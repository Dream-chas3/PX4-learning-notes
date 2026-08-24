# ROS 与 PX4 通信实现定点飞行

> 来源：《23-ROS与PX4通信在Gazebo实现定点飞行》 | ROS1 + MAVROS + Gazebo SITL

---

## 一、架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                      桌面计算机 (Ubuntu)                     │
│                                                             │
│  ┌──────────┐    ROS 话题/服务     ┌──────────┐             │
│  │ ROS 节点  │ ◄────────────────► │  MAVROS   │             │
│  │ offboard  │   geometry_msgs    │  (桥接)   │             │
│  │ _ctrl     │                    │           │             │
│  └──────────┘                     └─────┬─────┘             │
│                                         │                   │
│                                  MAVLink (UDP)              │
│                                         │                   │
│  ┌──────────────────────────────────────┼──────────────────┤
│  │                       Gazebo 仿真环境  │                  │
│  │  ┌──────────────────────────────────┐ │                  │
│  │  │           PX4 SITL               │◄┘                  │
│  │  │   uORB 消息总线                   │                    │
│  │  │   Commander → Navigator → Control │                    │
│  │  └──────────────┬───────────────────┘                    │
│  │                 ▼                                        │
│  │  ┌──────────────────────────────────┐                    │
│  │  │      Gazebo 无人机模型            │                    │
│  │  └──────────────────────────────────┘                    │
│  └──────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────┘
```

**数据流**：ROS 节点 → MAVROS → MAVLink(UDP) → PX4 uORB → Gazebo → 传感器数据原路返回

---

## 二、MAVROS 核心概念

### 2.1 MAVROS 是什么

| 角色 | 说明 |
|------|------|
| **协议转换** | ROS 消息 ↔ MAVLink 帧，双向翻译 |
| **运行方式** | 作为一个 ROS 节点运行 |
| **物理链路** | 串口（真实飞控）或 UDP（仿真） |

### 2.2 MAVROS 三种通信模式

```
MAVROS 通信方式
├── 话题（Topic）── 异步、实时、持续推送
│   ├── mavros/imu/data          → IMU数据
│   ├── mavros/local_position/pose → 本地位置
│   ├── mavros/state             → 飞控状态
│   └── mavros/setpoint_position/local → 目标位置指令
│
├── 服务（Service）── 同步、请求-响应
│   ├── mavros/cmd/arming        → 解锁/上锁
│   └── mavros/set_mode          → 切换飞行模式
│
└── 与 uORB 的关系
    MAVROS 通过 MAVLink 间接订阅/发布 PX4 内部 uORB 主题
```

### 2.3 常用消息速查

#### 订阅消息（PX4 → ROS）

| 话题 | 消息类型 | 获取内容 |
|------|----------|----------|
| `mavros/state` | `mavros_msgs::State` | 连接状态、飞行模式、解锁状态 |
| `mavros/local_position/pose` | `geometry_msgs::PoseStamped` | 本地 NED 坐标（x北/y东/z下） |
| `mavros/global_position/global` | `sensor_msgs::NavSatFix` | GPS 经纬高 |
| `mavros/imu/data` | `sensor_msgs::Imu` | 四元数姿态 + 角速度 + 加速度 |
| `mavros/manual_control/control` | `mavros_msgs::ManualControl` | 遥控器摇杆值 |

#### 发布消息（ROS → PX4）

| 话题 | 消息类型 | 控制内容 |
|------|----------|----------|
| `mavros/setpoint_position/local` | `geometry_msgs::PoseStamped` | NED 坐标系下期望位置 |
| `mavros/setpoint_velocity/cmd_vel` | `geometry_msgs::TwistStamped` | 三轴线速度 + 角速度 |
| `mavros/setpoint_attitude/attitude` | `geometry_msgs::PoseStamped` | 期望姿态（欧拉角/四元数） |
| `mavros/setpoint_position/global` | `mavros_msgs::GlobalPositionTarget` | GPS 坐标系期望位置 |

#### 服务调用

| 服务 | 消息类型 | 功能 |
|------|----------|------|
| `mavros/cmd/arming` | `mavros_msgs::CommandBool` | 解锁(request.value=true) / 上锁(false) |
| `mavros/set_mode` | `mavros_msgs::SetMode` | 切换飞行模式(request.custom_mode="OFFBOARD") |

---

## 三、Offboard 控制节点完整流程

### 3.1 六步流程

```
┌─────────────────────────────────────────────────────┐
│                  offboard_ctrl_node 主流程            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Step 1] 等待 FCU 连接                              │
│     while (!current_state_.connected) { wait; }     │
│         │                                           │
│         ▼                                           │
│  [Step 2] 连续发布初始 setpoint（2秒，20Hz）          │
│     ⚠️ PX4 安全要求：必须先发 setpoint 才能切 OFFBOARD │
│         │                                           │
│         ▼                                           │
│  [Step 3] 循环切换 OFFBOARD + 解锁（带重试机制）       │
│     while (!armed || mode != "OFFBOARD") {          │
│         每5秒重试 set_mode + arming                  │
│         同时持续发布 setpoint                         │
│     }                                               │
│         │                                           │
│         ▼                                           │
│  [Step 4] 航点导航主循环                             │
│     for each waypoint:                              │
│       发布目标坐标                                    │
│       计算当前位置与目标距离                           │
│       距离 < 0.5m → 到达 → 悬停0.8s → 下一个         │
│         │                                           │
│         ▼                                           │
│  [Step 5] 所有航点完成 → 保持最终位置 5 秒            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 核心代码（精简注释版）

