# 第二课学习笔记：从一次路径错误理解 CMake 与 Catkin

> 课程主题：C++ 编译、Makefile、CMake、Catkin，以及如何用 AI 辅助排错  
> 本次练习环境：Ubuntu，练习包 `DynamicX_Lesson2_CMake_Catkin_Practice.zip`  
> 记录原则：只把自己实际执行过的内容写成“已完成”；没有验证的步骤单独列为待完成。

## 1. 我这节课到底在学什么

刚开始看到 `g++`、`make`、`cmake` 和 `catkin` 时，我以为它们都是“把代码运行起来的命令”。实际操作后，我把它们分成了四层：

| 层次 | 工具或文件 | 作用 |
| --- | --- | --- |
| 源代码 | `.cpp`、`.hpp` | 写函数实现、声明和程序入口 |
| 编译器 | `g++` | 把 C++ 源文件编译、链接成可执行文件 |
| 构建规则 | `Makefile`、`CMakeLists.txt` | 说明有哪些目标、源文件和依赖 |
| ROS 构建组织 | catkin | 在 ROS1 中组织多个基于 CMake 的功能包及依赖 |

最关键的关系是：

```text
CMakeLists.txt --cmake--> Makefile --make--> g++ --编译和链接--> 可执行文件
ROS 功能包 ------catkin（内部仍使用 CMake）------^ 
```

因此，CMake 不是编译器，`make` 也不是 C++ 编译器。真正处理 C++ 代码的是 `g++`；CMake 和 Make 负责把越来越复杂的编译命令组织起来。

---

## 2. 从 C++ 编译过程理解报错位置

一个 C++ 程序生成可执行文件，大致经过四步：

1. **预处理**：展开头文件和宏；
2. **编译**：检查语法并生成汇编代码；
3. **汇编**：生成目标文件 `.o`；
4. **链接**：把目标文件和库组合成可执行文件。

以 `main.cpp` 和 `control_math.cpp` 为例，可以先分别编译：

```bash
g++ -Iinclude -c src/main.cpp -o main.o
g++ -Iinclude -c src/control_math.cpp -o control_math.o
```

再链接：

```bash
g++ main.o control_math.o -o control_demo
```

这里的 `-Iinclude` 是把 `include/` 加入头文件搜索路径，`-c` 表示只编译、不链接。

我现在会先根据关键词判断错误属于哪一层：

| 报错关键词 | 优先检查 |
| --- | --- |
| `No such file or directory` | 当前目录、文件名、头文件搜索路径 |
| `was not declared in this scope` | 声明、拼写、命名空间 |
| `multiple definition` | 同一个函数或变量是否被重复定义 |
| `undefined reference` | 实现文件或库是否参加链接 |
| `does not appear to contain CMakeLists.txt` | 交给 CMake 的源码目录是否正确 |
| `No targets specified and no makefile found` | 是否先成功执行过 CMake 配置 |

这比看到红色文字就重新安装软件更有效，因为不同阶段的问题要用不同办法解决。

---

## 3. Makefile 与 CMake 分别解决什么问题

### 3.1 Makefile：把重复命令写成依赖规则

一个简单 Makefile 可以写成：

```makefile
CXX := g++
CXXFLAGS := -Wall -Wextra -std=c++14 -Iinclude

control_demo: main.o control_math.o
	$(CXX) main.o control_math.o -o control_demo

main.o: src/main.cpp include/control_math.hpp
	$(CXX) $(CXXFLAGS) -c src/main.cpp -o main.o

control_math.o: src/control_math.cpp include/control_math.hpp
	$(CXX) $(CXXFLAGS) -c src/control_math.cpp -o control_math.o

.PHONY: clean
clean:
	rm -f control_demo main.o control_math.o
```

规则的基本形式是“目标依赖哪些文件，以及如何生成目标”。`make` 会比较文件修改时间，只重建需要更新的部分。Makefile 中命令前必须是 Tab，这是我需要特别注意的格式细节。

### 3.2 CMake：用更高层的方式描述项目

CMake 读取 `CMakeLists.txt`，再为当前平台生成 Makefile 或其他构建文件。一个对应的最小示例是：

```cmake
cmake_minimum_required(VERSION 3.10)
project(control_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(control_demo
    src/main.cpp
    src/control_math.cpp
)

target_include_directories(control_demo PRIVATE include)
target_compile_options(control_demo PRIVATE -Wall -Wextra)
```

我对几条命令的理解如下：

