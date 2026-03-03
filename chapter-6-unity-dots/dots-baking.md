# 数据转换机制：Baking 与 Authoring

在理解了 Unity Entities 的高能特性后，我们面临一个非常现实的工程难题：

Entities 底层要求极其严苛的纯数据架构（非托管结构体、连续无引用的内存块）。而 Unity 编辑器由于历史原因，是以可视化、面向对象的 `GameObject` 和 `MonoBehaviour` 为核心的。我们无法在检视面板（Inspector）里直接拖拽或配置底层长相丑陋的 `NativeArray`。

为了弥合“高效编辑（OOP）”与“高速运行（DOD）”之间的鸿沟，Unity 提出了目前最优雅的解决方案：**Baking（烘焙）架构**。

## 1. 原理：就像在烤面包

想象你在做面包：
1. **揉面团（Authoring）**：在面盆里，你向面粉中加入水、糖、酵母，揉捏成形。这个面团极其柔软，可以随意拉伸、合并（就像你能随意往 GameObejct 上挂一堆 MonoBehaviour，配置它们的 Inspector 数值）。
2. **入烤箱（Baking）**：你把面团放进烤箱进行高温烘烤。这是个不可逆的化学反应。
3. **出炉（Entities）**：烤好后的面包结构固定、松脆可口，直接用于高效的食用。在这个阶段，面团里那些多余的水分（面向对象的冗余开销）被全部蒸发。

在 Unity DOTS 中：
- 我们依然像往常一样在 Scene 和 Prefab 中使用 GameObject 创建场景，并且在上面挂载名为 **Authoring（创作数据）** 的 MonoBehaviour 组件。
- Unity 在后台会自动触发一个极速的 **Baking** 过程。
- 这个过程读取 Authoring 数据，然后在后台的 ECS 内存世界里，为你生成轻量级的纯数据 **Entity** 和对应的值类型 **Component**。

这就是我们在上一章配置环境时提到的 **Sub Scene（子场景）** 的真面目：把一个普通的 Scene 放到 Sub Scene 结构下，其实就是把它送进了 Unity 的免干预自动烤箱。

## 2. 第一步：定义底层纯数据组件 (IComponentData)

让我们用一个“旋转立方体”的小用例来跑通这个机制。首先，像往常一样，我们先建立底层的（要在 System 中被高速访问的）纯数据组件。

```csharp
using Unity.Entities;

// 这是我们在 ECS 运行时需要的组件
public struct RotationSpeed : IComponentData {
    public float RadiansPerSecond;
}
```

## 3. 第二步：编写 MonoBehaviour 面团 (Authoring)

我们无法直接把 `IComponentData` 挂载到层级视图的物体上。所以我们需要写一个传统的 `MonoBehaviour` 来让策划或关卡设计师在 Inspector 中进行配置。

```csharp
using UnityEngine;

// 这是一个普通的 MonoBehaviour，用于编辑器可视化配置
public class RotationSpeedAuthoring : MonoBehaviour {
    // 供人在面板中友好调节的角度（度）
    public float DegreesPerSecond = 360f; 
}
```

## 4. 第三步：编写翻译官 (Baker)

在旧版的 DOTS 中，这两个过程耦合得很深，并且依赖反射，存在很大隐患。在 1.0 版本后，Unity 引入了强大且明确的 `Baker<T>` 机制。

我们要在同一个文件中（或者另一个独立文件），编写一段代码，告诉 Unity 的烤箱：“当你烤熟这个 `RotationSpeedAuthoring` 面团时，请把它变成什么样的 `RotationSpeed` 组件数据。”

```csharp
using Unity.Entities;
using Unity.Mathematics;
using UnityEngine;

public class RotationSpeedAuthoring : MonoBehaviour {
    public float DegreesPerSecond = 360f; 
}

// Baker 必须继承自 Baker<绑定的Authoring类型>
public class RotationSpeedBaker : Baker<RotationSpeedAuthoring> {
    
    // Bake 函数就是烘焙的车间
    public override void Bake(RotationSpeedAuthoring authoring) {
        
        // 我们要得到转化后的核心标志：当前正在烘焙的 Entity ID
        // TransformUsageFlags 告诉底层这个物体需不需要进行坐标系维护（不维护最省性能）
        Entity entity = GetEntity(TransformUsageFlags.Dynamic);

        // 创建纯数据组件。我们在烘焙阶段顺手进行数学换算（把度转化为弧度），
        // 这样在每帧更新时（System），底层可以直接拿数据用，节省一次乘法带来的损耗。
        RotationSpeed speedData = new RotationSpeed {
            RadiansPerSecond = math.radians(authoring.DegreesPerSecond)
        };

        // 将数据紧紧粘合在生成的 Entity 身上
        AddComponent(entity, speedData);
    }
}
```

## 5. 运作流程闭环

完成这三个步骤后，你就可以像往常一样在传统的层级视图里，建一个有着 Transform 的 Cube 预制体，并且把 `RotationSpeedAuthoring` 脚本拖上去。

一旦你将这个 GameObject 拖进 `Sub Scene`，神奇的事情就发生了：
1. Unity 的后台进程捕获到 GameObject 变更。
2. 调用 `RotationSpeedBaker.Bake()`。
3. GameObject 上繁重的 Transform 组件被底层剥离成了极度优化的 `LocalTransform` 二进制 ECS 组件。
4. 你的 `RotationSpeedAuthoring` 上的数据被提取，转化成 `RotationSpeed`。
5. 这个重达几万字节的“庞大对象”，在 ECS 的内存池里缩影成了仅仅有（位置+旋转公式）不到几十个字节的纯血实体。

而这个实体，现在正饥渴地等待着它的调度管道：能够光速压榨它的 **System（系统）**。

了解了进入 ECS 世界的数据大门（Baking）后，下一章《**系统实现：SystemAPI 与数据查询**》，我们将开始揭开 Unity DOTS 运转的心脏部位——看看现代版的 System 语法究竟进化得有多么简洁。
