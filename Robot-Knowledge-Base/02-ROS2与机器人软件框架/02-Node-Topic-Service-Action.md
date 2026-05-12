# 02-Node-Topic-Service-Action

> 学习目标：理解 ROS2 最核心的通信模型。学完本文后，应能判断一个功能应该用 Topic、Service 还是 Action，并能用命令查看和调试它们。

---

## 1. 总体关系

ROS2 中最核心的概念可以先记成下面这句话：

```text
Node 是软件模块；Topic 是连续数据流；Service 是短请求；Action 是长任务。
```

四者关系：

```text
Node A 发布 Topic ─────→ Node B 订阅 Topic
Node C 请求 Service ───→ Node D 返回结果
Node E 发送 Action Goal → Node F 执行任务、反馈进度、返回结果
```

---

## 2. 对比总表

| 概念 | 通信特点 | 是否持续 | 是否有返回结果 | 是否可反馈进度 | 是否可取消 | 典型例子 |
|---|---|---:|---:|---:|---:|---|
| Node | 一个运行中的功能模块 | 是 | 取决于内部接口 | 取决于内部接口 | 取决于内部接口 | 雷达节点、导航节点、控制节点 |
| Topic | 发布-订阅，连续数据流 | 是 | 否 | 否 | 否 | `/scan`、`/odom`、`/cmd_vel` |
| Service | 请求-响应，一问一答 | 否 | 是 | 否 | 否 | 清除地图、重置仿真、查询状态 |
| Action | 目标-反馈-结果 | 是 | 是 | 是 | 是 | 导航到目标点、执行机械臂轨迹 |

---

## 3. Node：机器人软件模块

### 3.1 一句话解释

**Node 是 ROS2 中最小的运行单元。**

一个节点通常负责一个相对独立的功能。

例如：

```text
camera_driver_node      负责读取相机
lidar_driver_node       负责读取雷达
slam_node               负责建图和定位
planner_node            负责路径规划
controller_node         负责输出控制指令
motor_driver_node       负责驱动电机
```

### 3.2 Node 的常见组成

一个节点内部可以包含：

| 组成 | 作用 |
|---|---|
| Publisher | 发布 topic |
| Subscriber | 订阅 topic |
| Service Server | 提供 service |
| Service Client | 请求 service |
| Action Server | 执行长任务 |
| Action Client | 发送长任务目标 |
| Parameter | 保存可配置参数 |
| Timer | 定时执行函数 |

### 3.3 机器人中的 Node 例子

AMR：

```text
/lidar_node → 发布 /scan
/robot_state_publisher → 发布 /tf
/slam_toolbox → 订阅 /scan，发布 /map
/nav2_controller → 订阅路径，发布 /cmd_vel
/base_driver → 订阅 /cmd_vel，控制底盘电机
```

机械臂：

```text
/joint_state_broadcaster → 发布 /joint_states
/move_group → 负责规划
/trajectory_controller → 执行关节轨迹
/robot_state_publisher → 发布机械臂 TF
```

四足机器人：

```text
/imu_driver → 发布 IMU 数据
/joint_state_node → 发布关节角速度和力矩
/state_estimator → 估计机身状态
/gait_controller → 生成步态和足端轨迹
/leg_controller → 输出关节控制命令
```

### 3.4 Node 常用命令

```bash
ros2 node list
ros2 node info /node_name
```

示例：

```bash
ros2 node list
ros2 node info /turtlesim
```

`ros2 node info` 可以查看该节点发布了哪些 topic、订阅了哪些 topic、提供了哪些 service/action。

---

## 4. Topic：连续数据流通信

### 4.1 一句话解释

**Topic 适合传输连续变化的数据。**

例如：

```text
传感器数据、速度指令、位姿、关节状态、图像、点云、地图更新
```

### 4.2 Topic 的特点

| 特点 | 说明 |
|---|---|
| 发布-订阅 | 一个节点发布，多个节点可订阅 |
| 异步通信 | 发布方不需要等待接收方返回 |
| 适合连续数据 | 高频传感器、状态量、控制指令 |
| 依赖 Message 类型 | 每个 topic 都有固定消息格式 |

