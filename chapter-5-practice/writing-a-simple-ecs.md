# 构建基础 ECS 框架

理论的深度最终需要代码来检验。在深入讨论那些复杂的开源框架和商业引擎（如 Unity DOTS）之前，我们决定从零开始，使用 C# 语言手写一个最基础的 ECS 运行时（Runtime）。

这个框架麻雀虽小，五脏俱全。它将采用我们在第三部分学到的**稀疏集（Sparse Set）**&#x4F5C;为底层存储模型，舍弃复杂的泛型或宏魔法，以最为平实的手段向你展示 ECS 的运转机制。

## 1. 定义实体（Entity）

正如第二部分所讲，实体本身不能包含任何业务数据，它仅仅是一个只带有整数 ID 的“标签”。为了简化演示，我们暂时不实现 Generation 版本号机制，直接将其定义为一个整数的薄包装。

```csharp
using System;
using System.Collections.Generic;

// 实体仅仅是一个 uint 的别名
public struct Entity {
    public uint Id;
    
    public Entity(uint id) { Id = id; }
}

// 实体生成器：负责分发唯一 ID
public class EntityManager {
    private uint _nextId = 0;
    
    public Entity CreateEntity() {
        return new Entity(_nextId++);
    }
}
```

## 2. 定义组件（Component）的存储系统

组件是没有任何行为的值数据（Struct）。难点在于如何按照类型统一存储它们并保证快速查询，即我们要为每种组件实现一个基于 `Sparse Set` 原理的组件池（Component Pool）。

首先定义所有组件必须继承的基接口，以便我们能用统一的类型保存它们。

```csharp
public interface IComponent {}

// 业务类型的组件声明（纯数据，无逻辑）
public struct PositionComponent : IComponent {
    public float X, Y, Z;
}

public struct VelocityComponent : IComponent {
    public float X, Y, Z;
}
```

接下来，我们编写核心的**组件池（ComponentPool）**，它使用稀疏集的逻辑，确保内部维持着一个连续的 `密集数组（Dense Array）`。为了让代码不至于过度复杂，我们用 `Dictionary` 模拟稀疏黄页簿，用 `List` 模拟绝对连续的密集数组（虽然这增加了装箱开销，但逻辑最清晰）。

```csharp
public class ComponentPool<T> where T : struct, IComponent {
    // 密集数组：存放真正的连续数据
    public List<T> DenseComponents = new List<T>();
    
    // 与密集数组强对齐的实体列表：DenseEntities[i] 对应 DenseComponents[i]
    public List<Entity> DenseEntities = new List<Entity>();
    
    // 稀疏数组（字典模拟）：Key 为实体 ID，Value 为该实体在密集数组中的索引
    private Dictionary<uint, int> _sparseMap = new Dictionary<uint, int>();

    public void AddComponent(Entity entity, T component) {
        if (_sparseMap.ContainsKey(entity.Id)) {
            throw new Exception("Entity already has this component.");
        }
        
        // 追加在密致数组的最末尾
        int newIndex = DenseComponents.Count;
        DenseComponents.Add(component);
        DenseEntities.Add(entity);
        
        // 更新黄页登记
        _sparseMap[entity.Id] = newIndex;
    }

    public ref T GetComponent(Entity entity) {
        if (!_sparseMap.TryGetValue(entity.Id, out int index)) {
            throw new Exception("Component not found.");
        }
        
        // O(1) 立即通过数组提取连续内存
        // 注意由于 C# 的限制，要返回引用的真实指针可能需要更底层的 unsafe 代码。
        // 为了演示框架思路，我们这里利用 C# 7.0 的 ref array 语法结构。由于 List 不支持 ref 返回，
        // 我们用一个巧妙的结构来表示该如何获取。但在真正的工业 ECS 里往往用 NativeArray 指针操作。
        return ref System.Runtime.InteropServices.CollectionsMarshal.AsSpan(DenseComponents)[index];
    }
}
```

