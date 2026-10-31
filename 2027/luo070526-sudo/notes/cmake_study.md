---
tags:
  - type/lesson
  - topic/cmake
  - topic/catkin
  - status/done
date: 2026-10-29
---

# 第二课：CMake、Catkin 与构建验证

本课学习的是“源码怎样可靠地变成可执行程序或 ROS 节点”。

```text
C++ 源码
  → g++ 编译与链接
  → Makefile 管理依赖
  → CMake 生成构建系统
  → Catkin 组织 ROS 功能包
```

## 1. g++、Make、CMake 与 Catkin

| 工具 | 作用 | 不是 |
| --- | --- | --- |
| `g++` | 编译、汇编并链接 C++ | 项目管理工具 |
| `make` | 按 Makefile 的规则构建目标 | C++ 编译器 |
| CMake | 从 `CMakeLists.txt` 生成构建系统 | 编译器 |
| Catkin | 在 ROS1 中组织基于 CMake 的功能包 | ROS 节点本身 |

一个程序大致经历：

1. 预处理：展开头文件和宏；
2. 编译：检查语法并生成汇编；
3. 汇编：生成目标文件 `.o`；
4. 链接：把目标文件与库组合成可执行文件。

手动编译示例：

```bash
g++ -Iinclude -c src/main.cpp -o main.o
g++ -Iinclude -c src/diff_drive.cpp -o diff_drive.o
g++ main.o diff_drive.o -o dx_demo
```

`-Iinclude` 添加头文件搜索路径，`-c` 表示只编译、不执行链接。

## 2. Makefile

Makefile 把重复的编译命令写成“目标—依赖—生成方法”：

```makefile
CXX := g++
CXXFLAGS := -Wall -Wextra -std=c++17 -Iinclude

dx_demo: main.o diff_drive.o
	$(CXX) main.o diff_drive.o -o dx_demo

main.o: src/main.cpp include/diff_drive.h
	$(CXX) $(CXXFLAGS) -c src/main.cpp -o main.o

diff_drive.o: src/diff_drive.cpp include/diff_drive.h
	$(CXX) $(CXXFLAGS) -c src/diff_drive.cpp -o diff_drive.o

.PHONY: clean
clean:
	rm -f dx_demo main.o diff_drive.o
```

`make` 会根据文件修改时间只重建受影响的目标。规则中的命令必须用 Tab 缩进。

## 3. CMake

CMake 用更高层的方式描述目标、源码、头文件和库，再为当前平台生成 Makefile 等构建文件。

### 3.1 差速底盘工程

本次工程位于：

```text
~/下载/DynamicX_第二课_差速底盘实战/native/
├── CMakeLists.txt
├── include/
├── src/
│   ├── main.cpp
│   └── diff_drive.cpp
└── tests/
    └── test.cpp
```

差速底盘运动学：

```text
linear  = radius * (leftWheel + rightWheel) / 2
angular = radius * (rightWheel - leftWheel) / wheelBase
```

当左右轮角速度分别为 `8 rad/s`、`12 rad/s`，轮半径为 `0.05 m`，轮距为 `0.30 m` 时：

```text
linear  = 0.500000 m/s
angular = 0.666667 rad/s
```

### 3.2 典型 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(dx_diff_drive LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(dx_kinematics src/diff_drive.cpp)
target_include_directories(
  dx_kinematics
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
)

add_executable(dx_demo src/main.cpp)
target_link_libraries(dx_demo PRIVATE dx_kinematics)

enable_testing()
add_executable(dx_test tests/test.cpp)
target_link_libraries(dx_test PRIVATE dx_kinematics)
add_test(NAME diff_drive_math COMMAND dx_test)
```

关键关系：

- `add_library`：把运动学函数封装成可复用的库；
- `target_include_directories`：为目标添加头文件目录；
- `add_executable`：生成演示程序和测试程序；
- `target_link_libraries`：把运动学库链接到两个程序；
- `add_test`：把 `dx_test` 注册给 CTest。

这比把所有 `.cpp` 堆进一个可执行程序更容易复用和测试。

### 3.3 配置、构建、运行与测试

```bash
cd ~/下载/DynamicX_第二课_差速底盘实战/native
cmake -S . -B cmake_build
cmake --build cmake_build
./cmake_build/dx_demo
ctest --test-dir cmake_build --output-on-failure
```

- `-S .`：源码目录是当前目录；
- `-B cmake_build`：生成文件放入单独的构建目录；
- `cmake --build`：调用已经生成的构建系统；
- `ctest`：运行 CMake 注册的测试。

本机记录的结果是演示程序得到 `0.500000 m/s` 和 `0.666667 rad/s`，CTest 为 `1/1` 通过。

### 3.4 我遇到的路径错误

我曾在 `~/下载/build` 中执行：

```bash
cmake ..
make
```

报错：

```text
The source directory "/home/luoman/下载" does not appear to contain CMakeLists.txt.
No targets specified and no makefile found.
```

原因链：

```text
build 建错位置
  → cmake .. 指向 ~/下载
  → 该目录没有 CMakeLists.txt
  → CMake 没生成 Makefile
  → make 无目标可构建
