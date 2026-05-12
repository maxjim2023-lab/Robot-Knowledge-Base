# RViz 可视化

## 1. 一句话理解

RViz 是 ROS 2 的数据可视化工具。它本身不负责物理仿真，也不计算控制，只负责把 ROS 2 中的模型、坐标系、点云、图像、路径等数据展示出来。

在 Unitree Go2 调试中，RViz 的核心任务是确认：

- Go2 的 URDF 是否能被读取；
- TF 坐标树是否正常；
- Mesh 外观模型是否能加载；
- 拖动关节滑块后模型是否联动。

## 2. Go2 的最小 RViz 联调链路

```text
go2_description robot.launch.py
  -> robot_state_publisher
      -> /robot_description
      -> /tf
      -> /tf_static

joint_state_publisher_gui
  -> /joint_states

RViz RobotModel
  -> 读取 /robot_description + TF + Mesh
```

## 3. RViz 中需要关注的设置

| 设置 | Go2 调试中的建议 |
|---|---|
| Fixed Frame | 先设为 `base_link` |
| Displays | 添加 `RobotModel` 和 `TF` |
| RobotModel Description Source | 优先使用 `Topic` |
| RobotModel Description Topic | `/robot_description` |
| Status | 黄色/红色警告是排查入口 |

如果 `Topic` 方式无法显示模型，可以临时把 RobotModel 的 `Description Source` 改成 `File`，手动选择展开后的 URDF 文件，例如：

```text
/root/unitree_ws/install/go2_description/share/go2_description/urdf/go2_description.urdf
```

## 4. “只有骨架，没有肉身”的排查

这个问题很适合用来理解 RViz 的显示机制。

| 检查项 | 命令或动作 | 目标 |
|---|---|---|
| Mesh 是否存在 | `ls /root/unitree_ws/install/go2_description/share/go2_description/meshes` | 确认 `.dae` 或 `.stl` 文件已安装 |
| 工作空间是否 source | `source /root/unitree_ws/install/setup.bash` | 让 ROS 能解析 `package://go2_description/...` |
| RobotModel 状态 | 查看 RViz 左侧 Status | 找到具体丢失的文件或 frame |
| URDF 是否可读 | 手动选择 URDF 文件 | 区分是 topic 问题还是资源路径问题 |
| TF 是否正常 | 添加 TF display | 确认坐标树是否存在 |

判断逻辑：

```text
坐标轴会动，但 Mesh 不显示
  -> TF 链路大概率正常
  -> 优先查 Mesh 安装、路径和 RViz RobotModel
```

## 5. RViz 和 Gazebo 的分工

| 工具 | 主要用途 | 是否有物理仿真 |
|---|---|---|
| RViz | 看 ROS 数据、TF、模型、传感器、路径 | 否 |
| Gazebo | 做动力学仿真、碰撞、接触、传感器仿真 | 是 |

学习顺序建议：先在 RViz 中确认模型和 TF 正常，再进入 Gazebo 做动力学和控制实验。

## 6. 相关项目记录

- [[06-四足机器人/14-Unitree-Go2-ROS2项目笔记]]
- [[100-robot-learning-notes/UnitreeGo2调试记录]]