- `project()`：声明项目及使用的语言；
- `add_executable()`：定义要生成的可执行目标，并列出源文件；
- `target_include_directories()`：告诉某个目标去哪里找头文件；
- `target_link_libraries()`：声明目标需要链接的库；
- `target_compile_options()`：只给指定目标增加编译选项。

相比全局使用 `include_directories()`，`target_*` 写法能更清楚地说明配置属于哪个目标。

---

## 4. 我的真实操作与错误分析

这一部分记录的是我在 Ubuntu 终端中真实遇到的问题。它让我意识到：很多“编译错误”其实发生在编译之前，根因只是当前目录不对。

### 4.1 第一次错误：把中文目录当成英文目录

我先输入：

```bash
cd ~/Downloads
```

终端提示：

```text
bash: cd: /home/luoman/Downloads: 没有那个文件或目录
```

原因不是权限或 CMake，而是这台 Ubuntu 的下载目录实际叫 `~/下载`。Linux 路径必须与真实名称一致，不能凭习惯猜测。

我后来先回到正确目录并查看文件：

```bash
cd ~/下载
ls
```

这一步让我看到压缩包确实在 `~/下载` 中。

### 4.2 第二次错误：在错误目录执行 `unzip`

在家目录执行下面的命令时：

```bash
unzip DynamicX_Lesson2_CMake_Catkin_Practice.zip
```

终端提示找不到压缩包。原因仍然是相对路径：命令只会在当前目录寻找文件，而压缩包位于 `~/下载`。

在正确目录中，我执行了：

```bash
cd ~/下载
unzip DynamicX_Lesson2_CMake_Catkin_Practice.zip -d ~/DynamicX_Lesson2
```

这次终端显示已经解压出以下关键文件：

```text
~/DynamicX_Lesson2/dx_lesson2_practice/README.md
~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo/CMakeLists.txt
~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo/include/control_math.hpp
~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo/src/control_math.cpp
~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo/src/main.cpp
~/DynamicX_Lesson2/dx_lesson2_practice/ros_pkg/dx_turtle_control/CMakeLists.txt
~/DynamicX_Lesson2/dx_lesson2_practice/ros_pkg/dx_turtle_control/launch/demo.launch
~/DynamicX_Lesson2/dx_lesson2_practice/ros_pkg/dx_turtle_control/src/circle_node.cpp
```

我又使用 `find` 确认了 CMake 示例的准确位置：

```bash
find ~/DynamicX_Lesson2 -type d -name cmake_demo
```

输出为：

```text
/home/luoman/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo
```

### 4.3 第三次错误：在 `~/下载/build` 中运行 CMake

我当时仍停留在 `~/下载`，随后执行：

```bash
mkdir -p build
cd build
cmake ..
```

报错为：

```text
CMake Error: The source directory "/home/luoman/下载" does not appear to contain CMakeLists.txt.
```

紧接着执行 `make`，又得到：

```text
make: *** 没有指明目标并且找不到 makefile。停止。
```

这两个错误是同一条因果链：

1. `build` 建在了 `~/下载` 下面；
2. `cmake ..` 中的 `..` 因而指向 `/home/luoman/下载`；
3. 这个目录没有 `CMakeLists.txt`，所以 CMake 配置失败；
4. CMake 没有生成 Makefile，后面的 `make` 自然也无法工作。

这里的重点不是机械地再输一次 `make`，而是先用下面三条命令确认位置和文件：

```bash
pwd
ls
find .. -maxdepth 2 -name CMakeLists.txt
```

### 4.4 正确的构建位置与命令

下一步应该先进入**包含 `CMakeLists.txt` 的源码目录**：

```bash
cd ~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo
pwd
ls
```

确认能看到 `CMakeLists.txt`、`include/` 和 `src/` 后，再进行外部构建：

```bash
cmake -S . -B build
cmake --build build -j2
```

两条命令中：

- `-S .` 表示源码目录是当前目录；
- `-B build` 表示把生成文件放到当前目录的 `build/`；
- `cmake --build build` 调用已经生成的构建系统；
- `-j2` 表示最多并行执行两个构建任务。

这比先 `cd build` 再依赖 `..` 更直观，也更不容易因为所在目录变化而指错位置。

> 当前真实进度：已经完成压缩包定位、解压和源码路径确认；照片中最后一次构建仍因目录错误而失败。正确路径下的配置、编译和程序输出需要回到 Ubuntu 后继续执行并补充，不能把“计划执行”写成“已经成功”。

