# Unitree Go2 ROS2 项目笔记

## 1. 项目解决什么问题

本项目用于在 Windows 11 + WSL 2 + Docker + ROS 2 Humble 环境中加载 Unitree Go2，并通过 RViz、关节滑块和后续 Gazebo 仿真理解一台四足机器人的软件链路。

当前已跑通的核心能力：

- 在 Docker 容器中编译 `unitree_ros2` 和 `go2_description`；
- 使用 `go2_description` 加载 Go2 的 URDF、Xacro 和 Mesh；
- 使用 `robot_state_publisher` 发布机器人 TF；
- 使用 `joint_state_publisher_gui` 手动改变关节角；
- 在 RViz 中观察机器人模型、坐标系和关节状态。

这不是单纯的“显示模型”项目，而是一条很好的机器人学习主线：从环境搭建、模型描述、ROS 2 通信、TF 坐标树、RViz 调试，一直延伸到 Gazebo 仿真和基础控制。

## 2. 适合学习什么

| 学习方向 | 本项目里的具体对象 | 可沉淀笔记 |
|---|---|---|
| ROS 2 工程结构 | package、launch、topic、source、colcon | [[02-ROS2与机器人软件框架/01-ROS2总体理解]] |
| 模型描述 | URDF、Xacro、Link、Joint、Mesh | [[01-机器人基础/04-URDF-SDF-MJCF模型文件]] |
| 坐标变换 | `/joint_states`、`robot_state_publisher`、`/tf` | [[02-ROS2与机器人软件框架/04-TF坐标变换]] |
| 可视化调试 | RViz、RobotModel、Fixed Frame、TF | [[02-ROS2与机器人软件框架/05-RViz可视化]] |
| 四足结构 | Go2 躯干、四条腿、12 个主动自由度 | [[06-四足机器人/03-腿部结构与自由度]] |
| 问题排查 | Mesh 丢失、source 失效、Docker/WSL/RViz 问题 | [[14-问题库/03-报错与解决记录]] |
| 控制入门 | 站立、蹲起、关节摆动、低速行走 | [[06-四足机器人/08-关节PD与阻抗控制]] |

## 3. 运行环境

| 项目 | 当前记录 |
|---|---|
| 主机系统 | Windows 11 |
| Linux 层 | WSL 2 / Ubuntu |
| 容器 | Docker 容器 `unitree_lab` |
| ROS 版本 | ROS 2 Humble |
| 工作空间 | `/root/unitree_ws` 或宿主侧 `/home/jim/unitree_ws` |
| 核心包 | `go2_description`、`unitree_ros2`、`unitree_ros2_example` |

三层结构：

```text
Windows 11
  -> WSL 2 / Ubuntu
      -> Docker: unitree_lab
          -> ROS 2 Humble + Unitree Go2 packages
```

这个结构的好处是依赖隔离清楚。Windows 负责图形界面和日常操作，WSL 提供 Linux 环境，Docker 提供稳定可复现的 ROS 2 实验室。

## 4. 关键目录

| 路径 | 作用 |
|---|---|
| `/root/unitree_ws` | 容器内 ROS 2 工作空间 |
| `/root/unitree_ws/src` | 源码区 |
| `/root/unitree_ws/src/go2_description` | Go2 模型描述包 |
| `/root/unitree_ws/src/unitree_ros2` | Unitree ROS 2 通信和示例包 |
| `/root/unitree_ws/install` | 编译安装结果 |
| `/home/jim/unitree_ws` | WSL 宿主侧可见的工作空间 |
| `/home/jim/unitree_ws/Go2_API_AutoDoc.md` | 当前自动生成的话题接口记录 |

`go2_description` 中值得重点阅读：

| 文件/目录 | 学习重点 |
|---|---|
| `package.xml` | 包名、依赖关系 |
| `CMakeLists.txt` | Mesh、URDF、launch 如何被安装到 share 目录 |
| `launch/robot.launch.py` | 启动 `robot_state_publisher` 的方式 |
| `urdf/go2_description.urdf` | 展开后的完整机器人模型 |
| `xacro/robot.xacro`、`xacro/leg.xacro` | 机器人主体和腿部的模块化描述 |
| `meshes/*.dae` | RViz 中看到的外观模型 |

## 5. 标准启动命令

进入 WSL：

```bash
wsl -d Ubuntu
```

启动并进入容器：

```bash
docker start unitree_lab
docker exec -it unitree_lab /bin/bash
```

每个新终端都需要加载环境：

```bash
cd /root/unitree_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
```

启动机器人状态发布器：

```bash
ros2 launch go2_description robot.launch.py
```

启动 RViz：

```bash
export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA
export LC_NUMERIC=C
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
rviz2
```

启动关节滑块：

```bash
ros2 run joint_state_publisher_gui joint_state_publisher_gui
```

## 6. 核心数据流

```text
joint_state_publisher_gui
  -> /joint_states
      -> robot_state_publisher
          -> /tf + /tf_static + robot_description
              -> RViz RobotModel
```

理解这条链路非常关键：

- `joint_state_publisher_gui` 是关节角数据源；
- `/joint_states` 描述每个关节的角度、速度和力矩；
- `robot_state_publisher` 读取 URDF 和关节角，计算每个 link 的坐标变换；
- `/tf` 和 `/tf_static` 是机器人坐标树；
- RViz 读取 TF、URDF 和 Mesh 后显示完整机器人。

