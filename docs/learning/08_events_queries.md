# 08 · 事件、传感器、空间查询与角色移动

[返回学习路线](../learning_guide_zh.md) · 下一篇：[岛与并行](09_parallel.md)

目标：把物理结果接入业务，并区分“物理求解”“查询世界”和“控制角色”。

## 事件是 step 的输出

| 接口 | 常见用途 | 入口 |
| --- | --- | --- |
| `b2World_GetBodyEvents` | 同步发生运动的对象，处理睡眠通知 | [physics_world.c](../../src/physics_world.c) |
| `b2World_GetContactEvents` | 接触开始、结束以及达到条件的撞击 | 同上及 [contact.c](../../src/contact.c) |
| `b2World_GetSensorEvents` | 触发区进入/离开 | [sensor.c](../../src/sensor.c) |
| `b2World_GetJointEvents` | 读取关节事件 | 定义与条件见 [types.h](../../include/box2d/types.h) |

先完成 `b2World_Step`，再读取事件。事件数组由 World 持有，应及时消费，不要保存内部数组指针跨 step 使用。销毁 Body/Shape 等修改也可能使数据失效；若要在处理期间销毁对象，可先提取业务 ID 和动作，随后统一执行，并重新检查必要句柄。

结束事件中对象可能已经被销毁，读取相关属性前使用 `b2Shape_IsValid` 等检查。Contact ID 也是临时的，不适合作为长期业务标识。

先查看 `b2ShapeDef` 的 `enableContactEvents`、`enableHitEvents`、`enableSensorEvents` 以及 `isSensor`，不要假设所有通知默认开启。开始接触和高速撞击是两类事件；静静放在地面上不应不断触发“首次撞击”逻辑。

## Sensor 不参与实体碰撞求解

传感器同样拥有 Shape 几何与空间代理，但不建立普通实体 Contact，也不会产生支撑刚体的接触冲量。其重叠由专门路径维护，`b2OverlapSensors` 在 step 后部运行。

阅读 [sensor.h](../../src/sensor.h) 和 `sensor.c`，观察重叠结果如何比较、排序并生成 begin/end。当前实现还包含连续传感器相关处理，研究高速穿越时应对照本版本配置与测试，不要照搬旧文档的限制结论。

## 查询按需求选择

| 需求 | 工具 | 注意点 |
| --- | --- | --- |
| 区域选中候选物 | `b2World_OverlapAABB` | 包围盒粗筛，不等于精确形状重叠 |
| 精确形状重叠 | `b2World_OverlapShape` | 查看 proxy 和坐标系约定 |
| 激光/视线检测 | `b2World_CastRay` / `b2World_CastRayClosest` | translation 是扫掠位移；前者有回调控制 |
| 有体积物体的路径检测 | `b2World_CastShape` | 不是把物体简化成一条射线 |

签名见 [box2d.h](../../include/box2d/box2d.h)。本仓库空间查询有 `b2Pos origin` 参数，几何输入相对这个原点。局部几何查询与 World 查询的过滤、坐标系和回调含义不同。一般查询不会自动推进物理模拟。

## 两条角色移动路线

几何角色可视为由应用控制的胶囊：使用 `b2World_CastMover` 预测移动，使用 `b2World_CollideMover` 取得碰撞平面，再用 `b2SolvePlanes` 和 `b2ClipVector` 修正位置与速度。它本身不一定是 World 中的刚体。实现入口是 [mover.c](../../src/mover.c)，示例在 [geometric_mover.cpp](../../samples/geometric_mover.cpp)。

动态角色使用真实 Body，并通过速度控制、关节等与世界交互。对照 [dynamic_mover.cpp](../../samples/dynamic_mover.cpp) 与 [sample_character.cpp](../../samples/sample_character.cpp)，观察推动箱子、斜坡和移动平台上的表现。

几何控制便于定制手感，但不能自动获得与动态刚体完全相同的受力行为；应用要明确推动、平台跟随等规则。

## 实验与验收

在 [sample_events.cpp](../../samples/sample_events.cpp) 中做“落地计数 + Sensor 触发区”，先把事件输出到日志，再接业务。对照 [test_mover.c](../../test/test_mover.c) 理解平面约束。

验收：能解释踩中地面、穿过触发区、向下射线命中地面这三件事的数据来源为何不同。

参考：[角色手册](../character.md)、[模拟手册](../simulation.md)、[大世界与快照](11_advanced.md)。
