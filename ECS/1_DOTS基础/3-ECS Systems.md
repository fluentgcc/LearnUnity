# ECS systems

## Systems

**System** : 通常是一帧执行一次的简单代码单元。 继承ISystem。

```csharp
[BurstCompile]
public partial struct MySystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // on create. 第一次update之前，会调用。
    }
    
    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {
        //on destroy. system实例被从world中移除，或者world本身disposed。
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // per frame. 通常每帧调用一次。
    }
}
```

- 参数都是 SystemState 的引用
- 任一方法带BurstCompile，那么struct也要带BurstCompile。
- 每个System 实例从属于某一个world。
- 一个world下的entities，一般只允许该world自己的systems访问。（normally）(实际并没严格限制，所有代码都能访问任何world的entities)
- SystemState 包含world，和world的EntityManager。

## System groups

World中的systems是按System Groups组织的层级结构：每个Stytem Group 有子system和子system group。类似文件系统，含有文件和子目录一样。

- SystemGroup 有OnUpdate函数。可以重写。
- 默认OnUpdate行为是简单的按照伪随机排序更新组内所有子项。每次添加删除子项，都会重新排序。
- 可以通过对System添加UpdateBefore和UpdateAfter属性控制Updates顺序。

```csharp
public class MySystemGroup : ComponentSystemGroup
{
    protected override void OnUpdate()
    {
        // Update every child in sorted order
        base.OnUpdate()
    }
}


//通过UpdateBefore，UpdateAfter控制更新顺序。
[UpdateBefore(typeof(FooSystem))]    //sort MySystem before FooSystem
[UpdateAfter(typeof(BarSystem))]    //sort MySyster after BarSystem
public class MySystem : ISystem
{
    // ...
}

```

### 默认的System Group

通过Systems视图可以查看System Group，以及执行顺序。

![Entity metadata](pictures/3_ecs_system1.png)

这里有三个标准System Group， 会直接在unity main loop中直接Update：

- **initialization system group** 负责setup工作。
- **simulation system group** 用于游戏核心逻辑。
- **presentation system group** 用于render。

注意override System的update，可以选择性Update一些children, 或者一帧中更新多次。eg:

- **fixed step simulation group** 会尝试以固定时间间隔更新子项，这种一帧可能更新多次，也可能一帧不更新。

### Automatic bootstrap

Automatic bootstrapping 自动引导

- 自动创建默认world，并填充上述三个顶级system groups.
- project中的每种system和system group都会创建一个实例并加入到默认world，默认会全部放到simulation system group中。

```csharp
//将 MySystem 放到 FixedStepSystemGroup 中，而不是默认的SimulationSystemGroup
[UpdateInGroup(typeof(FixedStepSystemGroup))]
public class MySystem : ISystem
{
    //...
}
```

- 完全禁用 automatic bootstraping: 再脚本中添加宏定义#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP
- 此时需要自己负责创建world,system，group等全部实例。

## SystemState

System方法内，可以通过参数SystemState访问world，和EntityManager。
SystemState本身也有下面方法查询实体、获取组件句柄。

- GetEntityQuery()
- GetComponentTypeHandle\<T\>()

注意：**永远只通过SystemState来获取queries和component type handles，而不是EntityManager**
因为: SystemState’s methods register component types with the system. 如果直接用EntityManager则无法注册。

注册有啥作用？

Immediately Before a system updates:

- The Dependency job handle of SystemState is completed
- Then Dependency is assigned a handle combining the Dependency of all other systems which access any of the same component types.

 So to ensure that all jobs scheduled in a system properly depend upon conflicting jobs scheduled in other systems

- All jobs scheduled in a system update should directly or indirectly depend upon Dependency.
- Before the system update returns, Dependency should be assigned a handle that includes all jobs scheduled in the update

```csharp

// ... in a system
protected override void OnUpdate(ref SystemState state)
{
    var handle = new MyJob().Schedule(state.Dependdency)
    var otherHandle = new OtherJob().Schedule(handle) 
    state.Dependency = otherHandle; //the first job is included indirectly.
}

```
