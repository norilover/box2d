# 10 · 数学、内存、容器与平台基础

[返回学习路线](../learning_guide_zh.md) · 下一篇：[快照、录制与大世界](11_advanced.md)

目标：遇到内部辅助函数时知道该去哪里查，也理解这些基础设施为何影响性能与正确性。

## 必备数学和单位

| 概念 | 直觉 | 对应类型/用途 |
| --- | --- | --- |
| 位置与位移 | 点的位置与两个点的差不同 | `b2Pos` 与 `b2Vec2` |
| 点积 | 一个向量沿另一个方向的分量 | 接触法向速度、投影 |
| 二维叉积 | 两向量的有向面积；结合力臂产生转动 | 力矩与角速度变化 |
| 旋转 | 用 cos/sin 表示二维朝向 | `b2Rot`，避免误当作一个角度数值 |
| 变换 | 把局部点旋转并平移到另一坐标系 | `b2Transform`、`b2WorldTransform` |
| 逆质量、逆惯量 | 决定同样冲量产生多大速度变化 | 约束有效质量 |

阅读 [math_types.h](../../include/box2d/math_types.h)、[math_functions.h](../../include/box2d/math_functions.h) 和 [math_functions.c](../../src/math_functions.c)，先手算一个平移加 90 度旋转的点，再检查逆变换是否还原。

常规约定是米、千克、秒、弧度。屏幕像素应通过应用自己的比例转换。几何容差与引擎长度单位有关；不要把所有浮点比较机械地改成 `==`，也不要未经分析到处减小 epsilon。

## 基础设施地图

| 文件 | 职责 | 初读重点 |
| --- | --- | --- |
| [core.c](../../src/core.c)、[core.h](../../src/core.h) | 分配、断言、日志和内部基础配置 | 谁拥有内存、如何安装调试钩子 |
| [arena_allocator.c](../../src/arena_allocator.c) | step 临时内存的栈式分配 | 分配/释放顺序与 step 后的清空检查 |
| [container.h](../../src/container.h) | 内部动态数组等容器工具 | count 与 capacity、扩容、删除元素后的移动 |
| [id_pool.c](../../src/id_pool.c) | 槽位 ID 分配与回收 | free list 与对象 generation 的分工 |
| [bitset.c](../../src/bitset.c) | 紧凑记录状态标记与集合操作 | 多线程结果合并与稳定遍历 |
| [table.c](../../src/table.c) | 内部哈希集合/表支持 | 键、冲突、扩容与去重 |
| [qsort.h](../../src/qsort.h) | 内部排序辅助 | 候选/结果排序位置及比较规则 |
| [platform.h](../../src/platform.h) | 平台、原子与线程相关适配 | 先看接口，再看平台分支 |
| [timer.c](../../src/timer.c) | ticks 与耗时测量 | 测量单位与转换 |
| [config.h](../../include/box2d/config.h)、[base.h](../../include/box2d/base.h)、[constants.h](../../include/box2d/constants.h) | 对外配置、基础接口与常量 | 与 CMake 编译定义的关系 |

`arena_allocator` 文件里的类型叫 `b2Stack`。它主要服务临时工作内存，不是保存全部 Body 的永久对象池。`id_pool` 分配的是 ID 槽位，也不等于“创建了完整物理对象”。

## 阅读容器时特别注意

连续数组扩容会换地址；删除时若把末尾元素搬到空位，必须同步更新该元素的反向索引。这正是 Body 的 `localIndex` 和约束索引需要维护的原因。性能设计和对象生命周期在这里联系起来了。

编译配置也参与语义：`BOX2D_DISABLE_SIMD`、`BOX2D_AVX2`、`BOX2D_DOUBLE_PRECISION` 并不只是影响一个源文件。消费者、内部测试与库必须使用一致的相关定义。根 CMake 中抑制 FMA 收缩的配置服务于浮点结果一致性，别为了“更快”随意加 fast-math。

## 实验与验收

选择 [test_math.c](../../test/test_math.c)、[test_container.c](../../test/test_container.c)、[test_id.c](../../test/test_id.c)、[test_bitset.c](../../test/test_bitset.c)、[test_table.c](../../test/test_table.c) 各读一个最小测试。用调试器观察数组扩容或删除后索引如何变化。

验收：能说明 ID 有效不等于内部地址永远不变；能说出三种不同坐标量的单位；能解释临时栈与永久数组的生命周期差别。

参考：[Foundation](../foundation.md)、[Loose Ends](../loose_ends.md)。
