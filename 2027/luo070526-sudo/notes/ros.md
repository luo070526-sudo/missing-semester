# 第三课：ROS 架构与通信学习笔记

> 本节使用 ROS 1（Noetic）的命令和写法。学习重点不是背命令，而是先弄清楚：一个程序如何成为节点、节点怎样交换数据、小乌龟为什么会动、出错时应该从哪里查。

## 1. ROS 到底解决什么问题

一台机器人通常同时运行很多程序：相机采图、视觉识别、决策、底盘控制、电机通信等。如果每两个程序都自己约定连接方式，系统会很难维护。ROS 提供了一套统一的通信和管理机制，让不同程序按相同规则交换数据。

我把几个核心概念理解成下面这样：

| ROS 概念 | 简单理解 | 典型例子 |
| --- | --- | --- |
| 节点（Node） | 一个正在运行、负责具体任务的程序 | `/turtlesim`、`/teleop_turtle` |
| 话题（Topic） | 持续传递数据的频道，发布者和订阅者不必同步等待 | `/turtle1/cmd_vel` |
| 消息（Message） | 频道中数据的固定格式 | `geometry_msgs/Twist` |
| 服务（Service） | 一次请求对应一次回答 | `/spawn` 生成新乌龟 |
| 参数（Parameter） | 运行时配置 | 背景颜色 |
| Master | 帮节点注册并找到彼此 | `roscore` 启动它 |

ROS 不是代替 Ubuntu 的完整操作系统。它更像运行在 Linux 上的机器人软件中间件、工具链和生态。

## 2. 先看完整例子：小乌龟为什么会动

课堂上用键盘控制小乌龟时，实际发生的是一条完整的数据链：

```mermaid
flowchart TD
    A[按下方向键] --> B[turtle_teleop_key 读取按键]
    B --> C[向 /turtle1/cmd_vel 发布 Twist]
    C --> D[/turtlesim 节点订阅消息]
    D --> E[更新速度和位姿]
    E --> F[重绘小乌龟并发布 /turtle1/pose]
```

关键点：方向键不是直接控制屏幕上的乌龟。键盘节点先把“想要的线速度和角速度”写进消息，再由 turtlesim 节点接收和处理。

### 2.1 三个终端完成演示

终端 1：启动 ROS Master。

```bash
roscore
```

终端 2：启动仿真窗口。

```bash
rosrun turtlesim turtlesim_node
```

终端 3：启动键盘控制节点。

```bash
rosrun turtlesim turtle_teleop_key
```

最后要把输入焦点放在终端 3，再按方向键。课堂照片里出现的提示 `Use arrow keys to move the turtle` 就是在说明该终端正在读取按键。

### 2.2 小乌龟内部怎样处理命令

查看 turtlesim 的官方源代码可以看到，乌龟对象订阅相对名称 `cmd_vel`。它位于 `/turtle1` 命名空间下，所以实际话题名是 `/turtle1/cmd_vel`。

收到 `geometry_msgs/Twist` 后，回调函数读取：

- `linear.x`：前进或后退的线速度；
- `linear.y`：平面内另一方向的线速度；
- `angular.z`：绕竖直方向旋转的角速度。

随后更新循环根据速度和时间间隔计算新的位置与朝向，重新绘制小乌龟，并发布 `/turtle1/pose`。官方实现中，如果超过 1 秒没有再收到速度命令，速度会被清零，所以松开按键后乌龟不会一直运动。

这段内部处理把“按键—消息—回调—状态更新—画面变化”连成了完整闭环。

## 3. 不要混淆：包、可执行文件、节点、话题和消息类型

以这条命令为例：

```bash
rosrun turtlesim turtlesim_node
```

| 名称 | 本例中的值 | 作用 |
| --- | --- | --- |
| 包名 | `turtlesim` | 组织源代码、配置和依赖 |
| 可执行文件名 | `turtlesim_node` | 被 `rosrun` 找到并启动的程序 |
| 运行后的节点名 | `/turtlesim` | ROS 计算图中的身份 |
| 速度话题名 | `/turtle1/cmd_vel` | 数据传输频道 |
| 速度消息类型 | `geometry_msgs/Twist` | 规定频道中每个字段的结构 |

因此，可执行文件名和节点名可以不同。排查正在运行的程序时应看节点名，而不是只盯着启动命令。

### 3.1 查看节点

```bash
rosnode list
rosnode info /turtlesim
rosnode ping /turtlesim
```

- `rosnode list`：有哪些节点正在向 Master 注册；
- `rosnode info /turtlesim`：该节点发布、订阅了什么，以及提供了哪些服务；
- `rosnode ping /turtlesim`：节点是否还能响应。

需要从另一个终端结束节点时，可以使用：

```bash
rosnode kill /turtlesim
```

### 3.2 `Ctrl+C` 和 `q` 有什么区别

- 在 `turtle_teleop_key` 中按 `q`，是程序自己定义的退出按键；
- 在终端按 `Ctrl+C`，会向当前前台进程发送中断信号；
- roscpp 收到正常的关闭信号后，`ros::ok()` 会变为 `false`，`ros::spin()` 也会退出，节点可以结束清理工作。

