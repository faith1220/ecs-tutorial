# 并法处理：C# Job System 与 Burst 编译器

在上一章，我们使用 `SystemAPI.Query` 在主线程（Main Thread）中完成了实体的旋转逻辑。配合 `[BurstCompile]` 标签，单核性能已经得到了极大的优化。

但现代设备的 CPU 动辄 8 核 16 线程甚至更多，如果只让一个核心疯狂工作，其他核心全部闲置，这就违背了 ECS 架构的一大初衷：**数据并行处理（Data Parallelism）**。

本章将教你如何利用 **C# Job System**，仅需几行代码的改动，就能将计算任务安全地分发到所有空闲的 CPU 核心上并行执行。

## 1. 为什么 OOP 多线程像走钢丝？

在传统面向对象（OOP）中，如果我们在子线程去修改一个 `GameObject` 的 `Transform`，Unity 会直接抛出错误。原因在于：
1. **数据竞争（Data Race）**：多个线程可能同时读写同一个对象的内存地址，导致数据损坏。
2. **指针乱飞**：引用类型在堆内存中散落，子线程随时可能顺着藤摸到一个正在被主线程垃圾回收（GC）的对象，导致崩溃。
3. **沉重的锁（Locks）**：为了解决竞争，开发者必须手动编写 `lock(obj)` 语法，这会导致线程频繁阻塞甚至死锁（Deadlock），最终性能往往不如单线程。

## 2. Job System 的无锁魔法

回到 ECS 的世界。我们前面强调过，ECS 的组件数据是以扁平的、连续的小内存块（Chunk）严格按类型存放的。

在这个前提下，因为 `Job System` 和 `Entities` 底层是深度融合的：
它明确知道内存里的数据是一块长条切片。它完全可以做到：把这一长条切片切成 16 份。
将第 1 份丢给核心 A，第 2 份丢给核心 B... 它们都在操作纯正的值类型数据，没有任何交集，没有任何引用类型关联。**不需要任何一把锁，天然绝对并发安全。**

这就是 **Job System 的调度艺术**。

## 3. 从单线程升级到并行 Job

让我们将上一章在主线运行的 `RotationSystem` 改写为真正的多线程并行作业。

得益于 Unity 的进化，现在将一个 `foreach` 循环丢进子线程的操作已经简化到了极致的地步，这被称为 **IJobEntity** 机制。

### 步骤一：定义 Job 结构体

在系统的外部（或内部），我们声明一个专门处理核心运算的“任务（Job）包”。

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

// 1. 实现 IJobEntity 接口
// 2. 加上 BurstCompile 标签让它变成狂暴的机器码
[BurstCompile]
public partial struct RotateJob : IJobEntity {
    
    // 必须要从系统传过来的外部数据（比如只有每一帧才知道的 deltaTime）
    public float DeltaTime;

    // Execute 方法就是你的核心操作管线
    // 这里的参数列表，等同于上一章 SystemAPI.Query 里的查表签名！
    // RefRW 代表要修改的组件，RefRO 代表只读取的组件
    void Execute(ref LocalTransform transform, in RotationSpeed speed) {
        
        // 我们只在乎这一个小方块计算逻辑
        transform = transform.RotateY(speed.RadiansPerSecond * DeltaTime);
    }
}
```

注意到了吗？`Execute` 中没有 `for` 循环，也没有 `foreach`。你只需要写针对**单个实体**的转换公式，就像着色器（Shader）里写顶点计算一样。至于怎么把它切片发给多核 CPU 算，是底层去搞定的。

### 步骤二：在系统（System）中排期分发任务

现在回到我们的系统主控台 `OnUpdate` 中。我们不再亲自 `foreach` 下场干活了，而是把任务交给工人。

```csharp
[BurstCompile]
public partial struct RotationSystem : ISystem {
    
    [BurstCompile]
    public void OnUpdate(ref SystemState state) {
        
        // 实例化刚才定义的 Job，并填入所需的逐帧数据
        var job = new RotateJob {
            DeltaTime = SystemAPI.Time.DeltaTime
        };

        // 关键调用：ScheduleParallel
        // 这句话就是命令：“把这个任务，以极其暴力的多线程形式，分发给有这些组件的所有 Chunk 吧！”
        job.ScheduleParallel();
    }
}
```

## 4. 安全系统的严苛护卫

你可能会问：如果我不小心在两个不同的并行的 Job 里，同时要求去修改 `LocalTransform`（即声明了两个 `ref LocalTransform` 或 `RefRW`），导致竞争崩溃了怎么办？

答案是：**你的代码会在编辑器里直接报错，游戏根本运行不起来！**

Unity Job System 拥有极其严格的静态分析检查器（Safety System）。由于所有数据读写权限都是在参数标头（`in`, `ref`, `RefRW`, `RefRO`）中写死的声明：
1. **如果一个 Job 声明了只读（`in` / `RefRO`）**，在代码里哪怕你只是尝试给它赋一个值，也会在写代码的那一秒立刻报红提示只读错误。
2. **如果有两个 Job 在同一帧被调度了，且都对同一个组件声明了读写（`ref` / `RefRW`）**，框架在安排任务（Schedule）时就会抛出致命异常：检测到潜在的数据竞争！
3. 它会强迫你修改管线设计，或者通过设置任务的依赖项（JobHandle.CombineDependencies）强制让其中一个系统串行排在另一个系统后面等它算完。

这就是面向数据带来的终极福音：**在编译期就把多线程的鬼魅 bug 按死在摇篮里**。

## 5. 结语

简单的一个结构体定义和一句 `ScheduleParallel()`，你的单核帧率限制器轰然倒塌，你驯服了计算机里沉睡着的凶兽。

当我们把：
* 连续无洞的物理内存（Chunk）
* 无状态的数据流水线系统（System）
* 充分剥皮去枝只留数据（Baking）
* 压榨每一根流水线与处理器的机制（Job & Burst）

这所有的“魔法零件”最终都组装在了一起。你不再是那个需要跟 `GameObject`、GC 卡顿和指针报错苦苦搏斗的开发者了。如果你有胆识，大可肆意拉出一个数量达到十万级别的单位矩阵。

这便是我们本书的终章演练 —— 《**综合演练：海量实体同屏渲染与更新 Demo**》。在那里，你将写下你在 DOTS 工程里的第一部交响乐。
