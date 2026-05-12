# Unitree Go2 调试记录（结构化整理版）

> 适用场景：Windows 11 + WSL 2 + Docker + ROS 2 Humble 环境下，进行 Unitree Go2 的 RViz 可视化、关节滑块联调和后续 Gazebo 仿真。

## 0. 文档元信息

| 字段 | 内容 |
|---|---|
| Area Tag | Robot |
| Category | Project |
| Status | 进行中 |
| 创建日期 | 2026-03-13 11:25 |
| 当前用途 | Go2 仿真环境复盘、重启指南、常见问题排查 |

---

## 1. 当前目标与状态

### 1.1 当前目标

在 Windows 11 电脑上，通过 WSL 2 + Docker 容器运行 ROS 2 Humble，完成 Unitree Go2 的：

1. RViz2 中的机器人模型可视化；
2. `joint_state_publisher_gui` 关节滑块控制；
3. `robot_state_publisher` 发布 TF 坐标变换；
4. 后续 Gazebo 动力学仿真与基础运动控制。

### 1.2 已完成内容

| 模块 | 状态 | 说明 |
|---|---:|---|
| WSL 2 基础环境 | 已完成 | Windows 11 上运行 Ubuntu/WSL 2 |
| Docker 容器 | 已完成 | 容器名：`unitree_lab` |
| ROS 2 环境 | 已完成 | 容器内使用 ROS 2 Humble |
| Go2 模型包 | 已部署 | `go2_description` Humble 版 |
| Unitree ROS 2 包 | 已部署 | `unitree_ros2` |
| 编译 | 已成功 | `colcon build` 后 5 个包全部 Finished |
| TF/关节逻辑链路 | 基本通畅 | 滑块变化时 TF 坐标轴能够变化 |
| RViz 模型显示 | 待确认/修复 | 曾出现“只有骨架，没有 Mesh 肉身”的问题 |

### 1.3 当前核心待办

当前最重要的任务不是重新搭环境，而是确认 **RViz 是否能正确加载 Mesh 模型文件**。

典型现象：

- 如果能看到 Go2 银色外壳，说明 Mesh 路径和模型加载正常；
- 如果只能看到坐标轴或骨架，说明 TF 逻辑正常，但 RViz 没有找到或没有加载 Mesh 文件。

---

## 2. 系统架构说明

### 2.1 三层运行结构

| 层级 | 名称 | 作用 |
|---|---|---|
| 第一层 | Windows 11 | 主机系统，提供图形界面、显卡驱动、日常操作环境 |
| 第二层 | WSL 2 / Ubuntu | Linux 子系统，用于运行 Docker 和 Linux 工具链 |
| 第三层 | Docker 容器 `unitree_lab` | ROS 2 开发环境，隔离依赖，保证环境稳定 |

### 2.2 为什么使用 Docker

Docker 相当于一个独立的“机器人开发实验室”。ROS 2、Unitree 依赖、Python/C++ 库都安装在容器内，避免污染 Windows 或 WSL 宿主系统。

好处：

1. 环境可重复；
2. 依赖不容易乱；
3. 换电脑后更容易迁移；
4. 出问题时可以明确区分：是 Windows/WSL 问题，还是容器内部问题。

---

## 3. 关键目录与软件包

### 3.1 工作空间目录

| 路径 | 说明 |
|---|---|
| `/root/unitree_ws` | ROS 2 工作空间根目录 |
| `/root/unitree_ws/src` | 源码区，放置 ROS 2 包 |
| `/root/unitree_ws/build` | 编译中间文件 |
| `/root/unitree_ws/install` | 编译安装结果，运行时主要读取这里 |
| `/root/unitree_ws/log` | 编译与运行日志 |

### 3.2 核心软件包

| 包名 | 作用 |
|---|---|
| `unitree_ros2` | Unitree 机器人通信与控制相关包 |
| `go2_description` | Go2 机器人模型描述，包括 URDF/Xacro、Mesh、Launch 文件 |
| `robot_state_publisher` | 根据 URDF 和关节角度发布 TF 坐标变换 |
| `joint_state_publisher_gui` | 图形化关节滑块，用于手动改变关节角度 |
| `rviz2` | ROS 2 可视化工具，用于观察模型、TF、Topic 等 |

