# 实体行为模拟：移动与碰撞

在上一章中，我们手写了一个微型的 ECS 框架骨架。为了证明这套架构是切实可用的，本章我们将编写一个完整的测试程序。

在这个长篇案例中，我们将生成 10,000 个在 2D 空间内随机运动的小球实体，并利用我们自定义的基于连续组件内存的 ECS 框架，来处理它们的位移（Movement）以及简单的边界碰撞回弹（Collision）逻辑。

## 1. 扩充组件定义

之前的章节中我们定义了 `Position` 和 `Velocity`，在这我们将为它们降维至 2D，并针对功能补全字段。

```csharp
using System;

// 1. 位置组件：记录当前的二维坐标
public struct PositionComponent : IComponent {
    public float X, Y;
}

// 2. 速度组件：记录当前的速率向量
public struct VelocityComponent : IComponent {
    public float Vx, Vy;
}

// 3. 边界碰撞组件：标记该实体不仅会运动，还会与地图边界发生碰撞反弹
// 实际上这个组件哪怕没有数据，也可以作为一种“标签（Tag）”存在
public struct BoundsColliderComponent : IComponent {
    public float MinX, MaxX;
    public float MinY, MaxY;
    public float Radius; // 用于确定小球边缘
}
```

## 2. 完善系统逻辑

为了模拟游戏过程，我们将原有的 `MovementSystem` 和新增的 `BoundsCollisionSystem` 分别建立。这体现了 ECS 系统之间的解耦与逻辑管道化。

```csharp
// 负责处理所有具备 [Position + Velocity] 的实体的位移
public class MovementSystem : BaseSystem {
    public override void Update(Registry registry, float deltaTime) {
        var posPool = registry.GetPool<PositionComponent>();
        var velPool = registry.GetPool<VelocityComponent>();

        // 遍历所有的位置实体
        for (int i = 0; i < posPool.DenseEntities.Count; i++) {
            Entity entity = posPool.DenseEntities[i];
            
            try {
                ref PositionComponent pos = ref posPool.GetComponent(entity);
                ref VelocityComponent vel = ref velPool.GetComponent(entity);

                // 更新位置
                pos.X += vel.Vx * deltaTime;
                pos.Y += vel.Vy * deltaTime;
            } catch {
                // 如果没有拿到 Velocity，静默进行下一次循环
                continue; 
            }
        }
    }
}

// 负责处理那些具备 [Position + Velocity + Collider] 的实体的边界反弹
public class BoundsCollisionSystem : BaseSystem {
    public override void Update(Registry registry, float deltaTime) {
        var posPool = registry.GetPool<PositionComponent>();
        var velPool = registry.GetPool<VelocityComponent>();
        var boundsPool = registry.GetPool<BoundsColliderComponent>();

        for (int i = 0; i < boundsPool.DenseEntities.Count; i++) {
            Entity entity = boundsPool.DenseEntities[i];
            
            try {
                ref PositionComponent pos = ref posPool.GetComponent(entity);
                ref VelocityComponent vel = ref velPool.GetComponent(entity);
                ref BoundsColliderComponent bounds = ref boundsPool.GetComponent(entity);

                // 处理 X 轴反弹
                if (pos.X - bounds.Radius < bounds.MinX || pos.X + bounds.Radius > bounds.MaxX) {
                    vel.Vx = -vel.Vx; // 速度反向
                    
                    // 防穿模修正
                    if (pos.X - bounds.Radius < bounds.MinX) 
                        pos.X = bounds.MinX + bounds.Radius;
                    else
                        pos.X = bounds.MaxX - bounds.Radius;
                }

                // 处理 Y 轴反弹
                if (pos.Y - bounds.Radius < bounds.MinY || pos.Y + bounds.Radius > bounds.MaxY) {
                    vel.Vy = -vel.Vy; // 速度反向
                    
                    if (pos.Y - bounds.Radius < bounds.MinY) 
                        pos.Y = bounds.MinY + bounds.Radius;
                    else
                        pos.Y = bounds.MaxY - bounds.Radius;
                }
            } catch {
                continue;
            }
        }
    }
}
```

