# 01 · 从构建到第一个落地盒子

[返回学习路线](../learning_guide_zh.md) · 下一篇：[对象与数据](02_objects.md)

目标：运行一个不依赖图形界面的物理世界，再在调试器里看到盒子落地。本文对应本仓库代码，构建选项以根目录 [CMakeLists.txt](../../CMakeLists.txt) 为依据。

## 先补哪些基础

你需要能读懂 C 的结构体、指针、数组、枚举、函数指针和头文件，知道编译、链接与断点的区别。数学先掌握二维向量、点积、叉积、旋转和单位；暂时不用从完整的微分方程课程开始。相关符号见[数学与内存基础](10_foundation.md)。

本项目核心是 C17，samples 是 C++20；不是 Java 项目。CMake 至少需要 3.22，并需要可用的 C/C++ 编译器。Windows 可安装带 C++ 桌面开发工具的 Visual Studio，Linux 可使用 GCC 或 Clang。

## 第一步：只构建核心与测试

下面使用独立的构建目录，避免与已有 `build` 或另一操作系统的缓存混用。在 Windows PowerShell 中执行：

```powershell
Set-Location 'D:\Own\Learn\box2d'
cmake -S . -B build-learn-win -DBOX2D_SAMPLES=OFF -DBOX2D_UNIT_TESTS=ON -DBUILD_SHARED_LIBS=OFF -DBOX2D_DOCS=OFF
cmake --build build-learn-win --config Debug --target test
.\build-learn-win\bin\Debug\test.exe WorldTest
```

上面的可执行文件位置对应 Visual Studio 多配置生成器。如果使用 Ninja 单配置生成器，配置时增加 `-DCMAKE_BUILD_TYPE=Debug`，测试通常位于 `bin\test.exe`。

在 Linux/WSL 中，核心是 C 项目，可使用以下独立目录：

```bash
cmake -S . -B build-learn-linux -DCMAKE_BUILD_TYPE=Debug -DBOX2D_SAMPLES=OFF -DBOX2D_UNIT_TESTS=ON -DBUILD_SHARED_LIBS=OFF -DBOX2D_DOCS=OFF
cmake --build build-learn-linux --target test --parallel
./build-learn-linux/bin/test WorldTest
```

测试入口是 [test/main.c](../../test/main.c)。参数按**测试组全名**精确匹配：`WorldTest` 可以，`HelloWorld` 不是顶层组名。拼错会跳过测试，仍可能打印成功；必须看到 `test passed: WorldTest`。不带参数运行全部测试。本仓库没有用 `add_test` 注册这些测试，不要将 `ctest` 的“没有测试”当作验证通过。

## 第二步：读懂最小程序

下面是可单独编译的 `hello.c`，结构来自 [HelloWorld 测试](../../test/test_world.c)。

```c
#include <box2d/box2d.h>
#include <stdio.h>

int main(void)
{
    b2WorldDef worldDef = b2DefaultWorldDef();
    worldDef.gravity = (b2Vec2){0.0f, -10.0f};
    b2WorldId world = b2CreateWorld(&worldDef);

    b2BodyDef groundDef = b2DefaultBodyDef();
    groundDef.position = (b2Pos){0.0f, -10.0f};
    b2BodyId ground = b2CreateBody(world, &groundDef);
    b2Polygon groundBox = b2MakeBox(50.0f, 10.0f);
    b2ShapeDef groundShape = b2DefaultShapeDef();
    b2CreatePolygonShape(ground, &groundShape, &groundBox);

    b2BodyDef bodyDef = b2DefaultBodyDef();
    bodyDef.type = b2_dynamicBody;
    bodyDef.position = (b2Pos){0.0f, 4.0f};
    b2BodyId body = b2CreateBody(world, &bodyDef);
    b2Polygon box = b2MakeBox(1.0f, 1.0f);
    b2ShapeDef shapeDef = b2DefaultShapeDef();
    shapeDef.density = 1.0f;
    shapeDef.material.friction = 0.3f;
    b2CreatePolygonShape(body, &shapeDef, &box);

    for (int i = 0; i < 120; ++i)
    {
        b2World_Step(world, 1.0f / 60.0f, 4);
        b2Pos p = b2Body_GetPosition(body);
        if (i % 30 == 29)
        {
            printf("step=%d x=%.3f y=%.3f\n", i + 1, (double)p.x, (double)p.y);
        }
    }
    b2DestroyWorld(world);
    return 0;
}
```

`b2MakeBox` 接受半宽、半高，所以盒子边长为 2。地面顶面在 y=0，盒子静止时中心应接近 y=1，允许求解容差。`density` 决定质量，**动态类型由 `bodyDef.type` 指定**，仅把密度设为正数不会让静态刚体变动态。

要运行这段程序，可在仓库旁新建 `box2d-hello` 目录，将它保存为 `hello.c`，同时写入以下 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.22)
project(box2d_hello LANGUAGES C CXX)
add_subdirectory(../box2d box2d-build)
add_executable(hello hello.c)
set_target_properties(hello PROPERTIES C_STANDARD 17 C_STANDARD_REQUIRED YES)
target_link_libraries(hello PRIVATE box2d)
```

在新目录执行 `cmake -S . -B build`、`cmake --build build --config Debug`。Visual Studio 下运行 `build\Debug\hello.exe`，Linux 默认单配置生成器下运行 `./build/hello`。Box2D 作为子目录时不会默认构建 samples 和测试。这个示例按 C 编译，不要随意改成 `.cpp`。

## 第三步：打开可视化样例

从仓库根目录按 [README](../../README.md) 的 presets 构建。Windows：`cmake --preset windows`，然后 `cmake --build --preset windows-debug`，运行 `build\bin\Debug\samples.exe`。Linux：`cmake --preset linux-debug`，然后 `cmake --build --preset linux-debug`，运行 `./build/bin/samples`。

samples 配置会从网络获取 GLFW、ImGui、ImPlot 和文件选择器等依赖；Linux 还需要对应的窗口系统开发库。无显示环境先完成无窗口测试即可。初学时只看 Bodies、Stacking、Events、Joints 对应的场景，暂时跳过渲染实现。

## 练习与验收

1. 将初始高度由 4 改为 8，记录落地时间的变化。
2. 保持初始高度不变，将密度翻倍，预测自由落体阶段是否更快。
3. 在 `HelloWorld`、`b2World_Step`、`b2CreateBody` 设置断点，找到用户代码进入引擎的位置。

完成标志：能解释盒子最终为什么停在 y≈1，能独立运行 `WorldTest`，能区分物理单位与屏幕像素。继续读本地 [Hello Box2D](../hello.md) 时，若示例字段与头文件不同，以当前头文件和已核验的程序为准。
