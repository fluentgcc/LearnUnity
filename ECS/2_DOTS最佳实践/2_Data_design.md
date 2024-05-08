# 数据设计

- 如何组织数据高效使用DOTS
- blittable data和managed types
- 思考运行时数据访问

## 1. 理解如何设计数据以及背后原因

如演讲中所述[Building A Data-Oriented Future](https://youtu.be/u8B3j8rqYMw?t=1337)，提取下面的一些DOD设计原则。

- **转换某些数据所需的（全局）能量应该与意外量成正比。** 也就是说的前面说的针对最常见情况进行设计。
- **所有程序以及所有程序的所有部分的目的，都是将数据从一种形式转换成另一种形式。** DOD，要找出最合理的数据格式，并设计合理pipeline用来transform data.
- **如果不理解数据，就没有理解问题。**
- **不同的问题，需要不同的解决方案。** 没有普适的方案，不同问题不同分析。
- **如果数据不同，那么问题不同。** 不存在什么 'DOTS制作某某最佳方法'。问题的关键，取决于需要处理的数据。
- **如果不了解解决问题的成本，就是没有理解问题。** 制定约束条件，制定解决问题的性能目标。在这些约束下设计数据和pipeline。
- **如果不了解硬件，就无法推断解决问题的成本。** CPU核心数？内存大小？cache missing多花费多少CPU周期？CPU cache line大小，以及因内存布局浪费的性能？这些问题答案是设计的约束条件。

从OOP关注抽象和概括的心态转换到DOD思维。要点是要根据硬件和性能要求解决特定问题，尽可能高效设计数据、设计数据转换pipeline。

## 2. 预先设计数据

弄清需要哪些数据，以及实现行为的转换数据。

以经典的Breakout游戏为例，其数据设计表格

![游戏图片](./pictures/2_breakout.png)

### Data

![data table](./pictures/2_breakout_data.png)

### Data Transformations

![data transform table](./pictures/2_breakout_transform.png)

## 3. 为高效transformation设计数据

原则是高效利用CPU缓存（cache hit），减少cache miss。

减少使用高级抽象的容器，如字典、链表，多使用可预测的线性数据结构，如数组（即 NativeArray, NativeList）

ECS将组件数据打包存放在chunk中，处理整个chunk或者多个chunk，是高效的，但是，如果要找某个entity的某个组件，是不太高效的（random access）。要避免迭代entities，一次性访问组件数据。为最大化利用线性内存组织的优势，要通过EntityQuery来访问组件。

大量小组件优于少量大组件。不要把大量数据放到一个组件中，这样会导致不能有效利用缓存。

一般的cache line 是64 bytes，同一组件太多数据不能充分利用。

```csharp
//错误示范
public struct Ball : IComponentData
{
    public float2 Position;  
    public float2 Direction;  
    public float Speed;  
    public float2 Size;
    // ..etc...
}

```

不是所有的System都需要访问所有数据，这样导致了低效。这里各个属性做成单独组件更好。

## 4. 理解 Blittable 和 Burst 兼容

OOP（GameObject和MonoBehaviour）大部分用户代码在主线程，写多线程还需要考虑race conditions.

**Unity job system**使多线程更容易写，并且不用考虑竞争条件。因为它有个安全系统防止竞争条件发生。但是需要引入一个约束，jobs只能操作[**blittalbe data types**](https://en.wikipedia.org/wiki/Blittable_types).

**Blittable data**: [Blittable data](https://learn.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types) 常用byte,int,float,double都是blittalbe的。托管类型，引用等（如class,string,容器）都不是blittable的。应该设计成只用blittable数据。

blittable data的另外优势，是可以使用burst编译成高效native code。[Burst User Guide](https://docs.unity3d.com/Packages/com.unity.burst@latest?subfolder=/manual/index.html%23cnet-language-support)

## 5. 考虑数据是否read-only

System访问数据在不需写入数据的地方要设成只读，影响job并行。组件设计时也要分成只读，还是r/w。

如果数据在所有变换中都是只读，那数据应该存到[**BlobAsset**](https://docs.unity3d.com/Packages/com.unity.entities@latest?subfolder=/api/Unity.Entities.BlobBuilder.html)中，
BlobAsset是一种不可变的数据结构，存储在非托管内存中，可以含blittable数据，以及blittable数据组成的结构体、数组，或者BlobString存储的string。它比OOP assets可以更快的反序列化。

## 6. 考虑最常见情况，并针对优化

## 7. 不要使用 string

Burst和Jobs不支持string，用枚举，int，或者字符串计算的hash（burst中提供XXHash）。

必须使用字符串的，也可以用Collection包中的一些固定尺寸的sting类型。FixedString32Bytes 到 FixedString4096Bytes

BlobAsset中需要用string的，用BlobString类型。

## 8. Runtime data 不同于 authoring data

[数据转换](https://docs.unity3d.com/Packages/com.unity.entities@1.2/manual/conversion-intro.html)

关键点：

Authoring data 是为了灵活性优化。

- 人的可理解性以及可编辑性
- 版本控制
- 团队协作

Runtime data 是为了性能优化。

- Cache efficiency
- Loading time and streaming
- Distribution size

本指南侧重于创建和处理最佳的运行时数据结构，但是还需考虑authoring data的数据格式，以及如何从authoring data转换成runtime data.
