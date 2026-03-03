# 系统实现：SystemAPI 与数据查询

经过上一章的 Baking，我们已经在底层拥有了纯净的 Entity 和 `RotationSpeed` 组件。接下来，我们需要编写一台“发动机”让它们转起来。

在 Unity Entities 1.0 时代，编写原生的 C# 系统变得前所未有的简洁。这全靠 Unity 引入的一套名为 `SystemAPI` 的强大语法糖。本章我们将通过实现一个旋转系统，来看看现代 DOTS 中的查询与遍历到底长什么样。

## 1. 声明一个非托管系统结构体

在早期的 DOTS 中，系统一般继承自 `SystemBase` 类（C# class）。但在现在的极限优化流派里，我们更推荐使用 `ISystem` 接口的结构体（C# struct）。

由于 struct 是分配在栈（Stack）上的或是通过非托管内存分配的，加上特定的 `[BurstCompile]` 标签，它们完全免除了垃圾回收（GC），是真刀真枪的极致性能载体。

```csharp
using Unity.Entities;
using Unity.Transforms;      // 包含自带的 LocalTransform 组件
using Unity.Burst;

// 1. 使用 [BurstCompile] 告诉 Burst 编译器接管这个系统的汇编生成
// 2. 实现 ISystem 接口的 struct
[BurstCompile]
public partial struct RotationSystem : ISystem {
    
    // 初始化时调用（可选）
    [BurstCompile]
    public void OnCreate(ref SystemState state) {
        // 如果系统没有任何包含 RotationSpeed 和 LocalTransform 的实体，就不必运行
        state.RequireForUpdate<RotationSpeed>();
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state) {}

    // 核心更新循环，类似于 Update()
    [BurstCompile]
    public void OnUpdate(ref SystemState state) {
        // ... 我们将在下面填充它
    }
}
```

## 2. SystemAPI.Query 的优雅遍历

在 `OnUpdate` 中，我们需要获取所有长着“旋转速度”并且能被挪动位置的实体。
在第三部分，我们讲过多组件查询（尤其是底层是用 Archetype 时），涉及到复杂的 Chunk 抓取。但现在，面对开发者，Unity 把它封装成了一行极其优美的 `foreach` 代码：`SystemAPI.Query`。

让我们把 `OnUpdate` 填满：

```csharp
    [BurstCompile]
    public void OnUpdate(ref SystemState state) {
        
        // 1. 获取全局的时间增量。在 ECS 中时间是全局唯一的系统数据
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 2. 魔法查询与暴力遍历
        // 这里声明了该系统关心的签名：可改写的 LocalTransform，和只读的 RotationSpeed
        foreach (var (transform, speed) in 
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>()) 
        {
            // 通过 .ValueRW 修改读写结构的实际值
            // 通过 .ValueRO 获取只读结构的实际值
            transform.ValueRW = transform.ValueRO.RotateY(speed.ValueRO.RadiansPerSecond * deltaTime);
        }
    }
```

## 3. 语法糖背后的底层发生了什么？

写代码时，看到 `foreach` 很多开发者会警惕：这是一个在堆内存里分配迭代器（IEnumerator）并引发 GC 的行为吗？

**绝对不是！** 
这里的 `SystemAPI.Query` 是一个存在于**源码生成期（Source Generator）**的魔法宏。
在你保存代码进行编译时，Unity 的 Source Generator 会扫描你的代码结构。它会识别出你在查询 `LocalTransform` (可读写) 和 `RotationSpeed` (只读)。

然后它在后台悄悄把这行优雅的 `foreach` 翻译成了如下极端暴力的非托管代码逻辑（用第一视觉描述）：
1. 找出所有拥有这两个组件组合的原型内存块（Chunks）。
2. 将这些 Chunk 以 `NativeArray` 的形式全部提取出来。
3. 把那个简单的旋转算法 `RotateY`，像盖章一样用传统的 C-style 数组遍历，硬扣在内存指针上。

它完美实现了我们在第三部分要求的：**绝无指针追逐，全是顺着连续内存做 O(N) 盲扫。**

## 4. 读写权限（RefRW 与 RefRO）

在这里必须要重点解释代码中的 `RefRW` (Reference Read-Write) 和 `RefRO` (Reference Read-Only)。

还记得我们在讲“伪共享与多线程碰撞”时，提到的那个问题吗？如果不知道谁读谁写，强上多线程就会死锁或者数据报废。

明确标注 `RefRO`（只读）是现代 ECS 编程的核心修养：
当我们告诉系统，我们**只看**速度（`RefRO<RotationSpeed>`），但是**要改**方向（`RefRW<LocalTransform>`）时，ECS 底层框架就吃下了一颗定心丸。它知道如果在其他角落有个系统只改速度，不碰方向，那这俩系统完全就可以一起并行运行！

在单线程（主线程）的 `SystemAPI` 遍历中，读写修饰可能只决定你能点出啥属性。但在下一章我们要讲的 **Job System（多线程并发任务）** 中，这几句修饰词则是决定代码生死和能否通过编译的绝对安检闸门。

## 结语

简单、清晰、没有类的继承链、没有满天飞的虚函数表，只有结构体和纯粹的 `for` 循环数学暴力美学。这就是 Unity DOTS 中 System 编写的现状。

但是，上面的这个 `foreach` 循环仍然跑在你的那颗主 CPU 核心上（即便加了 Burst 加持单线程飞天，它依然只用了一个核）。我们要怎么把这群“旋转算法”丢给你的电脑甚至手机那闲置的十几个物理核心去一起算呢？

这就来到了 DOTS 的灵魂组件，也是传统 OOP 的禁区。下一章：《**并发处理：C# Job System 与 Burst 编译器**》，我们将教你最简单的一键多线程并行化改写方案！