_(备注：真实引擎里的稀疏集会用巨大的 `uint[]` 代替 `Dictionary` 以消灭哈希查找的微小开销，并使用非托管指针完成真正的零拷贝获取。本示例旨在表现原理)_

## 3. 全局容器：注册表（Registry）

现在我们需要一个全局总管（Registry），它负责管理所有的 `ComponentPool` 并在系统请求时分发特定的组件池。

```csharp
public class Registry {
    // 实体生成器
    public EntityManager EntityManager = new EntityManager();
    
    // 组件池集合：类型 -> 针对该类型的专属池
    private Dictionary<Type, object> _componentPools = new Dictionary<Type, object>();

    // 获取特定类型的组件池（如果没有则创建）
    public ComponentPool<T> GetPool<T>() where T : struct, IComponent {
        var type = typeof(T);
        if (!_componentPools.TryGetValue(type, out var pool)) {
            pool = new ComponentPool<T>();
            _componentPools[type] = pool;
        }
        return (ComponentPool<T>)pool;
    }

    // 语法糖：为实体添加组件
    public void AddComponent<T>(Entity entity, T component) where T : struct, IComponent {
        GetPool<T>().AddComponent(entity, component);
    }
    
    // 语法糖：获取实体的某个组件
    public delegate void RefAction<T>(ref T component);
    public ref T GetComponent<T>(Entity entity) where T : struct, IComponent {
        return ref GetPool<T>().GetComponent(entity);
    }
}
```

## 4. 定义系统（System）

有了存放数据的水池后，我们来定义专门处理数据的抽水机——系统（System）。

系统没有数据，只有方法。它接受 `Registry` 和每一帧的时间（DeltaTime），向库里申请相关的密集数组，开始疯狂且不受打扰的循环计算（批处理）。

```csharp
public abstract class BaseSystem {
    // 系统只需实现 Update 逻辑
    public abstract void Update(Registry registry, float deltaTime);
}

// 具体的运动系统，严格遵守无状态原则
public class MovementSystem : BaseSystem {
    public override void Update(Registry registry, float deltaTime) {
        // 第一步：向 ECS 框架索要 Position 和 Velocity 的连续组件池
        var posPool = registry.GetPool<PositionComponent>();
        var velPool = registry.GetPool<VelocityComponent>();

        // 第二步批处理：为了简化多组件相交查询，我们以包含实体数较少的密集数组为基准遍历。
        // （真实框架中会提供一个完美的 View / Query 函数交集产生器）
        var entities = posPool.DenseEntities;

        for (int i = 0; i < entities.Count; i++) {
            Entity currentEntity = entities[i];
            
            // 假设：我们用 try 的方式看看它身上有没有 Velocity
            // 虽然这里引发了由于交集不完美导致的轻微查询开销，但对单组件池（posPool）的访问依然是严格连续的。
            try {
                ref PositionComponent pos = ref posPool.GetComponent(currentEntity);
                ref VelocityComponent vel = ref velPool.GetComponent(currentEntity);

                // 纯碎的业务逻辑核心：没有对象调用，只有数据公式
                pos.X += vel.X * deltaTime;
                pos.Y += vel.Y * deltaTime;
                pos.Z += vel.Z * deltaTime;
            } catch {
                // 这个实体只有坐标没速度（比如一棵树），忽略不计
                continue; 
            }
        }
    }
}
```

## 构建总结

在这一章，仅用了不到 100 行代码，我们就搭建了一个不含有一丝传统的面向对象包袱的结构。

在这个袖珍引擎中：

* `Entity` 是透明且随时能增减的标签数字。
* `Component` 是没有任何赘肉的纯值类型数据。
* `Registry` 和其内部的 `ComponentPool` 维持着按类型分类的“纯血内存组”。
* `MovementSystem` 展示了最标准的数据流驱动范式。

然而，空有框架毫无意义。在下一章，我们将编写一段驱动程序，把这辆“车”开起来，并加上诸如碰撞计算这样稍具复杂度的业务逻辑系统，完成一个完整的游戏循环！