### 4.3 机器人中的典型 Topic

| Topic | 消息类型示例 | 含义 |
|---|---|---|
| `/scan` | `sensor_msgs/msg/LaserScan` | 激光雷达扫描数据 |
| `/camera/image_raw` | `sensor_msgs/msg/Image` | 相机原始图像 |
| `/odom` | `nav_msgs/msg/Odometry` | 里程计 |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | 底盘速度指令 |
| `/joint_states` | `sensor_msgs/msg/JointState` | 关节位置、速度、力矩 |
| `/tf` | `tf2_msgs/msg/TFMessage` | 坐标变换 |
| `/map` | `nav_msgs/msg/OccupancyGrid` | 栅格地图 |

### 4.4 Topic 常用命令

查看 topic：

```bash
ros2 topic list
ros2 topic list -t
```

查看 topic 内容：

```bash
ros2 topic echo /topic_name
```

查看频率：

```bash
ros2 topic hz /topic_name
```

查看带宽：

```bash
ros2 topic bw /topic_name
```

查看信息：

```bash
ros2 topic info /topic_name
```

手动发布：

```bash
ros2 topic pub /topic_name <msg_type> "<data>"
```

### 4.5 手动发布速度指令示例

以 turtlesim 为例：

```bash
ros2 topic pub /turtle1/cmd_vel geometry_msgs/msg/Twist \
"{linear: {x: 1.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.5}}"
```

如果是 AMR，`/cmd_vel` 一般含义如下：

```text
linear.x   前进/后退速度
angular.z  原地旋转角速度
```

对于差速底盘，常用的速度控制就是：

```text
/cmd_vel → 底盘驱动节点 → 左右轮转速
```

---

## 5. Service：短请求和短响应

### 5.1 一句话解释

**Service 适合“一次请求、一次响应”的任务。**

例如：

```text
清除轨迹、重置仿真、保存地图、切换模式、读取一次状态
```

### 5.2 Service 的特点

| 特点 | 说明 |
|---|---|
| 请求-响应 | Client 发送请求，Server 返回结果 |
| 通常时间较短 | 不适合长时间运动任务 |
| 有明确返回值 | 成功/失败、查询结果、执行结果 |
| 同步感更强 | 调用者通常关心这次调用是否完成 |

### 5.3 机器人中的 Service 例子

| Service | 含义 |
|---|---|
| `/reset` | 重置仿真 |
| `/clear` | 清除 turtlesim 轨迹 |
| `/map_saver/save_map` | 保存地图 |
| `/controller_manager/switch_controller` | 切换控制器 |
| `/set_parameters` | 设置节点参数 |

### 5.4 Service 常用命令

```bash
ros2 service list
ros2 service type /service_name
ros2 service find <service_type>
ros2 service call /service_name <service_type> "<request>"
```

示例：

```bash
ros2 service call /clear std_srvs/srv/Empty "{}"
```

### 5.5 什么时候不用 Service

不要用 Service 传连续数据，例如：

```text
雷达数据、图像数据、实时速度指令、连续关节状态
```

这些应该使用 Topic。

也不要用 Service 执行长时间任务，例如：

```text
导航到 10 米外的目标点
机械臂执行一整段轨迹
四足机器人完成一段复杂动作
```

这些更适合 Action。

---

## 6. Action：长时间任务

### 6.1 一句话解释

**Action 适合执行需要一段时间、需要反馈进度、可能需要取消的任务。**

例如：

```text
导航到目标点
机械臂执行轨迹
抓取一个物体
让机器人完成一个动作序列
```

### 6.2 Action 的结构

Action 通常包含三类信息：

```text
Goal      目标
Feedback  执行过程反馈
Result    最终结果
```

例如导航任务：

```text
Goal：目标位姿
Feedback：当前距离目标还有多远
Result：是否成功到达
```

机械臂轨迹任务：

