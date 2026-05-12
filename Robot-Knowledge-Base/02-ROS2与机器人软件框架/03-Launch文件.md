# 03-Launch文件

> 学习目标：理解 launch 文件为什么是 ROS2 工程化启动的入口。学完本文后，应能看懂常见 `.launch.py` 文件，并能写一个简单 launch 文件启动多个节点。

---

## 1. 一句话解释

**Launch 文件用于一次性启动、配置和组织多个 ROS2 节点。**

如果没有 launch 文件，一个机器人系统可能要手动打开多个终端：

```bash
ros2 run robot_state_publisher robot_state_publisher
ros2 run joint_state_publisher joint_state_publisher
ros2 run rviz2 rviz2
ros2 run gazebo_ros gzserver
ros2 run nav2_controller controller_server
ros2 run nav2_planner planner_server
```

有了 launch 文件，可以一条命令启动：

```bash
ros2 launch my_robot_bringup bringup.launch.py
```

---

## 2. Launch 文件解决什么问题

Launch 文件主要解决：

| 问题 | Launch 的作用 |
|---|---|
| 节点太多 | 一次启动多个节点 |
| 参数太多 | 自动加载 YAML 参数文件 |
| 需要重映射 topic | 在启动时配置 remapping |
| 需要命名空间 | 支持 namespace，方便多机器人 |
| 需要组合其他启动文件 | 支持 include 其他 launch 文件 |
| 需要条件启动 | 支持 if/unless 条件 |
| 需要启动 RViz/Gazebo/Nav2/MoveIt2 | 统一管理启动流程 |

可以把 launch 文件理解为：

```text
机器人系统的启动剧本
```

---

## 3. Launch 文件常见类型

ROS2 中常见 launch 文件格式：

| 格式 | 后缀 | 特点 | 建议 |
|---|---|---|---|
| Python Launch | `.launch.py` | 功能最强，最常见 | 初学优先掌握 |
| XML Launch | `.launch.xml` | 结构清晰 | 能看懂即可 |
| YAML Launch | `.launch.yaml` | 简洁 | 了解即可 |

当前大多数复杂项目会使用 Python launch，因为它支持条件、参数处理、路径拼接和复杂逻辑。

---

## 4. 最小 Launch 文件示例

### 4.1 文件路径

建议放在 package 的 `launch/` 目录下：

```text
my_robot_bringup/
├─ package.xml
├─ setup.py 或 CMakeLists.txt
└─ launch/
   └─ turtlesim_demo.launch.py
```

### 4.2 示例内容

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            name='turtlesim'
        ),
        Node(
            package='turtlesim',
            executable='turtle_teleop_key',
            name='teleop_turtle'
        )
    ])
```

核心结构：

```text
generate_launch_description()
    ↓
LaunchDescription([...])
    ↓
Node(...), Node(...), IncludeLaunchDescription(...)
```

---

## 5. 如何运行 Launch 文件

通用命令：

```bash
ros2 launch <package_name> <launch_file.py>
```

示例：

```bash
ros2 launch my_robot_bringup turtlesim_demo.launch.py
```

如果 launch 文件不在已安装的 package 中，也可以直接运行文件路径：

```bash
ros2 launch ./turtlesim_demo.launch.py
```

但工程中更推荐通过 package 启动。

---

## 6. Node 启动参数详解

常用写法：

```python
Node(
    package='demo_nodes_cpp',
    executable='talker',
    name='my_talker',
    namespace='robot1',
    output='screen',
    parameters=[{'use_sim_time': True}],
    remappings=[('/chatter', '/robot1/chatter')]
)
```

| 字段 | 含义 |
|---|---|
| `package` | 节点所在功能包 |
| `executable` | 要运行的可执行程序 |
| `name` | 节点启动后的名字 |
| `namespace` | 命名空间，多机器人常用 |
| `output` | 日志输出位置，常用 `screen` |
| `parameters` | 加载参数 |
| `remappings` | topic/service 名称重映射 |
| `arguments` | 启动参数 |

---

## 7. 参数文件 YAML

### 7.1 为什么需要参数文件

机器人项目中参数很多，例如：

```text
机器人尺寸
传感器位置
控制频率
速度上限
规划器参数
PID 参数
仿真时间开关
```

这些不适合写死在代码里，通常放在 YAML 中。

### 7.2 YAML 参数示例

```yaml
my_node:
  ros__parameters:
    use_sim_time: true
    max_linear_speed: 0.5
    max_angular_speed: 1.0
    robot_radius: 0.22
```

### 7.3 Launch 中加载 YAML

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    config_file = os.path.join(
        get_package_share_directory('my_robot_bringup'),
        'config',
        'robot_params.yaml'
    )

    return LaunchDescription([
        Node(
            package='my_robot_package',
            executable='my_node',
            name='my_node',
            parameters=[config_file],
            output='screen'
        )
    ])
```

---

## 8. Remapping：重映射 topic 名称

### 8.1 为什么需要 remapping

