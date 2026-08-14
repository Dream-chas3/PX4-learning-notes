---
tags:
  - ROS
  - MAVROS
  - PX4
  - Gazebo
  - QGC
  - Offboard
  - 节点开发
  - 源码学习
date: 2026-08-13
---

# ROS 节点开发实战：MAVROS 控制 PX4 无人机

> 来源：自写 ROS 节点控制无人机方向总结 | ROS1 + MAVROS + PX4(v1.14) + Gazebo SITL + QGC

---

## 一、总体架构：一个 ROS 节点如何"驱动"整架无人机

先建立全局认知。你写的 ROS 节点**并不直接控制电机**，它只是通过 MAVROS 这座"桥"，把位置指令翻译成 MAVLink 消息，发给 PX4 飞控，由 PX4 内部完成控制解算。

```
┌──────────────────────────────────────────────────────────────────────┐
│                        桌面计算机 (Ubuntu)                            │
│                                                                      │
│   ┌───────────────┐   ROS 话题/服务    ┌──────────────┐              │
│   │  你的 ROS 节点  │ ◄────────────────►│    MAVROS    │              │
│   │  (offboard.py) │  setpoint/state   │   (桥接节点)  │              │
│   └───────────────┘                    └──────┬───────┘              │
│                                               │ MAVLink (UDP/TCP)    │
│  ┌────────────────────────────┐               │                      │
│  │ Gazebo (SITL 仿真环境)      │               │                      │
│  │  ┌──────────────────────┐  │               │                      │
│  │  │  PX4 飞控 (软件在环)   │◄─┘               │                      │
│  │  │  内部: EKF2→位置控制   │                  │                      │
│  │  │  →姿态→速率→混控→电机  │                  │                      │
│  │  └──────────────────────┘                  │                      │
│  └────────────────────────────┘               │                      │
│                                               ▼                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  QGC 地面站 (可同时连接, 只读监视 + 手动接管)              │       │
│  └──────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

**关键结论（务必记住）：**

| 概念 | 说明 |
|------|------|
| 你的节点发什么 | `setpoint`（期望位置/速度/姿态），不是电机 PWM |
| PX4 做什么 | 根据 setpoint 做位置→姿态→速率的级联控制，输出到电机 |
| MAVROS 是什么 | 一个常驻的 ROS 节点，负责 ROS 话题 ↔ MAVLink 消息互转 |
| QGC 的角色 | 地面站，可以**同时**连进来监视/接管，不冲突 |

---

## 二、环境准备

### 2.1 三个组件的作用与安装

| 组件 | 作用 | 安装/启动 |
|------|------|-----------|
| **MAVROS** | ROS ↔ MAVLink 桥 | `sudo apt install ros-noetic-mavros ros-noetic-mavros-extras` |
| **PX4 SITL** | 软件在环仿真（飞控跑在电脑上） | 源码在 `~/workspace/PX4-Autopilot`，用 `make px4_sitl gazebo-classic` 启动 |
| **QGC** | 地面站监视/接管 | 官网下载 AppImage，直接运行 |

### 2.2 安装 MAVROS 及 GeographicLib 数据集

```bash
sudo apt install ros-noetic-mavros ros-noetic-mavros-extras

# 关键！安装地理数据（否则 mavros 起不来，报 missing dataset）
wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
sudo bash ./install_geographiclib_datasets.sh
```

### 2.3 启动仿真环境（顺序很重要）

```bash
# 终端 1：启动 PX4 SITL + Gazebo（默认四旋翼 iris）
cd ~/workspace/PX4-Autopilot
make px4_sitl gazebo-classic

# 终端 2：启动 MAVROS（连接到 SITL 的默认 UDP 端口）
roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"

