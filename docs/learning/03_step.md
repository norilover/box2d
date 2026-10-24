# 03 · 沿着 b2World_Step 走完一帧

[返回学习路线](../learning_guide_zh.md) · 下一篇：[几何与距离](04_geometry.md)

目标：把文件串成数据流。入口是 [physics_world.c](../../src/physics_world.c) 的 `b2World_Step`，下面描述正常的正时间步路径；零时间步会跳过 `b2Solve`。

```text
用户创建/修改对象、施加力
          ↓
b2World_Step(world, dt, subStepCount)
  清理本步事件、锁定 world、初始化任务状态
          ↓
  b2UpdateBroadPhasePairs → 候选形状对、创建 contact
          ↓
  b2Collide → 更新流形、接触状态、约束组织
          ↓
  b2Solve → 约束准备、子步、连续碰撞、最终状态等
          ↓
  完成尚未结束的树任务
          ↓
  b2OverlapSensors → 传感器重叠变化
          ↓
  事件缓冲切换、统计、录制校验（若启用）、解锁
          ↓
用户读取事件、同步显示和业务状态
```

这是主线程的逻辑顺序；某些内部任务会并发执行。CCD 属于求解流程的一部分，不能认为所有碰撞处理都发生在 `b2Collide` 内。

## 每一站看什么

| 站点 | 输入 | 输出 / 建议观察内容 |
| --- | --- | --- |
| `b2UpdateBroadPhasePairs` | 空间树和移动标记、过滤配置 | 候选对、contact 数量；AABB 重叠不保证几何接触 |
| `b2Collide` | 已存在的 contact 和当前形状变换 | 接触点、法线、状态变化；缓存可能被复用 |
| `b2Solve` | 活动刚体、接触与关节约束、`b2StepContext` | 更新后的运动状态、冲量、休眠与移动事件 |
| `b2OverlapSensors` | 传感器与其他 Shape 的几何关系 | begin/end 重叠事件，不产生实体接触冲量 |
| step 返回之后 | 完成本步的 world | `b2World_GetBodyEvents` 等公开接口可读的结果 |

`b2StepContext` 定义在 [solver.h](../../src/solver.h)，是一次 step 的临时上下文，不是另一个永久 World。重点关注 `dt`、`h`、`subStepCount`、`states` 和 `sims`。

## 时间步、子步、绘制帧不要混淆

`dt=1/60` 表示模拟时间前进 1/60 秒；`subStepCount=4` 表示求解子步 `h=dt/4`。它不是旧版 API 的“速度迭代次数”和“位置迭代次数”。增加子步也不等于在外部调用四次完整 `b2World_Step`，宽相更新与事件边界等行为不同。

交互程序可累计真实经过的时间，每攒够一个固定 `dt` 就调用一次 step，再独立绘制。发生卡顿时应限制单次渲染帧的追赶工作并明确超额时间策略，避免越追赶越卡；不建议直接把很大的渲染帧间隔传给物理模拟。渲染插值是显示层的事。

## 用一个落地过程做断点记录

先设置 `workerCount=0` 使用串行路径，避免首次调试就进入工作线程同步。只观察一个动态盒子和一块地面：

1. 远离地面时，记录位置、速度和接触数。
2. 接近地面时，在 `b2CreateContact` 与 `b2UpdateContact` 观察形状对、流形点和 separation。
3. 停稳时，对比法向冲量、速度和休眠状态。
4. 在 step 返回后读取事件，解释为什么“存在 contact”和“本步产生 hit 事件”不是同一个条件。

验收产物：画一张包含上述四站的调用图，每条箭头写出传递的数据，而不只是函数名。

参考：[模拟手册](../simulation.md)、[接触生命周期](05_collision.md)、[求解器](06_solver.md)。
