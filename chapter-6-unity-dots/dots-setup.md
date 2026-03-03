# 开发环境配置

在上一章中我们介绍了 Unity DOTS 的核心架构。要真正在 Unity 中使用这套技术栈，我们需要进行一些特定的环境配置。因为 DOTS 相关组件目前并不是 Unity 默认安装的一部分，而是作为 Package（包）独立存在的。

本章将指导你如何从零开始，在一个纯净的 Unity 工程中搭建支持 DOTS 的开发环境。

## 1. 引擎版本选择

Unity DOTS 经历了漫长的迭代，API 变化曾经非常大。
目前最稳妥、最建议的开发版本是 **Unity 2022.3 LTS（长期支持版）或更高版本（如 Unity 6）**。
在这个版本段，Entities 包已经正式迈入了 `1.0.0` 以上的 Release 阶段，核心 API（如 SystemAPI 和 Baker）已经非常稳定。

建议使用 Unity Hub 安装最新的 `2022.3.x` LTS 版本，并确保安装了对应操作系统的 Build Support 模块（如 Windows Build Support (IL2CPP) / Mac Build Support (IL2CPP)）。

## 2. 新建工程并修改架构设置

1. 打开 Unity Hub，点击 **New project** (新建项目)。
2. 选择 **3D (URP)** 或 **3D (HDRP)** 模板（DOTS 中的实体渲染库目前强依赖于现代可编程渲染管线，极其不建议使用传统 Built-in 管线）。
3. 命名你的项目（如 `DOTS_Tutorial_Project`），然后点击 Create (创建)。

一旦项目打开，为了更好地支持 Burst 编译和数学计算，我们需要先去检查一下工程设置：
- 打开 `Edit -> Project Settings -> Player`。
- 在 `Other Settings` 下找到 `Api Compatibility Level`，确保选择的是 **.NET Standard 2.1** 或 **.NET Framework**。
- (可选但推荐点) 若有 IL2CPP 相关的设置，为了发布时的极致性能，最终往往会使用 IL2CPP 而不是 Mono 后端。

## 3. 安装关键的 Package 依赖

DOTS 是由一系列零碎的包组合而成的，但 Unity 提供了一个统合的大包。

1. 打开菜单栏的 `Window -> Package Manager`。
2. 在 Package Manager 窗口的左上角，点击 `+` 号。
3. 选择 **Add package by name** (通过名称添加包)。
4. 在弹出的输入框中，输入首个必装的包名：
   `com.unity.entities.graphics`
   （或者你可以直接输入 `com.unity.entities`，通常安装 graphics 包会自动将其依赖的 entities 基础包一同装上）。
5. 稍等片刻让 Unity 去下载并进行编译。

当这个核心的 Entities 包（包含 Unity.Entities, Unity.Collections, Unity.Jobs 等组件）安装完毕后，我们需要再确保几个相关插件已启用：
- 再次点击 `+` 选择 **Add package by name**。
- 输入 `com.unity.physics` (Unity 的无状态物理引擎，如果你需要物理模拟)。
- 输入 `com.unity.mathematics` (这是必定会装的基础包，提供了与着色器极其相似、对 Burst 极度友好的数学库 `float3`, `int2` 等)。
- 检查 `com.unity.burst` 是否已经存在且更新到最新版本。

## 4. 验证环境：你好，实体世界！

安装完成后，我们如何验证 DOTS 已经被正确整合进了我们的项目？

**方式 1：查看 Hierarchy 结构改变**
在一个安装了 Entities 的工程中，当你想要创建一个能被 ECS 处理的 SubScene (子场景) 时。
你可以在 `Hierarchy` 面板的空白处右键点击：
你会发现菜单里多了一项：`New Sub Scene`。
这是 DOTS 开发的专属标志。只有放入 Sub Scene 里的对象，才会在游戏运行时被执行底层的大规模烘焙（Baking），转化为那一串串高速的纯数据。

**方式 2：开启 Entities Hierarchy 窗口**
在编辑器顶部菜单栏点击 `Window -> Entities -> Hierarchy`。
这个新窗口长得和普通的游戏物体结构窗口一样，但内部是空的。
当你运行游戏时（并且有实体生成），这个面板里会列出当前世界上存活的纯数据实体（只是一堆带着数字 ID 的图标），并且能查看它们身上挂载了哪些内存紧凑的组件数据。

**方式 3：体验 Unity Mathematics 的快感**
在任何 C# 脚本的开头加入：
```csharp
using Unity.Mathematics;
```
现在，你就可以在代码中使用 `float3` 代替沉重的 `Vector3`，使用 `math.sin()` 替代 `Mathf.Sin()`。这些小写开头且类型确定的数据结构，是未来撰写极速管线、迎合 Burst 编译器 SIMD 向量化优化的专属“原材料”。

## 结语

环境配置完毕后，你的工程底层就已经更换了一台 V8 引擎。
但问题来了，Unity 的整个编辑器 UI 和我们的开发习惯，依然是重度依赖“往 Hierarchy 里拖物体、挂 MonoBehaviour 脚本”的。

既然 ECS 的底层是那么反人类的“纯数据数组”和“无状态函数”，我们要如何把这种直观的编辑体验（拖拽可视化）与底层的深奥架构（Archetype / Chunk）连接起来呢？

答案是通过一套被称为 **Baking（烘焙）与 Authoring（数据创作）** 的奇妙机制。这将是我们下一章研究的核心。