```cpp
#include <ros/ros.h>
#include <geometry_msgs/PoseStamped.h>
#include <mavros_msgs/State.h>
#include <mavros_msgs/SetMode.h>
#include <mavros_msgs/CommandBool.h>
#include <array>
#include <vector>
#include <cmath>

class OffboardCtrl {
public:
    OffboardCtrl(ros::NodeHandle &nh)
        : nh_(nh), current_pose_received_(false), waypoint_index_(0)
    {
        // 可配置参数
        nh_.param("rate", rate_, 20.0);
        nh_.param("arrival_threshold", arrival_threshold_, 0.5);
        nh_.param("offboard_timeout", offboard_timeout_, 5.0);
        nh_.param("arm_timeout", arm_timeout_, 5.0);
        nh_.param("stabilize_time", stabilize_time_, 0.8);

        // 订阅者
        state_sub_  = nh_.subscribe<mavros_msgs::State>("mavros/state", 10, &OffboardCtrl::stateCb, this);
        pose_sub_   = nh_.subscribe<geometry_msgs::PoseStamped>("mavros/local_position/pose", 10, &OffboardCtrl::poseCb, this);

        // 发布者
        setpoint_pub_ = nh_.advertise<geometry_msgs::PoseStamped>("mavros/setpoint_position/local", 10);

        // 服务客户端
        arming_client_    = nh_.serviceClient<mavros_msgs::CommandBool>("mavros/cmd/arming");
        set_mode_client_  = nh_.serviceClient<mavros_msgs::SetMode>("mavros/set_mode");

        // 预设航点 {x, y, z}（NED 坐标，单位：米）
        waypoints_.push_back({1.0, 1.0, 1.0});
        waypoints_.push_back({2.0, 1.0, 2.0});
        waypoints_.push_back({2.0, 3.0, 1.0});
        waypoints_.push_back({5.0, 5.0, 1.0});
    }

    void run() {
        ros::Rate r(rate_);

        // ── Step 1: 等待飞控连接 ──
        ROS_INFO("Waiting for FCU connection...");
        while (ros::ok() && !current_state_.connected) {
            ros::spinOnce(); r.sleep();
        }
        ROS_INFO("FCU connected!");

        // ── Step 2: 发布初始 setpoint 2 秒（PX4 前置要求）──
        geometry_msgs::PoseStamped sp;
        if (current_pose_received_) sp = current_pose_;
        else { sp.pose.position.z = 1.0; }
        for (int i = 0; i < int(rate_ * 2.0) && ros::ok(); ++i) {
            sp.header.stamp = ros::Time::now();
            setpoint_pub_.publish(sp);
            ros::spinOnce(); r.sleep();
        }

        // ── Step 3: 切换 OFFBOARD + 解锁 ──
        mavros_msgs::SetMode offb_set_mode;
        offb_set_mode.request.custom_mode = "OFFBOARD";
        mavros_msgs::CommandBool arm_cmd;
        arm_cmd.request.value = true;
        ros::Time last_request = ros::Time::now();

        while (ros::ok() && !(current_state_.armed && current_state_.mode == "OFFBOARD")) {
            // 每 5 秒重试模式切换
            if (current_state_.mode != "OFFBOARD"
                && (ros::Time::now() - last_request).toSec() > offboard_timeout_) {
                set_mode_client_.call(offb_set_mode);
                last_request = ros::Time::now();
            }
            // 每 5 秒重试解锁
            else if (!current_state_.armed
                     && (ros::Time::now() - last_request).toSec() > arm_timeout_) {
                arming_client_.call(arm_cmd);
                last_request = ros::Time::now();
            }
            // 持续发布 setpoint（OFFBOARD 模式下必须不间断！）
            geometry_msgs::PoseStamped cur_sp = getActiveWaypointPose();
            cur_sp.header.stamp = ros::Time::now();
            setpoint_pub_.publish(cur_sp);
            ros::spinOnce(); r.sleep();
        }

        // ── Step 4: 航点导航 ──
        while (ros::ok() && waypoint_index_ < (int)waypoints_.size()) {
            geometry_msgs::PoseStamped target = getActiveWaypointPose();
            target.header.stamp = ros::Time::now();
            setpoint_pub_.publish(target);

            if (current_pose_received_) {
                double d = distance3(
                    current_pose_.pose.position.x,
                    current_pose_.pose.position.y,
                    current_pose_.pose.position.z,
                    target.pose.position.x,
                    target.pose.position.y,
                    target.pose.position.z
                );
                if (d <= arrival_threshold_) {
                    ROS_INFO("Reached waypoint %d! dist=%.3f m", waypoint_index_, d);
                    waypoint_index_++;
                    // 悬停 stabilize_time_ 秒
                    ros::Time t0 = ros::Time::now();
                    while (ros::ok() && (ros::Time::now() - t0).toSec() < stabilize_time_) {
                        setpoint_pub_.publish(getActiveWaypointPose());
                        ros::spinOnce(); r.sleep();
                    }
                }
            }
            ros::spinOnce(); r.sleep();
        }

        // ── Step 5: 最终位置保持 5 秒 ──
        for (int i = 0; i < int(rate_ * 5.0) && ros::ok(); ++i) {
            geometry_msgs::PoseStamped f = getActiveWaypointPose();
            f.header.stamp = ros::Time::now();
            setpoint_pub_.publish(f);
            ros::spinOnce(); r.sleep();
        }
        ROS_INFO("Mission completed!");
    }

private:
    // 回调函数
    void stateCb(const mavros_msgs::State::ConstPtr& msg) { current_state_ = *msg; }
    void poseCb(const geometry_msgs::PoseStamped::ConstPtr& msg) {
        current_pose_ = *msg;
        current_pose_received_ = true;
    }

    // 获取当前航点
    geometry_msgs::PoseStamped getActiveWaypointPose() {
        geometry_msgs::PoseStamped p;
        p.header.frame_id = "map";
        if (waypoint_index_ < (int)waypoints_.size()) {
            p.pose.position.x = waypoints_[waypoint_index_][0];
            p.pose.position.y = waypoints_[waypoint_index_][1];
            p.pose.position.z = waypoints_[waypoint_index_][2];
        } else {
            auto &w = waypoints_.back();
            p.pose.position.x = w[0];
            p.pose.position.y = w[1];
            p.pose.position.z = w[2];
        }
        p.pose.orientation.w = 1.0;
        return p;
    }

    // 3D 距离
    static double distance3(double x1, double y1, double z1,
                            double x2, double y2, double z2) {
        return std::sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2) + (z1-z2)*(z1-z2));
    }

    // 成员变量
    ros::NodeHandle nh_;
    ros::Subscriber state_sub_, pose_sub_;
    ros::Publisher setpoint_pub_;
    ros::ServiceClient arming_client_, set_mode_client_;
    mavros_msgs::State current_state_;
    geometry_msgs::PoseStamped current_pose_;
    bool current_pose_received_;
    double rate_, arrival_threshold_, arm_timeout_, offboard_timeout_, stabilize_time_;
    std::vector<std::array<double,3>> waypoints_;
    int waypoint_index_;
};

int main(int argc, char **argv) {
    ros::init(argc, argv, "offboard_ctrl_node");
    ros::NodeHandle nh;
    OffboardCtrl ctrl(nh);
    ctrl.run();
    return 0;
}
```

