# 存储结构：原型模式 (Archetype)

在上一章中我们看到，稀疏集（Sparse Set）在处理单组件查询时表现完美，但在处理多组件相交（Intersection）查询时，由于不同组件密集数组中实体的物理顺序不一致，重新引入了指针跳跃的开销。

为了达到极致的、100% 紧致的多组件缓存命中率，Unity DOTS、Flecs 等重量级工业 ECS 框架选择了一种更为刚猛的内存管理模型：**原型模式（Archetype）**。

## 1. 什么是原型 (Archetype)？

在原型模式中，当我们在建立底层数组时，**不再是以单一组件**（如只建一个全是 Position 的数组）为基准，而是**以组件的组合（即原型）**&#x4E3A;基准来建立内存块（Chunk）。

“原型”就是指**一个实体身上挂载的所有组件类型的唯一集合**。

举个例子：

* 如果实体 1 和实体 2 都只有 `[Position, Velocity]`，它们属于**同一个原型：Archetype A**。
* 如果实体 3 除了 `[Position, Velocity]`，还多挂载了一个 `[Health]`，那么它就属于**全新的另一个原型：Archetype B**。

## 2. 内存块 (Chunk) 与完全紧身的 SoA

在原型理论下，ECS 框架在分配内存时，会为每一种**原型**申请若干固定大小的独立内存块（在 Unity DOTS 中叫 Chunk，通常是 16KB）。

在这个属于 Archetype A 的 Chunk 中，只被允许存入完全拥有 `[Position, Velocity]` 的实体数据，并且严格按照高度对齐的 SoA 长条排列：

```
Chunk 1 (Archetype A: [Position, Velocity]):
--------------------------------------------------
Entity ID Array:  | ID_1 | ID_2 | ID_... |
Position Array:   | Pos1 | Pos2 | Pos... |
Velocity Array:   | Vel1 | Vel2 | Vel... |
--------------------------------------------------
```

当一个系统（System）的签名是查询 `[Position, Velocity]` 时发生什么？ 框架会直接抛给它所有属于 Archetype A 的 Chunk。接下来，系统进行批处理时，**`Pos1` 和 `Vel1` 在所属的各自小数组内的索引是 100% 锁定一致的。**

系统在 `Position` 数组里往下读一个元素的同一微秒，在 `Velocity` 数组里往下读一个元素，拿到的绝对是同一个实体的组件数据。

这就彻底消灭了多组件查询时的任何一次查表与指针追逐。多组件遍历变成了在同构 Chunk 内做纯粹的 O(N) 暴力直行。这就是原型模式对性能做出的究极承诺。

## 3. 代价：沉重的结构变化（Structural Change）

俗话说，没有免费的午餐。原型模式为了换取这种变态级别的绝对一致性查询，付出了极高的写入代价。

假设实体 1 刚开始只有 `[Position, Velocity]`，它快乐地生活在 Archetype A 的 Chunk 里。 现在，游戏逻辑决定给实体 1 附加一个流血 BUFF，即调用了 `AddComponent<Health>(Entity_1)`。

在稀疏集里，做这层操作很容易，往后追加数据就行。但在原型模式下，一场巨大的“内存搬家”惨案发生了！

实体 1 现在的组合变成了 `[Position, Velocity, Health]`，这不符合 Archetype A 了。框架必须：

1. 找出或新建一个适合 `[Position, Velocity, Health]` 的内存块（即 Archetype B 的 Chunk）。
2. 把实体 1 的 `Position` 从 Chunk A **拷贝**到 Chunk B。
3. 把实体 1 的 `Velocity` 从 Chunk A **拷贝**到 Chunk B。
4. 在 Chunk B 初始化 `Health` 数据。
5. 在 Chunk A 中，由于实体 1 搬家留下了一个空洞，框架必须把 Chunk A 里最后一位实体的数据全套**拷贝**过来填补这个洞（类似上章讲的 Swap and Pop，但拷贝的数据量巨大）。

这一系列操作，在 Unity DOTS 术语中被称为**结构性改变（Structural Change）**。它极其昂贵，不仅涉及大量内存数据的复制，甚至会引发主线程阻塞和 Job System 的同步锁。

## 4. 如何在工业界驾驭 Archetype？

既然运行时添加或删除组件代价如此巨大，那我们在实际编程时应该怎么做呢？

工业界总结出的最佳实践是：**前期做好大包大揽，避免运行时的频繁卸装。**

如果一个单位偶尔需要中毒掉血。不要在它不中毒时把 `Poison` 组件删掉，中毒时再加回来（引发两次巨大的结构搬家）。 正确的做法是：在实体诞生时，就挂载 `[Position, Velocity, Health, Poison]`，让它一直呆在这个 Archetype 中。当不中毒的时候，可以：

1. 把 `Poison` 组件里的持续时间数值置为 0，然后在 System 开头写个 `if (Poison.duration <= 0) continue;`（有少部分性能损耗）。
2. 或者在 Unity DOTS 中引入针对原型的“开关组件”（Enableable Components）特性，从更底层的位掩码（Bitmask）去过滤计算，而不移动物理内存位置。

## 总结

到这里，ECS 世界里两派最重要的内存组织结构你已经全部了然于胸了。

* **稀疏集（Sparse Set，如 EnTT）**：适合组件增删极为频繁，对象组合类型繁多的游戏。牺牲了少量多组件联查的带宽，换取了高度灵活的组装自由。
* **原型（Archetype，如 DOTS, Flecs）**：适合那些在生命周期中挂载组件极少发生变化的同质化海量单位。它接受添加组件时的“剧烈疼痛”，确保了核心运行图景下的每一根内存管都是被塞得严防死守、水泄不通的。

这两个底层结构的揭秘，标志着你已经跨过了 ECS 这门技术的最大学习门槛。接下来&#x7684;**《第四部分：硬件级性能剖析》**，我们将真正下探到决定这些算法生死的基石——CPU 高速缓存微架构，看看什么叫真正的“在钢铁上刻字母”。
