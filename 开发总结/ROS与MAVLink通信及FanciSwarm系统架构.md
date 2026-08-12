---
tags:
  - ROS
  - MAVROS
  - MAVLink
  - FanciSwarm
  - Mcontroller
  - 无人机架构
  - 伴随计算机
date: 2026-08-12
---

# ROS 与 MAVLink 通信及 FanciSwarm 系统架构

> 来源：ChatGPT 对话 + FanciSwarm 固件源码分析
> 关键修正：FanciSwarm 飞控运行的是 **Mcontroller-v7**（自研），不是 PX4

---

## 一、先纠正一个关键认知

> ❓ **我之前以为**：FanciSwarm = STM32H743 上跑 PX4

**实际情况**：

| | PX4 | FanciSwarm（Mcontroller-v7） |
|------|-----|------|
| 开发者 | Pixhawk 社区（开源） | 幻思创新 Fancinnov（自研） |
| RTOS | NuttX | FreeRTOS |
| 消息机制 | uORB（发布/订阅） | 直接函数调用 + MAVLink |
| 模块系统 | 独立模块动态加载 | 编译时静态链接 |
| 构建系统 | cmake + make | STM32CubeIDE |
| 核心库 | 全部开源 | **libMcontroller-v7-FanciSwarm.a**（闭源） |
| 参数系统 | YAML 参数定义 | 自定义 C++ struct 参数表 |
| MAVLink | ✅ 支持 | ✅ 支持（原生） |
| ROS 通信 | 通过 MAVROS | **同样通过 MAVROS**（都走 MAVLink） |

> 💡 **关键结论**：虽然内部架构完全不同，但从 ROS/MAVROS 的角度看，**通信协议都是 MAVLink**，所以你的 ROS 控制代码（`offboard_ctrl.cpp`）基本可以复用。

---

## 二、核心通信链路（仿真 vs FanciSwarm 真机）

```
ROS 节点（你的代码）
    ↓
MAVROS
    ↓
MAVLink（统一协议）
    ↓
┌──────────────────────────────────────┐
│ 仿真：PX4 SITL（电脑上模拟运行）      │
│ 真机：Mcontroller-v7（STM32H743）     │
└──────────────────────────────────────┘
    ↓
无人机（Gazebo / 真实电机）
```

中间三层（ROS → MAVROS → MAVLink）完全一样，所以**你的 ROS 代码不改也能用**。

改动的是最底层——飞控固件本身：
- PX4 改 `src/modules/` → 对 FanciSwarm **无意义**
- Mcontroller 改 `Maincontroller/` → 对 PX4 **无意义**

---

## 三、FanciSwarm 系统架构

### 3.1 完整分层

```
        Ubuntu 电脑（开发/地面站）
        ROS + MAVROS
                │
                │ WiFi（MAVLink over UDP）
                │ 或 USB 串口
                ↓
        ┌───────────────────┐
        │   树莓派（伴随计算机）  │
        │   Linux + ROS        │
        │   视觉 / AI / 规划    │
        └───────────────────┘
                │
                │ 串口 MAVLink（/dev/ttyAMA0）
                │ 转接线连接飞控扩展板
                ↓
        ┌───────────────────┐
        │ STM32H743（飞控）    │
        │ Mcontroller-v7     │
        │ FreeRTOS           │
        │ 实时姿态/电机控制    │
        └───────────────────┘
                │
              电机
```

### 3.2 各组件分工

| 组件 | 运行什么 | 擅长 | 不擅长 |
|------|----------|------|--------|
| **STM32H743** | Mcontroller-v7 固件 | 400Hz 姿态控制、电机 PWM、传感器融合 | AI、视觉、复杂路径规划 |
| **树莓派** | Linux + ROS + OpenCV | YOLO、SLAM、深度学习、ROS 节点 | 实时控制 |
| **Ubuntu 电脑** | ROS + MAVROS | 开发调试、Offboard 控制、仿真 | — |

---

## 四、有无树莓派对 ROS 通信的影响

> ❓ **我的疑问**：没有树莓派是不是就无法进行真机和 ROS 之间的联动了？

**结论：不是。树莓派只是"ROS 运行在哪里"的问题。**