### 3.3 编译与运行

```bash
# 1. 创建工作空间
mkdir -p ~/catkin_ws/src && cd ~/catkin_ws/src
catkin_init_workspace

# 2. 创建功能包
catkin_create_pkg offboard_ctrl roscpp rospy geometry_msgs mavros_msgs std_msgs tf

# 3. 将 offboard_ctrl_node.cpp 放入 src/offboard_ctrl/src/

# 4. CMakeLists.txt 添加
add_executable(offboard_ctrl_node src/offboard_ctrl_node.cpp)
target_link_libraries(offboard_ctrl_node ${catkin_LIBRARIES})
add_dependencies(offboard_ctrl_node ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

# 5. 编译
cd ~/catkin_ws && catkin_make

# 6. 环境变量（写入 ~/.bashrc）
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
```

```bash
# ── 启动顺序（三个终端）──

# 终端1: 启动 PX4 SITL + Gazebo
cd ~/PX4_1.14.3
make px4_sitl_default gazebo

# 终端2: 启动 MAVROS
roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"

# 终端3: 启动 offboard 控制节点
rosrun offboard_ctrl offboard_ctrl_node
```

---

## 四、PX4 PID 参数整定分析

### 4.1 控制层级回顾

```
操控指令 → 位置控制(外环) → 速度控制 → 角速度控制(内环) → 电机
           ↑                 ↑          ↑
      航点跟踪精度         响应快慢    姿态稳定性
```

