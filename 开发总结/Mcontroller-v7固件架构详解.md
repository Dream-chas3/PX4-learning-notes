---
tags:
  - Mcontroller-v7
  - FanciSwarm
  - 飞控固件
  - STM32H7
  - FreeRTOS
  - MAVLink
  - 固件架构
date: 2026-08-12
---

# Mcontroller-v7 固件架构详解

> 来源：FanciSwarm 固件源码分析（`~/桌面/FanciSwarm_Drone/Mcontroller-v7-FanciSwarm-main/`）
> 固件全称：Mcontroller-v7-FanciSwarm — 基于 Mcontroller 跨模态机器人运动控制器 v7

---

## 一、固件概览

| 项目 | 值 |
|------|-----|
| 固件名 | Mcontroller-v7-FanciSwarm |
| 开发者 | 幻思创新 Fancinnov（作者：JackyPan） |
| MCU | STM32H743 |
| RTOS | FreeRTOS |
| 构建工具 | STM32CubeIDE |
| 核心库 | `libMcontroller-v7-FanciSwarm.a`（闭源） |
| 烧录工具 | STM32CubeProgrammer V2.15 |
| 通信协议 | MAVLink（原生支持） |
| 源码托管 | GitHub（持续更新） |

---

## 二、四层架构

与开发指南文档描述一致：

```
┌─────────────────────────────────┐
│    主控层（Maincontroller）       │  ← 飞行模式、任务逻辑
│    mode_althold / mode_poshold   │
│    mode_autonav / mode_stabilize │
├─────────────────────────────────┤
│    中间层（Cpplibrary）           │  ← 算法库
│    EKF / PID / AHRS / 滤波器     │
├─────────────────────────────────┤
│    系统层（FreeRTOS）             │  ← 实时多线程调度
├─────────────────────────────────┤
│    硬件层（Clibrary）             │  ← HAL 封装、外设驱动
│    config.h / hal.h / ringbuffer │
└─────────────────────────────────┘
```

---

## 三、目录结构

```
Mcontroller-v7-FanciSwarm-main/
├── Maincontroller/           ← 飞控核心 + 飞行模式
│   ├── inc/
│   │   ├── maincontroller.h  ← 主接口（姿态/位置/MAVLink/参数）
│   │   └── common.h          ← PID 默认参数 + 全局常量定义
│   ├── src/
│   │   ├── maincontroller.cpp
│   │   ├── mode_stabilize.cpp
│   │   ├── mode_althold.cpp
│   │   ├── mode_poshold.cpp
│   │   ├── mode_autonav.cpp
│   │   ├── mode_mecanum.cpp
│   │   └── common.cpp
│   └── demo/                 ← 学习示例
│       ├── demo_ekf.cpp
│       ├── demo_pid.cpp
│       ├── demo_filter.cpp
│       ├── demo_fdcan.cpp
│       └── demo_uwb.cpp
│
├── Cpplibrary/               ← C++ 算法库（头文件）
│   └── include/
│       ├── ahrs/              ← 姿态估计
│       ├── ekf/               ← 扩展卡尔曼滤波
│       │   ├── ekf_baro.h
│       │   ├── ekf_rangefinder.h
│       │   ├── ekf_odometry.h
│       │   ├── ekf_gnss.h
│       │   └── ekf_wind.h
│       ├── attitude/          ← 姿态控制
│       ├── position/          ← 位置控制
│       ├── pid/               ← PID 控制器
│       ├── filter/            ← 滤波器（低通/微分）
│       ├── motors/            ← 电机驱动
│       ├── compass/           ← 磁力计校准
│       ├── accel/             ← 加速度计校准
│       ├── flash/             ← 数据存储
│       ├── math/              ← 数学库
│       └── uwb/               ← UWB 定位
│
├── Clibrary/                  ← C 硬件抽象层
│   └── include/
│       ├── config.h           ← 核心配置（串口模式/PWM/通信）
│       ├── define.h           ← 宏定义
│       ├── hal.h              ← 硬件抽象层
│       └── ringbuffer.h       ← 环形缓冲区
│
├── Core/                      ← STM32 CMSIS 启动文件
├── Drivers/                   ← STM32H7 HAL 驱动
├── Middlewares/               ← FreeRTOS + USB 库
├── FATFS/                     ← 文件系统（SD 卡日志）
└── USB_DEVICE/                ← USB 虚拟串口
```

---

## 四、飞行模式

定义在 `common.h` 中：

| 模式枚举 | 名称 | 功能 |
|----------|------|------|
| `MODE_STABILIZE` | 自稳模式 | 姿态控制，无位置保持 |
| `MODE_ALTHOLD` | 定高模式 | 保持高度，姿态可控 |
| `MODE_POSHOLD` | 定点模式 | 保持位置 + 高度 |
| `MODE_AUTONAV` | 自动导航 | 轨迹飞行、航点任务 |
| `MODE_PERCH` | 栖息模式 | 壁面/地面栖息 |
| `MODE_MECANUM_A` | 麦轮角度控制 | UGV 模式 |
| `MODE_MECANUM_V` | 麦轮速度控制 | UGV 模式 |
| `MODE_UGV_A/V` | 地面车模式 | UGV |

对应源文件：
- `mode_stabilize.cpp` — 自稳
- `mode_althold.cpp` — 定高
- `mode_poshold.cpp` — 定点
- `mode_autonav.cpp` — 自主导航
- `mode_mecanum.cpp` — 麦轮/UGV