---

## 5. 源码目录与 build 目录为什么要分开

练习项目中，源码目录大致是：

```text
cmake_demo/
├── CMakeLists.txt
├── include/
│   └── control_math.hpp
└── src/
    ├── control_math.cpp
    └── main.cpp
```

构建后会增加：

```text
cmake_demo/build/
├── CMakeCache.txt
├── CMakeFiles/
├── Makefile
└── 可执行文件
```

源码是我需要维护的内容，`build/` 中则主要是 CMake 生成的缓存、中间文件和构建结果。把它们分开有三个好处：

1. 源码目录更干净；
2. 构建异常时可以明确重建 `build/`；
3. Git 提交时更容易排除生成文件。

因此项目的 `.gitignore` 通常应忽略 `build/`。如果需要清理，应先用 `pwd` 确认目标，避免在错误目录删除文件。

---

## 6. Catkin：把同样的构建思想放进 ROS1

Catkin 建立在 CMake 之上。普通 CMake 项目解决“怎样编译一个或多个 C++ 目标”，Catkin 进一步解决“ROS 工作空间中怎样组织多个功能包、声明 ROS 依赖并让系统找到它们”。

一个常见工作空间结构是：

```text
catkin_ws/
├── src/       # 自己维护的功能包源码
├── build/     # 构建中间文件
├── devel/     # 开发环境和生成结果
└── logs/      # catkin_tools 日志
```

ROS 功能包中的两个核心配置文件分工不同：

| 文件 | 作用 |
| --- | --- |
| `package.xml` | 声明包名、版本、维护者、许可证和依赖 |
| `CMakeLists.txt` | 说明怎样查找依赖、编译节点和链接库 |

例如代码使用 `roscpp` 和 `geometry_msgs` 时，这些依赖既要在 `package.xml` 中声明，也要在 `CMakeLists.txt` 的 `find_package(catkin REQUIRED COMPONENTS ...)` 中配置。只改一个文件，可能导致依赖信息不一致。

### 6.1 建立工作空间的命令逻辑

以下是我接下来要在安装好 ROS1 和 catkin_tools 的环境中验证的流程：

```bash
source /opt/ros/noetic/setup.bash
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws
catkin init
catkin build
source devel/setup.bash
```

每一步的目的分别是：加载 ROS 基础环境、创建源码目录、初始化工作空间、构建、再把本工作空间生成的环境叠加到当前终端。

练习包已经给出了 `dx_turtle_control`，复制到工作空间时应复制整个功能包目录，而不是只复制一个 `.cpp` 文件。构建前需要检查：

```bash
find ~/DynamicX_Lesson2/dx_lesson2_practice/ros_pkg/dx_turtle_control -maxdepth 2 -type f
```

确认包结构后再放入 `~/catkin_ws/src/`，然后构建指定包。新开终端时还要重新 `source`，否则即使编译成功，`rosrun` 或 `roslaunch` 也可能找不到包。

### 6.2 `source` 的含义

`source` 不是“安装 ROS”，而是在当前 shell 中加载环境变量。排查包找不到时，我会先查看：

```bash
echo "$ROS_DISTRO"
echo "$ROS_PACKAGE_PATH"
echo "$CMAKE_PREFIX_PATH"
```

另外，`catkin_make` 和 `catkin build` 是两种不同的构建方式。一个工作空间内最好固定使用一种，避免生成目录布局混在一起。

---

## 7. 我现在形成的排错顺序

以后遇到构建失败，我按下面的顺序处理：

1. **确认我在哪里**：`pwd`；
2. **确认文件是否真的存在**：`ls`、`find`；
3. **确认 CMake 的源码目录**：该目录必须包含 `CMakeLists.txt`；
4. **区分配置与构建**：先让 `cmake -S ... -B ...` 成功，再执行构建；
5. **阅读第一条有效错误**：最后一行往往只是“任务失败”；
6. **按阶段排查**：头文件问题看 include，链接问题看 target 和 library，ROS 包问题再看依赖及 `source`；
7. **修改后重新验证**：不能只因为报错消失就认为功能正确，还要运行目标并观察输出。

这次错误的根因可以总结为：

```text
没有先确认当前目录
        ↓
build 建在 ~/下载
        ↓
cmake .. 把 ~/下载 当成源码目录
        ↓
找不到 CMakeLists.txt，配置失败
        ↓
没有生成 Makefile，make 继续失败
```

---

