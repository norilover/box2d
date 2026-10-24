# 09 · 岛、休眠、图着色、线程与 SIMD

[返回学习路线](../learning_guide_zh.md) · 下一篇：[基础设施](10_foundation.md)

目标：理解模拟如何减少工作量，以及哪些剩余工作能并行。建议已完成一次单线程 step 跟踪后再读。

## 四种组织各管什么

| 组织 | 作用 | 源码 |
| --- | --- | --- |
| Island | 保存通过约束相连的刚体关系，支持整组休眠/唤醒与连接变化 | [island.c](../../src/island.c) |
| Solver set | 按静态、禁用、活动、睡眠状态紧凑存放模拟数据 | [solver_set.c](../../src/solver_set.c) |
| Constraint graph | 将活动约束安排到可安全并行处理的颜色组 | [constraint_graph.c](../../src/constraint_graph.c) |
| Stage / Block | 把某一阶段的数据切成工作块并同步执行 | [solver.h](../../src/solver.h)、[solver.c](../../src/solver.c) |

Island 和颜色不是同一个分组。一个大堆叠可能是一个岛，但其中仍有可并行处理的独立约束。反过来，睡眠岛可以直接离开活动求解路径。

## 休眠如何减少工作

把刚体看成顶点、接触或关节看成边，岛类似连通分量。静态地面不会把所有落在其上的独立物体都串成一个必须共同唤醒的大岛。当前实现维护持久岛，连接增加、删除时需要合并或在合适时机拆分。

`solver_set.h` 中的实际枚举为：0 静态、1 禁用、2 活动、3 起为睡眠集合。活动 Body 的热数据连续放置；睡眠时转移出去，唤醒再转回来。跟踪 `b2TrySleepIsland` 与 `b2WakeSolverSet`，注意迁移会改变内部索引和地址。

## 图着色解决写入冲突

两个约束若同时修改同一个动态 Body 的速度，就不能直接并行更新。着色把这种冲突约束分开；同色中不存在相应写入冲突时可以并行。静态和运动学数据如何共享，要看当前的可写状态规则。

颜色数量有限，放不进普通颜色组的约束走 overflow 路径。初读时把它当作保证完整求解的兜底，随后再研究它对吞吐的影响。不能简单把每个 Contact 扔进任意线程后就期待结果正确。

## 当前仓库内置调度器

在 `b2CreateWorld` 可看到三条路径：

1. `workerCount > 0` 且提供完整 enqueue/finish 回调：使用外部任务系统。
2. 否则 `workerCount > 1`：创建 [scheduler.c](../../src/scheduler.c) 中的内置调度器。
3. 其余情况：串行回退，内部 worker 数为 1。

[parallel_for.c](../../src/parallel_for.c) 把范围拆块，通过原子操作领取工作；求解器还使用阶段同步和可复用的 `b2SyncBlock`。按“每块是否只执行一次”“下一阶段是否等待上一阶段完成”两个问题阅读，比先研究所有原子指令更有效。

[simd.h](../../src/simd.h) 则让一条指令处理多个约束的数据通道。SIMD 是单个执行流里的数据并行，多线程是多个执行流；两者能叠加。先认清 gather/scatter 与宽约束的输入输出，再看具体 SSE2、Neon、AVX2 分支。

## 确定性比没有数据竞争更严格

即使互斥锁保护了数组，线程抢到插入位置的顺序仍可能改变约束顺序，进而改变浮点结果。检查线程局部 bitset、候选排序和结果合并，理解它们如何固定可观察顺序。

可读作者的 [Simulation Islands](https://box2d.org/posts/2023/10/simulation-islands/) 与 [Determinism](https://box2d.org/posts/2024/08/determinism/) 了解设计背景；文章年份早于本仓库部分功能，尤其回滚能力应看当前快照实现。

## 实验与验收

运行 `ThreadTest` 与 `DeterminismTest`（见[工程与验证](12_engineering.md)），阅读 [test_thread.c](../../test/test_thread.c)、[test_determinism.c](../../test/test_determinism.c)。重建相同初始 World，对比 0、2、4 个 worker 的状态结果与耗时；小场景线程越多可能越慢。

验收：能分别解释 Island、set、color、block 的作用，能指出“线程安全”为什么不能自动推出“结果确定”。
