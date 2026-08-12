---
tags:
  - ROS
  - catkin
  - 功能包
  - CMakeLists
  - 多节点
date: 2026-08-12
---

# ROS 功能包与多节点开发

> 来源：ChatGPT 对话补充整理
> 主题：如何在 catkin workspace 中添加新的 cpp 文件、创建功能包、CMakeLists 配置、编译运行

---

## 一、核心问题

> ❓ **我的疑问**：当我想写别的程序的时候，操作还是一样的吗？还是得再创建一个新的文件夹，还是说创一个 cpp 文件就可以了？然后 rosrun 的时候运行新的节点？我还需要进行别的操作吗？就是 CMake？

**结论：在同一个功能包里，只需要添加新的 cpp 文件 + 修改 CMakeLists.txt，不需要新建包。**

---

## 二、两种扩展方式

### 方式 A：在已有包里添加新节点（推荐）

适用于功能相近的程序。

**操作步骤：**

**1. 写新的 cpp 文件**

```
~/catkin_ws/src/offboard_ctrl/src/
├── offboard.cpp        ← 已有的
└── trajectory.cpp      ← 新增的
```

**2. 修改 CMakeLists.txt**

在 `CMakeLists.txt` 中添加：

```cmake
# 新增可执行文件
add_executable(trajectory_node
  src/trajectory.cpp
)

# 链接库
target_link_libraries(trajectory_node
  ${catkin_LIBRARIES}
)
```

**3. 编译**

```bash
cd ~/catkin_ws
catkin_make
```

**4. 运行**

```bash
rosrun offboard_ctrl trajectory_node
```

### 方式 B：创建新功能包

适用于功能完全不同的程序。

```bash
cd ~/catkin_ws/src
catkin_create_pkg <包名> <依赖1> <依赖2> ...
```

---

## 三、`catkin_create_pkg` 命令详解

> ❓ **我的疑问**：`catkin_create_pkg offboard_ctrl roscpp rospy geometry_msgs mavros_msgs std_msgs tf` 有什么讲究吗？执行这个语句会有什么变化？

### 3.1 命令格式

```bash
catkin_create_pkg <功能包名> <依赖列表>
```

每个依赖的含义：

| 依赖 | 类型 | 作用 |
|------|------|------|
| `roscpp` | 核心库 | C++ ROS 接口（NodeHandle、Publisher、Subscriber） |
| `rospy` | 核心库 | Python ROS 接口（如果你用 Python 写节点） |
| `geometry_msgs` | 消息包 | 位置/姿态/速度等消息类型（如 `PoseStamped`、`Twist`） |
| `mavros_msgs` | 消息包 | MAVROS 专用消息（如 `State`、`CommandBool`） |
| `std_msgs` | 消息包 | 标准消息类型（如 `Header`、`String`） |
| `tf` | 库 | 坐标变换库（坐标系之间的转换） |

### 3.2 执行后会发生什么

```bash
catkin_create_pkg offboard_ctrl roscpp rospy geometry_msgs mavros_msgs std_msgs tf
```

自动生成：

```
~/catkin_ws/src/offboard_ctrl/
├── CMakeLists.txt        ← 自动生成（已写好依赖声明）
├── package.xml           ← 自动生成（包信息和依赖）
├── include/              ← 头文件目录（空）
└── src/                  ← 源码目录（空）
```

关键生成内容：

**package.xml** — 声明依赖：
```xml
<depend>roscpp</depend>
<depend>rospy</depend>
<depend>geometry_msgs</depend>
<depend>mavros_msgs</depend>
<depend>std_msgs</depend>
<depend>tf</depend>
```

**CMakeLists.txt** — 声明构建规则：
```cmake
find_package(catkin REQUIRED COMPONENTS
  roscpp
  rospy
  geometry_msgs
  mavros_msgs
  std_msgs
  tf
)
```

### 3.3 依赖写多了怎么办？

