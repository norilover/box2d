# 11 · 快照、录制回放与大世界坐标

[返回学习路线](../learning_guide_zh.md) · 下一篇：[工程、样例与验证](12_engineering.md)

目标：认识当前仓库的扩展能力及其边界。它们不是第一周的必修内容，但深入学习本项目时不能遗漏。

## 快照、录制、回放分别是什么

| 功能 | 保存或执行的内容 | 入口 |
| --- | --- | --- |
| Snapshot | step 边界的一份内部模拟状态 | [snapshot.c](../../src/snapshot.c)、`b2World_GetSnapshot` |
| Recording | 初始快照，随后记录 API 操作及校验等信息 | [recording.c](../../src/recording.c)、[recording_ops.inl](../../src/recording_ops.inl) |
| Replay | 恢复初态并按记录驱动引擎，检查结果 | [replay.c](../../src/replay.c)、`b2ValidateReplay`、`b2CreateReplay` |

快照不仅包含位置和速度，还涉及接触缓存、岛、空间树和 ID 池等内部状态。仅将 Body 移回旧位置，不等于完成同样的恢复。

`b2World_Restore` 在已有 World 中恢复，保留该 World 的身份；`b2CreateWorldFromSnapshot` 创建新 World，原 World 的句柄不能直接作为新 World 的句柄使用。两者都需要关注宿主数据与回调配置，不能把业务对象指针当作可移植的存档内容。

## 从最小验证读起

1. 阅读 [recording.md](../recording.md)，在 step 边界调用 `b2World_StartRecording`。
2. 正常创建对象、修改状态和推进模拟，再停止录制。
3. 使用 `b2ValidateReplay` 检查，或在 samples 中打开 `.b2rec`。
4. 对照 [test_recording.c](../../test/test_recording.c) 找到记录、重放、比对的测试；快照单独看 [test_snapshot.c](../../test/test_snapshot.c)。

库的录制缓冲与 World 生命周期分开，停止录制后仍需要由应用销毁缓冲。实际程序还要检查创建、加载、保存、恢复的返回值，避免把手册中强调流程的简写当成完整的文件错误处理。

## 格式与确定性边界

当前快照包含原始结构布局，加载会检查布局、指针宽度、字节序、格式与精度等兼容条件。它不是承诺任意引擎版本都能加载的通用业务存档协议。

`userData` 不会作为有效业务指针恢复；自定义摩擦/恢复系数混合回调需要按约定重新安装。当前录制不支持使用 preSolve 或 customFilter 回调的情形。完整约定见本地录制手册，研究时必须将这些条件纳入实验。

2024 年的确定性文章讨论过当时缺少回滚机制的情况；当前仓库已有快照/恢复代码。另一方面，这也不等于项目已经提供完整的多人网络同步系统：输入历史、业务状态、通信协议与修正策略仍属于应用。

## 大世界不是把全部 float 换成 double

启用 CMake 选项 `BOX2D_DOUBLE_PRECISION=ON` 后，世界位置 `b2Pos` 和 `b2WorldTransform` 的平移部分使用双精度；局部几何、速度、旋转、接触求解以及动态树仍主要使用单精度。

这是把大数值留在世界坐标边界，将局部运算放回附近坐标系的设计。读 [math_types.h](../../include/box2d/math_types.h)、[body.h](../../src/body.h) 中的 `b2MakeRelativeSweep`，再看 [shape.c](../../src/shape.c) 的保守包围盒计算。

查询的 `origin` 参数尤其重要：在远离原点时，把附近 Body 或相机位置作为基点，查询几何使用相对坐标。不要先把绝对位置强制缩成 `b2Vec2` 再相减，这会提前丢失精度。

双精度边界也不代表无限精度的全部查询。浮点树在远处会产生更粗的包围盒与更多候选；本地 [large_worlds.md](../large_worlds.md) 还说明了射线/形状扫掠擦边时的树遍历限制。研究范围与精度时应包含这些边界条件。

## 实验与验收

用两个独立构建目录比较默认模式和双精度模式，分别运行 `LargeWorldTest`。不要复用不同精度的录制文件；布局不同会被拒绝。

快照实验：在第 N 步保存，继续 M 步，恢复后再执行相同 M 步，比较测试所采用的状态指标。保持输入、回调与配置一致，并观察 Body 句柄在原地恢复和新建 World 时的差异。

验收：能解释为什么位姿列表不足以重建完整模拟；能区分世界位置和局部向量的精度职责。

参考：[test_large_world.c](../../test/test_large_world.c)、[sample_replay.cpp](../../samples/sample_replay.cpp)、本地[录制手册](../recording.md)与[大世界手册](../large_worlds.md)。
