# ECS

## GameObject存在的问题

### GameObject是啥？

UnityEngine.GameObjet

- C# 类
- UnityEngine.Component实例的容器
- 总带一个 Transform component
- 可以是其他GameObject的chid

UnityEngine.Component

- C# 类
- 特殊方法，如Update()，会在game loop的特定时刻调用
- 可能引用任何一些托管对象（GameObject, other Component and assets）

### GameObject问题

- 托管对象不能用于Burst
- 托管对象不应该用于jobs(一般情况)
- 托管对象有GC(garbage collected)，消耗性能
- 创建销毁GameObject和Component相当消耗性能
- 托管对象在内存分布上相当的分散，循环效率低下，因为它并不像array那样内存对齐分布，会触发相当多cache misses。

## Entity

Entity

- 一个int id
- 有关联component(每种component类型最多只允许一个)
- 没有内建的父子关系的概念

Unity.Entities.IComponentData

- C# struct (or class)
- 可以通过 id 引用 Entities, 而不是托管类型（usually）
- No methods(usually)。 通常是纯数据没有方法，但是也没严格限制
- 通过Query高效访问。按照array紧密打包，通过query批量访问高效。
- 用class可以包含其他托管对象，但是这种用法，托管对象会依旧存在GameObject所有的效率问题，因此这种用法必须严格限制，非必要不使用

## sample

```csharp
//表示运动的组件
public struct Movement : IComponentData
{
    //两个非托管的成员变量
    public float3 Direction;
    public float speed;
}

```

**World** : a collection of entities

- 可以有多个world，一般一个也够用。
- 一个entity在一个world中有唯一id，但其他wordld可能有相同id，query时注意world。

**EntityManager** : manages the entities of a world.

- CreateEntity()
- DestroyEntity()
- AddComponent()
- RemoveComponent()
- GetComponent()
- SetComponent()

### 实体与它们的组件是如何存储的

**Archtype** : 原型， stores all entities of a world which have a specific set of component types.

**Chunk** : 块， uniformly-sized blocks within an archetype where the entities and components are stored.

- 一个world中的所有entities被划分成多个Archtype，每个Archtype存储所有具有特定组件组合的所有实体。
- 在每个原型中，实体以及组件存储在大小均匀分布的内存块中，称之为Chunk。

![World sample](./pictures/2_ecs_1.png)

- 一个World下有三个Archtype。
- 左边Archtype有四个Chunk，存储所有有ABC三种组件的实体。
- 中间Archtype有两个Chunk，存储所有有AB两种组件的实体。
- 中间Archtype有三个Chunk，存储所有有AC两种组件的实体。
- 向实体增删组件，实体需要移动到新的archtype。例如向含AB的两种组件的实体添加组件C，则改组件需要从中间的Archtype中的Chunk移动到左边Archtype下的Chunk中。
- 作为用户不用靠考虑上述移动实体细节，由EntityManager负责。
- structural change operations: 指这种可能改变原型和块结构的操作。简单读取和修改不是结构改变操作。

### Chunk 内存储结构

![Archtype ABC下的Chunk](./pictures/2_ecs_2.png)

- 一个Chunk中Entities数量取决于component大小和数量，但是Entities数量上限总是128，Chunk内存大小16KB。
- 有四个array,一个用来存Entity id, 另外三个分别存ABC三个组件。
- 删除chunk中的某一个实体，那么最后一个entity会移动到对应位置填补空缺。

![Entity metadata](./pictures/2_ecs_3.png)

- 为能通过id查找实体， EntityManager 维护一个entity metadata的array。
- entity metadata 一个Chunk指针，一个index, 一个Version number。

**Query** : efficiently find all entities with a specified set of componeent types

### 可以访问 entities 的jobs

**IJobChunk** : a job that iterates over the chunks matching a query.

**IJobEntity** : a job that iterates over the entities matching a query.

- IJobChunk: 传递一个Query，遍历符合条件的Chunk。
- IJobEntity: 传递-一个Query，遍历符合条件的Entity。
- IJobChunk: 使用不是很方便，但是提供低级别控制。
- IJobEntity: 使用更简便，但有时需要用IJobChunk。

### baking

**baking** : build time conversion of GameObjects into serialized entities

**Entity Subscene** : a scene whose GameObjects are baked.

**Baker** : a class that bakes GameObject components of a specific type.
