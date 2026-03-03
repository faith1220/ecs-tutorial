# Summary

* [前言](README.md)

## 第一部分：设计范式演进
* [面向对象编程（OOP）的性能瓶颈](chapter-1-rethinking/oop-dilemma.md)
* [面向数据设计（DOD）引言](chapter-1-rethinking/dod-introduction.md)

## 第二部分：ECS 核心元素解析
* [实体（Entity）：标识符的本质](chapter-2-core/entity.md)
* [组件（Component）：连续的数据容器](chapter-2-core/component.md)
* [系统（System）：无状态的逻辑处理](chapter-2-core/system.md)

## 第三部分：核心术语与底层概念
* [重要专有名词解析 (Glossary)](chapter-3-glossary/terminology.md)
* [内存布局对比：AoS 与 SoA](chapter-3-glossary/aos-vs-soa.md)
* [存储结构：稀疏集 (Sparse Set)](chapter-3-glossary/sparse-set.md)
* [存储结构：原型模式 (Archetype)](chapter-3-glossary/archetype.md)

## 第四部分：硬件级性能剖析
* [内存层级与 CPU 缓存机制](chapter-4-under-the-hood/memory-and-cpu-cache.md)
* [缓存未命中 (Cache Miss) 的影响](chapter-4-under-the-hood/cache-miss.md)

## 第五部分：框架实现与演练
* [构建基础 ECS 框架](chapter-5-practice/writing-a-simple-ecs.md)
* [实体行为模拟：移动与碰撞](chapter-5-practice/practical-example.md)

## 第六部分：Unity DOTS 应用指南
* [Unity DOTS 架构概述](chapter-6-unity-dots/dots-intro.md)
* [开发环境配置](chapter-6-unity-dots/dots-setup.md)
* [数据转换机制：Baking 与 Authoring](chapter-6-unity-dots/dots-baking.md)
* [系统实现：SystemAPI 与数据查询](chapter-6-unity-dots/dots-system-api.md)
* [并发处理：C# Job System 与 Burst 编译器](chapter-6-unity-dots/dots-jobs.md)
* [综合演练：海量实体同屏渲染与更新](chapter-6-unity-dots/dots-demo.md)

* [总结与拓展](conclusion.md)