### 3.3 机器人模型文件类型

| 文件/目录 | 常见后缀 | 作用 |
|---|---|---|
| URDF / Xacro | `.urdf`, `.xacro` | 定义机器人 Link、Joint、坐标关系和关节限制 |
| Meshes | `.dae`, `.stl` | 提供真实 3D 外观模型，也就是“肉身/皮肤” |
| Launch | `.launch.py` | 一键启动多个 ROS 2 节点 |
| Config | `.yaml`, `.rviz` | 保存参数、RViz 配置、话题设置等 |
| Package/CMake | `package.xml`, `CMakeLists.txt` | 描述包信息、依赖关系和安装规则 |

---

## 4. 标准启动流程

> 建议严格按照本节流程执行。每打开一个新终端，都需要重新 `source` 环境。

### 4.1 Windows 终端：进入 WSL

```powershell
wsl -d Ubuntu
```

### 4.2 WSL 终端：启动并进入 Docker 容器

```bash
docker start unitree_lab
docker exec -it unitree_lab /bin/bash
```

如果 Docker 权限异常，可在 WSL 中执行：

```bash
sudo chmod 666 /var/run/docker.sock
```

### 4.3 容器内：加载 ROS 2 和工作空间环境

```bash
cd /root/unitree_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
```

验证包是否能被识别：

```bash
ros2 pkg list | grep -E "go2|unitree"
```

---

## 5. RViz 可视化联调流程

### 5.1 终端 1：启动机器人状态发布器

作用：发布机器人 URDF 和 TF 坐标变换。

```bash
docker exec -it unitree_lab /bin/bash
cd /root/unitree_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch go2_description robot.launch.py
```

### 5.2 终端 2：启动 RViz2

作用：观察机器人模型、TF、RobotModel 状态。

```bash
docker exec -it unitree_lab /bin/bash
export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA
export LC_NUMERIC=C
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
rviz2
```

RViz 内部设置：

1. 将 `Fixed Frame` 改为 `base_link`；
2. 点击 `Add`，添加 `RobotModel`；
3. 如果看不见模型，检查左侧 `RobotModel -> Status` 是否有黄色或红色警告；
4. 如果 RobotModel 的 `Description Source` 使用 `Topic` 不成功，可以临时切换为 `File` 并手动选择 URDF 文件。

### 5.3 终端 3：启动关节滑块 GUI

作用：手动改变关节角度，观察 RViz 中 TF/模型是否联动。

```bash
docker exec -it unitree_lab /bin/bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
ros2 run joint_state_publisher_gui joint_state_publisher_gui
```

### 5.4 预期现象

| 操作 | 正常现象 |
|---|---|
| 启动 `robot_state_publisher` | 无明显报错，持续发布机器人描述与 TF |
| 启动 RViz2 | 能打开图形窗口 |
| Fixed Frame 改为 `base_link` | Fixed Frame 不再报红 |
| 添加 RobotModel | 能显示 Go2 模型或至少显示模型结构 |
| 拖动关节滑块 | RViz 中关节、坐标轴或模型姿态发生变化 |

---

## 6. 找回 RViz “肉身”的排查流程

当 RViz 中只有坐标轴或骨架，没有 Go2 外壳时，按下面顺序排查。

### 6.1 确认 Mesh 文件是否存在

```bash
ls /root/unitree_ws/install/go2_description/share/go2_description/meshes
```

判断：

| 结果 | 含义 | 处理 |
|---|---|---|
| 能看到 `.dae` / `.stl` 文件 | Mesh 已安装 | 继续检查 RViz 设置或 URDF 路径 |
| 目录为空或不存在 | Mesh 未被安装 | 检查源码包、CMake 安装规则，重新编译 |

### 6.2 确认工作空间已 source

```bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
```

很多“只有骨架没有肉身”的问题，本质是没有 source 工作空间，导致 ROS 2 找不到 `package://go2_description/...` 对应的资源路径。

### 6.3 RViz 手动指定 URDF 文件

在 RViz 左侧 RobotModel 中操作：