# 终端 3（可选）：启动 QGC，连 UDP 14550
./QGroundControl.AppImage
```

**端口速记：**

| 端口 | 用途 |
|------|------|
| 14540 | MAVROS → PX4 的 UDP |
| 14550 | QGC → PX4 的 UDP（默认） |
| 14557 | PX4 的 onboard 端口 |

---

## 三、MAVROS 核心话题与服务速查

这是写节点的"API 手册"，先收藏这张表。

### 3.1 你主要"订阅"的话题（读状态）

| ROS 话题 | 消息类型 | 内容 |
|----------|----------|------|
| `/mavros/state` | `mavros_msgs/State` | 连接状态、armed、mode（**必须订阅**） |
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | 本地坐标系位置（ENU） |
| `/mavros/local_position/velocity_local` | `geometry_msgs/TwistStamped` | 本地速度 |
| `/mavros/global_position/global` | `sensor_msgs/NavSatFix` | 经纬度（GPS） |
| `/mavros/imu/data` | `sensor_msgs/Imu` | 姿态/角速度 |
| `/mavros/rc/in` | `mavros_msgs/RCIn` | 遥控器输入 |
| `/mavros/battery` | `sensor_msgs/BatteryState` | 电池 |

### 3.2 你主要"发布"的话题（发指令）

| ROS 话题 | 消息类型 | 内容 |
|----------|----------|------|
| `/mavros/setpoint_position/local` | `geometry_msgs/PoseStamped` | 期望位置 setpoint |
| `/mavros/setpoint_raw/local` | `mavros_msgs/PositionTarget` | 原始 setpoint（可指定速度/加速度） |
| `/mavros/setpoint_velocity/cmd_vel` | `geometry_msgs/TwistStamped` | 期望速度 |

### 3.3 你主要"调用"的服务（动作类）

| ROS 服务 | 类型 | 作用 |
|----------|------|------|
| `/mavros/cmd/arming` | `mavros_msgs/CommandBool` | 解锁/上锁 |
| `/mavros/set_mode` | `mavros_msgs/SetMode` | 切换飞行模式（如 OFFBOARD） |
| `/mavros/cmd/takeoff` | `mavros_msgs/CommandTOL` | 起飞 |
| `/mavros/cmd/land` | `mavros_msgs/CommandTOL` | 降落 |
| `/mavros/param/set` | `mavros_msgs/ParamSet` | 设置参数 |
| `/mavros/param/get` | `mavros_msgs/ParamGet` | 读取参数 |

---

## 四、Offboard 控制的完整流程（核心逻辑）

写节点的本质是掌握下面这个状态机：

```
        ┌─────────────┐
        │  1. 订阅状态  │◄─────── 持续读 /mavros/state
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ 2. 预发 setpoint│◄──── 先发几秒 setpoint（关键！）
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ 3. 切 OFFBOARD│◄──── 调用 /mavros/set_mode
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │   4. 解锁     │◄──── 调用 /mavros/cmd/arming
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ 5. 持续发 setpoint│◄── 高频循环 (≥2Hz，建议 20Hz)
        └─────────────┘
```

> ⚠️ **两个高频踩坑点：**
> 1. **必须先持续发 setpoint，再切 OFFBOARD 模式**——PX4 在 OFFBOARD 模式下若持续一段时间（约 500ms）收不到 setpoint 会自动退出 OFFBOARD，导致无法解锁。
> 2. **setpoint 频率必须 ≥ 2Hz**，否则同样触发"失去 offboard 心跳"。

---

## 五、示例 1：Python 节点（完整可运行，起飞悬停）

把下面代码存为 `offb_node.py`，放进你的功能包 `scripts/` 目录并 `chmod +x`。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
offb_node.py —— 最简单的 Offboard 控制节点
功能：解锁 -> 切 OFFBOARD -> 飞到 (0,0,2) 悬停
"""
import rospy
from geometry_msgs.msg import PoseStamped
from mavros_msgs.msg import State
from mavros_msgs.srv import CommandBool, SetMode

# 全局状态
current_state = State()

def state_cb(msg):
    """回调：持续刷新飞控状态"""
    global current_state
    current_state = msg

if __name__ == "__main__":
    rospy.init_node("offb_node", anonymous=True)
    rate = rospy.Rate(20.0)  # 20Hz，远高于 2Hz 下限

    # 1. 订阅状态
    rospy.Subscriber("/mavros/state", State, state_cb)

    # 2. 发布 setpoint
    local_pos_pub = rospy.Publisher(
        "/mavros/setpoint_position/local", PoseStamped, queue_size=10)

    # 3. 服务客户端
    arming_client  = rospy.ServiceProxy("/mavros/cmd/arming", CommandBool)
    set_mode_client = rospy.ServiceProxy("/mavros/set_mode", SetMode)

    # 等待 MAVROS 与 PX4 建立连接
    rospy.loginfo("等待 MAVROS 连接...")
    while not rospy.is_shutdown() and not current_state.connected:
        rate.sleep()
    rospy.loginfo("已连接！")

    # 构造目标点：(0, 0, 2) 米（ENU 本地坐标）
    pose = PoseStamped()
    pose.pose.position.x = 0.0
    pose.pose.position.y = 0.0
    pose.pose.position.z = 2.0

    # 关键：先持续发 setpoint，让 PX4 "记住"有 offboard 指令
    for _ in range(100):
        local_pos_pub.publish(pose)
        rate.sleep()

    last_request = rospy.Time.now()

    while not rospy.is_shutdown():
        # 4. 切 OFFBOARD 模式（间隔 5s 防重复请求）
        if current_state.mode != "OFFBOARD" and \
                (rospy.Time.now() - last_request) > rospy.Duration(5.0):
            if set_mode_client(0, "OFFBOARD").mode_sent:
                rospy.loginfo("已切到 OFFBOARD 模式")
            last_request = rospy.Time.now()

        # 5. 解锁（必须先切模式再解锁）
        elif not current_state.armed and \
                (rospy.Time.now() - last_request) > rospy.Duration(5.0):
            if arming_client(True).success:
                rospy.loginfo("已解锁")
            last_request = rospy.Time.now()

        # 6. 持续发布 setpoint（整个循环都在发）
        local_pos_pub.publish(pose)
        rate.sleep()
```