同一个节点在不同机器人或不同项目中，topic 名称可能不一样。

例如某节点默认发布：

```text
/cmd_vel
```

但你的系统希望使用：

```text
/robot1/cmd_vel
```

这时可以用 remapping。

### 8.2 示例

```python
Node(
    package='demo_nodes_cpp',
    executable='talker',
    name='talker',
    remappings=[('/chatter', '/robot1/chatter')]
)
```

含义：

```text
节点内部使用 /chatter
系统实际变成 /robot1/chatter
```

---

## 9. Launch Arguments：启动参数

### 9.1 作用

Launch Arguments 允许启动时传参：

```bash
ros2 launch my_robot_bringup robot.launch.py use_sim_time:=true robot_name:=go2
```

### 9.2 示例

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    use_sim_time = LaunchConfiguration('use_sim_time')

    return LaunchDescription([
        DeclareLaunchArgument(
            'use_sim_time',
            default_value='true',
            description='Use simulation clock if true'
        ),
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            name='turtlesim',
            parameters=[{'use_sim_time': use_sim_time}],
            output='screen'
        )
    ])
```

运行：

```bash
ros2 launch my_robot_bringup turtlesim.launch.py use_sim_time:=false
```

---

## 10. IncludeLaunchDescription：包含其他 launch 文件

### 10.1 为什么需要 include

复杂机器人系统通常不是一个 launch 文件写到底，而是分层组织：

```text
bringup.launch.py
├─ description.launch.py
├─ gazebo.launch.py
├─ control.launch.py
├─ navigation.launch.py
└─ rviz.launch.py
```

这样更清晰，也便于复用。

### 10.2 示例

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    gazebo_launch = os.path.join(
        get_package_share_directory('my_robot_sim'),
        'launch',
        'gazebo.launch.py'
    )

    return LaunchDescription([
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(gazebo_launch)
        )
    ])
```

---

## 11. 条件启动

### 11.1 典型场景

```text
use_rviz=true 时启动 RViz
use_sim=true 时启动 Gazebo
use_camera=true 时启动相机节点
```

### 11.2 示例

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.conditions import IfCondition
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    use_rviz = LaunchConfiguration('use_rviz')

    return LaunchDescription([
        DeclareLaunchArgument('use_rviz', default_value='true'),
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            condition=IfCondition(use_rviz),
            output='screen'
        )
    ])
```

运行时关闭 RViz：

```bash
ros2 launch my_robot_bringup display.launch.py use_rviz:=false
```

---

## 12. namespace：多机器人常用

如果同时启动两个机器人，不能都叫 `/cmd_vel`、`/odom`，否则会冲突。

命名空间示例：

```python
Node(
    package='turtlesim',
    executable='turtlesim_node',
    namespace='robot1',
    name='sim'
)
```

启动后节点名可能变成：

```text
/robot1/sim
```

topic 也可能变成：

```text
/robot1/turtle1/cmd_vel
```

多机器人常见结构：

```text
/robot1/cmd_vel
/robot1/odom
/robot1/scan

/robot2/cmd_vel
/robot2/odom
/robot2/scan
```

---

## 13. 典型机器人项目的 Launch 结构

### 13.1 AMR 启动结构

```text
amr_bringup/launch/bringup.launch.py
    ↓
启动 robot_state_publisher
启动 lidar driver
启动 base driver
启动 slam 或 localization
启动 nav2
启动 rviz
```

数据链路：

```text
/scan + /odom + /tf → SLAM/Nav2 → /cmd_vel → base_driver
```

### 13.2 机械臂启动结构

```text
arm_bringup/launch/demo.launch.py
    ↓
启动 robot_state_publisher
启动 ros2_control_node
启动 joint_state_broadcaster
启动 trajectory_controller
启动 move_group
启动 rviz
```

数据链路：

```text
目标位姿 → MoveIt2 → FollowJointTrajectory Action → 控制器 → 关节
```

### 13.3 四足机器人仿真启动结构

```text
quadruped_bringup/launch/sim.launch.py
    ↓
加载机器人模型
启动仿真器
启动 joint_state_publisher / robot_state_publisher
启动控制器
启动状态估计
启动步态控制
启动 RViz
```

数据链路：

```text
IMU/关节/接触状态 → 状态估计 → 步态控制 → 关节命令
```

---

## 14. Python package 中安装 launch 文件

如果你创建的是 Python 类型 package，需要在 `setup.py` 中安装 launch 文件，否则 `ros2 launch` 可能找不到。

示例：

```python
import os
from glob import glob
from setuptools import setup

package_name = 'my_robot_bringup'