---

## 五、串口通信配置

定义在 `Clibrary/include/config.h`：

```c
// 串口模式选项
#define CONFIG_COMM    0x00  // WiFi 配置模式（发 AT 指令用）
#define DEV_COMM       0x01  // 自定义开发模式
#define MAV_COMM       0x02  // MAVLink 模式 ← ROS 通信用这个
#define GPS_COMM       0x03  // GPS
#define TFMINI_COMM    0x04  // TFmini 激光测距仪
#define LC302_COMM     0x05  // LC302 光流
#define TF2MINI_COMM   0x06  // TF2mini 激光测距仪

// 默认配置
#define COMM_0 MAV_COMM        // USB 口 → MAVLink
#define COMM_1 MAV_COMM        // 串口1 → MAVLink
#define COMM_2 LC302_COMM      // 串口2 → 光流
#define COMM_3 GPS_COMM        // 串口3 → GPS
#define COMM_4 MAV_COMM        // 串口4 → MAVLink
#define COMM_UWB DEV_COMM      // UWB → 自定义开发模式
```

> ⚠️ **配置 WiFi 模块时**：需要把 `COMM_0 MAV_COMM` 改为 `COMM_0 CONFIG_COMM`，这样才能通过串口发 AT 指令。配完之后**必须改回来**并重新编译烧录。

---

## 六、传感器与算法

### EKF 滤波器

| EKF 模块 | 用途 |
|----------|------|
| `ekf_baro` | 气压计高度估计 |
| `ekf_rangefinder` | 激光测距仪高度 |
| `ekf_odometry` | 里程计（光流/视觉） |
| `ekf_gnss` | GNSS 定位 |
| `ekf_wind` | 风估计 |

### PID 控制（默认参数在 `common.h`）

```
姿态控制（串级 PID）：
  外环：角度 P（ROLL_PITCH_YAW_P = 4.5）
  内环：角速率 PID（P=0.05, I=0.15, D=0.0015）

位置控制（串级 PID）：
  外环：位置 P（POS_XY_P = 1.0, POS_Z_P = 1.0）
  中环：速度 PID（VEL_XY: P=2.0, I=0.4, D=0.8）
  内环：加速度 PID（ACC_Z: P=0.5, I=0.3, D=0）
```

### 传感器

| 传感器 | 数据接口 | 状态 |
|--------|----------|------|
| IMU（加速度计 + 陀螺仪） | SPI | ✅ 内置 |
| 磁力计 | I2C/SPI | ✅ 内置 |
| 气压计 | I2C | ✅ 内置 |
| 激光测距（ToF） | I2C | ✅ 可选 |
| 光流模块 | 串口 | ✅ 可选 |
| UWB（DWM1000） | SPI | ✅ 可选 |
| GNSS/GPS | 串口 | ⬜ 可选 |

---

## 七、参数系统

Mcontroller-v7 使用自定义 C++ struct 参数系统（类似 ArduPilot 风格），定义在 `common.h` 的 `parameter` 结构体中：

- 每个参数有唯一 `num`（ID 号）
- 支持类型：UINT8/16/32、FLOAT、VECTOR3F、FLOAT_PID 等
- 参数可持久化存储到 Flash
- 用户自定义参数从 `num=401` 开始

示例：
```cpp
struct angle_max{
    uint16_t num=6;           // 参数 ID
    dataflash_type type=FLOAT; // 类型
    float value=30.0f;        // 默认值（度）
}angle_max;
```

---

## 八、与 PX4 的关键区别

| 方面 | PX4 | Mcontroller-v7 |
|------|-----|----------------|
| **消息机制** | uORB 发布/订阅 | 直接函数调用 |
| **模块隔离** | 独立模块，松耦合 | 编译时静态链接 |
| **RTOS** | NuttX（类 POSIX） | FreeRTOS（轻量） |
| **构建** | cmake → make | STM32CubeIDE |
| **核心库** | 全部开源 | 闭源 .a 库 |
| **参数系统** | YAML 定义 | C++ struct |
| **控制器** | PX4 标准控制器 | 自研 PID 串级控制 |
| **学习资料** | PX4 官方文档丰富 | 仅官方开发指南 |
| **MAVLink** | ✅ | ✅ |

---

## 九、开发流程

```
修改源码（Maincontroller/ 或 Clibrary/）
        ↓
STM32CubeIDE 编译
        ↓
生成 .hex 文件
        ↓
STM32CubeProgrammer V2.15 烧录
        ↓
重启飞控
```

> ⚠️ 编译需要依赖包 `libMcontroller-v7-FanciSwarm.a`，需从官网下载最新版本放到工程根目录。

---

## 十、与 ROS 的关系

虽然是自研固件，但通过 MAVLink 与 ROS 通信：

```
树莓派 ROS 节点
    ↓ 串口 /dev/ttyAMA0 @ 460800 bps
MAVLink 协议
    ↓ USB 串口 (COMM_0 = MAV_COMM)
Mcontroller-v7 飞控
```

树莓派的 `datalink_serial.py` 通过 pymavlink 直接与飞控 MAVLink 通信，不经过 MAVROS。

电脑端也可以用 MAVROS 通过 WiFi（Mlink-esp 透传）连接飞控。
