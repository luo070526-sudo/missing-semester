# CMake 与 Catkin 学习笔记

> 第二课主题：从 C++ 编译过程理解 Makefile、CMake 和 catkin，并思考怎样正确使用大模型辅助学习。

## 0. 学习前后的理解变化

上课前我把 `g++`、`make` 和 `cmake` 都简单理解成“编译代码的工具”，所以看到一串命令时只会照着输入。看完课堂演示和补充资料后，我现在更愿意把它们分成三个层次：

```text
C++ 源文件
   ↓ 由 g++ 完成预处理、编译、汇编和链接
可执行文件

Makefile --交给--> make --调用--> g++
CMakeLists.txt --交给--> cmake --生成--> Makefile（或其他构建文件）
```

也就是说，CMake 本身不是编译器。它负责描述项目并生成构建系统，最后真正把 C++ 源文件变成目标文件的仍然是编译器。

这节课对我最有用的一点，是让我知道报错不能只看成“代码错了”，而要先判断它发生在编译、链接、构建配置还是环境配置阶段。

---

## 1. 从一条 `g++` 命令开始

一个 C++ 程序从源代码变成可执行文件，大致会经过四个阶段：

| 阶段 | 常用选项 | 典型输出 | 我的理解 |
| --- | --- | --- | --- |
| 预处理 | `-E` | `.i` | 展开头文件和宏，处理 `#include`、`#define` |
| 编译 | `-S` | `.s` | 把预处理后的代码翻译成汇编代码 |
| 汇编 | `-c` | `.o` | 把汇编代码变成目标文件 |
| 链接 | 无固定单阶段选项 | 可执行文件 | 把多个目标文件和库组合起来 |

课堂中使用的分阶段命令可以写成：

```bash
g++ -E main.cpp -o main.i
g++ -S main.i -o main.s
g++ -c main.s -o main.o
g++ main.o -o main
./main
```

平时写 `g++ main.cpp -o main`，只是让编译器驱动程序一次性完成了这些步骤。

### 1.1 头文件声明和源文件实现

多文件项目可以拆成：

```text
project/
├── add.h
├── add.cpp
└── main.cpp
```

`add.h` 放函数声明：

```cpp
#pragma once

int add(int a, int b);
```

`add.cpp` 放函数实现：

```cpp
#include "add.h"

int add(int a, int b)
{
    return a + b;
}
```

`main.cpp` 只需要包含头文件并调用函数：

```cpp
#include <iostream>
#include "add.h"

int main()
{
    std::cout << add(1, 2) << std::endl;
    return 0;
}
```

正确的分离编译和链接方式是：

```bash
g++ -c main.cpp -o main.o
g++ -c add.cpp -o add.o
g++ main.o add.o -o demo
./demo
```

### 1.2 我重点记住的链接错误

如果只执行：

```bash
g++ main.o -o demo
```

可能会出现类似下面的错误：

```text
undefined reference to `add(int, int)`
```

一开始我容易把所有红色报错都当成“语法错误”，但这里 `main.cpp` 已经能编译成 `main.o`，说明声明和语法基本通过了。真正的问题是链接时没有加入包含 `add` 实现的 `add.o`。

我目前的判断方法是：

1. `No such file or directory`：先检查路径、文件名和头文件搜索目录；
2. `was not declared in this scope`：先检查声明、命名空间和拼写；
3. `multiple definition`：检查是否重复定义；
4. `undefined reference`：优先检查实现文件或库有没有参与链接。

---

## 2. Makefile：把重复命令写成规则

手动输入几条 `g++` 命令还能接受，但文件多以后很容易遗漏。Makefile 的作用是写清楚“目标依赖什么，以及如何生成目标”。

```makefile
CXX = g++
CXXFLAGS = -Wall -Wextra -std=c++14

demo: main.o add.o
	$(CXX) main.o add.o -o demo

main.o: main.cpp add.h
	$(CXX) $(CXXFLAGS) -c main.cpp

add.o: add.cpp add.h
	$(CXX) $(CXXFLAGS) -c add.cpp

.PHONY: clean
clean:
	rm -f demo main.o add.o
```

这里每条规则可以概括为：

```text
目标: 依赖
<Tab>生成目标的命令
```

