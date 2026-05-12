# URDF / SDF / MJCF 模型文件

## 1. 一句话理解

机器人模型文件用来描述机器人“由哪些刚体组成、这些刚体如何连接、外观是什么、物理属性是什么”。不同仿真和机器人生态会使用不同格式。

| 格式 | 常见生态 | 适合做什么 |
|---|---|---|
| URDF | ROS / RViz / robot_state_publisher | 描述机器人结构、link、joint、visual、collision |
| Xacro | ROS | 用宏和参数生成 URDF，减少重复 |
| SDF | Gazebo | 描述仿真世界、机器人、传感器、插件和物理属性 |
| MJCF | MuJoCo | 描述 MuJoCo 中的机器人、关节、执行器和仿真参数 |

## 2. Go2 项目中的模型文件

在 `go2_description` 中可以看到：

| 文件/目录 | 作用 |
|---|---|
| `urdf/go2_description.urdf` | 展开后的 Go2 URDF 模型 |
| `urdf/go2_description.urdf.xacro` | 可参数化生成 URDF 的 Xacro |
| `urdf/leg.urdf.xacro` | 腿部结构描述 |
| `urdf/transmission.urdf.xacro` | 控制和传动相关描述 |
| `xacro/robot.xacro` | 机器人主体结构 |
| `xacro/leg.xacro` | 腿部模块化结构 |
| `meshes/*.dae` | RViz 中显示的外观模型 |

## 3. Link、Joint、Mesh 的关系

| 概念 | 含义 | Go2 中的直觉例子 |
|---|---|---|
| Link | 一个刚体部件 | 躯干、大腿、小腿、足端 |
| Joint | 两个 link 之间的连接和运动关系 | 髋关节、膝关节 |
| Mesh | 挂在 link 上的外观模型 | `trunk.dae`、`thigh.dae`、`calf.dae` |
| TF | link 坐标系之间的实时变换 | 拖动关节后腿部坐标系变化 |

可以把 URDF 理解成机器人骨架图，把 Mesh 理解成外观皮肤，把 TF 理解成骨架当前姿态。

## 4. `package://` 路径

URDF 中常见 Mesh 路径形式：

```text
package://go2_description/meshes/trunk.dae
```

这不是普通文件路径，而是 ROS 包资源路径。它要求当前终端已经加载工作空间：

```bash
source /opt/ros/humble/setup.bash
source /root/unitree_ws/install/setup.bash
```

如果没有 source，RViz 可能找不到 `go2_description` 包，于是出现“有骨架、没有外观”的现象。

## 5. 学习 Go2 模型时的阅读顺序

```text
package.xml
  -> CMakeLists.txt
      -> launch/robot.launch.py
          -> urdf/go2_description.urdf.xacro
              -> xacro/robot.xacro
                  -> xacro/leg.xacro
                      -> meshes/*.dae
```

阅读目标不是一次看懂所有标签，而是先建立三件事：

- 机器人有哪些 link；
- link 之间由哪些 joint 连接；
- 每个 link 的 visual mesh 从哪里加载。

## 6. 相关项目记录

- [[06-四足机器人/14-Unitree-Go2-ROS2项目笔记]]
- [[02-ROS2与机器人软件框架/05-RViz可视化]]
