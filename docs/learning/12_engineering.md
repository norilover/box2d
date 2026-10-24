# 12 · 工程目录、可视化样例、测试与性能实验

[返回学习路线](../learning_guide_zh.md) · 下一篇：[参考读物](13_reading.md)

目标：知道算法之外的代码怎样组成可运行、可调试、可测量的项目。

## 目录职责

| 目录/文件 | 职责与入口 | 学习优先级 |
| --- | --- | --- |
| [include/box2d](../../include/box2d) | 公共接口，应用只应依赖这一层 | 必修 |
| [src/CMakeLists.txt](../../src/CMakeLists.txt) | 核心源文件、编译定义、安装与链接配置 | 基础 |
| [samples/main.cpp](../../samples/main.cpp)、[sample.cpp](../../samples/sample.cpp)、[sample.h](../../samples/sample.h) | 窗口、场景选择、公共模拟循环和设置 | 基础 |
| `samples/sample_*.cpp` | 场景构造、交互、演示和问题复现 | 随模块阅读 |
| [draw.c](../../samples/draw.c)、[draw_text.cpp](../../samples/draw_text.cpp)、[shader.c](../../samples/shader.c)、[data](../../samples/data) | 调试显示、文字、着色器及场景素材 | 后读 |
| [shared](../../shared) | benchmarks、determinism、human、utils 等共享场景与工具 | 中级 |
| [benchmark/main.c](../../benchmark/main.c) | 无窗口基准入口，参数解析与耗时输出 | 中级 |
| [test/main.c](../../test/main.c) | 注册并筛选测试组 | 必修 |
| [extern](../../extern) | 本地第三方辅助代码；其他部分依赖由 CMake 拉取 | 按需 |
| [scripts/api_dump.py](../../scripts/api_dump.py) | API 信息提取，与 API 工作流配合 | 后读 |
| [.github/workflows](../../.github/workflows) | 跨配置构建、测试及 API 检查流程 | 中级 |
| [docs/CMakeLists.txt](../CMakeLists.txt) | Doxygen 文档构建 | 按需 |
| [build.zig](../../build.zig)、[Package.swift](../../Package.swift) | Zig、Swift 构建/接入方式 | 可选 |

`shared` 不负责 OpenGL 渲染；渲染主要在 `samples`。核心库可以独立使用，不需要把示例窗口系统带进自己的应用。

## 样例怎样读

先在界面找到一个场景，再搜索对应文件里的场景注册与构造函数。把创建 Body/Shape/Joint 的代码与每帧交互分开看，最后再回到公共 `Sample` 基类确认 step 在哪里执行。

| 想观察的内容 | 推荐文件 |
| --- | --- |
| 刚体与堆叠 | [sample_bodies.cpp](../../samples/sample_bodies.cpp)、[sample_stacking.cpp](../../samples/sample_stacking.cpp) |
| 几何、形状、碰撞 | [sample_geometry.cpp](../../samples/sample_geometry.cpp)、[sample_shapes.cpp](../../samples/sample_shapes.cpp)、[sample_collision.cpp](../../samples/sample_collision.cpp) |
| 高速运动和反弹 | [sample_continuous.cpp](../../samples/sample_continuous.cpp)、[sample_restitution.cpp](../../samples/sample_restitution.cpp) |
| 业务与角色 | [sample_events.cpp](../../samples/sample_events.cpp)、[sample_character.cpp](../../samples/sample_character.cpp) |
| 世界配置、复现与稳健性 | [sample_world.cpp](../../samples/sample_world.cpp)、[sample_issues.cpp](../../samples/sample_issues.cpp)、[sample_robustness.cpp](../../samples/sample_robustness.cpp) |
| 性能、确定性与回放 | [sample_benchmark.cpp](../../samples/sample_benchmark.cpp)、[sample_determinism.cpp](../../samples/sample_determinism.cpp)、[sample_replay.cpp](../../samples/sample_replay.cpp) |

## 测试是可执行的阅读材料

按[入门篇](01_start.md)构建 `test` 后，将下表组名作为第一个参数。例：Windows 的 `.\build-learn-win\bin\Debug\test.exe DistanceTest`，Linux 的 `./build-learn-linux/bin/test DistanceTest`。

| 组名（精确匹配） | 对应主题 |
| --- | --- |
| `WorldTest`、`ShapeTest`、`ChainTest` | 对象与场景生命周期、形状与地形 |
| `CollisionTest`、`DistanceTest`、`DynamicTreeTest` | 几何、距离与宽相数据结构 |
| `RestitutionTest`、`MoverTest` | 反弹、角色几何辅助 |
| `ThreadTest`、`DeterminismTest` | 调度与结果一致性 |
| `SnapshotTest`、`RecordingTest`、`LargeWorldTest` | 快照、录制、大坐标 |
| `MathTest`、`ContainerTest`、`IdTest`、`BitSetTest`、`TableTest` | 内部基础设施 |
| `InvalidInputTest` | 输入检查与非法参数行为 |

阅读一个测试时先找场景与断言，再回源码解释为什么应满足断言。测试通过证明的是它覆盖的条件，不能把单一场景推广成所有情况下的保证。完整测试组还包括回放浏览等细项，见 `test/main.c`。

## 性能实验怎样做

单独配置 Release 构建，启用 `BOX2D_BENCHMARKS=ON`，关闭不需要的 samples 和测试。例如 Linux：

```bash
cmake -S . -B build-learn-bench -DCMAKE_BUILD_TYPE=Release -DBOX2D_SAMPLES=OFF -DBOX2D_UNIT_TESTS=OFF -DBOX2D_BENCHMARKS=ON
cmake --build build-learn-bench --target benchmark --parallel
./build-learn-bench/bin/benchmark --help
./build-learn-bench/bin/benchmark --list
```

Windows 用独立的构建目录，并在 build 命令加 `--config Release`；Visual Studio 生成器对应 `bin\Release\benchmark.exe`。按 help 确认线程、场景、重复次数等参数后再开始长时间运行。

先固定场景、初态、模拟时长、构建模式、硬件与线程数。每次只改变一个变量，记录 `b2World_GetProfile` 与 `b2World_GetCounters`，区分碰撞、求解、休眠与显示的成本。记录预热、重复次数和波动，不把一次最快结果当作普遍性能。仓库中已有的 CSV 来自特定硬件，只能作背景材料。

需要阶段跟踪时再启用 `BOX2D_PROFILE` / Tracy；需要调试完整性时看 `BOX2D_VALIDATE`；需要内存诊断时看 `BOX2D_SANITIZE`。诊断构建有额外成本，比较时保持配置一致。

## 生成文档与验收

当前中文系列已加入 Doxygen 输入列表。安装 Doxygen 与 Graphviz 的 `dot` 后，可以配置独立目录：

```bash
cmake -S . -B build-learn-docs -DBOX2D_SAMPLES=OFF -DBOX2D_UNIT_TESTS=OFF -DBOX2D_DOCS=ON
cmake --build build-learn-docs --target doc
```

无需生成 HTML 也能直接在 Markdown 阅读器中沿链接学习。验收：能选择一个对应模块的测试组，能定位一个样例的场景创建代码，能记录一次可复查的性能实验条件。
