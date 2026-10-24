# 07 · 关节：把运动规则变成约束

[返回学习路线](../learning_guide_zh.md) · 下一篇：[事件、查询与角色](08_events_queries.md)

目标：能为一个机械结构选择关节，并沿代码理解准备、热启动与求解。先读 [types.h](../../include/box2d/types.h) 中的 `b2JointDef` 与各派生定义，再读 [joint.c](../../src/joint.c) 的创建和分派。

## 当前仓库有哪些关节

| 类型 | 直觉与用途 | 实现 |
| --- | --- | --- |
| Distance | 约束两锚点距离，可用于拉杆与弹簧 | [distance_joint.c](../../src/distance_joint.c) |
| Revolute | 两物体围绕铰点相对旋转，可配角度限制/电机 | [revolute_joint.c](../../src/revolute_joint.c) |
| Prismatic | 沿指定轴平移的滑轨 | [prismatic_joint.c](../../src/prismatic_joint.c) |
| Weld | 将相对平移和角度联系起来，可带柔性 | [weld_joint.c](../../src/weld_joint.c) |
| Wheel | 悬架轴运动与车轮旋转 | [wheel_joint.c](../../src/wheel_joint.c) |
| Motor | 驱动相对运动，具体轴与目标见定义 | [motor_joint.c](../../src/motor_joint.c) |
| Mover | 面向移动控制的速度目标约束 | [mover_joint.c](../../src/mover_joint.c) |
| Pogo | 沿接触法线支撑角色的弹性约束，可限制拉力与压力 | [pogo_joint.c](../../src/pogo_joint.c) |
| Filter | 控制指定刚体之间的碰撞关系，不承担普通运动约束求解 | [joint.c](../../src/joint.c) |

不要只按 README 或旧版教程中的关节列表找文件：当前定义含 `base` 公共字段，关节帧和参数也会随版本变化。搜索 `b2Default...JointDef` 和 `b2Create...Joint` 能找到本版本用法。

## 从距离关节入手

设两个锚点间距离为 d，目标长度为 L，则误差可写成 `C = d - L`。求解器需要的是减小这个误差及其变化速度的方法，而不是每帧强行把两个 Body 摆到指定坐标。

按下面顺序阅读 Distance 实现：

1. 创建：公共 Body 关联、局部帧与具体参数如何进入 `b2JointSim`。
2. Prepare：把锚点与质量信息转换为当前求解所需的数据。
3. WarmStart：把缓存冲量作用到刚体速度。
4. Solve：计算相对速度、限制和弹簧项，更新冲量。

理解这一条主线后，再看 Revolute：标量距离约束变成多个受限自由度，但质量、锚点、速度和累计冲量的思路可以复用。

## 参数怎么理解

Limit 限制允许的范围；Motor 提供受最大力或力矩限制的驱动；Spring 用频率和阻尼调节响应。电机有输出上限时，目标速度不一定立即达到。角速度通常以弧度每秒计，UI 中的角度显示不代表内部单位也是度。

锚点和局部关节帧属于各 Body 的坐标系。创建时在世界空间看上去重合，不意味着直接复制同一个向量到两个局部帧也正确。核对当前定义，参考样例中的世界/局部变换过程。

## 实验与验收

在 [sample_joints.cpp](../../samples/sample_joints.cpp) 找到对应关节构造；车辆可参考 [car.cpp](../../samples/car.cpp)。先做单摆，再做弹簧连接，最后组合滑轨与电机。

每次只改一个参数，记录振荡周期、衰减或最大驱动力的变化。验收：能解释约束为什么改变速度，能找出哪个自由度被保留，能说明 Motor 与直接设置位置的区别。

参考：[模拟手册](../simulation.md) 的 Joints，以及[参考资料](13_reading.md)中的 Understanding Constraints、Soft Constraints。
