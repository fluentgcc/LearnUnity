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
        // on create. 第一次updata之前，会调用。
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
