# 综合演练：海量实体同屏渲染与更新 Demo

经过前面几章对 Unity DOTS 核心机制（Baking、SystemAPI、Job System）的学习，你已经掌握了开发高性能 ECS 应用的所有必要武器。

作为本教程的压轴大戏，我们将把所有的知识点串联起来，在 Unity 中从零实现一个极具视觉冲击力的经典 ECS 测试场景：**在同屏生成并驱动 100,000 个实体（海量方块）进行随机运动，同时保持丝滑的 60 FPS 以上帧率。**

如果使用传统的 `GameObject` 和 `MonoBehaviour`，即使不写任何逻辑代码，仅仅是实例化 10 万个对象在场景里，也足以让大多数顶配电脑卡成幻灯片。而现在，我们要用数据导向设计（DOD）的力量击穿这面性能墙。

## 1. 制作基础预制体 (Prefab)

在进行代码编写前，我们先准备好需要被海量复制的基础“面团”。

1. 在 Unity 场景中，右键创建一个普通的 `3D Object -> Cube`，命名为 `UnitPrefab`。
2. 确保它有一个合适的材质（建议开启 GPU Instancing 以极大地降低渲染 Draw Call）。
3. 为它添加上一章我们编写的 `RotationSpeedAuthoring` 脚本（设个几十度的旋转速度即可）。
4. 将这个 `UnitPrefab` 拖拽到 Project 目录中变成一个预制体，然后从场景中将其删除。

## 2. 编写生成器组件 (Spawner)

为了能够一次性生成十万个实体，我们需要在 ECS 宇宙中配置一台“实体复印机”。

首先，定义底层的生成器数据组件：

```csharp
using Unity.Entities;

public struct Spawner : IComponentData {
    // 指向我们要复制的原型 Entity
    public Entity Prefab;
    // 生成的数量
    public int Count;
    // 生命范围
    public float SpawnRadius;
}
```

然后，编写对应的面包模具（Authoring 和 Baker）：

```csharp
using UnityEngine;
using Unity.Entities;

public class SpawnerAuthoring : MonoBehaviour {
    // 在 Inspector 中拖入我们刚才制作的 UnitPrefab
    public GameObject PrefabToSpawn;
    public int NumberOfEntitiesToSpawn = 100000; // 宣战 10 万！
    public float SpawnRadius = 50f;
}

public class SpawnerBaker : Baker<SpawnerAuthoring> {
    public override void Bake(SpawnerAuthoring authoring) {
        Entity entity = GetEntity(TransformUsageFlags.None);
        
        AddComponent(entity, new Spawner {
            // GetEntity() 也能在预制体上调用，这会将 GameObject 预制体注册进 ECS 的烘焙管线
            Prefab = GetEntity(authoring.PrefabToSpawn, TransformUsageFlags.Dynamic),
            Count = authoring.NumberOfEntitiesToSpawn,
            SpawnRadius = authoring.SpawnRadius
        });
    }
}
```

在 Unity 场景里新建一个空的 GameObject，命名为 `SpawnerObject`。挂载 `SpawnerAuthoring` 脚本，并将 `UnitPrefab` 拖入引用槽。最后，**将这个 `SpawnerObject` 拖入你创建好的 Sub Scene（子场景）中完成 Baking。**

## 3. 编写生成系统：ECB (Entity Command Buffer) 的亮相

我们如何把这个生成器跑起来呢？我们需要一个 `SpawnerSystem`。

但在多线程或者 `foreach` 遍历期间，我们**绝对不能**直接调用 `EntityManager.Instantiate` 去创建实体。回忆一下第三部讲的 原型模式 (Archetype) —— 创建新实体意味着底层 Chunk 的内存结构（Structural Change）正在发生剧烈变动。如果在你一边遍历数组一边往数组里塞新东西，内存游标会彻底崩溃。

在 DOTS 中，进行结构性改变的最佳实践是使用 **Entity Command Buffer (ECB，实体命令缓冲)**。
ECB 就像是一本备忘录：系统在多线程高速公路上飙车时，把“等会儿帮我生成一个实体”的请求写在备忘录上。等飙车结束（某个安全的主线程同步点同步时），ECS 框架再统一集中处理备忘录上的创建/销毁命令。这叫**延迟执行（Deferred Execution）**。

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