我之前对 `make` 的理解是“重新编译整个项目”，现在知道它会比较文件的修改时间，只重建已经过期的目标。例如只修改 `add.cpp`，通常只需要重新生成 `add.o`，再重新链接，不必重新编译 `main.cpp`。

常用命令：

```bash
make
make clean
make -j4
```

其中 `make -j4` 表示最多并行执行 4 个任务，但并行并不能修复错误的依赖关系。Makefile 中最容易忽略的问题是命令行开头必须是 **Tab**，不能随便用空格代替。

---

## 3. CMake：描述项目，而不是手写所有编译命令

Makefile 与平台和具体构建工具关系较紧。CMake 通过 `CMakeLists.txt` 描述项目，再生成当前平台可使用的 Makefile、Ninja 文件或 IDE 工程。

### 3.1 最小 CMake 项目

```cmake
cmake_minimum_required(VERSION 3.16)
project(cmake_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(demo
    main.cpp
    add.cpp
)

target_compile_options(demo PRIVATE -Wall -Wextra)
```

推荐使用外部构建目录：

```bash
cmake -S . -B build
cmake --build build -j$(nproc)
./build/demo
```

这样生成的缓存和中间文件都在 `build/` 中，源代码目录不会被大量构建文件弄乱。需要彻底重新配置时，也能明确处理 `build/`，而不是在项目目录里寻找各种缓存文件。

### 3.2 头文件目录和链接库

如果项目结构变成：

```text
project/
├── CMakeLists.txt
├── include/add.h
└── src/
    ├── add.cpp
    └── main.cpp
```

可以写成：

```cmake
add_library(math_utils src/add.cpp)

target_include_directories(math_utils
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

add_executable(demo src/main.cpp)
target_link_libraries(demo PRIVATE math_utils)
```

我对 `PUBLIC`、`PRIVATE` 和 `INTERFACE` 的记法是：

| 关键字 | 当前目标使用 | 依赖它的目标使用 |
| --- | --- | --- |
| `PRIVATE` | 是 | 否 |
| `PUBLIC` | 是 | 是 |
| `INTERFACE` | 否 | 是 |

刚开始学习时直接写 `include_directories()` 看起来比较省事，但它会影响较大的目录范围。`target_include_directories()` 和 `target_link_libraries()` 把配置绑定到具体目标，后面排查依赖时更清楚。

### 3.3 我的 CMake 排错顺序

1. 先看终端中出现的**第一条真正错误**，不要只看最后一句 `Failed`；
2. 确认当前位置和项目根目录：`pwd`、`ls`、`tree -L 3`；
3. 如果配置阶段失败，检查 `CMakeLists.txt`、路径和 `find_package()`；
4. 如果编译阶段失败，使用 `cmake --build build --verbose` 查看实际命令；
5. 头文件错误检查 `target_include_directories()`；
6. `undefined reference` 检查 `add_library()`、`add_executable()` 和 `target_link_libraries()`；
7. 修改了缓存相关内容仍然异常时，再考虑重新建立构建目录。

---

## 4. Catkin：在 ROS1 中组织多个 CMake 功能包

Catkin 建立在 CMake 之上，主要帮助 ROS1 管理工作空间、功能包和包之间的依赖。它不是 CMake 的替代品，而是在 ROS 项目中加入了一套约定。

### 4.1 工作空间结构

```text
catkin_ws/
├── src/       # 自己需要维护的源码
├── build/     # 构建中间文件
├── devel/     # 开发环境与生成结果
└── logs/      # catkin_tools 日志
```

我需要注意：平时主要修改 `src/` 中的内容，不应该手动修改 `build/`、`devel/` 和 `logs/` 里的生成文件。

使用 `catkin_tools` 的基本流程：

```bash
source /opt/ros/noetic/setup.bash

mkdir -p ~/catkin_ws/src
cd ~/catkin_ws
catkin init
catkin build
source devel/setup.bash
```

课堂 PPT 中容易误读成 `cd /catkin_ws &mkdir src`。我认为更安全的写法是 `mkdir -p ~/catkin_ws/src`：