### 架构 1：电脑直连飞控（无树莓派）✅ 可以

```
Ubuntu 电脑
    ROS + MAVROS
        ↓ MAVLink（WiFi UDP 或 USB 串口）
STM32H743（Mcontroller-v7）
        ↓
      电机
```

飞控本身支持 MAVLink 串口通信（COMM_0 = MAV_COMM over USB），电脑可以直接 `roslaunch mavros` 连上飞控。

### 架构 2：树莓派作为伴随计算机 ✅ 更强

```
树莓派（无人机上）
    ROS + MAVROS + 视觉/AI
        ↓ 串口 MAVLink
STM32H743（Mcontroller-v7）
```

无人机飞出去后不需要电脑跟着，树莓派负责高级任务。

| | 架构 1（电脑直连） | 架构 2（树莓派伴随） |
|------|------|------|
| ROS 在哪跑 | 电脑 | 树莓派 |
| 能 Offboard 控制？ | ✅ | ✅ |
| 能视觉/AI？ | ❌ | ✅ |
| 通信距离 | 受 WiFi 限制 | 自主飞行 |

---

## 五、与 PX4+MAVROS+Gazebo 开发的区别总结

| | PX4 SITL + Gazebo | FanciSwarm 真机 |
|------|------|------|
| 飞控固件 | PX4（开源） | **Mcontroller-v7**（自研） |
| 飞控运行位置 | 电脑上模拟 | STM32H743 芯片 |
| 飞控开发方式 | 修改 PX4 源码 → cmake 编译 | 修改 Mcontroller 源码 → STM32CubeIDE 编译 → .hex 烧录 |
| 核心库 | 全部可见 | **libMcontroller-v7-FanciSwarm.a**（闭源 .a 库） |
| MAVLink | ✅ | ✅ |
| MAVROS | ✅ | ✅ |
| ROS 控制代码 | ✅ | ✅（**基本可以复用**） |
| 硬件 | Gazebo 模拟 | 真实电机、传感器 |
| 风险 | 无 | 有（注意安全） |

---

## 六、学习路线调整

### 修正后的路线

```
PX4 概念学习（uORB/MAVLink/飞控架构）
        ↓                          ← 你在这里学到的概念仍然有用
MAVROS + ROS 控制
        ↓                          ← 这部分仿真和真机通用
FanciSwarm 真机通信
        ↓
Mcontroller-v7 固件开发           ← 这是真机特有的，跟 PX4 无关
        ↓
树莓派 ROS + 视觉
```

### 分阶段建议

| 阶段 | 内容 | 说明 |
|------|------|------|
| **1** | PX4 SITL + MAVROS + Offboard 控制 | 学习 ROS↔飞控 通信链路（通用） |
| **2** | 电脑直连 FanciSwarm 真机 | MAVROS 连接目标换成真实飞控 |
| **3** | Mcontroller-v7 固件开发 | 修改 `Maincontroller/`，重新编译烧录 |
| **4** | 树莓派 ROS 迁移 | 把 ROS 节点搬到无人机上 |

> 💡 PX4 和 Mcontroller 虽然内部不同，但 MAVLink 是统一语言。你现在学 MAVROS 和 ROS Offboard 控制的知识**直接适用于 FanciSwarm**。

---

## 七、扩展知识

### 7.1 为什么 FanciSwarm 自研而不是直接用 PX4？

- PX4 的 NuttX + uORB 架构相对重量级
- FreeRTOS 更轻量，适合微型无人机
- 自研可以完全控制硬件层（STM32H7 HAL 直接操作）
- 闭源核心库保护商业竞争力
- 仍然兼容 MAVLink，不封闭生态

### 7.2 通信方式汇总

| 方式 | 物理介质 | MAVLink 通道 | 适用场景 |
|------|----------|-------------|----------|
| USB 串口 | USB 线 | COMM_0（MAV_COMM） | 调试、近距离烧录 |
| WiFi | Mlink-esp 模块 | 串口透传 | 飞行控制、ROS 连接 |
| 树莓派串口 | GPIO 转接线 | `/dev/ttyAMA0` @ 460800 | 树莓派 ↔ 飞控内部通信 |