如果拖动滑块后 RViz 中坐标轴会动，说明关节状态和 TF 链路基本通了；如果只有骨架没有外壳，通常是 Mesh 路径或资源安装问题。

## 7. Go2 模型理解

Go2 可以先从“躯干 + 四条腿”理解：

| 部分 | 常见 Link/Mesh | 直觉理解 |
|---|---|---|
| 躯干 | `trunk` / `base` | 机器人主体，承载电池、计算、传感器 |
| 髋部 | `hip` | 决定腿向身体侧面摆动 |
| 大腿 | `thigh` | 连接髋部和膝部，是主要摆动段 |
| 小腿 | `calf` | 连接膝部和足端 |
| 足端 | `foot` | 与地面接触，是运动控制的重要末端 |

四足机器人通常每条腿 3 个主动关节，四条腿共 12 个主动自由度。学习时可以先把每条腿看成一个 3 自由度机械臂：输入是 3 个关节角，输出是足端位置。

## 8. 当前已掌握内容

- 已知道 Docker 容器和 ROS 2 工作空间的分工；
- 已知道每个终端都需要 `source` ROS 2 和工作空间环境；
- 已知道 RViz 显示机器人依赖 URDF、TF 和 Mesh 三部分；
- 已知道 `/joint_states` 到 `/tf` 的基本数据流；
- 已知道“只有骨架没有肉身”时，优先检查 Mesh 文件、安装路径和 RViz RobotModel 状态；
- 已经具备继续进入 Gazebo 和基础控制脚本的条件。

## 9. 下一步实验路线

### 阶段 1：RViz 模型完整显示

目标：能看到 Go2 的完整银色外壳，而不是只有坐标轴或骨架。

检查：

```bash
ls /root/unitree_ws/install/go2_description/share/go2_description/meshes
ros2 topic echo /robot_description --once
ros2 topic list | grep -E "joint|tf|robot"
```

记录到：

- [[02-ROS2与机器人软件框架/05-RViz可视化]]
- [[14-问题库/03-报错与解决记录]]

### 阶段 2：关节滑块联动

目标：拖动 `joint_state_publisher_gui` 后，RViz 中模型或坐标系跟着变化。

检查：

```bash
ros2 topic echo /joint_states
ros2 topic echo /tf
```

记录到：

- [[02-ROS2与机器人软件框架/04-TF坐标变换]]
- [[06-四足机器人/03-腿部结构与自由度]]

### 阶段 3：Gazebo 动力学仿真

目标：在仿真环境中让 Go2 具备物理属性和关节状态输出。

启动参考：

```bash
ros2 launch unitree_gazebo gazebo.launch.py rname:=go2
```

如果图形界面卡顿：

```bash
ros2 launch unitree_gazebo gazebo.launch.py rname:=go2 headless:=true
```

记录到：

- [[03-仿真软件/01-仿真软件对比]]
- [[06-四足机器人/11-Unitree-Go2-MuJoCo笔记]]

### 阶段 4：基础控制脚本

目标：写最小脚本读取关节状态，并尝试发布简单控制命令。

建议实验：

- 读取 `/joint_states` 并打印 12 个关节名称；
- 做单腿关节摆动；
- 做四腿同步蹲起；
- 记录站立、蹲起、摆腿对应的关节变化；
- 对照 PD 控制理解目标角、当前角、误差和反馈。

## 10. 暂时没看懂的部分

| 问题 | 当前理解 | 下一步 |
|---|---|---|
| Go2 真机通信和仿真接口是否完全一致 | 目前主要验证了模型和 ROS 2 可视化链路 | 阅读 `unitree_ros2_example` 的 Go2 示例 |
| Gazebo 控制插件如何接入关节 | 需要模型中的 transmission 和 ros2_control 配置 | 读 `urdf/transmission.urdf.xacro` 和 `config/ros_control` |
| Sport Client 和低层关节控制的区别 | 前者更像高层运动模式，后者更接近电机/关节控制 | 阅读 `go2_sport_client.cpp` 和 `low_level_ctrl.cpp` |
| 步态控制如何从关节命令生成 | 需要足端轨迹、状态估计和接触判断 | 结合 CHAMP 或 Unitree 示例继续看 |

## 11. 知识库填充建议

优先填这些卡片：

1. [[02-ROS2与机器人软件框架/04-TF坐标变换]]：用 Go2 的 `/joint_states -> /tf` 解释 TF。
2. [[02-ROS2与机器人软件框架/05-RViz可视化]]：记录 RobotModel、Fixed Frame、Mesh 排查。
3. [[01-机器人基础/04-URDF-SDF-MJCF模型文件]]：用 `go2_description` 解释 URDF/Xacro/Mesh。
4. [[06-四足机器人/03-腿部结构与自由度]]：用 Go2 解释 12 自由度四足结构。
5. [[14-问题库/03-报错与解决记录]]：沉淀“只有骨架没有 Mesh 外壳”的排查流程。

## 12. 相关笔记

- [[100-robot-learning-notes/UnitreeGo2调试记录]]
- [[00-总览与路线图/04-开源项目索引]]
- [[02-ROS2与机器人软件框架/07-常用命令速查]]
- [[06-四足机器人/13-四足机器人常见问题]]