**运行方式：**

```bash
# 先启动仿真 + MAVROS（见 2.3），然后：
rosrun 你的功能包名 offb_node.py
```

---

## 六、示例 2：让无人机"动起来"——按路径飞行

悬停没意思，改成飞一个正方形航点。只改 setpoint 部分即可：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""路径飞行：起飞 -> 依次飞 4 个角 -> 回到原点"""
import rospy, math
from geometry_msgs.msg import PoseStamped
from mavros_msgs.msg import State
from mavros_msgs.srv import CommandBool, SetMode

current_state = State()

def state_cb(msg):
    global current_state
    current_state = msg

rospy.init_node("path_node", anonymous=True)
rate = rospy.Rate(20.0)
rospy.Subscriber("/mavros/state", State, state_cb)
pub = rospy.Publisher("/mavros/setpoint_position/local", PoseStamped, queue_size=10)
arm = rospy.ServiceProxy("/mavros/cmd/arming", CommandBool)
set_mode = rospy.ServiceProxy("/mavros/set_mode", SetMode)

while not rospy.is_shutdown() and not current_state.connected:
    rate.sleep()

def send_pose(x, y, z):
    p = PoseStamped()
    p.pose.position.x = x
    p.pose.position.y = y
    p.pose.position.z = z
    return p

# 预发 setpoint
for _ in range(100):
    pub.publish(send_pose(0, 0, 2)); rate.sleep()

last = rospy.Time.now()
while not rospy.is_shutdown():
    if current_state.mode != "OFFBOARD" and (rospy.Time.now()-last) > rospy.Duration(5.0):
        set_mode(0, "OFFBOARD"); last = rospy.Time.now()
    elif not current_state.armed and (rospy.Time.now()-last) > rospy.Duration(5.0):
        arm(True); last = rospy.Time.now()
    pub.publish(send_pose(0, 0, 2)); rate.sleep()

# 执行正方形路径（2x2 米，边长 2m）
waypoints = [(2, 0, 2), (2, 2, 2), (0, 2, 2), (0, 0, 2)]
for (x, y, z) in waypoints:
    t0 = rospy.Time.now()
    while (rospy.Time.now() - t0) < rospy.Duration(5.0):  # 每个点停 5 秒
        pub.publish(send_pose(x, y, z))
        rate.sleep()
rospy.loginfo("路径完成！")
```

---

## 七、示例 3：用 `setpoint_raw` 做速度控制

位置 setpoint 需要 PX4 做位置控制；若你只想发**速度**（比如手柄式控制），用 `setpoint_raw`：

```python
from mavros_msgs.msg import PositionTarget

# 速度控制 setpoint：指定用速度坐标系，只控制 vx/vy/vz
raw = PositionTarget()
raw.coordinate_frame = PositionTarget.FRAME_LOCAL_NED   # 本地 NED 坐标
raw.type_mask = PositionTarget.IGNORE_PX | PositionTarget.IGNORE_PY | \
                PositionTarget.IGNORE_PZ |             # 忽略位置
                PositionTarget.IGNORE_AFX | PositionTarget.IGNORE_AFY | \
                PositionTarget.IGNORE_AFZ | PositionTarget.IGNORE_YAW_RATE
raw.velocity.x = 1.0   # 北向 1 m/s
raw.velocity.y = 0.0
raw.velocity.z = 0.0

# 发布到 raw 话题
raw_pub = rospy.Publisher("/mavros/setpoint_raw/local", PositionTarget, queue_size=10)
```

> `type_mask` 用于"忽略"你不关心的量。位置 setpoint 与速度 setpoint 的切换全靠它，写错会导致飞机行为异常。

---

## 八、QGC 集成与联合调试

QGC 可以和你自己的节点**同时**连到同一架仿真飞机，互不冲突：

| 场景 | 做法 |
|------|------|
| 监视 | QGC 上实时看飞机位置、姿态、模式、日志 |
| 手动接管 | 在 QGC 里切到"手动/位置模式"，你的 offboard 指令即被覆盖 |
| 对飞行模式 | 在 QGC 参数页确认 `COM_RC_IN_MODE`、`EKF2_*` 等关键参数 |

**推荐的调试组合拳：**

```
① 你自己的节点发 setpoint（看代码日志）
② MAVROS 转成 MAVLink（rostopic echo /mavlink/to 看发了啥）
③ QGC 上观察飞机实际动作（最直观）
④ PX4 内部日志（.ulog）事后分析
```

用 `rostopic echo` 快速验证数据流：

```bash
rostopic echo /mavros/state            # 看连接/模式/解锁状态
rostopic echo /mavros/local_position/pose   # 看飞机当前位置
rostopic hz /mavros/setpoint_position/local  # 看你的 setpoint 频率够不够
```

---

## 九、话题 ↔ MAVLink ↔ uORB 对照表（进阶理解）

理解这一层，你才能"看穿"整条链路：

| 你发的 ROS 话题 | MAVLink 消息 | PX4 内部 uORB 主题 |
|-----------------|--------------|-------------------|
| `/mavros/setpoint_position/local` | `SET_POSITION_TARGET_LOCAL_NED` | `position_setpoint_triplet` |
| `/mavros/setpoint_raw/local` | `SET_POSITION_TARGET_LOCAL_NED` | `position_setpoint_triplet` |
| `/mavros/setpoint_velocity/cmd_vel` | `SET_POSITION_TARGET_LOCAL_NED` | `position_setpoint_triplet` |
| `/mavros/cmd/arming` | `COMMAND_LONG (MAV_CMD_COMPONENT_ARM_DISARM)` | `vehicle_command` |
| `/mavros/set_mode` | `SET_MODE` | `vehicle_command` |
| `/mavros/local_position/pose`（回读） | `LOCAL_POSITION_NED` | `vehicle_local_position` |
| `/mavros/state`（回读） | `HEARTBEAT` / `STATUSTEXT` | `vehicle_status` |

> 这张表是连接"ROS 世界"和"PX4 世界"的钥匙。后面读 PX4 源码时，你会反复遇到右列的 uORB 主题。

---

## 十、常见坑与排查清单

| 现象 | 原因 | 解决 |
|------|------|------|
| 切不了 OFFBOARD | 没先持续发 setpoint | 先循环发 setpoint 再切模式 |
| 解锁失败 | 参数 `COM_RCL_EXCEPT` 未设、或未满足条件 | 检查 `COM_ARM_WO_GPS` 等参数 |
| 飞机起飞后立刻掉高度 | setpoint 频率 < 2Hz | 提高循环频率到 20Hz |
| MAVROS 起不来报 dataset 错误 | 没装 GeographicLib 数据 | 跑 `install_geographiclib_datasets.sh` |
| QGC 连不上 | 端口错 | QGC 连 `udp://:14550` |
| `rostopic echo` 无数据 | MAVROS 没连上 PX4 | 检查 `fcu_url` 端口 |

---

## 十一、下一步进阶路线

1. **读 PX4 控制链源码**（见配套笔记《PX4 源码导读与学习路线图》）——理解 setpoint 在飞控内部如何变成电机 PWM。
2. **改用 C++ 写节点**——大项目用 C++ 更规范，`ros::Rate` + 类封装。
3. **接入视觉**——用 `mavros/setpoint_velocity/cmd_vel` 发视觉解算的速度。
4. **多机集群**——每架飞机一个 MAVROS 实例 + 不同 `target_system`。
