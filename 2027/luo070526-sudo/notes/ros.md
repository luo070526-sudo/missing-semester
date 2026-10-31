---
tags:
  - type/lesson
  - topic/ros
  - status/done
date: 2026-10-29
---

# 第三课：ROS 架构与通信

ROS 解决的是“多个机器人程序如何分工并交换数据”。本课用 turtlesim 串起节点、话题、消息、服务、参数和 Catkin。

```text
输入节点产生指令
  → ROS 通信传递数据
  → 处理节点更新状态
  → 状态再次发布给其他节点
```

## 1. ROS 计算图

一台 RoboMaster 机器人可能同时运行相机驱动、视觉识别、状态估计、决策、底盘控制和电机通信。ROS 把这些功能拆成节点，并用统一接口连接。

| 概念 | 含义 | 例子 |
| --- | --- | --- |
| Package | 组织源码、配置和依赖 | `turtlesim` |
| Executable | 编译得到的可执行程序 | `turtlesim_node` |
| Node | 加入 ROS 计算图的运行中进程 | `/turtlesim` |
| Topic | 持续传输数据的通道 | `/turtle1/cmd_vel` |
| Message | 话题中数据的结构 | `geometry_msgs/Twist` |
| Service | 一次请求对应一次响应 | `/spawn` |
| Parameter | 运行时配置 | 背景颜色 |

命令：

```bash
rosrun turtlesim turtlesim_node
```

其中第一个 `turtlesim` 是包名，`turtlesim_node` 是可执行文件名，运行后的默认节点名是 `/turtlesim`。

### 1.1 Master

```bash
roscore
```

节点先向 Master 注册名称、话题、服务和通信地址。订阅者找到发布者后，消息主要在节点之间直接传输。

因此 Master 更像“登记处”，而不是每条消息都必须经过的中转站。

## 2. turtlesim：从按键到运动

需要三个终端：

```bash
# 终端 1
roscore

# 终端 2
rosrun turtlesim turtlesim_node

# 终端 3
rosrun turtlesim turtle_teleop_key
```

把输入焦点放在终端 3，再按方向键。

内部过程：

```text
方向键
  → turtle_teleop_key 生成 Twist
  → 发布到 /turtle1/cmd_vel
  → /turtlesim 接收消息
  → 更新速度、位姿和画面
  → 发布 /turtle1/pose
```

查看消息结构：

```bash
rosmsg show geometry_msgs/Twist
```

`Twist` 中包含 `linear` 和 `angular` 两个三维向量。但 turtlesim 的运动回调主要读取：

- `linear.x`：前进或后退；
- `angular.z`：在二维平面内转向。

消息中存在 `linear.y`，不代表订阅端一定会使用它。理解通信时需要同时看消息定义和接收端源码。

### 2.1 观察节点与数据

```bash
rosnode list
rosnode info /turtlesim
rosnode ping /turtlesim
rostopic list
rostopic info /turtle1/cmd_vel
rostopic type /turtle1/cmd_vel
rostopic echo /turtle1/pose
rqt_graph
```

### 2.2 结束节点

```bash
rosnode kill /turtlesim
```

- `Ctrl+C`：向当前终端的前台进程发送中断信号；
- `q`：`turtle_teleop_key` 自己定义的退出键；
- `rosnode kill`：从其他终端请求指定节点关闭。

## 3. Topic：发布与订阅

Topic 适合速度、图像、雷达和位姿等连续数据。发布者与订阅者正确连接需要：

1. 话题名一致；
2. 消息类型一致；
3. 节点注册到同一个 Master；
4. 双方网络地址可访问。

### 3.1 发布者：发送 hello

```cpp
#include <ros/ros.h>
#include <std_msgs/String.h>

int main(int argc, char** argv) {
  ros::init(argc, argv, "talker");
  ros::NodeHandle nh;

  ros::Publisher pub =
      nh.advertise<std_msgs::String>("/chatter", 10);
  ros::Rate rate(1);

  while (ros::ok()) {
    std_msgs::String msg;
    msg.data = "hello";
    pub.publish(msg);
    ROS_INFO("publish: %s", msg.data.c_str());
    rate.sleep();
  }
  return 0;
}
```

- `advertise` 声明话题和消息类型；
- `10` 是发送队列长度，不是频率；
- `ros::Rate(1)` 表示循环约为 1 Hz；
- `publish` 发送消息。

### 3.2 订阅者：接收并打印

```cpp
#include <ros/ros.h>
#include <std_msgs/String.h>

void callback(const std_msgs::String::ConstPtr& msg) {
  ROS_INFO("I heard: %s", msg->data.c_str());
}

int main(int argc, char** argv) {
  ros::init(argc, argv, "listener");
  ros::NodeHandle nh;

  ros::Subscriber sub =
      nh.subscribe("/chatter", 10, callback);
  ros::spin();
  return 0;
}
```

消息到达后执行 `callback`。`ros::spin()` 让节点持续等待并处理回调；删掉它以后，`main` 很快结束。

### 3.3 命令行发布速度

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

`-r 10` 表示以 10 Hz 重复发布。`linear.x = 2.0` 与 `angular.z = 1.0` 让乌龟一边前进一边转向。

若出现：

```text
ERROR: invalid topic type: 10
```

说明 `10` 被放到了消息类型的位置。正确顺序是：

```text
rostopic pub [选项] 话题名 消息类型 消息内容
```

## 4. Service 与 Parameter

### 4.1 Service

Service 适合“一次请求并等待结果”，例如生成新乌龟。

