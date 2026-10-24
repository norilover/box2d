# 04 · 几何、凸包、距离与连续碰撞

[返回学习路线](../learning_guide_zh.md) · 下一篇：[宽相与接触](05_collision.md)

目标：理解碰撞系统使用的几何工具。许多函数可以脱离 World 使用，因此这一模块很适合先写几个输入固定的小实验。

## 文件地图与阅读顺序

| 顺序 | 文件 | 解决的问题 |
| --- | --- | --- |
| 1 | [collision.h](../../include/box2d/collision.h) | 几何、距离、投射、流形等公开结构与函数约定 |
| 2 | [geometry.c](../../src/geometry.c) | 点包含、质量属性、AABB、射线与各类形状的计算 |
| 3 | [hull.c](../../src/hull.c) | 将点集整理成有效凸包，检查几何合法性 |
| 4 | [distance.c](../../src/distance.c) | GJK 距离、Shape Cast、Time of Impact |
| 5 | [manifold.c](../../src/manifold.c) | 为具体形状组合生成局部接触流形 |

圆用中心和半径表示；胶囊是线段沿半径向外扩张；凸多边形要求凸性和正确的顶点组织。凹多边形通常需要拆成多个凸 Shape；Chain 适合边界地形，邻接顶点用于处理接缝。

## 先分清四个问题

- overlap：给定两个形状，现在是否重叠？
- distance：现在最近距离是多少，最近点在哪里？
- cast：让点或形状沿给定运动扫过，何时命中？
- manifold：为了求解物理接触，需要哪些接触点、法线和分离距离？

一次 AABB 查询只能提供粗筛结果。距离为零也不能替代完整的接触流形。射线从形状内部开始时是否报告命中，必须按具体 API 的约定理解，不能假定所有 cast 都返回“出口”。

## GJK 可以怎样入门

把两个凸形状的问题转成 Minkowski 差与原点的距离问题。support 函数给出某个方向上最远的顶点；GJK 逐步维护一个 simplex，在二维里可由点、线段或三角形组成，并寻找更接近原点的组合。

先看 `b2ShapeDistance` 的输入与输出，再看 `b2SimplexCache` 如何复用上次搜索信息，最后进入 simplex 更新逻辑。先不要同时推导 TOI。缓存是加速信息，不是“上一帧距离直接当作本帧答案”。

## 连续碰撞为什么存在

高速物体在两个离散位置之间可能完整穿过薄墙，即 tunneling。`b2TimeOfImpact` 研究一段运动中的相遇时间；`b2SolveContinuous` 在 [solver.c](../../src/solver.c) 中将这种几何计算接入模拟。

`isBullet` 和 `enableContinuous` 影响连续处理路径。子步能降低单次位移，但不能把“多做几个子步”当作任意高速场景不会穿透的证明。旋转、目标类型和初始重叠也会影响结果。

## 实验与验收

1. 读 [test_distance.c](../../test/test_distance.c)，平移两个圆或方盒，手算一组简单距离再核对输出。
2. 读 [test_collision.c](../../test/test_collision.c)，检查 `b2CollideCircles`，理解半径与 separation 的关系，再看多边形裁剪。
3. 在 [sample_geometry.cpp](../../samples/sample_geometry.cpp) 与 [sample_continuous.cpp](../../samples/sample_continuous.cpp) 中观察距离、扫掠和高速碰撞。

验收：能分别选择距离查询、Shape Cast 和 manifold 来解决“离墙多远”“这一段移动会不会撞墙”“接触后怎样约束刚体”三个问题。

参考：[碰撞手册](../collision.md)；[参考资料](13_reading.md) 中的 GJK、Continuous Collision、Contact Manifolds 讲义。
