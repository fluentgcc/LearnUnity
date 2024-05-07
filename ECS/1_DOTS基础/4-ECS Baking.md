# Baking

Scene并不能直接包含entities，但是可以通过SubScene来加载entities。

- **SubScene** a Monobehaviour referencing a scene asset to be baked.
- **Baking** 将SubScene中的GameObjects转换成serialized entities.

右键 New SubScene。

运行时只会加载baked entities，而不是GameObject.

## The Baking process

1. 为Subscene创建一个baking world。
2. 为每个GameObject创建一个entity。
3. 带有Baker的每个GameObject component会通过调用Baker的bake方法进行处理。
4. 运行the baking wrold 的 systems。//可以额外创建entity,但是不含在序列化中。
5. 序列化the baking world.

## Creating a Baker

```csharp
public struct Movement : IComponentData
{
    public float3 Value;
}

public class MovementAuthoring : MonoBehaviour
{
    public Vector3 Direction;

    class Baker : Baker<MovementAuthoring>
    {
        public override void Bake(MovementAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlages.Dynamic);
            
            AddComponent(entity, new Movement {
                Value = authing.Direction
            });
        }
    }
}
```

```csharp
//通过配置传递实例化多个Monster Prefab的个数。
public struct Config : IComponentData
{
    public Entity MonsterPrefab;
    public int NumMoster;
}

public class ConfigAuthoring : Monobehaviour
{
    public GameObject MonsterPrefab;
    public int NumMonsters;
    
    class Baker : Baker<ConfigAuthoring>
    {
        public override void Bake(ConfigAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.None);
            
            AddComponent(entity, new Config
            {
                MonsterPrefab = GetEntity(authoring.MonsterPrefeb, transformUsageFlages.Dynamic),
                NumMonsters = authoring.NumMonsters
            });
        }
    }

}
```

### Baker methods that register dependencies

- Dependson()
- GetComponent()
- GetChild()
- GetParent()
- GetComponentInChildren()
- GetComponentInParent()
- CreateAdditionalEntity()    //这个会包含在序列化中。
