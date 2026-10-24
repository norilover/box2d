# 06 · 冲量、子步与约束求解器

[返回学习路线](../learning_guide_zh.md) · 下一篇：[关节](07_joints.md)

目标：理解盒子落地后为什么不继续下落，以及为什么堆叠问题需要迭代。先读 [solver.h](../../src/solver.h)，再沿 [solver.c](../../src/solver.c) 的 `b2Solve` 和 `b2SolverTask` 进入 [contact_solver.c](../../src/contact_solver.c)。

## 最少的物理知识

力通过一段时间改变速度；冲量直接表达动量变化。对质量 m 的刚体，线速度变化为 `Δv = J / m`。接触点偏离质心时，冲量还会通过力臂改变角速度，二维形式为 `Δω = I⁻¹ × cross(r, J)`。

一对物体的接触求解同时更新双方，方向相反。有效质量把两边的逆质量、逆惯量和接触力臂组合起来，不只是把两个质量相加。

法向约束阻止继续向内运动，普通接触不应把两物体吸在一起；切向摩擦抵抗滑动，其可用冲量受法向支撑限制。恢复系数描述反弹，还受速度阈值等条件影响。

## 先读概念上的单点约束

忽略旋转、摩擦与柔性参数后，可把一次法向速度修正理解为：

```text
vn = 两物体接触点相对速度在法线方向上的分量
delta = -effectiveMass * vn
newImpulse = max(oldImpulse + delta, 0)
appliedImpulse = newImpulse - oldImpulse
将 appliedImpulse 沿法线作用到两边
```

这是帮助理解的简化式，不是本仓库完整实现。关键在于钳制**累计冲量**，本轮增量可以为负，以修正前面过大的冲量。真正实现还涉及预测接触、柔性误差修正、旋转、摩擦、滚动阻力和反弹。

## 本仓库的阶段顺序

```text
准备关节与接触约束
    ↓
对每个子步：
  IntegrateVelocities  外力、重力、阻尼等影响速度
  WarmStart            应用已缓存的约束冲量
  Solve                带误差修正求解约束
  IntegratePositions   使用速度推进位置增量与旋转增量
  Relax                去掉相应偏置后再调整约束速度
    ↓
按条件执行 Restitution
    ↓
StoreImpulses，供后续 step 使用
```

阶段名来自 `b2SolverStageType`；实际执行有按颜色分组、重复迭代和 overflow 路径。第一次阅读可以先不进入 SIMD 指令，但要认清哪些操作在每个子步内，哪些在子步循环之后。

Warm start 使用上一状态提供的较好初值；多次处理约束让局部修正向堆叠的其他部分传播。子步缩短每次运动距离，也有助于减少非线性误差。这些作用不同，不能统一称作“提高精度的次数”。

## Softness 的三个量

`b2MakeSoft` 根据频率 hertz、阻尼比 zeta 和子步 h，得到 `biasRate`、`massScale`、`impulseScale`。读代码时先确认它们分别影响误差反馈、有效质量项和历史冲量项，再去推导公式。频率与阻尼给的是约束响应参数，不等同于渲染帧率。

可配合作者的 [Solver2D](https://box2d.org/posts/2024/02/solver2d/) 理解软约束、子步和 relaxation 的设计动机；文章讨论多种实验求解器，具体流程仍以当前 `b2SolverTask` 为准。

## 实验与验收

选 [sample_stacking.cpp](../../samples/sample_stacking.cpp) 的堆叠场景，分别使用 1、4、8 个子步，记录稳定性和 step 耗时。固定总模拟时间、材料、初始姿态和休眠配置，避免把不同实验混为一谈。

再读 [test_restitution.c](../../test/test_restitution.c)，研究反弹测试怎样控制初始条件。用 [sample_restitution.cpp](../../samples/sample_restitution.cpp) 观察恢复系数变化。

验收：能从“接触相对速度 → 冲量 → 速度变化 → 位置变化”讲清一轮更新，并解释为什么不能简单把所有发生重叠的物体直接移回上一帧。

参考：[模拟手册](../simulation.md)、[参考资料](13_reading.md)中的 Sequential Impulses、Understanding Constraints 和 Box2D-Lite。
