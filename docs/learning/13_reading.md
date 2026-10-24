# 13 · 参考读物：按问题阅读，而不是一次读完

[返回学习路线](../learning_guide_zh.md)

下面按本仓库学习任务组织资料。外部链接于 2026-10-24 核对；代码依据为本地提交 `956ce4e`，根 CMake 声明版本为 3.2.0。在线手册和 main 分支会继续变化，具体函数签名以本仓库头文件为准。

## 第一层：先读本地手册

| 阅读时机 | 资料 | 带着什么问题读 |
| --- | --- | --- |
| 第一遍建立全局图 | [overview.md](../overview.md)、[hello.md](../hello.md)、[samples.md](../samples.md) | 最小应用有哪些对象与调用步骤？ |
| 接入应用时 | [simulation.md](../simulation.md) | 对象、时间步、过滤、关节与事件怎么使用？ |
| 碰撞模块前 | [collision.md](../collision.md) | 几何、查询、流形有哪些输入输出约定？ |
| 内存与调试时 | [foundation.md](../foundation.md)、[loose_ends.md](../loose_ends.md)、[questions.md](../questions.md) | 单位、分配、断言和常见问题是什么？ |
| 进阶功能时 | [character.md](../character.md)、[recording.md](../recording.md)、[large_worlds.md](../large_worlds.md) | 角色、快照与坐标精度有哪些边界？ |
| 遇到旧代码时 | [migration.md](../migration.md)、[release_notes_v310.md](../release_notes_v310.md) | 旧 API 与当前对象模型哪里不同？ |

[在线官方手册](https://box2d.org/documentation/) 可用于浏览 API 索引；它说明手册对应关联发布版本，不保证与任意 main 提交一致。即使都显示 3.2.0，也要核对字段与函数参数。

## 第二层：作者讲义，按模块挑选

这些讲义可从作者的 [Publications](https://box2d.org/publications/) 总目录找到。它们用于理解算法，不是本仓库逐行代码说明。

| 建议顺序 | 资料 | 对应模块与阅读任务 |
| --- | --- | --- |
| 1 | [Numerical Integration · GDC 2009](https://box2d.org/files/ErinCatto_NumericalIntegration_GDC2009.pdf) | 读第 03、06 篇时，理解离散推进与误差 |
| 2 | [Sequential Impulses · GDC 2006](https://box2d.org/files/ErinCatto_SequentialImpulses_GDC2006.pdf) | 第 06 篇；重点理解累计冲量、摩擦与热启动 |
| 3 | [Contact Manifolds · GDC 2007](https://box2d.org/files/ErinCatto_ContactManifolds_GDC2007.pdf) | 第 04、05 篇；碰撞如何变成稳定的接触信息 |
| 4 | [Computing Distance using GJK · GDC 2010](https://box2d.org/files/ErinCatto_GJK_GDC2010.pdf) | 第 04 篇；support、simplex、最近点 |
| 5 | [Understanding Constraints · GDC 2014](https://box2d.org/files/ErinCatto_UnderstandingConstraints_GDC2014.pdf) | 第 06、07 篇；从运动规则到约束方程 |
| 6 | [Soft Constraints · GDC 2011](https://box2d.org/files/ErinCatto_SoftConstraints_GDC2011.pdf) | 第 06、07 篇；频率、阻尼与柔性约束 |
| 7 | [Continuous Collision · GDC 2013](https://box2d.org/files/ErinCatto_ContinuousCollision_GDC2013.pdf) | 第 04 篇；离散漏碰撞与连续时间处理 |
| 8 | [Dynamic Bounding Volume Hierarchies · GDC 2019，完整版](https://box2d.org/files/ErinCatto_DynamicBVH_Full.pdf) | 第 05 篇；空间树与包围盒层次结构 |

不必先推完全部公式。每次挑一个能对应当前实验的问题，例如“为什么冲量增量可以为负”，读完后回源码找对应变量。

## 第三层：设计文章与小型实现

| 资料 | 用途 | 版本提醒 |
| --- | --- | --- |
| [Box2D-Lite](https://github.com/erincatto/box2d-lite) | 求解器代码太长时，先读一个较小的物理引擎建立直觉 | 教学实现，不是本仓库 API 或并行架构 |
| [Solver2D，2024](https://box2d.org/posts/2024/02/solver2d/) | 比较求解方法，理解子步、软约束与 relaxation | 文中有多个实验方案，不能全当作当前生产实现 |
| [Simulation Islands，2023](https://box2d.org/posts/2023/10/simulation-islands/) | 理解岛、休眠与图连接管理的动机 | 对照当前 `island.c`、`solver_set.c` 阅读 |
| [Determinism，2024](https://box2d.org/posts/2024/08/determinism/) | 理解线程顺序和浮点环境为什么影响结果 | 当时关于缺少回滚机制的描述不适用于已有 snapshot/restore 的本仓库 |

## 第四层：系统参考书

《Real-Time Collision Detection》，Christer Ericson，2005。适合在学习第 04、05 篇时查几何基础、包围体、相交测试与数值稳健性；不必为了运行 Box2D 先整本读完。作者的[配套网站](https://realtimecollisiondetection.net/)提供目录、参考资料与勘误，阅读实现细节时记得核对勘误。

本地 [reading.md](../reading.md) 还列有 Gino van den Bergen 的《Collision Detection in Interactive 3D Environments》（2004），可作为更深入的碰撞检测参考。两本书都涉及 3D，应用到本项目时应先理解二维版本的几何与坐标约定。

## 怎样判断一篇教程是否适用

遇到 `b2World` C++ 对象、直接保存 `b2Body*` 或 `Step(dt, velocityIterations, positionIterations)` 的教程，先按旧版资料处理。若看到 Shape 材料字段、查询原点或 Joint 定义与当前头文件不同，也先核对版本。可以借用数学解释，但应重新用当前 API 完成实验。

每读完一份资料，留下三行笔记：它解释了什么问题；本仓库哪个函数体现这个思路；自己用哪个实验核对了理解。