所以课堂终端里出现的 `^C` 表示终端收到了 `Ctrl+C`，它不是发给小乌龟的运动指令。

## 4. 话题通信：发布者和订阅者怎样配对

话题通信适合连续数据，例如相机图像、雷达数据、位姿和速度指令。发布者只负责发送，订阅者只负责接收；双方能否连接，关键是：

1. 话题名相同；
2. 消息类型相同；
3. 两个节点都已经正常运行并完成注册。

### 4.1 发布者：每秒发送一次 `hello`

课堂照片中的核心代码可以整理为：

```cpp
#include <ros/ros.h>
#include <std_msgs/String.h>

int main(int argc, char** argv) {
  ros::init(argc, argv, "talker");
  ros::NodeHandle nh;

  ros::Publisher pub = nh.advertise<std_msgs::String>("/chatter", 10);
  ros::Rate rate(1);

  while (ros::ok()) {
    std_msgs::String msg;
    msg.data = "hello";
    pub.publish(msg);
    rate.sleep();
  }
  return 0;
}
```

逐句理解：

- `ros::init`：初始化 ROS，并把当前进程注册为名叫 `talker` 的节点；
- `ros::NodeHandle`：后续创建发布者、订阅者等 ROS 对象的入口；
- `advertise`：声明要在 `/chatter` 上发布 `std_msgs/String`；
- `10`：发布队列长度。发送过快、消息来不及处理时，队列满后旧消息可能被丢弃；
- `ros::Rate(1)`：循环频率为 1 Hz；
- `publish`：真正把消息发出。

### 4.2 订阅者：收到消息后打印

```cpp
#include <ros/ros.h>
#include <std_msgs/String.h>

void callback(const std_msgs::String::ConstPtr& msg) {
  ROS_INFO("I heard: %s", msg->data.c_str());
}

int main(int argc, char** argv) {
  ros::init(argc, argv, "listener");
  ros::NodeHandle nh;

  ros::Subscriber sub = nh.subscribe("/chatter", 10, callback);
  ros::spin();
  return 0;
}
```

- `subscribe`：订阅 `/chatter`，消息到达时交给 `callback`；
- 第二个 `10`：订阅队列长度，不是消息类型；
- `ros::spin()`：让节点持续等待并执行回调，否则 `main` 很快结束，订阅者也就退出了；
- 回调函数应尽量短。耗时任务会阻塞后续回调，造成延迟或丢消息。

`ros::` 是 C++ 的命名空间限定符，表示这里使用的是 ROS C++ 客户端库中的成员。

### 4.3 Master 在其中做什么

节点启动后先向 Master 注册自己的名称、话题和通信地址。订阅者查询到发布者地址后，会和发布者建立直接连接。之后大部分话题数据直接在两个节点之间传输，并不经过 Master。

所以 `roscore` 的主要作用是“登记和牵线”，不是替每条消息做中转。

## 5. 不写代码也能检查和发布话题

先查看系统里有哪些话题：

```bash
rostopic list
```

再确认速度话题的连接信息和消息类型：

```bash
rostopic info /turtle1/cmd_vel
rostopic type /turtle1/cmd_vel
rosmsg show geometry_msgs/Twist
```

查看小乌龟不断发布的位姿：

```bash
rostopic echo /turtle1/pose
```

以 10 Hz 发布速度，让乌龟一边前进一边转弯：

```bash
rostopic pub -r 10 /turtle1/cmd_vel geometry_msgs/Twist "linear:
  x: 2.0
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 1.0"
```

这里的顺序是：发布选项、话题名、消息类型、消息内容。

### 5.1 课堂报错：`ERROR: invalid topic type: 10`

这个错误说明命令行把 `10` 解析到了“消息类型”的位置，但合法类型应该类似 `geometry_msgs/Twist`。遇到它时不需要盲目改数字，按下面顺序检查：

```bash
rostopic type /turtle1/cmd_vel
rosmsg show geometry_msgs/Twist
```

确认类型后，再完整输入上面的 `rostopic pub -r 10 ...`。这里 `-r 10` 的含义是以 10 Hz 重复发送，`10` 不是消息类型。

## 6. 服务通信：发出请求，等待一次结果

服务适合“调用一次并希望得到明确回应”的操作，例如生成一只新乌龟、清空轨迹。它与话题的区别是：

| 对比项 | 话题 Topic | 服务 Service |
| --- | --- | --- |
| 交互方式 | 发布后不等待某个订阅者回答 | 客户端发请求，服务端返回响应 |
| 常见用途 | 速度、图像、位姿等连续数据 | 触发一次动作或查询一次结果 |
| 关系 | 可一对多 | 一次调用对应一次应答 |

查看服务和 `/spawn` 的类型：

```bash
rosservice list
rosservice type /spawn
rossrv show turtlesim/Spawn
```

生成名为 `turtle2` 的新乌龟：

```bash
rosservice call /spawn "x: 2.0
y: 3.0
theta: 0.0
name: 'turtle2'"
```

