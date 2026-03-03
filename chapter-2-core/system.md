# 系统（System）：无状态的逻辑处理

到目前为止，我们已经了解了 ECS 中的“实体（Entity）”仅仅是无意义的 ID，而“组件（Component）”是毫无行为能力的纯数据结构。如果将 ECS 比作一座现代化的工厂，那么实体就是传送带上的托盘标签，组件就是托盘里的各种原材料。

那么，是谁在加工这些材料？答案就是机器手臂——**系统（System）**。它是 ECS 架构中唯一包含业务逻辑的地方。

## 1. 无状态的数据处理器

在 OOP 中，状态和行为被封装在同一个对象盒子里。但在 ECS 中，**系统本质上是一个无状态（Stateless）的函数或操作管线**。

“无状态”是指，系统本身不应该在自己的内部缓存或持有任何与单个实体相关的数据。例如，一个负责移动所有实体的 `MovementSystem`，其内部绝不能定义一个叫 `List<Entity> myMovingEntities` 的成员变量来私自记录它要处理谁。

系统的唯一工作模式，是在每一帧（或特定的时间刻）通过向 ECS 框架发出\*\*“查询指令（Query）”\*\*，获取当前符合条件的组件流，然后进行批量的数学运算。

通常，一个系统的核心结构非常简单：

```csharp
public class MovementSystem {
    // 系统不持有任何单个实体状态，只有处理逻辑
    public void Update(float deltaTime, ComponentManager registry) {
        
        // 1. 发起查询：我要所有的 Position 和 Velocity 
        var query = registry.GetEntitiesWith<Position, Velocity>();
        
        // 2. 遍历流水线并转换数据
        foreach (var entity in query) {
            ref Position pos = ref registry.GetComponent<Position>(entity);
            ref Velocity vel = ref registry.GetComponent<Velocity>(entity);
            
            // 核心的业务计算
            pos.x += vel.vx * deltaTime;
            pos.y += vel.vy * deltaTime;
            pos.z += vel.vz * deltaTime;
        }
    }
}
```

## 2. 基于签名（Signature）的自动过滤

系统是如何知道它应该处理哪些实体的？这是通过\*\*组件签名（Component Signature）\*\*来实现的。

ECS 架构的美妙之处在于其\*\*“隐式订阅”\*\*的特性。在 OOP 的事件驱动或观察者模式中，对象通常需要显式地把自己注册给某个管理器。但在 ECS 中，一个系统只需要声明它关心的组件类型（即它的 Signature），框架就会自动将所有满足条件的实体投喂给它。

例如：

* `MovementSystem` 的签名是：`[Position, Velocity]`。无论是玩家、敌人、还是飞行的子弹，只要实体身上挂载了这两个组件，它就会自动在每一帧被该系统运算并产生位移。
* `RenderSystem` 的签名是：`[Position, RenderMesh]`。那么子弹（没有 RenderMesh）将被自动忽略，而静止的树木（有 Position 和 RenderMesh，没有 Velocity）则只会被渲染系统捕捉，而不会被运动系统干扰。

这种完全解耦的机制，极大地方便了功能模块的横向扩展。你可以随时向系统中添加一个新能力，而无需修改现有的任何代码逻辑。如果想给主角加上中毒的掉血效果，只需挂载一个 `PoisonComponent`，并新建一个签名是 `[Health, Poison]` 的 `PoisonSystem` 即可。

## 3. 从逻辑中心到管线设计

在大型项目中，系统并不是随意执行的，它们构成了严格的**执行管线（Execution Pipeline）**。

由于数据和逻辑被分离，ECS 常常被抱怨“逻辑太散”。为了解决这个问题，所有的系统必须依照严格的顺序进行调度（Scheduling）。比如在游戏循环中，典型的调度顺序可能是：

1. **输入系统（InputSystem）**：读取手柄按键，修改 `Velocity`。
2. **物理系统（PhysicsSystem）**：根据 `Velocity` 修改 `Position`。
3. **相机系统（CameraSystem）**：根据玩家的 `Position` 修改屏幕的渲染矩阵。
4. **渲染系统（RenderSystem）**：读取 `Position` 和 `RenderMesh` 进行画面绘制。

这种单向的、流水线式的数据处理（Data flow），不仅减少了不同模块之间的状态耦合（比如输入系统绝不直接调用物理引擎的方法），而且让代码极度可测试，也让多线程改造变得可能。

## 4. 并行化的极限：无锁多线程

在 OOP 代码中很难实现安全的并行（Multi-threading），因为不同的对象之间往往存在着错综复杂的指针引用，极易发生竞态条件（Race Condition），导致你需要编写大量的死锁保护代码（Locks/Mutex）。

到了 ECS 的领域，多线程变得异常简单。回顾上面讲的“签名（Signature）”。ECS 框架通过静态分析各个系统的签名，可以在调度时完美地判断出哪些系统可以并行执行：

* `System A` 写入 `Position`。
* `System B` 读取 `Position`，写入 `Collision`。
* `System C` 读取 `Health`，写入 `Health`。

框架会立刻算出结论：

1. A 和 B 存在数据依赖（都操作 `Position`），它们必须串行调度（先 A 后 B）。
2. C 并不关心 `Position` 和 `Collision`，它关注的是一片完全不同的连续内存区域（`Health`）。因此，C 完全可以与 A、B 在不同的 CPU 线程中**无锁并行计算**。

这是传统 OOP 架构几乎无法想象的便利性。通过数据在系统签名层面的明确声明，框架接管了最令人头疼的多核 CPU 任务调度工作。

## 总结

至此，ECS 的三大支柱已经拼图完毕：

* **实体（Entity）**&#x662F;用来绑定数据的索引键。
* **组件（Component）**&#x662F;按类型聚拢连续存放的数据“肉身”。
* **系统（System）**&#x662F;由组件签名驱动的无状态逻辑流水线，它在其中像压路机一样滚过连续的内存，贪婪地进行着批处理。

但这三者的结合并不是终点。在下一部分&#x7684;**《核心术语与底层概念》**&#x4E2D;，我们将剖析这台机器的内部底盘齿轮，解答两个让无数初学者晕头转向的硬核名词：用来加速组件查询的 **Sparse Set（稀疏集）**，以及 Unity DOTS 和 Flecs 广泛采用的终极内存排列模型 —— **Archetype（原型模式）**。