## 3. 组装主心骨：引擎游戏循环串联

所有的底层部件已经完毕，最后我们需要构建 `Game Application`（也就是 Main 函数的等价物），它负责初始化数据世界并驱动 System 的流转。

```csharp
public class ECSDemoApp {
    private Registry _registry;
    private List<BaseSystem> _systems;

    public void Initialize() {
        _registry = new Registry();
        _systems = new List<BaseSystem>();

        // 1. 注册系统管线：注意调度的顺序很关键
        _systems.Add(new MovementSystem());       // 第一步：移动
        _systems.Add(new BoundsCollisionSystem());// 第二步：检测碰撞修正速度和位置
        
        // 2. 初始化实体工厂（创建测试用例）
        Random rand = new Random();
        int entityCount = 10000; // 我们可以瞬间配置 1 万个单位

        for (int i = 0; i < entityCount; i++) {
            Entity ball = _registry.EntityManager.CreateEntity();
            
            // 组装组件：让这个小球具有移动与反弹属性
            _registry.AddComponent(ball, new PositionComponent { 
                X = (float)(rand.NextDouble() * 800), 
                Y = (float)(rand.NextDouble() * 600) 
            });
            
            _registry.AddComponent(ball, new VelocityComponent { 
                Vx = (float)((rand.NextDouble() - 0.5) * 50), 
                Vy = (float)((rand.NextDouble() - 0.5) * 50) 
            });

            _registry.AddComponent(ball, new BoundsColliderComponent { 
                MinX = 0, MaxX = 800, 
                MinY = 0, MaxY = 600,
                Radius = 5.0f
            });
        }
        
        Console.WriteLine($"世界初始化完成。共生成了 {entityCount} 个活动实体。");
    }

    // 这就类似于 Unity 的 Update() 主循环
    public void GameLoopUpdate(float deltaTime) {
        // 让所有的 System 按顺序无情地碾压密集数组
        foreach (var system in _systems) {
            system.Update(_registry, deltaTime);
        }
    }
}
```

## 结语与理论印证

虽然上面是一段简化的 C# 仿代码演示，但仔细审视，这正是数据导向设计（DOD）理念在软件工程中的完美投射：

1. **没有多态与虚函数的迟滞**：我们的 10,000 个小球，不是实例化的 GameObject 或 Player 对象，它们仅仅是薄薄的 10,000 个 `uint` 类型数字。它们的内存极度轻盈，生成速度远超任何传统的 Class `new` 对象操作。
2. **极小化缓存未命中（Cache Thrashing）**：`MovementSystem` 在运行时抓取的是两个底层通过 `List<T>` 实现的密致数值数组。当我们在处理小球的位置和速度时，没有加载任何无谓的材质渲染或逻辑状态数据。CPU 可以毫无阻碍地发挥高速预读取的优势。
3. **管线的独立与低耦合**：`BoundsCollisionSystem` 被剥离在一个彻底独立的流水工序里。你若需要添加渲染，则直接新起一个专门读 `Position` 的渲染系统放入末尾即可；若需要对小球增加重力，完全不需要碰小球现有的代码，只需向其挂载新组件并新建系统去运算即可。

至此，“怎么手写一个小巧且五脏俱全的 ECS”这一章节已经正式告一段落。

通过这五大部分从底层心智再到极简框架的手搓演练，相信你对数据驱动的高速模式已经成竹在胸。最后&#x7684;**《第六部分》**，我们将把视角彻底转向工业级标准，通过目前全球最成体系化的 **Unity DOTS (Data-Oriented Technology Stack)** 技术栈，看看商业引擎是如何将这一套架构“武装到牙齿”的。