其中 `x`、`y`、`theta` 是初始位置和朝向，返回值中的 `name` 是最终生成的乌龟名称。课堂照片里命令行自动列出的 `x、y、theta、name`，就是 `/spawn` 请求的数据结构。

## 7. 参数：修改配置，不负责连续传数据

参数服务器适合保存背景颜色、阈值等配置。它不适合代替高频话题。

查看所有参数：

```bash
rosparam list
```

把 turtlesim 背景改为白色：

```bash
rosparam set /turtlesim/background_r 255
rosparam set /turtlesim/background_g 255
rosparam set /turtlesim/background_b 255
rosservice call /clear
```

`rosparam set` 修改参数，`/clear` 让 turtlesim 重新读取背景颜色并刷新画面。参数较多时通常写进 YAML，再由 launch 文件统一加载，避免每次手动输入。

## 8. ROS 软件包和工作空间只需先掌握这些

一个常见 catkin 工作空间如下：

```text
catkin_ws/
├── src/
├── build/
└── devel/
```

- `src`：源代码和软件包；
- `build`：CMake 构建过程产生的中间文件；
- `devel`：编译后供当前工作空间使用的环境和可执行结果。

软件包中最重要的两个文件：

- `package.xml`：包名、版本、作者和依赖；
- `CMakeLists.txt`：如何编译、链接和安装。

在工作空间根目录编译并加载环境：

```bash
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

如果 `rosrun` 找不到刚编译的包，先检查当前终端是否执行了 `source devel/setup.bash`。

## 9. 从小乌龟连接到真实机器人控制

小乌龟例子中，`/turtle1/cmd_vel` 只是让仿真程序更新画面；真实机器人中，同样的速度指令还要经过控制器和硬件接口。

```mermaid
flowchart TD
    A[/cmd_vel 目标速度] --> B[Controller 计算控制量]
    B --> C[controller_manager 调度]
    C --> D[RobotHW read / write]
    D --> E[CAN 或 EtherCAT]
    E --> F[电机执行]
    F --> D
```

可以把 ros_control 的一次循环记成三步：

1. `read`：从编码器、电机驱动等硬件读取状态；
2. `update`：控制器根据目标和当前状态计算输出；
3. `write`：把输出通过 CAN、EtherCAT 等总线发给电机。

因此，ROS 话题负责模块之间传递“目标和状态”，控制器负责计算，RobotHW 负责把统一接口翻译成真实硬件通信。

## 10. 一套实用的排错顺序

遇到“程序启动了但没有反应”时，我应该按数据流逐层检查，而不是反复重启：

1. **节点是否存在**

   ```bash
   rosnode list
   ```

2. **目标节点发布和订阅了什么**

   ```bash
   rosnode info /turtlesim
   ```

3. **话题是否存在、名称是否完全一致**

   ```bash
   rostopic list
   ```

4. **消息类型是否正确**

   ```bash
   rostopic type /turtle1/cmd_vel
   rosmsg show geometry_msgs/Twist
   ```

5. **话题里是否真的有数据**

   ```bash
   rostopic echo /turtle1/pose
   rostopic hz /turtle1/pose
   ```

6. **发布者和订阅者是否连上**

   ```bash
   rostopic info /turtle1/cmd_vel
   rqt_graph
   ```

7. **如果是服务，再查服务类型和请求字段**

   ```bash
   rosservice type /spawn
   rossrv show turtlesim/Spawn
   ```

这套顺序对应“进程—连接—类型—数据—调用”，能快速判断问题是在节点没运行、名字写错、类型不匹配，还是根本没有数据。

## 11. 这节课我需要能独立回答的问题

- 为什么方向键不是直接控制小乌龟，而是先发布 `Twist` 消息？
- `/turtle1/cmd_vel` 和 `geometry_msgs/Twist` 分别是话题名还是消息类型？
- 为什么发布者和订阅者的话题名、消息类型都必须一致？
- `ros::spin()` 为什么不能随便删掉？
- `Ctrl+C`、`q` 和 `rosnode kill` 分别在什么场景下结束节点？
- Master 完成连接后，话题数据是否仍然要经过 Master？
- 什么情况用 Topic，什么情况用 Service，什么情况用 Parameter？
- 真实机器人中，Controller、RobotHW 和 CAN/EtherCAT 各自负责什么？
- 看到 `invalid topic type: 10` 时，应该先检查哪个参数位置？

如果这些问题能不看笔记讲清楚，说明已经掌握了第三课的主线，而不是只记住几条命令。

## 参考资料

- 课程第三课 PPT《ROS 架构与通信》及课堂演示照片
- [ROS Noetic 官方 talker 示例](https://github.com/ros/ros_tutorials/blob/noetic-devel/roscpp_tutorials/talker/talker.cpp)
- [ROS Noetic 官方 listener 示例](https://github.com/ros/ros_tutorials/blob/noetic-devel/roscpp_tutorials/listener/listener.cpp)
- [turtlesim 官方运动处理源码](https://github.com/ros/ros_tutorials/blob/noetic-devel/turtlesim/src/turtle.cpp)
- [ros_control 官方仓库](https://github.com/ros-controls/ros_control/tree/noetic-devel)