```bash
rosservice list
rosservice type /spawn
rossrv show turtlesim/Spawn
rosservice call /spawn "x: 2.0
y: 3.0
theta: 0.0
name: 'turtle2'"
```

使用服务前先用 `rossrv show` 查看请求和响应字段，不要凭截图猜参数。

### 4.2 Parameter

Parameter 保存阈值、增益和颜色等运行配置，不适合高频数据。

```bash
rosparam list
rosparam get /turtlesim/background_r
rosparam set /turtlesim/background_r 255
rosparam set /turtlesim/background_g 255
rosparam set /turtlesim/background_b 255
rosservice call /clear
```

### 4.3 选择通信方式

| 需求 | 方式 |
| --- | --- |
| 相机持续发布图像 | Topic |
| 持续发送底盘速度 | Topic |
| 生成一只乌龟并获得结果 | Service |
| 设置控制增益 | Parameter |
| 长时间任务，需要进度和取消 | Action |

## 5. C++ 节点怎样被构建

```text
节点源码
  → CMakeLists.txt 规定编译与链接
  → package.xml 声明包和依赖
  → Catkin 构建
  → source 工作空间环境
  → rosrun 或 roslaunch 启动
```

典型目录：

```text
dx_ros_practice/
├── CMakeLists.txt
├── package.xml
└── src/
    ├── talker.cpp
    └── listener.cpp
```

核心 CMake：

```cmake
find_package(catkin REQUIRED COMPONENTS
  roscpp
  std_msgs
)

catkin_package()

include_directories(${catkin_INCLUDE_DIRS})

add_executable(talker src/talker.cpp)
target_link_libraries(talker ${catkin_LIBRARIES})

add_executable(listener src/listener.cpp)
target_link_libraries(listener ${catkin_LIBRARIES})
```

构建与运行：

```bash
source /opt/ros/noetic/setup.bash
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
catkin_create_pkg dx_ros_practice roscpp std_msgs
cd ~/catkin_ws
catkin_make
source devel/setup.bash
rosrun dx_ros_practice talker
```

另一个终端：

```bash
source /opt/ros/noetic/setup.bash
source ~/catkin_ws/devel/setup.bash
rosrun dx_ros_practice listener
```

验证：

```bash
rosnode list
rostopic info /chatter
rostopic echo /chatter
rostopic hz /chatter
```

只看到“没有报错”还不够；还要确认节点、连接、消息内容和频率。

## 6. 本次 Docker 与 ROS 实操

### 6.1 宿主机与容器

在 Ubuntu 宿主机查看和进入容器：

```bash
sudo docker ps -a
sudo docker exec -it noetic-gui bash
```

出现 `root@...#` 后已经位于容器内部，不要再次执行 `docker exec`。

容器内检查环境：

```bash
echo "$ROS_DISTRO"
source /opt/ros/noetic/setup.bash
source ~/dx_lesson2_ws/devel/setup.bash
```

宿主机找不到 `/opt/ros/noetic/setup.bash`，不代表容器中的 ROS 没有安装。

### 6.2 已验证的节点与话题

本次已查询到：

```text
/dx_velocity_node
/dx_demo/cmd_vel
```

继续验证应使用：

```bash
rosnode info /dx_velocity_node
rostopic info /dx_demo/cmd_vel
rostopic type /dx_demo/cmd_vel
rostopic echo -n 1 /dx_demo/cmd_vel
```

若 `roscore` 提示已有 Master，不要重复启动，先用 `rosnode list` 与 `rostopic list` 检查现有系统。

### 6.3 图形显示问题

本次 turtlesim 出现过：

```text
Authorization required
could not connect to display :0
Qt xcb error
```

这属于容器图形转发问题，优先检查：

- 宿主机 X11 授权；
- `DISPLAY`；
- `/tmp/.X11-unix` 是否挂载。

它不等于 ROS 安装失败。当前小乌龟图形显示尚未完成修复验证。

## 7. 在 RoboMaster 中的位置

| 课堂例子 | 比赛系统 |
| --- | --- |
| 键盘发布速度 | 决策节点发布底盘目标 |
| `/turtle1/cmd_vel` | 底盘速度目标话题 |
| turtlesim 更新状态 | 控制器计算电机输出 |
| `/turtle1/pose` | 定位或状态估计结果 |

卡尔曼滤波可以作为状态估计节点：订阅编码器、IMU 或视觉测量，发布融合后的机器人状态。`ros_control` 的控制循环可以记为：

1. `read`：读取传感器与电机状态；
2. `update`：计算控制量；
3. `write`：通过 CAN 等接口发送电机命令。

ROS 解决数据如何传递，算法解决收到数据后如何计算。

## 8. 固定排错顺序

| 顺序 | 检查内容 | 命令 |
| --- | --- | --- |
| 1 | Master 是否存在 | `rosnode list` |
| 2 | 节点是否运行 | `rosnode ping /turtlesim` |
| 3 | 发布订阅是否连接 | `rosnode info`、`rqt_graph` |
| 4 | 话题名是否正确 | `rostopic list` |
| 5 | 消息类型是否正确 | `rostopic type`、`rosmsg show` |
| 6 | 是否真的有数据 | `rostopic echo`、`rostopic hz` |
| 7 | 服务字段是否正确 | `rosservice type`、`rossrv show` |
| 8 | 包是否构建并加载 | `catkin_make`、`source devel/setup.bash` |

“乌龟不动”时，按节点、话题、类型、数据、订阅端处理的顺序检查，不要直接重装 ROS。