写了不用的依赖（如 `rospy` 和 `tf`）**不会报错**，只是会让编译时多检查几个包。但这是一种工程上的不规范。

**最佳实践**：只写实际用到的依赖。

---

## 四、添加节点后的完整编译运行流程

### 4.1 目录结构示例

```
~/catkin_ws/src/
├── fcu_core/              ← 官方包（不改它）
│   └── ...
│
└── offboard_ctrl/         ← 你自己的包
    ├── CMakeLists.txt
    ├── package.xml
    └── src/
        ├── offboard.cpp       ← 定点飞行
        └── trajectory.cpp     ← 新加的轨迹规划
```

### 4.2 CMakeLists.txt 完整写法

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(offboard_ctrl)

find_package(catkin REQUIRED COMPONENTS
  roscpp
  geometry_msgs
  mavros_msgs
  std_msgs
)

catkin_package()

include_directories(
  ${catkin_INCLUDE_DIRS}
)

# 节点1：定点飞行
add_executable(offboard_node
  src/offboard.cpp
)
target_link_libraries(offboard_node
  ${catkin_LIBRARIES}
)

# 节点2：轨迹规划（新增）
add_executable(trajectory_node
  src/trajectory.cpp
)
target_link_libraries(trajectory_node
  ${catkin_LIBRARIES}
)

# 节点3：更多...
```

### 4.3 完整操作流程

```bash
# 1. 写代码
vim ~/catkin_ws/src/offboard_ctrl/src/trajectory.cpp

# 2. 修改 CMakeLists.txt（添加 add_executable + target_link_libraries）

# 3. 编译
cd ~/catkin_ws
catkin_make

# 4. source 环境
source ~/catkin_ws/devel/setup.bash

# 5. 运行
rosrun offboard_ctrl trajectory_node
```

---

## 五、工程习惯建议

### 不要直接改官方包

```
~/catkin_ws/src/
├── fcu_core/              ← 官方包，保持原样
└── my_offboard/           ← 你的包，随便改
```

原因：以后更新官方代码不会覆盖你自己的程序。

> 这和 PX4 开发的原则一样：不要在 `src/modules/` 里乱改，最好在 `src/examples/` 下加自己的模块。

### 功能包命名建议

| 包名 | 用途 |
|------|------|
| `offboard_ctrl` | Offboard 控制相关（起飞、降落、位置控制） |
| `trajectory_planner` | 轨迹规划相关 |
| `vision_detector` | 视觉检测相关 |

---

## 六、扩展知识

### 6.1 `rosrun` vs `roslaunch`

| 命令 | 用途 | 示例 |
|------|------|------|
| `rosrun <包名> <节点名>` | 运行单个节点 | `rosrun offboard_ctrl offboard_node` |
| `roslaunch <包名> <launch文件>` | 启动多个节点 + 参数配置 | `roslaunch mavros px4.launch` |

### 6.2 `catkin_make` 做了什么

```
catkin_make
  ├── 1. 扫描 src/ 下所有包的 CMakeLists.txt
  ├── 2. 生成 Makefile
  ├── 3. 编译所有 cpp → .o 目标文件
  ├── 4. 链接 → 可执行文件
  └── 5. 输出到 devel/lib/<包名>/
```

编译后你的可执行文件在：

```
~/catkin_ws/devel/lib/offboard_ctrl/
├── offboard_node     ← rosrun 实际运行的就是它
└── trajectory_node
```

### 6.3 节点名 vs 可执行文件名

```cmake
add_executable(trajectory_node src/trajectory.cpp)
#              ↑ 这是可执行文件名
```

```bash
rosrun offboard_ctrl trajectory_node
#                       ↑ 必须和 CMakeLists 里 add_executable 的第一个参数一致
```

但节点在 ROS 网络中的名字可以不同（通过 `ros::init` 设置）：

```cpp
ros::init(argc, argv, "my_trajectory_controller");  // 节点名
// 可执行文件是 trajectory_node，但节点在网络中叫 my_trajectory_controller
```