setup(
    name=package_name,
    version='0.0.0',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
        (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='your_name',
    maintainer_email='your_email@example.com',
    description='Robot bringup package',
    license='Apache-2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [],
    },
)
```

编译：

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

---

## 15. CMake package 中安装 launch 文件

如果是 C++ / CMake 类型 package，需要在 `CMakeLists.txt` 中加入：

```cmake
install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY config
  DESTINATION share/${PROJECT_NAME}
)
```

然后：

```bash
colcon build
source install/setup.bash
```

---

## 16. 实验：写一个 turtlesim launch 文件

### 16.1 创建 package

```bash
cd ~/ros2_ws/src
ros2 pkg create my_robot_bringup --build-type ament_python
cd my_robot_bringup
mkdir launch
```

### 16.2 创建文件

```bash
nano launch/turtlesim_demo.launch.py
```

写入：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            name='turtlesim',
            output='screen'
        ),
        Node(
            package='turtlesim',
            executable='turtle_teleop_key',
            name='teleop_turtle',
            output='screen'
        )
    ])
```

### 16.3 修改 setup.py

在 `data_files` 中加入 launch 安装规则：

```python
import os
from glob import glob
```

并加入：

```python
(os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
```

### 16.4 编译运行

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
ros2 launch my_robot_bringup turtlesim_demo.launch.py
```

### 16.5 验证

```bash
ros2 node list
ros2 topic list
```

---

## 17. 常见报错与排查

### 17.1 找不到 package

报错类似：

```text
Package 'my_robot_bringup' not found
```

排查：

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
ros2 pkg list | grep my_robot_bringup
```

原因通常是：

```text
没有编译
没有 source install/setup.bash
package.xml/setup.py/CMakeLists.txt 配置有问题
```

### 17.2 找不到 launch 文件

报错类似：

```text
file 'xxx.launch.py' was not found in the share directory
```

排查：

```bash
ros2 pkg prefix my_robot_bringup
find install -name "*.launch.py"
```

原因通常是：

```text
launch 文件没有被安装到 share 目录
setup.py 或 CMakeLists.txt 没有安装 launch 目录
```

### 17.3 找不到 executable

报错类似：

```text
executable 'xxx' not found
```

排查：

```bash
ros2 pkg executables <package_name>
```

原因通常是：

```text
executable 名字写错
节点没有安装
entry_points 或 install(TARGETS) 配置错误
```

### 17.4 参数不生效

排查：

```bash
ros2 param list
ros2 param get /node_name param_name
```

常见原因：

```text
节点名称和 YAML 中的节点名不一致
YAML 缩进错误
参数类型错误
launch 中没有正确加载参数文件
```

### 17.5 use_sim_time 问题

仿真中很多节点需要：

```yaml
use_sim_time: true
```

否则可能出现：

```text
TF 时间不同步
RViz 显示异常
Nav2 行为异常
```

---

## 18. 阅读别人 launch 文件的顺序

建议按下面顺序阅读：

```text
1. 找 generate_launch_description()
2. 看 DeclareLaunchArgument 定义了哪些启动参数
3. 看 Node 启动了哪些节点
4. 看 IncludeLaunchDescription 包含了哪些子 launch
5. 看 parameters 加载了哪些 YAML
6. 看 remappings 改了哪些 topic 名
7. 看 condition 决定哪些节点是否启动
8. 看 namespace 是否影响 topic 和 node 名称
```

不要一开始纠结所有 Python 语法，先看系统启动了什么。

---

## 19. Launch 文件在后续项目中的学习重点

| 项目 | launch 阅读重点 |
|---|---|
| TurtleBot3 | Gazebo、robot_state_publisher、Nav2 如何一起启动 |
| Nav2 | planner、controller、bt_navigator、costmap 参数如何加载 |
| MoveIt2 | move_group、robot_description、controller 如何启动 |
| Unitree 仿真 | 模型、控制器、仿真器、可视化如何连接 |
| 四足 CHAMP | gait、controller、state_estimator 的启动关系 |

---

## 20. 学习检查清单

完成本文后，你应该能回答：

- Launch 文件解决什么问题？
- `.launch.py` 的核心函数是什么？
- `Node(...)` 中 package、executable、name 分别是什么？
- 参数 YAML 如何加载？
- remapping 是什么？
- namespace 的作用是什么？
- 如何用 launch argument 控制是否启动 RViz？
- 为什么复杂机器人项目要 include 多个 launch 文件？
- launch 文件找不到时应该检查哪里？

---

## 21. 本文总结

Launch 文件的核心价值：

```text
把机器人系统从“多个终端手动启动”，变成“一个工程化启动入口”。
```

掌握 launch 文件后，你看开源项目时就能快速判断：

```text
这个项目启动了哪些节点？
加载了哪些模型和参数？
topic 有没有重映射？
是否启动仿真、RViz、控制器、导航或规划模块？
```

这对于后续学习 TurtleBot3、MoveIt2、Unitree Go2/G1 仿真都非常重要。

---

## 22. 后续关联文档

建议继续阅读：

- [[01-ROS2总体理解]]
- [[02-Node-Topic-Service-Action]]
- [[04-TF坐标变换]]
- [[05-RViz可视化]]
- [[06-ros2_control控制框架]]
- [[08-常见报错与解决]]
