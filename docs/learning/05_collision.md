# 05 · 宽相、动态树与 Contact 生命周期

[返回学习路线](../learning_guide_zh.md) · 下一篇：[约束求解](06_solver.md)

目标：解释一个形状对怎样从“值得检查”变成“可以交给求解器的接触”。

## 宽相为什么重要

对 N 个 Shape 两两检测需要约 N(N−1)/2 次比较。宽相用空间包围盒排除大量不可能接触的组合，再将少量候选交给窄相。

[dynamic_tree.c](../../src/dynamic_tree.c) 管理动态 AABB 树；树的内部节点包围子树，查询只进入与查询区域有关的分支。叶子对应空间代理。较宽松的包围盒允许物体做小幅移动时减少索引维护，代价是增加假阳性候选。

[broad_phase.c](../../src/broad_phase.c) 管理模拟中的树、过滤与候选配对。当前实现包含树自配对和跨树配对任务、线程局部 pair keys 汇总等步骤；不要直接套用旧教程中“遍历 move buffer 查询树”的逐行解释。

## 跟踪一对 Shape

```text
创建 Shape → 创建 broad-phase proxy
运动使边界需要更新 → 更新/标记空间代理
b2UpdateBroadPhasePairs → 过滤、汇总候选、创建 Contact
b2Collide / b2UpdateContact → 几何计算、缓存与接触状态更新
接触约束进入求解组织 → 冲量写回缓存
分离或销毁等条件 → 移除 Contact、产生相应状态变化
```

这里的“接触约束”包含 speculative contact 的情况：允许在尚有小间距时准备约束，防止下一段运动穿透。Contact 对象存在、流形有点、业务收到 begin/hit 事件，是不同层次的概念。

## 三层数据不要混用

| 数据 | 在哪里读 | 关注点 |
| --- | --- | --- |
| `b2Contact` | [contact.h](../../src/contact.h) | 稳定身份、两个 Shape、Body 关联、集合/颜色索引 |
| `b2ContactSim` | 同上 | 求解与窄相需要的材料、缓存、流形等 |
| `b2LocalManifold` / 公开接触流形 | [collision.h](../../include/box2d/collision.h) | 点、法线、分离距离和坐标系约定 |

当前几何碰撞函数使用局部流形。例如 `b2CollidePolygons` 接受两个多边形及相对变换，不能用旧版接受两个世界变换的代码直接替换。先读函数声明和注释，再观察局部接触信息怎样转换给求解器或公开查询。

流形中的点标识帮助跨帧匹配接触并复用冲量。这是堆叠稳定性的组成部分；若把每一帧都看成全新的碰撞，便难以理解 warm start。

## 过滤和地形接缝

在 [shape.h](../../src/shape.h) 的 `b2ShouldShapesCollide` 观察 `categoryBits`、`maskBits` 与 `groupIndex`：两侧掩码都要允许；相同且非零的 groupIndex 有优先规则。Body 类型、关节连接关系和自定义过滤还会进一步影响候选是否接受。

Chain 邻接顶点用于避免相邻线段的内部顶点形成不合理法线。阅读 [test_chain.c](../../test/test_chain.c) 与本地碰撞手册的 ghost collision 说明，比直接研究全部链段算法更容易建立直觉。

## 实验与验收

使用 [test_dynamic_tree.c](../../test/test_dynamic_tree.c)、[sample_collision.cpp](../../samples/sample_collision.cpp)、[sample_shapes.cpp](../../samples/sample_shapes.cpp)。一次只修改掩码、AABB 尺寸或几何间距中的一个，分别记录候选、流形与事件。

验收：能解释“AABB 相交但没有实体接触”为何正常；能找到 Contact 创建、更新、销毁的源码入口；能说明冲量缓存为什么需要接触点匹配。

参考：[碰撞手册](../collision.md)、[参考资料](13_reading.md)中的 Dynamic Bounding Volume Hierarchies 与 Contact Manifolds。