[BurstCompile]
public partial struct SpawnerSystem : ISystem {

    [BurstCompile]
    public void OnCreate(ref SystemState state) {
        state.RequireForUpdate<Spawner>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state) {
        
        // 我们只希望生成一次，完成后就关闭该系统
        state.Enabled = false;

        // 获取该系统中唯一存在的单个生成的组件
        var spawner = SystemAPI.GetSingleton<Spawner>();

        // 申请一个备忘录 (ECB)
        // Allocator.Temp 指示这个内存极其短暂，用完就立刻回收，对 GC 没有任何负担
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // 为了散布方块，我们建一个随机数生成器 (Mathematics 库的高效随机数)
        var random = Unity.Mathematics.Random.CreateFromIndex(1234);

        // 我们在主线程快速发出指令（这只是写入备忘录，还不是真的申请内存新建对象）
        for (int i = 0; i < spawner.Count; i++) {
            // 在 ECB 记下一笔：复制实体
            Entity spawnedEntity = ecb.Instantiate(spawner.Prefab);
            
            // 计算随机位置
            float3 randomPos = random.NextFloat3Direction() * random.NextFloat(0, spawner.SpawnRadius);
            
            // 在 ECB 记下一笔：修改刚复制的实体的 LocalTransform 组件
            ecb.SetComponent(spawnedEntity, LocalTransform.FromPosition(randomPos));
        }

        // 把备忘录交给经理，强制立即执行刚才记录的几万条实例化和修改组件命令 (回放播放)
        // 这个动作会锁定内存并引发海量拷贝，但我们只在初始化时做一次。
        ecb.Playback(state.EntityManager);
        ecb.Dispose(); 
    }
}
```

## 4. 驱动海量运算的 Job 引擎

如果你在上一部分（`dots-jobs.md`）已经顺利编写好了 `RotateJob` 和 `RotationSystem`。那么你这里的拼图已经完美闭环了。

当 `SpawnerSystem` 初始化完成那 100,000 个带着 `LocalTransform` 和 `RotationSpeed` 组件的实体后，底层庞大而规整的 Archetype Chunk 内存块已经被堆满。

而在下一帧，你编写的多线程 `RotationSystem` 里的 `Job.ScheduleParallel()` 将会如同砍瓜切菜一般，跨越你所有的 CPU 核心，毫无阻塞、毫无引用纠缠地将这十万个坐标点的旋转矩阵重新算出病铺设回去。

通过 Unity 的 `Burst Compiler` 的编译支持。这些循环会被编译成 SSE/AVX 向量级指令。每一次 CPU 滴答，都能同时计算 4 到 8 个实体的旋转。

## 5. 见证奇迹的时刻

点击 Unity 编辑器顶部的 `Play` 按钮运行游戏。

你会看到屏幕上瞬间爆发出 10 万个漂浮且不断自转的三维立方体。它们像一个庞大的、遵循统一物理法则的星云一样在屏幕里运动。

此时，你可以打开 Unity 的 **Profiler（性能分析器）**：
- 观察 CPU 耗时，那个巨大的 Update() 开销彻底消失了。
- 取而代之的是底部的 **Job 线柱**，你会看到各个 `Worker Thread`（工作线程）非常均匀、毫无间隙地塞满了绿色条块。这证明无论你电脑有多少核，都已经在一同狂奔！
- 查看 GC 内存分配指标（GC Alloc），那是一个大写而漂亮的 **0 B**！游戏运行期间没有任何堆内存抖动。你的系统甚至可以永远跑下去而不会触发哪怕一毫秒的主线程垃圾回收卡顿。

这就是 ECS，这便是属于数据导向架构的神迹。

通过将人类思维中繁复的“对象”，解构为纯粹流动在这个 16KB 区块里的“数字流”。你真正掌握了计算机硬件之所以被称为计算机器的本源真理。

至此，关于 ECS 从基础到工业级巅峰的所有实战演练正式收官！
如果还有没想通的概念，欢迎随时翻阅前面章节。在最后，我们有一份关于这趟重塑认知之旅的简短《**总结与拓展**》。
