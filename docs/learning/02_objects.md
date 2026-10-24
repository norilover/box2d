# 02 · World、Body、Shape 与数据布局

[返回学习路线](../learning_guide_zh.md) · 下一篇：[一帧模拟](03_step.md)

目标：从“会调用 API”走到“知道数据存在哪里”。先读 [simulation.md](../simulation.md) 的对象介绍，再对照 [box2d.h](../../include/box2d/box2d.h)、[types.h](../../include/box2d/types.h) 和 [id.h](../../include/box2d/id.h)。

## 对象分别负责什么

| 对象 | 职责 | 生命周期与入口 |
| --- | --- | --- |
| World | 持有全部物理对象、事件、空间索引和求解状态 | `b2CreateWorld` / `b2DestroyWorld`，见 [physics_world.c](../../src/physics_world.c) |
| Body | 类型、运动、质量、惯量及所附对象的关联 | `b2CreateBody` / `b2DestroyBody`，见 [body.c](../../src/body.c) |
| Shape | 圆、胶囊、多边形等几何，以及密度、材料、过滤属性 | `b2CreatePolygonShape` 等，见 [shape.c](../../src/shape.c) |
| Contact | 引擎自动维护的形状对、流形和冲量缓存 | [contact.c](../../src/contact.c)，用户一般不直接创建 |
| Joint | 用户指定的刚体约束或连接关系 | [joint.c](../../src/joint.c) |
| Chain | 多段地形及邻接信息 | 创建与销毁同样在 `shape.c`，不是独立的 `chain.c` |

一个 Body 可以有多个 Shape，例如车身由几个凸多边形组成；它们共享刚体运动。Shape 的位置通常定义在 Body 的局部坐标系中。几何原点、刚体原点、质心不一定重合。

| Body 类型 | 使用方式 |
| --- | --- |
| static | 地面等静止对象，不由力驱动 |
| kinematic | 用户指定速度的运动对象，不受力推动 |
| dynamic | 有有限质量，由重力、力、冲量和约束共同驱动 |

不要据此推断所有类型组合都会形成实体接触；还要看 `b2ShouldBodiesCollide`、Shape 过滤和 Sensor 路径。

## 为什么用 ID

应用拿到 `b2BodyId`，不是内部 `b2Body*`。ID 中的索引、world 信息和 generation 用于定位并检查对象。销毁之后再复用内部槽位，不应让旧句柄误指向新对象；generation 有限，也不应把句柄当作永久业务主键。

`B2_IS_NON_NULL` 只检查是否为空，不能替代 `b2Body_IsValid`。销毁 World 会连带销毁其中对象，但不会自动清空应用自己保存的句柄变量。

## 一个 Body 为什么分成三部分

阅读 [body.h](../../src/body.h) 和 [solver_set.h](../../src/solver_set.h)：

```text
应用 b2BodyId
    → world->bodies 中的 b2Body（身份、关联、setIndex、localIndex）
    → world->solverSets[setIndex].bodySims[localIndex]（b2BodySim）
    → 若处于 awake set，再访问 bodyStates[localIndex]（b2BodyState）
```

`b2BodySim` 保存变换、质心、逆质量、力等模拟数据；`b2BodyState` 保存求解热点数据，例如速度、位置增量和旋转增量。睡眠或静态对象不需要占用与活动对象相同的求解状态。

这种组织减少求解循环中无关字段的访问，并让活动数据连续存放。代价是移动或删除元素后必须更新索引，唤醒也可能使内部指针失效。因此调试时可以观察指针，应用代码应始终使用公开 ID。

## 阅读顺序与练习

1. 在 [types.c](../../src/types.c) 读 `b2DefaultWorldDef`、`b2DefaultBodyDef`、`b2DefaultShapeDef`，理解为什么只用 `{0}` 不够。
2. 跟踪 `b2CreateBody`：分配 ID、选择 solver set、初始化 Body 与模拟数据。
3. 跟踪 `b2CreatePolygonShape`：几何关联到 Body，质量更新，创建宽相代理。
4. 对照 [test_world.c](../../test/test_world.c) 的 `TestIsValid` 观察创建、销毁前后的句柄有效性。

实验：在同一 Body 上放两个偏离原点的 Shape，查看质心与转动效果。验收时解释“为什么修改 Shape 密度会影响整个 Body 的运动”，以及“为什么连续数组中的地址不适合作为公开身份”。

延伸阅读：[基础手册](../foundation.md)、[岛与并行](09_parallel.md)。