### 4.2 角速度控制器参数（内环核心）

| 参数 | 含义 | 调大效果 | 调小效果 |
|------|------|----------|----------|
| `MC_ROLLRATE_P` | 滚转角速度比例 | 响应更快 | 响应变慢 |
| `MC_ROLLRATE_I` | 滚转角速度积分 | 消除静差 | 静差增大 |
| `MC_ROLLRATE_D` | 滚转角速度微分 | 抑制震荡 | 震荡加剧 |
| `MC_PITCHRATE_P/I/D` | 俯仰角速度 PID | 通常与滚转相同 | |
| `MC_YAWRATE_P/I/D` | 偏航角速度 PID | 偏航响应可稍缓 | |

> **案例**：将 `MC_ROLLRATE_P` 从 0.15 → 0.2，角速度响应变快，但可能引入震荡。

### 4.3 速度控制器参数

| 参数 | 含义 | 调大效果 |
|------|------|----------|
| `MPC_XY_VEL_P_ACC` | XY 轴速度比例 | 提高水平响应性，可能超调 |
| `MPC_XY_VEL_I_ACC` | XY 轴速度积分 | 减少风扰等稳态误差 |
| `MPC_XY_VEL_D_ACC` | XY 轴速度微分 | 抑制超调和震荡 |
| `MPC_Z_VEL_P_ACC` | Z 轴速度比例 | 提高垂直响应性 |
| `MPC_Z_VEL_I_ACC` | Z 轴速度积分 | 减少高度稳态误差 |
| `MPC_Z_VEL_D_ACC` | Z 轴速度微分 | 抑制垂直震荡 |

> **案例**：将 `MPC_XY_VEL_P_ACC` 从 1.8 → 2.4，航点跟踪更敏捷，但超调增大。

### 4.4 PID 整定原则

```
调参顺序（从内到外）：
  角速度控制器 → 速度控制器 → 位置控制器

角速度控制器是最内环，必须先调稳，否则外层毫无意义。

P 参数：先调 P，让系统快速响应
D 参数：出现震荡后加 D，抑制震荡
I 参数：最后加 I，消除静差但不能太大（积分饱和）

调参工具：飞行日志 → logs.px4.io 分析
```

---

## 五、关键踩坑点

| # | 坑 | 原因 | 解决 |
|---|-----|------|------|
| 1 | **切 OFFBOARD 失败** | PX4 必须先连续收到 setpoint 才能切 | 切模式前先发布 2 秒 setpoint |
| 2 | **OFFBOARD 飞行中突然退出** | 飞控 500ms 没收到 setpoint 就自动退出 | 主循环中**每次迭代**都要 publish |
| 3 | **解锁不了** | OFFBOARD 模式下未收到稳定 setpoint | 解锁循环中也持续发 setpoint |
| 4 | **无人机飘走** | 位置控制需要 GPS 或视觉定位 | 确保 `mode_req_local_position` 满足 |
| 5 | **航点到达判定不准** | 阈值太小时无人机永远到不了 | `arrival_threshold` 设 0.3~0.5m |
| 6 | **编译报错找不到 mavros_msgs** | 未安装 mavros 包 | `sudo apt install ros-noetic-mavros` |

---

## 六、核心要点总结

| # | 要点 |
|---|------|
| 1 | **MAVROS = ROS ↔ PX4 的桥梁**，通过 MAVLink/UDP 双向通信 |
| 2 | **Topic 异步**（传感器数据流），**Service 同步**（解锁/模式切换） |
| 3 | **OFFBOARD 模式下 setpoint 不能断**（> 2Hz，500ms 超时即退出） |
| 4 | 流程：等待连接 → 预发 setpoint → 切 OFFBOARD → 解锁 → 航点导航 |
| 5 | PID 调参**从内到外**：角速度 → 速度 → 位置 |
| 6 | 飞行日志导入 `logs.px4.io` 可视化分析，辅助整定 |