1. 找到 `Description Source`；
2. 从 `Topic` 切换为 `File`；
3. 点击 `Description File` 右侧的 `...`；
4. 选择类似路径：

```text
/root/unitree_ws/install/go2_description/share/go2_description/urdf/go2_description.urdf
```

如果手动指定 URDF 后模型能显示，说明问题集中在 robot_description 话题或环境路径上。

---

## 7. Gazebo 仿真启动流程

当 RViz 模型显示正常后，再进入 Gazebo 动力学仿真。

### 7.1 启动 Gazebo

```bash
cd /root/unitree_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch unitree_gazebo gazebo.launch.py rname:=go2
```

### 7.2 显存不足或卡顿时使用 Headless 模式

```bash
ros2 launch unitree_gazebo gazebo.launch.py rname:=go2 headless:=true
```

### 7.3 Gazebo 启动后的检查

新开一个终端进入容器：

```bash
docker exec -it unitree_lab /bin/bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
ros2 topic list
```

查看关节状态：

```bash
ros2 topic echo /joint_states
```

如果 `/joint_states` 有持续输出，说明仿真与 ROS 2 通信基本正常。

---

## 8. 编译与重编译规范

### 8.1 标准编译流程

必须在工作空间根目录执行：

```bash
cd /root/unitree_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

### 8.2 推荐使用 `--symlink-install`

`--symlink-install` 不会把文件完整复制到 `install`，而是建立软链接。

优点：

1. 修改 Python 脚本、URDF、Mesh、Launch 后通常不需要完整重编；
2. `install` 中能实时反映源码区变动；
3. 适合模型调试和脚本开发阶段。

### 8.3 只编译某一个包

```bash
colcon build --packages-select go2_description --symlink-install
```

### 8.4 清理后重编译

当曾经在宿主机和容器里混合编译，或者出现路径冲突时，执行：

```bash
cd /root/unitree_ws
rm -rf build install log
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

---

## 9. 常见问题与处理方法

### 9.1 找不到 `ament_cmake`

现象：编译时报 `ament_cmake` 找不到。

原因：没有加载 ROS 2 环境。

处理：

```bash
source /opt/ros/humble/setup.bash
```

然后重新执行 `colcon build`。

### 9.2 `Fixed Frame [map] does not exist`

原因：RViz 默认 Fixed Frame 是 `map`，但当前模型没有发布 `map` 坐标系。

处理：

```text
RViz -> Global Options -> Fixed Frame -> base_link
```

### 9.3 只有骨架，没有模型外壳

常见原因：

1. 没有 `source install/setup.bash`；
2. Mesh 文件没有被安装到 `install` 目录；
3. URDF 中 Mesh 路径错误；
4. RViz RobotModel 没有正确读取 `robot_description`。

优先处理：

```bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
ls /root/unitree_ws/install/go2_description/share/go2_description/meshes
```

### 9.4 RViz 或 Gazebo 卡顿

可能原因：WSL 2 图形渲染、Docker 渲染缓存、显存占用较高。

处理建议：

```bash
export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA
export LC_NUMERIC=C
```

Gazebo 可优先使用：

```bash
ros2 launch unitree_gazebo gazebo.launch.py rname:=go2 headless:=true
```

必要时重启 WSL：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

### 9.5 ROS 2 Topic/Node 状态异常

可重启 ROS 2 daemon：

```bash
ros2 daemon stop
ros2 daemon start
```

---

## 10. ROS 2 基础概念速查

### 10.1 为什么每个终端都要 `source`

Linux 环境变量是进程隔离的。每个新终端都是一个新的 shell，会话之间不会自动继承你手动加载的 ROS 2 路径。

所以每个终端都需要执行：

```bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
```

可以理解为：给当前终端戴上“ROS 2 眼镜”，让它知道 `ros2` 命令在哪里、Go2 模型包在哪里。

### 10.2 什么是 ROS 2 Package

Package 是 ROS 2 软件组织的最小功能单元。一个包通常包含：