```text
Goal：关节轨迹
Feedback：当前执行到第几个轨迹点
Result：轨迹是否执行成功
```

### 6.3 Action 常用命令

```bash
ros2 action list
ros2 action info /action_name
ros2 action send_goal /action_name <action_type> "<goal>"
```

带反馈输出：

```bash
ros2 action send_goal /action_name <action_type> "<goal>" --feedback
```

### 6.4 机器人中的典型 Action

| Action | 常见系统 | 含义 |
|---|---|---|
| `/navigate_to_pose` | Nav2 | 导航到目标位姿 |
| `/follow_joint_trajectory` | ros2_control / MoveIt2 | 执行关节轨迹 |
| `/compute_path_to_pose` | Nav2 | 计算到目标点的路径 |
| `/gripper_cmd` | 夹爪控制 | 控制夹爪开合 |

---

## 7. Topic、Service、Action 怎么选

### 7.1 决策规则

```text
数据是否连续变化？
  是 → Topic
  否 → 继续判断

任务是否很快完成？
  是 → Service
  否 → Action

任务是否需要反馈/取消？
  是 → Action
```

### 7.2 例子判断

| 需求 | 推荐通信方式 | 原因 |
|---|---|---|
| 雷达每秒输出 10 次扫描 | Topic | 连续传感器数据 |
| 底盘每秒接收 50 次速度指令 | Topic | 连续控制指令 |
| 查询一次电池电量 | Service 或 Topic | 偶尔查询用 Service，持续显示用 Topic |
| 保存当前地图 | Service | 一次性请求 |
| 导航到房间门口 | Action | 长任务，可反馈可取消 |
| 机械臂执行抓取轨迹 | Action | 长任务，需要知道执行结果 |
| 重置仿真环境 | Service | 短请求 |
| 发布相机图像 | Topic | 高频大数据流 |

---

## 8. Message、Service、Action 类型

### 8.1 Message 类型

Topic 传输的是 Message。

查看消息结构：

```bash
ros2 interface show geometry_msgs/msg/Twist
ros2 interface show sensor_msgs/msg/LaserScan
ros2 interface show nav_msgs/msg/Odometry
```

例如 `Twist`：

```text
Vector3 linear
Vector3 angular
```

它常用于表示速度：

```text
linear.x   前进速度
linear.y   横向速度
linear.z   垂向速度
angular.z  偏航角速度
```

### 8.2 Service 类型

查看 service 结构：

```bash
ros2 interface show std_srvs/srv/Empty
```

Service 文件通常分成请求和响应两部分：

```text
Request
---
Response
```

### 8.3 Action 类型

查看 action 结构：

```bash
ros2 interface show nav2_msgs/action/NavigateToPose
```

Action 文件通常分成三部分：

```text
Goal
---
Result
---
Feedback
```

---

## 9. 实验：用 turtlesim 理解四个概念

### 9.1 启动 turtlesim

终端 1：

```bash
source /opt/ros/humble/setup.bash
ros2 run turtlesim turtlesim_node
```

终端 2：

```bash
source /opt/ros/humble/setup.bash
ros2 run turtlesim turtle_teleop_key
```

### 9.2 查看 Node

```bash
ros2 node list
```

预期看到：

```text
/turtlesim
/teleop_turtle
```

查看节点详情：

```bash
ros2 node info /turtlesim
```

重点观察：

```text
Publishers
Subscribers
Service Servers
Action Servers
```

### 9.3 查看 Topic

```bash
ros2 topic list
ros2 topic echo /turtle1/cmd_vel
ros2 topic echo /turtle1/pose
```

你会看到：

```text
/turtle1/cmd_vel  是控制输入
/turtle1/pose     是乌龟状态输出
```

### 9.4 调用 Service

清除轨迹：

```bash
ros2 service call /clear std_srvs/srv/Empty "{}"
```

生成新的乌龟：

```bash
ros2 service call /spawn turtlesim/srv/Spawn \
"{x: 2.0, y: 2.0, theta: 0.0, name: 'turtle2'}"
```