```

因此，构建前先检查：

```bash
pwd
ls
find .. -maxdepth 2 -name CMakeLists.txt
```

## 4. Catkin

Catkin 建立在 CMake 之上，负责组织 ROS1 工作空间和功能包。

```text
catkin_ws/
├── src/       # 功能包源码
├── build/     # 构建中间文件
├── devel/     # 开发环境和可执行结果
└── logs/      # catkin_tools 日志
```

功能包中的两个核心文件：

| 文件 | 作用 |
| --- | --- |
| `package.xml` | 包信息、许可证与依赖 |
| `CMakeLists.txt` | 查找依赖、编译目标和链接库 |

依赖应同时在两个文件中正确声明。只改其中一个，可能导致构建或依赖解析异常。

### 4.1 典型 ROS 包 CMakeLists.txt

```cmake
find_package(catkin REQUIRED COMPONENTS
  roscpp
  geometry_msgs
)

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES dx_kinematics
  CATKIN_DEPENDS roscpp geometry_msgs
)

include_directories(
  include
  ${catkin_INCLUDE_DIRS}
)

add_executable(dx_velocity_node src/velocity_node.cpp)
target_link_libraries(
  dx_velocity_node
  dx_kinematics
  ${catkin_LIBRARIES}
)
```

### 4.2 工作空间构建

```bash
source /opt/ros/noetic/setup.bash
mkdir -p ~/dx_lesson2_ws/src
cd ~/dx_lesson2_ws
catkin_make
source devel/setup.bash
```

`source` 不是安装 ROS，而是把环境变量加载到当前 Shell。新开终端后需要重新加载。

`catkin_make` 和 `catkin build` 是两种构建方式，同一工作空间内最好固定使用一种。

### 4.3 本次 Catkin 实操

工作空间包含：

- `dx_kinematics`：运动学计算库；
- `dx_velocity_node`：计算并发布速度消息的节点。

第一次构建报错：

```text
fatal error: diff_drive.h: No such file or directory
```

真实头文件位于 `include/dx_kinematics/diff_drive.h`，所以引用应写为：

```cpp
#include "dx_kinematics/diff_drive.h"
```

重新 `catkin_make` 并加载环境后：

```bash
roslaunch dx_velocity_node demo.launch
```

终端持续显示：

```text
v=0.500 m/s, w=0.667 rad/s
```

另一终端验证通信：

```bash
source /opt/ros/noetic/setup.bash
source ~/dx_lesson2_ws/devel/setup.bash
rosnode list
rostopic type /dx_demo/cmd_vel
rostopic echo -n 1 /dx_demo/cmd_vel
```

实际消息包含：

```text
linear.x  = 0.5
angular.z = 0.6666666666666667
```

这证明节点不仅打印了计算结果，还真正发布了 `geometry_msgs/Twist`。

## 5. 测试与参数边界

“正常数据算对”不等于程序可靠。运动学函数还应处理：

- `radius <= 0`；
- `wheelBase <= 0`；
- 输入出现 `NaN` 或 `Inf`；
- 有限输入导致结果溢出；
- 负轮速表示倒车，应当允许。

典型检查：

```cpp
if (!std::isfinite(leftWheel) || !std::isfinite(rightWheel) ||
    !std::isfinite(radius) || !std::isfinite(wheelBase) ||
    radius <= 0.0 || wheelBase <= 0.0) {
  throw std::invalid_argument("Invalid wheel parameters");
}
```

改进测试覆盖直行、转弯、原地旋转、静止、倒车以及非法参数，共 14 个场景。

证据必须区分：

| 内容 | 验证状态 |
| --- | --- |
| 本机 CMake 构建、运行、原始 CTest | 已完成 |
| 本机 Catkin 双包构建与 Topic 验证 | 已完成 |
| 加强版 14 项测试 | AI 独立 Linux 环境通过 |
| 本机执行加强版修复脚本 | 待复现 |

## 6. 使用 AI 辅助排错

有效流程是：

```text
提供真实路径和第一条错误
  → 判断错误发生在哪一层
  → 进行最小修改
  → 重新构建
  → 运行程序
  → 用测试和 ROS 工具验证
```

AI 适合解释报错和补充边界情况，但最终应以 `pwd`、`ls`、编译输出、CTest 结果和 ROS 消息为准。不能把 AI 环境通过写成本机已经完成。

## 7. 固定排错顺序

1. 用 `pwd` 确认所在目录；
2. 用 `ls`、`find` 确认文件存在；
3. 确认源码目录中有 `CMakeLists.txt`；
4. 先让 CMake 配置成功，再执行构建；
5. 头文件错误检查 include 路径；
6. `undefined reference` 检查目标与链接库；
7. ROS 包找不到时检查依赖和 `source`；
8. 构建成功后仍要运行目标并检查输出。