| 内容 | 作用 |
|---|---|
| `package.xml` | 包名称、版本、依赖关系 |
| `CMakeLists.txt` | 编译和安装规则 |
| `src/` | C++ 或 Python 源码 |
| `launch/` | 启动脚本 |
| `urdf/` | 机器人模型结构 |
| `meshes/` | 机器人 3D 外观模型 |
| `config/` | 参数文件 |

### 10.3 GUI / State Publisher / RViz 的关系

| 节点 | 常用命令 | 角色 | 核心任务 |
|---|---|---|---|
| GUI 控制器 | `joint_state_publisher_gui` | 数据源 | 发布 `/joint_states`，表示每个关节角度 |
| 状态发布器 | `robot_state_publisher` | 计算中心 | 根据 URDF 和关节角度计算 TF 坐标变换 |
| RViz2 | `rviz2` | 显示器 | 读取 TF 和 Mesh，显示机器人模型 |

数据流：

```text
joint_state_publisher_gui
        ↓ /joint_states
robot_state_publisher
        ↓ /tf + robot_description
RViz2
        ↓
可视化机器人模型
```

### 10.4 机器人 = 坐标轴集合 + 外观模型

在 RViz/算法世界中，机器人首先是一组坐标系：

- Link 对应刚体/零件；
- Joint 定义父 Link 和子 Link 的相对运动；
- TF 表示各 Link 坐标系之间的变换；
- Mesh 只是挂载在这些坐标系上的外观模型。

对比：

| 维度 | 虚拟世界：RViz/算法 | 现实世界：真机 |
|---|---|---|
| 主体 | 坐标轴 Frames | 金属/碳纤维零件 Links |
| 核心 | 数学变换 TF | 电机力矩 Torque |
| 运动本质 | 矩阵数值变化 | 电机带动物理位移 |
| 视觉外观 | 挂载 Mesh 模型 | 实际外壳与结构件 |

---

## 11. 常用命令速查表

| 目的 | 命令 |
|---|---|
| 进入 WSL | `wsl -d Ubuntu` |
| 关闭 WSL | `wsl --shutdown` |
| 启动容器 | `docker start unitree_lab` |
| 进入容器 | `docker exec -it unitree_lab /bin/bash` |
| 退出容器 | `exit` |
| 加载 ROS 2 | `source /opt/ros/humble/setup.bash` |
| 加载工作空间 | `source /root/unitree_ws/install/setup.bash` |
| 编译全部包 | `colcon build --symlink-install` |
| 编译指定包 | `colcon build --packages-select go2_description --symlink-install` |
| 查看包 | `ros2 pkg list | grep go2` |
| 查看 Topic | `ros2 topic list` |
| 查看关节状态 | `ros2 topic echo /joint_states` |
| 启动 RViz | `rviz2` |
| 启动关节 GUI | `ros2 run joint_state_publisher_gui joint_state_publisher_gui` |
| 重启 ROS 2 daemon | `ros2 daemon stop && ros2 daemon start` |

---

## 12. 下一步建议

建议按下面顺序推进，避免跳步导致问题定位困难。

### 阶段 1：确认 RViz 静态显示

目标：Go2 模型能够在 RViz 中完整显示，包括 Mesh 外壳。

检查点：

1. Fixed Frame = `base_link`；
2. RobotModel 无红色错误；
3. Mesh 文件目录非空；
4. 拖动视角能看到完整 Go2 模型。

### 阶段 2：确认关节滑块联动

目标：拖动 `joint_state_publisher_gui` 后，RViz 中关节姿态跟随变化。

检查点：

1. `/joint_states` 有输出；
2. `/tf` 有输出；
3. RViz 中模型或坐标轴跟随变化。

### 阶段 3：进入 Gazebo 动力学仿真

目标：启动 Gazebo 场景，并加载 Go2。

检查点：

1. Gazebo 能打开或 headless 模式能启动；
2. `/joint_states` 有输出；
3. 没有持续刷屏的 plugin/package 错误。

### 阶段 4：编写基础控制脚本

目标：基于 Python 或 ROS 2 Topic，实现基础控制验证。

建议先做：

1. 读取 `/joint_states`；
2. 发布简单控制指令；
3. 实现站立、关节摆动、低速行走等最小动作；
4. 记录每个 Topic 的输入输出关系，逐步形成知识库。