- `~/catkin_ws` 位于当前用户家目录，不需要把工作空间建到根目录；
- `&&` 表示前一条成功后再执行下一条，单个 `&` 会把命令放到后台；
- 使用 `-p` 可以一次建立多级目录。

### 4.2 创建功能包

```bash
cd ~/catkin_ws/src
catkin_create_pkg test_pkg roscpp rospy std_msgs

cd ~/catkin_ws
catkin build test_pkg
source devel/setup.bash
rospack find test_pkg
```

一个常见功能包中比较重要的文件有：

| 文件或目录 | 作用 |
| --- | --- |
| `CMakeLists.txt` | 描述怎样查找依赖、编译节点和链接库 |
| `package.xml` | 声明包名、版本、维护者、许可证和依赖 |
| `src/` | C++ 源文件 |
| `include/<包名>/` | 对外头文件 |
| `launch/` | 启动多个节点和设置参数 |
| `msg/`、`srv/` | 自定义消息和服务 |

### 4.3 `package.xml` 与 `CMakeLists.txt` 要对应

例如节点使用了 `roscpp` 和 `std_msgs`，通常需要在 `package.xml` 声明依赖：

```xml
<buildtool_depend>catkin</buildtool_depend>
<depend>roscpp</depend>
<depend>std_msgs</depend>
```

同时在 `CMakeLists.txt` 中查找组件并链接：

```cmake
find_package(catkin REQUIRED COMPONENTS
    roscpp
    std_msgs
)

catkin_package()

include_directories(
    ${catkin_INCLUDE_DIRS}
)

add_executable(hello_node src/hello_node.cpp)
target_link_libraries(hello_node ${catkin_LIBRARIES})
```

我目前的理解是：`package.xml` 更像“这个包对外声明自己需要什么”，`CMakeLists.txt` 则负责“构建时具体怎样找到和使用这些依赖”。只改其中一个，可能出现依赖信息不一致。

### 4.4 `source` 为什么不能忘

```bash
source /opt/ros/noetic/setup.bash
source ~/catkin_ws/devel/setup.bash
```

`source` 会在当前终端加载环境变量。新开终端后，如果没有重新加载环境，可能出现：

```text
catkin: command not found
rospack find 找不到刚创建的包
rosrun 找不到可执行文件
```

排查时我会先查看：

```bash
echo $ROS_DISTRO
echo $ROS_PACKAGE_PATH
echo $CMAKE_PREFIX_PATH
```

另外，`catkin_make` 与 `catkin build` 会产生不同的工作空间布局。一个工作空间里最好固定使用一种方式，避免两套生成目录互相干扰。

---

## 5. 四份补充资料带给我的补充认识

### 5.1 Make、Makefile 与 CMake 的关系

知乎资料和课堂内容解决了我一开始最混乱的问题：

- `gcc/g++` 是编译工具；
- Makefile 是构建规则文件；
- `make` 读取 Makefile 并决定执行哪些命令；
- CMake 读取 `CMakeLists.txt`，生成适合当前环境的构建文件。

我以前会问“CMake 和 Make 哪个更快”，现在觉得这个问题不够准确，因为它们处在不同层次。更应该问：项目如何描述依赖、最终选择了什么生成器、底层实际执行了哪些编译命令。

### 5.2 菜鸟教程中的 CMake 基础

这份资料帮助我把课堂上的命令整理成一个较完整的顺序：

1. `cmake_minimum_required()` 指定最低版本；
2. `project()` 定义项目；
3. `add_executable()` 或 `add_library()` 创建目标；
4. `target_include_directories()` 添加头文件搜索路径；
5. `target_link_libraries()` 建立链接关系；
6. 用外部 `build` 目录完成配置和构建。

比起记很多零散变量，我更想先掌握“目标”和“目标之间的依赖”这条主线。

### 5.3 Catkin Tutorials

Catkin 教程让我看到 ROS 功能包不是把 `.cpp` 文件随便放进一个目录就结束了。工作空间、功能包、清单文件和环境叠加共同决定 ROS 能否找到并运行节点。

我目前把基本流程总结成：

```text
建立工作空间 → 创建功能包 → 声明依赖 → 编写节点
→ 配置 CMakeLists.txt → 构建 → source 环境 → 运行与验证
```

