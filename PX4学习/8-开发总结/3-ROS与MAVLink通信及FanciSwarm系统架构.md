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

|         | PX4            | FanciSwarm（Mcontroller-v7）             |
| ------- | -------------- | -------------------------------------- |
| 开发者     | Pixhawk 社区（开源） | 幻思创新 Fancinnov（自研）                     |
| RTOS    | NuttX          | FreeRTOS                               |
| 消息机制    | uORB（发布/订阅）    | 直接函数调用 + MAVLink                       |
| 模块系统    | 独立模块动态加载       | 编译时静态链接                                |
| 构建系统    | cmake + make   | STM32CubeIDE                           |
| 核心库     | 全部开源           | **libMcontroller-v7-FanciSwarm.a**（闭源） |
| 参数系统    | YAML 参数定义      | 自定义 C++ struct 参数表                     |
| MAVLink | ✅ 支持           | ✅ 支持（原生）                               |
| ROS 通信  | 通过 MAVROS      | **同样通过 MAVROS**（都走 MAVLink）            |

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

### 7.3 TCP/UDP 与 MAVLink 的关系

#### 先搞清楚 TCP 和 UDP 是什么

TCP 和 UDP 是**传输层**协议，负责把数据从一台设备传到另一台设备。MAVLink 是**应用层**协议，负责定义数据的格式和含义。

```
应用层：    MAVLink（心跳包、位置、指令...）
              ↓
传输层：    TCP 或 UDP（负责把数据送过去）
              ↓
网络层：    IP（负责寻址，找到目标设备）
              ↓
物理层：    WiFi / 串口 / 以太网
```

| | TCP | UDP |
|------|-----|------|
| 全称 | Transmission Control Protocol | User Datagram Protocol |
| 类比 | **打电话**：拨号→接通→确认对方在听→逐句说→挂断 | **寄明信片**：写完扔邮筒，不管对方收没收到 |
| 连接方式 | 面向连接（先三次握手建立连接） | 无连接（直接发） |
| 可靠性 | ✅ 保证送达、顺序不乱、不重复 | ❌ 不保证，可能丢包、乱序 |
| 速度 | 慢（确认、重传机制开销大） | 快（无额外开销） |
| 适用场景 | 网页、文件下载、邮件 | 视频直播、VoIP、在线游戏、**无人机遥测** |

#### MAVLink 可以跑在多种传输层上

MAVLink 本身不绑定任何传输方式。常见的组合：

| 传输方式 | 用的协议 | 在 FanciSwarm 中哪里用到 |
|----------|----------|------------------------|
| **串口（UART）** | 不需要 TCP/UDP，直接收发字节流 | 树莓派 ↔ 飞控（`/dev/ttyAMA0`） |
| **UDP over WiFi** | UDP | 电脑 ↔ Mlink-esp WiFi 模块 ↔ 飞控 |
| **UDP over WiFi** | UDP | PX4 SITL 仿真（端口 14550/14555） |
| **TCP over WiFi** | TCP | 树莓派视频推流（`rpicam-vid --listen -o tcp://...`） |

#### 为什么无人机通信优先用 UDP 而不是 TCP？

**核心原因：低延迟 > 绝对可靠。**

无人机每秒发几十上百条 MAVLink 消息（位置、姿态、状态）。如果用的是 TCP：

```
飞控发第 100 号包 → 丢了 → TCP 要求重传 → 卡住 → 第 101~110 号包全部排队等待
→ 地面站画面卡顿，位置信息延迟
```

UDP 的处理方式：

```
飞控发第 100 号包 → 丢了 → 不管，继续发 101 号 → 地面站最多丢一帧
→ 几乎感觉不到
```

> 💡 **飞控遥测宁可丢一个包，也不能因为等重传而卡住。** 一个位置数据包丢了，下一个几十毫秒就到；TCP 重传可能让整个链路延迟失控。

#### 但也有例外

| 场景 | 用什么 | 为什么 |
|------|--------|--------|
| 实时飞行数据（IMU/姿态/GPS） | UDP | 延迟敏感，偶尔丢包可接受 |
| 固件烧录 / 参数写入 | TCP 或串口 | 一个字节都不能错 |
| 视频推流（rpicam-vid） | TCP | 视频编码数据量大，关键帧丢了画面全花 |

所以你在文档里看到 `tcp://0.0.0.0:8888`（树莓派推流用 TCP）和 `mavlink_channel_t` 设为网络模式（飞控数据走 UDP）并不矛盾——**各自选了最适合的传输方式**。