## 8. 我如何使用 AI 辅助排错

这次我向 AI 提供了终端照片并询问这些命令在做什么。AI 帮我把报错串成了“目录错误 → CMake 配置失败 → Makefile 未生成”的因果链，也解释了源码目录与构建目录的区别。

但最终判断仍要以终端中的 `pwd`、`ls`、`find` 和真实输出为准。更有效的提问格式应该包含：

```text
系统与环境：Ubuntu / ROS 版本
目标：想构建哪个目录或功能包
当前目录：pwd 的输出
执行命令：完整命令
第一条错误：不要只截最后一行
相关结构：tree 或 find 的输出
希望 AI 做什么：先解释原因，再给最小修复和验证命令
```

我还需要警惕以下问题：

- AI 可能默认下载目录叫 `Downloads`，但我的机器实际是 `下载`；
- AI 可能把 ROS1 的 catkin 与 ROS2 的 colcon 混用；
- AI 给出的示例路径不一定与我的解压位置相同；
- “命令看起来合理”不等于已经在我的机器上验证；
- 删除目录、改权限或安装软件前，必须先确认目标和影响范围。

所以我更愿意把 AI 当成解释报错和整理思路的助教，而不是代替终端验证。

---

## 9. 早期进度记录（保留历史快照）

> 本节保留了最初练习包阶段的真实记录，**不是当前最新进度**。2026-10-29 完成的差速底盘工程、ROS 通信和代码检查见第 10 节。

### 已完成并有终端记录

- [x] 安装 C++ 基本构建工具；
- [x] 找到中文下载目录中的练习压缩包；
- [x] 将练习包解压到 `~/DynamicX_Lesson2`；
- [x] 用 `find` 定位 `cmake_demo` 的实际路径；
- [x] 分析错误构建目录导致的两条连续报错；
- [x] 理清 `g++`、Makefile、CMake 和 Catkin 的层次关系。

### 尚未完成，不能假装已经成功

- [ ] 在正确的 `cmake_demo` 目录执行配置和构建；
- [ ] 记录生成的可执行文件名称及真实运行输出；
- [ ] 将 `dx_turtle_control` 放入 catkin 工作空间并构建；
- [ ] 启动 `demo.launch`，观察节点、话题及小乌龟运动；
- [ ] 保存上述步骤的终端输出或截图，继续补充本笔记。

我下一次打开 Ubuntu 后会依次执行：

```bash
cd ~/DynamicX_Lesson2/dx_lesson2_practice/cmake_demo
pwd
ls
cmake -S . -B build
cmake --build build -j2
find build -maxdepth 2 -type f -executable
```

最后一条命令用于寻找构建生成的可执行文件。找到后再运行它，并把真实输出补进本节，而不是预先猜测程序名称或结果。

---

## 10. 2026-10-29 后续实操与鲁棒性改进

> **证据范围**：CMake、Catkin 和 ROS 的部分结果有本人 Ubuntu 终端截图；加强版代码与 14 项测试由 AI 在独立 Linux 环境运行验证。两种验证环境必须区分，不能将“AI 环境通过”误写为“本人电脑已经执行修复脚本”。以下为技术事实补充，个人心得仍需自行补写。

### 10.1 新实战工程：差速底盘运动学

这次使用的新工程路径是 `~/下载/DynamicX_第二课_差速底盘实战/native`，与第 4 节的早期 `cmake_demo` 不同。根据双轮差速底盘无侧滑的简化模型，已知左右轮角速度 `leftWheel`、`rightWheel`、轮半径 `radius` 和轮距 `wheelBase`，计算：

```text
linear  = radius * (leftWheel + rightWheel) / 2
angular = radius * (rightWheel - leftWheel) / wheelBase
```

示例采用 `leftWheel = 8 rad/s`、`rightWheel = 12 rad/s`、`radius = 0.05 m`、`wheelBase = 0.30 m`，得到：

```text
linear = 0.500000 m/s
angular = 0.666667 rad/s
```

本机截图记录了下列命令成功执行，CTest 结果为 1/1 通过：

```bash
cd ~/下载/DynamicX_第二课_差速底盘实战/native
cmake -S . -B cmake_build
cmake --build cmake_build
./cmake_build/dx_demo
ctest --test-dir cmake_build --output-on-failure
```

### 10.2 实际 CMakeLists.txt 的构建关系

新工程在 `CMakeLists.txt` 中定义如下构建目标：