### 9.5 观察 Action

turtlesim 中可用 action：

```bash
ros2 action list
```

执行旋转动作：

```bash
ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute \
"{theta: 1.57}" --feedback
```

这里体现了 Action 的特点：

```text
给定目标角度 → 执行旋转 → 持续反馈 → 返回结果
```

---

## 10. 一个完整机器人系统的数据链路例子

### 10.1 AMR 导航链路

```text
/lidar_node
  发布 /scan
        ↓
/slam_toolbox 或 /amcl
  订阅 /scan、/tf
  发布 /map、/tf
        ↓
/nav2_planner
  根据地图和目标点生成全局路径
        ↓
/nav2_controller
  生成 /cmd_vel
        ↓
/base_driver
  控制底盘电机
```

Topic：

```text
/scan
/odom
/tf
/map
/cmd_vel
```

Service：

```text
保存地图
切换生命周期状态
清除代价地图
```

Action：

```text
/navigate_to_pose
```

### 10.2 机械臂运动链路

```text
/user_goal_pose
        ↓
/move_group
  逆运动学 + 轨迹规划 + 碰撞检测
        ↓
/follow_joint_trajectory action
        ↓
/joint_trajectory_controller
        ↓
/robot_driver
```

Topic：

```text
/joint_states
/tf
/planning_scene
```

Service：

```text
查询 IK
设置规划场景
切换控制器
```

Action：

```text
/follow_joint_trajectory
```

---

## 11. QoS：初学必须知道但不用深挖

QoS 是 Quality of Service，决定 Topic 通信的可靠性和历史数据策略。

常见参数：

| QoS 项 | 含义 |
|---|---|
| Reliability | 可靠传输还是尽力而为 |
| Durability | 新订阅者是否能收到历史数据 |
| History | 保留多少历史消息 |
| Depth | 队列长度 |

典型理解：

```text
传感器数据：宁愿丢一点旧数据，也要实时 → Best Effort 常见
控制命令：希望稳定收到 → Reliable 常见
静态地图/静态TF：希望后加入节点也能拿到 → Transient Local 常见
```

如果 topic 明明存在但 echo 不到数据，可能和 QoS 不匹配有关。

---

## 12. 常见误区

| 误区 | 正确理解 |
|---|---|
| Topic 有返回值 | Topic 是发布-订阅，没有请求响应 |
| Service 可以用来做导航 | 导航是长任务，更适合 Action |
| Action 只是高级 Service | Action 有目标、反馈、结果，还能取消 |
| 一个节点只能发一个 topic | 一个节点可以有多个 publisher/subscriber/service/action |
| `/cmd_vel` 一定代表轮速 | `/cmd_vel` 是速度指令，底层驱动再转成轮速或关节速度 |
| 所有通信都越可靠越好 | 高频传感器数据更重视实时性，不一定追求每帧可靠 |

---

## 13. 学习检查清单

完成本文后，你应该能回答：

- Node 是什么？
- Topic、Service、Action 的区别是什么？
- `/scan`、`/odom`、`/cmd_vel` 分别是什么类型的数据？
- 为什么导航到目标点更适合 Action？
- 为什么相机图像更适合 Topic？
- 如何查看一个节点订阅和发布了哪些 topic？
- 如何用命令手动发布一个速度指令？
- 如何调用一个 service？
- 如何发送一个 action goal？

---

## 14. 本文总结

最重要的记忆：

```text
Node：功能模块
Topic：连续数据流
Service：短请求响应
Action：长任务，带反馈和结果
```

在后续学习 AMR、机械臂、四足、双足时，都要先画出：

```text
哪些节点在运行？
节点之间通过哪些 topic/service/action 连接？
每条数据链路的输入、处理、输出是什么？
```

---

## 15. 后续关联文档

建议继续阅读：

- [[01-ROS2总体理解]]
- [[03-Launch文件]]
- [[04-TF坐标变换]]
- [[05-RViz可视化]]
- [[07-常用命令速查]]