如果以后出现“能编译但是 `rosrun` 找不到”的情况，我不会立刻重装 ROS，而会先检查可执行目标是否生成、包名和节点名是否一致，以及当前终端是否正确 `source`。

### 5.4 《The Emperor Has No Clothes》与 Agent

这篇文章把编程 Agent 拆成了一个不神秘的循环：

1. 大模型读取当前任务和上下文；
2. 大模型决定调用读取、列目录或编辑文件等工具；
3. 本地程序真正执行工具；
4. 执行结果返回给大模型；
5. 大模型继续判断下一步。

我最大的收获是：**真正操作文件和运行命令的是工具，不是大模型本身。** 因此 Agent 能做多大范围的修改，取决于它被授予的目录、工具和权限。以后让 Agent 帮我处理 ROS 工程时，应该先限定工作目录，并要求它在修改后给出差异和验证命令。

---

## 6. 我如何使用大模型辅助这一课

本次整理使用了大模型辅助梳理概念和检查结构，但我不希望把“得到一份答案”当成“我已经学会”。我认为比较有效的交互顺序是：

1. 先自己写出当前理解和具体疑问；
2. 让大模型判断问题属于编译、链接、CMake 还是 catkin 环境；
3. 要求它解释第一条错误，而不是一次重写整个项目；
4. 让它给出最小修改方案和验证命令；
5. 自己在终端执行，比较预期结果和真实输出；
6. 把最后确认过的结论写回笔记。

例如，比起只问“为什么编译失败”，我会尽量提供：

```text
环境：Ubuntu 20.04、ROS Noetic、C++14
目标：使用 catkin build 编译 test_pkg
执行命令：catkin build test_pkg
第一条错误：fatal error: ros/ros.h: No such file or directory
相关文件：package.xml、CMakeLists.txt、src/hello_node.cpp
要求：先判断错误阶段，再给最小修改和验证步骤。
```

这样既方便模型定位，也迫使我自己先观察环境和错误信息。

### 我认为需要警惕的地方

- 大模型可能编造不存在的命令、包名或参数；
- 它可能把 ROS1 的 catkin 和 ROS2 的 colcon 混在一起；
- 它可能给出能运行但破坏目录结构的“暴力修复”；
- 如果直接让它写完整笔记，我会获得一份看起来正确但自己解释不出来的内容；
- 涉及删除、安装、权限和大范围修改时，应先检查目标路径和差异。

所以我希望把大模型当成“可以追问的助教和排错搭档”，而不是替我完成理解过程的人。

---

## 7. 本课自检

- [x] 能说出预处理、编译、汇编和链接的区别
- [x] 能解释 `undefined reference` 为什么通常属于链接问题
- [x] 能读懂简单 Makefile 的目标、依赖和命令
- [x] 能写出最小 `CMakeLists.txt`
- [x] 能解释 CMake、Make 和编译器之间的关系
- [x] 能说出 catkin 工作空间与功能包的基本结构
- [x] 知道 `package.xml` 和 `CMakeLists.txt` 需要保持依赖一致
- [x] 知道新终端中为什么需要重新 `source`
- [x] 能给大模型提供环境、命令、首条错误和相关文件
- [ ] 还需要在 Ubuntu 中独立完成一次从空目录到 ROS 节点运行的全过程

## 8. 下一步计划

1. 不看笔记重新建立一个双文件 C++ 项目；
2. 分别用 Makefile 和 CMake 构建，并故意漏掉 `add.o` 观察错误；
3. 在 `~/catkin_ws/src` 创建一个最小 ROS C++ 功能包；
4. 故意删除一次依赖声明，比较配置、编译和运行阶段的不同错误；
5. 把自己真实遇到的错误、原因和解决命令继续补充到本文件。

## 参考资料

- [ROS Wiki：catkin Tutorials](https://wiki.ros.org/catkin/Tutorials)
- [知乎：课程指定补充资料](https://zhuanlan.zhihu.com/p/476697014)
- [菜鸟教程：CMake 基础](https://www.runoob.com/cmake/cmake-basic.html)
- [Mihail Eric：The Emperor Has No Clothes](https://www.mihaileric.com/The-Emperor-Has-No-Clothes/)