```cmake
cmake_minimum_required(VERSION 3.10)
project(dx_diff_drive LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(dx_kinematics src/diff_drive.cpp)
target_include_directories(dx_kinematics PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_executable(dx_demo src/main.cpp)
target_link_libraries(dx_demo PRIVATE dx_kinematics)

enable_testing()
add_executable(dx_test tests/test.cpp)
target_link_libraries(dx_test PRIVATE dx_kinematics)
add_test(NAME diff_drive_math COMMAND dx_test)
```

| 指令 | 在本项目中的作用 |
| --- | --- |
| `cmake_minimum_required()` | 规定最低 CMake 版本为 3.10 |
| `project()` | 声明 C++ 工程 `dx_diff_drive` |
| `CMAKE_CXX_STANDARD` | 要求 C++17 标准 |
| `add_library()` | 将运动学计算封装为 `dx_kinematics` 库 |
| `target_include_directories(... PUBLIC ...)` | 暴露库头文件目录，使用库的目标也能继承该路径 |
| `add_executable()` | 分别创建演示程序 `dx_demo` 与测试程序 `dx_test` |
| `target_link_libraries(... PRIVATE ...)` | 将库链接给指定程序，依赖不向下游传播 |
| `enable_testing()` 与 `add_test()` | 将 `dx_test` 注册为 CTest 测试 |

我需要理解的是“**运动学函数作为库被多个可执行目标复用**”，而非把所有 `.cpp` 文件都堆进一个程序。

### 10.3 Catkin 双功能包与 ROS Topic 真实验证

在 ROS Noetic Docker 容器 `noetic-gui` 内，工作空间为 `~/dx_lesson2_ws`，包含两个功能包：

- `dx_kinematics`：运动学计算库；
- `dx_velocity_node`：依赖前者、发布 ROS 速度消息的节点。

第一次 `catkin_make` 编译时报：

```text
fatal error: diff_drive.h: No such file or directory
```

原因：ROS 包的头文件实际位于 `include/dx_kinematics/diff_drive.h`，而实现文件使用 `#include "diff_drive.h"`，与构建时暴露的头文件搜索路径不一致。随后在容器中将引用改成：

```cpp
#include "dx_kinematics/diff_drive.h"
```

重新执行 `catkin_make` 并加载 `devel/setup.bash` 后，`roslaunch dx_velocity_node demo.launch` 成功运行，终端连续显示：

```text
v=0.500 m/s, w=0.667 rad/s
```

另一个容器终端验证：

```bash
source /opt/ros/noetic/setup.bash
source ~/dx_lesson2_ws/devel/setup.bash
rosnode list
rostopic echo -n 1 /dx_demo/cmd_vel
```

实际截图显示 `/dx_velocity_node` 和 `/rosout`；`/dx_demo/cmd_vel` 的 `linear.x = 0.5`、`angular.z = 0.6666666666666667`。这证明程序不只在终端打印数字，还真正通过 ROS Topic 发布了 `geometry_msgs/Twist` 消息。

另外，成功执行 `catkin_create_pkg my_training_pkg roscpp std_msgs` 创建新包。**是否完成此新包的二次编译与 `rospack find` 验证，应以之后的终端结果补充，不能凭“已创建”推断“已构建”。**

### 10.4 代码检查：原计算函数缺少参数合法性校验

原函数在正常参数下计算正确，但没有处理以下非法输入：

| 输入 | 潜在问题 | 建议行为 |
| --- | --- | --- |
| `radius <= 0` | 不符合轮半径的物理约束 | 拒绝参数 |
| `wheelBase <= 0` | 轮距非法，零轮距还会引发浮点除零 | 拒绝参数 |
| 轮速、半径或轮距出现 `NaN`、`Inf` | 可能产生非法结果 | 拒绝参数 |
| 有限输入导致结果溢出 | 输出无穷大而未被察觉 | 报告溢出 |
| 轮速为负 | 可以表示倒车或反向旋转 | **允许**，不应误判为非法 |

改进后的 `native/src/diff_drive.cpp` 保留原运动学公式，增加 `std::isfinite` 检测、`radius > 0` 与 `wheelBase > 0` 检测；非法输入抛出 `std::invalid_argument`，计算结果非有限值时抛出 `std::overflow_error`。例如：

```cpp
if (!std::isfinite(leftWheel) || !std::isfinite(rightWheel) ||
    !std::isfinite(radius) || !std::isfinite(wheelBase) ||
    radius <= 0.0 || wheelBase <= 0.0) {
    throw std::invalid_argument("Invalid wheel parameters");
}
```

此处代码段是逻辑示意，完整版本还会对 `result.linear`、`result.angular` 再检查一次。真实机器人节点还需要合理捕获异常或在输入边界校验，不能让未处理异常直接导致控制节点退出。

### 10.5 CTest 由正常计算扩展为异常与边界测试

改进版 `native/tests/test.cpp` 不再只测正常数据，而是覆盖 **14 个场景**：

| 组别 | 场景 | 数量 |
| --- | --- | ---: |
| 正常运动 | 左转、直行、原地旋转、静止、倒车 | 5 |
| 非法参数及溢出 | 零/负轮半径、零/负轮距、无穷轮速、NaN 轮速、无穷半径、NaN 轮距、算术溢出 | 9 |

测试中还将普通 `assert()` 改为显式检查，使某些启用 `NDEBUG` 的构建方式不会将关键测试完全关闭。测试程序通过返回码 `0` 表示全部成功，非零表示失败。

**独立 Linux 环境实测**（不是本人 Ubuntu 修复后的回传截图）：

```text
linear = 0.500000 m/s
angular = 0.666667 rad/s
14 passed; 0 failed
1/1 Test #1: diff_drive_math ... Passed
100% tests passed, 0 tests failed out of 1
```

这里的“14”表示 `dx_test` 内部的检查场景，“1/1”表示 CTest 注册了一个 `diff_drive_math` 测试程序，二者并不矛盾。`CMakeLists.txt` 无需修改，因为原本已经将测试目标正确注册到 CTest。

**本机待复现**：之前提供的 `DynamicX_Lesson2_apply_fix.sh` 能备份并更新本机 `native/src/diff_drive.cpp`、`native/tests/test.cpp`，随后执行 CMake 和 CTest；目前没有本人在 Ubuntu 执行此脚本的终端截图，因此不能标记为已在本机完成。

### 10.6 AI 辅助开发：最小修复、执行证据与边界

今天的工作过程可概括为：

```text
读取真实报错 → 识别阶段与根因 → 修改最小必要代码
    → 重新构建 → 运行目标 → CTest 验证 → 记录证据
```

- AI 的作用是帮助定位 `#include` 路径问题、梳理参数边界、设计补充测试；但结果必须回到真实终端核实。
- 本机 Codex CLI 已打开登录界面，随后网页授权返回 `unsupported_country_region_territory`，因此**不能记作已成功登录 Codex 或用其直接修改本机文件**。
- 不应把 AI 在独立环境得出的“测试通过”直接当成本机证据，也不能把原本的 CMake `native` 改动误认为已自动同步到 ROS 容器副本。
- 遇到 `No such file or directory`，要先区分是 **目录错误、头文件路径错误，还是 Docker 容器名称不正确**；三者的修复方式不同。

### 10.7 本阶段进度与后续个人补充

- [x] 本机 CMake 配置、编译、运行和最初的 CTest 测试通过；
- [x] 本机 Catkin 双包构建、ROS 节点运行和 Topic 消息验证通过；
- [x] `my_training_pkg` 功能包已创建；
- [x] AI 在独立 Linux 环境验证参数检查改进及 14 项测试通过；
- [ ] 本机运行修复脚本并保存 14 项测试输出；
- [ ] 按真实终端输出确认 `my_training_pkg` 是否二次编译、识别成功；
- [ ] 自己补充修改前后差异与新学到的知识点；
- [ ] 结合课件指定阅读资料补全独立理解和反思。

> **留给自己补写**：为什么“构建成功”不等于“计算结果可靠”？为什么测过正常运动，还需要测试零轮距、NaN 和溢出？为什么在机器人控制软件中必须区分“程序抛异常”和“系统安全处理异常”？这些问题应在自己真正跑过修复脚本以后，用个人语言回答。

---

## 11. 参考资料

- [ROS Wiki：catkin Tutorials](https://wiki.ros.org/catkin/Tutorials)
- [知乎：Make、Makefile 与 CMake](https://zhuanlan.zhihu.com/p/476697014)
- [菜鸟教程：CMake 基础](https://www.runoob.com/cmake/cmake-basic.html)
- [Mihail Eric：The Emperor Has No Clothes](https://www.mihaileric.com/The-Emperor-Has-No-Clothes/)

> 说明：参考资料用于核对概念；第 4 节的命令、路径和报错来自我本次实际操作记录。
