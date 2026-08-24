---
title: FanciSwarm源码解读
tags: [FanciSwarm, STM32H7, FreeRTOS, 飞控, 无人机]
创建时间: 2026-08-13
源码版本: Mcontroller-v7-FanciSwarm-main
芯片: STM32H743VITx (Cortex-M7)
---

# FanciSwarm 飞控固件源码解读

> 本文档基于 `Mcontroller-v7-FanciSwarm-main` 工程源码整理，用于理解 FanciSwarm 微型集群无人机的飞控固件架构、工作流程与重点模块。

## 目录
- [1. 工程定位与核心结论](#1-工程定位与核心结论)
- [2. 文件夹地图（分层架构）](#2-文件夹地图分层架构)
- [3. 系统启动流程](#3-系统启动流程)
- [4. 多任务调度与数据流](#4-多任务调度与数据流)
- [5. 重点模块逐个解读](#5-重点模块逐个解读)
- [6. 串级 PID 控制结构](#6-串级-pid-控制结构)
- [7. 飞行模式状态机](#7-飞行模式状态机)
- [8. 配置系统](#8-配置系统)
- [9. 学习路线建议](#9-学习路线建议)

---

## 1. 工程定位与核心结论

FanciSwarm 是**幻思创新（Fancinnov）**开发的微型集群无人机。本固件基于 **Mcontroller 跨模态机器人运动控制器 v7** 开发，运行在 **STM32H743VI（Cortex-M7，480MHz）** 上，使用 **FreeRTOS** 作为实时操作系统，工程由 **STM32CubeIDE** 组织。

架构上大量借鉴 **ArduPilot Copter**：PID 参数命名（`AC_ATC_MULTI_RATE_ROLL_P`）、标志位结构 `ap_t`、飞行模式名（stabilize/althold/poshold/autonav）、状态机 `AltHoldModeState` 都与 ArduPilot 一脉相承。

> [!important] 最重要的一个事实
> 根目录的 `libMcontroller-v7-FanciSwarm.a`（约 353MB）是**预编译的闭源静态库**。所有核心算法的**实现**（EKF、AHRS、PID、滤波器、电机混控、姿态解算、GNSS/UWB 解析）都编译进了这个 `.a` 文件，**看不到源码**。
>
> 开源可见的只有：
> - `Clibrary/include`、`Cpplibrary/include` 里的**头文件**（接口 + 注释，相当于"API 文档"）
> - `Maincontroller/` 里的**飞控应用层**（飞行模式、主循环、校准、日志）
> - `Maincontroller/demo/` 里的**官方示例**（每个模块的最小用法）
>
> 因此学习定位是：**学飞控架构、流程、状态机、以及库的调用方式**，而非啃 EKF/PID 内部公式（不开放）。另外 `hal.h` 头部注明：代码仅限 Fancinnov 生产的 Mcontroller 上使用。

---

## 2. 文件夹地图（分层架构）

工程从底到顶分为五层：

| 层级 | 文件夹 | 说明 | 阅读优先级 |
|---|---|---|---|
| 应用层 | `Maincontroller/` | **飞控业务逻辑（开源精华）**：飞行模式、主循环、校准、日志 | ⭐⭐⭐ 核心精读 |
| 应用示例 | `Maincontroller/demo/` | 每个库模块的**最小使用示例** | ⭐⭐⭐ 入门入口 |
| 库接口层 | `Cpplibrary/include/` `Clibrary/include/` | 闭源算法库的**头文件**（EKF/PID/AHRS/数学库 API） | ⭐⭐ 当手册查 |
| 硬件抽象层 | `Core/` | STM32CubeMX 生成的初始化 + **FreeRTOS 任务** | ⭐⭐ 重点看 freertos.c/main.c |
| 驱动层 | `Drivers/` | ST 官方 HAL/CMSIS 库 | ✖ 用到再查 |
| 中间件层 | `Middlewares/` `FATFS/` `USB_DEVICE/` | FreeRTOS、文件系统、USB | ✖ 忽略 |
| 其他 | `build/` `.settings/` `.project` `.cproject` | 编译产物与工程配置 | ✖ 忽略 |
| 链接脚本 | `STM32H743VITX_FLASH.ld` / `RAM.ld` | 内存布局 | 进阶再瞄 |

`Maincontroller/` 内的源文件（这是真正要读的部分）：

```
Maincontroller/
├── inc/
│   ├── maincontroller.h    # 主控制器头文件：全局对象声明、函数原型、枚举、ap_t 标志位
│   └── common.h            # 所有 cpp 的公共头：PID 默认参数、parameter 参数系统、常量宏
├── src/
│   ├── maincontroller.cpp  # 核心胶水(5168行)：对象初始化、校准、模式调度、姿态/位置接口
│   ├── common.cpp          # 全局状态变量、mode_update 调度、云台相机协议
│   ├── mode_stabilize.cpp  # 自稳模式
│   ├── mode_althold.cpp    # 定高模式
│   ├── mode_poshold.cpp    # 定点模式（含返航/降落/航线）
│   ├── mode_autonav.cpp    # 自动导航模式（含 offboard 指令）
│   └── mode_mecanum.cpp    # 麦克纳姆轮底盘模式
└── demo/
    ├── demo_filter.cpp     # 低通滤波器示例
    ├── demo_pid.cpp        # 单环/串级/二维 PID 示例
    ├── demo_ekf.cpp        # EKF 示例
    ├── demo_uwb.cpp        # UWB 示例
    └── demo_fdcan.cpp      # CAN 通信示例
```

---

## 3. 系统启动流程

### 3.1 启动入口 `Core/Src/main.c`

`main()` 的执行顺序：

```
MPU_Config()                 # 配置内存保护单元(MPU)，保护 0x24000000 开始的 512KB AXI SRAM
SCB_EnableICache()           # 使能指令缓存
SCB_EnableDCache()           # 使能数据缓存
HAL_Init()                   # 复位所有外设，初始化 Flash 接口和 SysTick
SystemClock_Config()         # 系统时钟：HSE → PLL → 480MHz（PLLM=3, PLLN=120, PLLP=2）
PeriphCommonClock_Config()   # 外设时钟：PLL2(SPI/FDCAN/USART)、PLL3(I2C/USB/ADC)
MX_xxx_Init()                # 初始化各外设：GPIO/DMA/MDMA/ADC/FDCAN/I2C/SDMMC/SPI/TIM/UART/USART
HAL_TIM_Base_Start_IT(...)   # 启动 9 个定时器中断（TIM1/3/4/6/7/8/13/14/16）
osKernelInitialize()         # 初始化 FreeRTOS 内核
MX_FREERTOS_Init()           # 创建所有 FreeRTOS 任务
osKernelStart()              # 启动调度器（从此不再返回）
```

> [!note] 关键点
> 定时器中断在这里就启动了，但任务真正运行是在 `osKernelStart()` 之后。中断回调里只是用 `osThreadFlagsSet()` 唤醒对应任务（见 [freertos.c](#4-多任务调度与数据流)）。

`main.c` 中的两个关键回调：

- `HAL_TIM_IC_CaptureCallback`：PPM 遥控信号捕获（TIM17）
- `HAL_TIM_PeriodElapsedCallback`：各定时器中断 → 分发到不同频率的 `TIM_xxx_HZ_Callback()`
- `HAL_GPIO_EXTI_Callback`：8 个外部 GPIO 中断（编码器/通用 IO）

### 3.2 系统初始化 `Core/Src/freertos.c` 的 `InitTask`

这是**一次性初始化线程**，优先级最高，完成所有传感器/外设初始化后自毁（`osThreadTerminate`）：

```
MX_USB_DEVICE_Init()     # USB 虚拟串口
reset_usb() / config_comm()
adc_init()               # ADC（电压/电流检测）
FRAM_Init()              # 铁电存储器（参数持久化）
update_dataflash()       # 从 FRAM 读参数
set_comm_bandrate()      # 设置串口波特率
RC_Input_Init(SBUS)      # 遥控输入
wifi_init()              # WiFi
IMU_Init() / MAG_Init()  # 惯性/磁传感器
BARO_Init()              # 气压计
vl53lxx_init()           # VL53Lxx 激光测距
MAG2_Init() / tf2mini_init()
motors_init()            # 电机
attitude_init()          # 姿态控制器
pos_init()               # 位置控制器
uwb_init()               # UWB（失败则终止 uwbTask）
encoder_init()           # 编码器（USE_ENCODER）
initialed_task=true      # 标记初始化完成
osThreadTerminate(initTaskHandle)  # 自毁
```

---

## 4. 多任务调度与数据流

### 4.1 任务与频率

系统采用 **"定时器中断 → 任务标志位 → 任务执行"** 的经典 FreeRTOS 模式。定时器中断不直接干活，只负责唤醒任务。

| 定时器 | 频率 | 唤醒的任务 | 任务职责 |
|---|---|---|---|
| TIM16 | 1000Hz | （仅计数 `time_ms++`） | 系统时间基准 |
| TIM6 | 400Hz | `loop400hzTask` | **主控制循环** |
| TIM7 | 200Hz | `loop200hzTask` + `offboardTask` | 通信回调 / offboard 发送 |
| TIM13 | 100Hz | `loop100hzTask` | 解锁检查 + 油门 |
| TIM14 | 50Hz | `loop50hzTask` | 传感器外设（UWB/光流/测距/磁） |
| — | 定时 | `heartbeatTask` `mavSendTask` `gnssTask` `imuTask` `magTask` `buzzerTask` `sdLogTask` `idleTask` | 心跳、数据分发、GNSS、IMU、磁、蜂鸣器、SD 日志、调试 |

### 4.2 主控制循环（最重要）

`Loop400hzTask` 是**飞控的核心**，每 2.5ms（400Hz）执行一次：

```cpp
osThreadFlagsWait(1, osFlagsWaitAny, osWaitForever);  // 等待 400Hz 中断唤醒
ahrs_update();            // 1. 姿态解算（AHRS）
ekf_baro_alt();           // 2. 气压高度 EKF 融合
/*** 以下可改 ***/
encoder_position_update();// 3. 编码器里程计更新
ekf_odom_xy();            // 4. 里程计 XY 融合
ekf_gnss_xy();            // 5. GNSS XY 融合
mode_update();            // 6. 飞行模式控制器（核心业务）
set_motors_value();       // 7. 输出到电机
set_servos_value();       // 8. 输出到舵机
```

### 4.3 完整数据流图

```
                 ┌─────────────────────────────────────────────────────┐
                 │                   传感器输入层                       │
                 │  IMU(IMUTask,500Hz)  Baro  Mag(MagTask,100Hz)      │
                 │  GNSS(GnssTask,50Hz) UWB 光流 测距 遥控器RC         │
                 └──────────────┬──────────────────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────────────────┐
                 │              信号滤波与状态估计                      │
                 │  LowPassFilter → AHRS(四元数EKF) → EKF_Baro/        │
                 │  EKF_Rangefinder/EKF_Odometry/EKF_GNSS/EKF_Wind     │
                 │  输出: 姿态(roll/pitch/yaw) + 位置/速度(xyz)          │
                 └──────────────┬──────────────────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────────────────┐
                 │              飞行模式控制器 (400Hz)                  │
                 │  mode_update() → stabilize/althold/poshold/autonav   │
                 │  根据遥控/地面站/offboard 指令产生目标姿态+目标油门    │
                 └──────────────┬──────────────────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────────────────┐
                 │              串级 PID 控制                          │
                 │  位置环 → 速度环 → 加速度环 → 姿态角环 → 角速率环     │
                 │  Attitude_Multi + PosControl                        │
                 └──────────────┬──────────────────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────────────────┐
                 │              执行器输出                              │
                 │  Motors 混控 → PWM → 电调/舵机                      │
                 └─────────────────────────────────────────────────────┘
```

---

## 5. 重点模块逐个解读

### 5.1 配置层 `Clibrary/include/config.h`

这是**编译期配置**，改这里需要重新编译（部分运行期配置需在 APP 上设）。

| 配置块 | 关键宏 | 说明 |
|---|---|---|
| PWM 频率 | `PWM_MOTOR_FREQ 400` | 电机 400Hz、数字舵机 320Hz、模拟舵机 50Hz |
| 电调脉宽 | `PWM_ESC_MIN/MAX` | 1000~2000us，`SPIN_ARM 1100` |
| 遥控输入 | `RC_INPUT_SBUS` `RC_INPUT_CHANNELS 14` | 使用 SBUS，14 通道，1000~2000us |
| 串口模式 | `CONFIG_COMM/DEV_COMM/MAV_COMM/GPS_COMM/...` | 各串口的协议模式 |
| 串口分配 | `COMM_0=MAV` `COMM_1=MAV` `COMM_2=LC302(光流)` `COMM_3=GPS` `COMM_4=MAV` | 5 个物理串口用途 |
| 传感器开关 | `USE_MAG/GNSS/UWB/FLOW/ODOMETRY/VINS/MOTION/2D_LIDAR` | 各定位源使能开关 |
| 抗风 | `USE_WIND 0` | 是否启用抗风估计 |
| 云台相机 | `USE_A8MINI 0` `USE_C12 0` | 云台相机协议 |
| 解锁检查 | `PREARM_CHECK 1` | 预解锁检查 |
| Flash 存储 | `USE_FRAM 2` `ADDR_FLASH` | 参数存储位置（FRAM 或内部 Flash bank2） |
| 版本号 | `VERSION_HARDWARE 718` `VERSION_FIRMWARE 2026071501` | 硬件/固件版本 |

### 5.2 硬件抽象层 `Clibrary/include/hal.h` / `define.h`

- `hal.h`：包含所有 STM32 HAL 头 + Mavlink + 配置，定义传感器数据结构（`MPU6000_Data`、`MPU9250_Data`、`ICM20608_Data`、`QMC5883_Data` 等），以及大量底层函数（`IMU_Get_Data`、`BARO_Get_Date`、`Motor_Set_Value`、`RC_Input_Loop` 等）。**注意头部的版权声明**。
- `define.h`：数学/物理常量（`GRAVITY_MSS`、`RADIUS_OF_EARTH`、WGS84 椭球参数、大气参数），很多直接来自 ArduPilot。

### 5.3 姿态解算 `Cpplibrary/include/ahrs/ahrs.h`

`AHRS` 类是**基于 EKF 的四元数姿态解算**（不是简单的互补滤波）。从成员变量可看出实现思路：

- 四元数 `q0~q3` 表示姿态
- 用 **加速度计** 和 **磁罗盘** 两组观测做 EKF 校正（两组 `Hk`/`zk` 观测方程）
- 过程噪声 `Q_PredNoise` 很大（`1e6`），观测噪声 `Rk1=0.8`（加速度）、`Rk2=0.4`（磁）
- 输出 `quaternion2`，以及 `vib_value/vib_angle` 振动检测

调用：`ahrs_update()`（内部实现闭源，在 `.a` 里）。

### 5.4 传感器融合 `Cpplibrary/include/ekf/*.h`

多个独立的一维/二维 EKF，各自融合一种定位源：

| EKF 类 | 融合对象 | 输出 | 构造参数含义 |
|---|---|---|---|
| `EKF_Baro` | 气压计 | 高度 | `(dt, Q, R_pos, R_vel, ...)` 观测/预测方差 |
| `EKF_Rangefinder` | 激光测距 | 高度 | `(dt, Q, R_pos, R_vel)` |
| `EKF_Odometry` | 里程计 | XY | 6 个方差参数 |
| `EKF_GNSS` | GPS | XY | 8 个方差参数 |
| `EKF_Wind` | 风 | 风速 | — |

> [!tip] EKF 参数含义（以 demo_ekf.cpp 注释为准）
> `Q` = 观测数据方差，`R` = 预测数据方差。方差越小代表越"信任"该数据源。调参就是调这些方差。

初始化示例（maincontroller.cpp）：

```cpp
EKF_Baro *ekf_baro = new EKF_Baro(_dt, 0.01, 1.0, 0.000016, 0.000016);
EKF_GNSS *ekf_gnss = new EKF_GNSS(_dt, 0.0016, 0.0016, 0.0016, 0.0016, 0.000016, ...);
```

### 5.5 姿态控制器 `Cpplibrary/include/attitude/attitude_multi.h`

`Attitude_Multi` 是**多旋翼姿态控制类**（ArduPilot `AC_AttitudeControl_Multi` 的移植）。核心方法：

| 方法 | 作用 |
|---|---|
| `input_euler_angle_roll_pitch_yaw()` | 输入目标欧拉角（roll/pitch/yaw 角度） |
| `input_euler_angle_roll_pitch_euler_rate_yaw()` | 输入目标 roll/pitch 角度 + yaw 角速度 |
| `input_rate_bf_roll_pitch_yaw()` | 输入机体坐标系角速度 |
| `rate_controller_run()` | **运行最底层角速率 PID，输出给电机** |
| `set_throttle_out()` | 设置输出油门 |
| `bf_feedforward()` | 机体坐标前馈开关 |

内部包含 3 个角速率 PID（`_pid_rate_roll/pitch/yaw`）+ 3 个角度 P（`_p_angle_roll/pitch/yaw`），这就是**角速率环 + 角度环**的串级结构。

### 5.6 位置控制器 `Cpplibrary/include/position/position.h`

`PosControl` 是**位置控制类**（ArduPilot `AC_PosControl` 的移植），分 Z 轴（高度）和 XY 轴（水平）两套控制器：

- **Z 轴**：位置 P → 速度 P → 加速度 PID（`_p_pos_z` → `_p_vel_z` → `_pid_accel_z`）
- **XY 轴**：位置 P → 速度 PID_2D（`_p_pos_xy` → `_pid_vel_xy`），最终输出目标 roll/pitch 倾角

核心方法：

| 方法 | 作用 |
|---|---|
| `update_z_controller()` | 高度控制器（输入当前高度/速度） |
| `update_xy_controller()` | 水平控制器（输入当前位置/速度，输出 `get_roll()/get_pitch()`） |
| `set_alt_target_from_climb_rate_ff()` | 用爬升速度前馈更新目标高度 |
| `set_pilot_desired_acceleration()` | 把遥控摇杆换算成期望加速度 |
| `calc_desired_velocity()` | 计算期望速度 |

### 5.7 参数系统 `Maincontroller/inc/common.h`

这是**全工程最长的头文件**（1090 行），核心是一个 `parameter` 结构体实例 `param`。每个参数是一个结构体字段，包含：

```cpp
struct rate_roll_pid {
    uint16_t num=20;              // 参数 ID（地面站通过 num 寻址）
    dataflash_type type=FLOAT_PID;// 参数类型
    float value_p=...;            // 默认值
    // ...
} rate_roll_pid;
```

- 参数类型有 `FLOAT`、`FLOAT_PID`、`VECTOR3F`、`UINT8`、`UINT16_CHANNLE_8` 等（`dataflash_type` 枚举）
- 前半部分是**所有 PID 默认增益**（`AC_ATC_MULTI_RATE_ROLL_P` 等），能看出整个串级 PID 结构
- 后半部分是**所有可调参数**（角度上限、爬升速度、返航高度、UWB 锚点坐标、电压增益……）
- 用户自定义参数从 `num=401` 开始，避免与官方冲突

### 5.8 核心胶水 `Maincontroller/src/maincontroller.cpp`（5168 行）

这是**最大的文件**，把所有模块粘合起来。主要分块：

1. **全局对象初始化**（约 111-120 行）：`new` 出 `ahrs`、`ekf_*`、`motors`、`attitude`、`pos_control` 等核心对象并传入参数
2. **各种 `get_xxx()` 接口**：把库里的姿态/位置/速度/目标暴露给模式层（对应 `maincontroller.h` 里的声明）
3. **`mode_update()` 调度**（实际在 common.cpp）：根据 `robot_sub_mode` 分发到各模式
4. **校准流程**：加速度计/陀螺仪/罗盘校准
5. **日志系统**：`Logger_*`、`Log_To_File`（写 SD 卡）
6. **安全保护**：振动检测、倾角过大自动停桨、低电量返航/降落

### 5.9 云台相机协议 `Maincontroller/src/common.cpp`

`common.cpp` 除了定义全局状态变量（`robot_state`、`robot_sub_mode` 等），还实现了两个云台相机的串口控制协议：

- **A8mini**：0x55 0x66 帧头 + CMD + CRC16 校验（变焦、回中、偏航/俯仰角、拍照录像）
- **C12 热成像**：ASCII 命令 `#TPU...` + 累加和校验

---

## 6. 串级 PID 控制结构

这是理解飞控最关键的部分。整套控制是**串级（cascade）PID**，从外到内：

```
               位置目标(期望xyz)
                    │
   ┌────────────────▼────────────────┐
   │ 位置环 P      (pos_z_p/pos_xy_p) │  → 输出目标速度
   └────────────────┬────────────────┘
                    ▼
   ┌────────────────────────────────┐
   │ 速度环 P/PID  (vel_z_p/vel_xy)  │  → 输出目标加速度/倾角
   └────────────────┬────────────────┘
                    ▼
   ┌────────────────────────────────┐
   │ 加速度环 PID  (accel_z_pid)     │  → 输出油门   [仅 Z 轴]
   └────────────────┬────────────────┘
                    ▼
   ┌────────────────────────────────┐
   │ 姿态角环 P   (angle_roll/pitch) │  → 输出目标角速度
   └────────────────┬────────────────┘
                    ▼
   ┌────────────────────────────────┐
   │ 角速率环 PID (rate_roll/pitch/  │  → 输出 PWM 给电机
   │                yaw_pid)          │
   └────────────────────────────────┘
```

对应关系（代码位置）：

| 环 | PID 对象 | 默认参数（common.h） | 头文件 |
|---|---|---|---|
| 角度 roll/pitch/yaw | `P` | `AC_ATTITUDE_CONTROL_ANGLE_*_P = 4.5` | attitude_multi.h |
| 角速率 roll/pitch | `PID` | `AC_ATC_MULTI_RATE_RP_P=0.05 I=0.15 D=0.0015` | attitude_multi.h |
| 角速率 yaw | `PID` | `AC_ATC_MULTI_RATE_YAW_P=0.18 I=0.018 D=0` | attitude_multi.h |
| 位置 Z | `P` | `POSCONTROL_POS_Z_P = 1.0` | position.h |
| 速度 Z | `P` | `POSCONTROL_VEL_Z_P = 5.0` | position.h |
| 加速度 Z | `PID` | `POSCONTROL_ACC_Z_P=0.5 I=0.3 D=0` | position.h |
| 位置 XY | `P` | `POSCONTROL_POS_XY_P = 1.0` | position.h |
| 速度 XY | `PID_2D` | `POSCONTROL_VEL_XY_P=2.0 I=0.4 D=0.8` | position.h |

> [!tip] 为什么用串级
> 外环（位置/速度）是慢环，内环（姿态/角速率）是快环。快环先把姿态稳定住，慢环再修正位置，这样既稳又准。学习时可以先只看最内层的角速率环（自稳模式的本质），再逐层往外理解。

---

## 7. 飞行模式状态机

### 7.1 模式体系（三层）

`common.h` 里定义了三种模式的枚举：

| 枚举 | 作用 | 取值 |
|---|---|---|
| `ROBOT_MAIN_MODE` | **机器人形态**（跨模态核心） | `MODE_AIR`（空中）/`MODE_MECANUM`/`MODE_SPIDER`/`MODE_UGV` |
| `ROBOT_SUB_MODE` | **飞行/运动子模式** | `MODE_STABILIZE`/`MODE_ALTHOLD`/`MODE_POSHOLD`/`MODE_AUTONAV`/`MODE_PERCH`/`MODE_MECANUM_*`/`MODE_SPIDER_*`/`MODE_UGV_*` |
| `ROBOT_STATE` | **飞行器运行状态** | `STATE_NONE/TAKEOFF/FLYING/FLYING_VIRTUAL/LANDED/STOP/CLIMB/DRIVE` |

这体现了"跨模态"：同一套控制器既能飞（AIR），也能驱动麦克纳姆轮底盘（MECANUM）、蜘蛛机器人（SPIDER）、UGV。

### 7.2 各模式功能

| 模式 | 文件 | 功能 | 复杂度 |
|---|---|---|---|
| **Stabilize** | mode_stabilize.cpp | 纯手动自稳：遥控直接给目标倾角，无位置/高度闭环 | ⭐ 最简单 |
| **AltHold** | mode_althold.cpp | 定高：Z 轴自动，XY 手动，含起飞/降落状态机 | ⭐⭐ |
| **PosHold** | mode_poshold.cpp | 定点：XYZ 全闭环，含返航/降落/航线，3 个子模式 | ⭐⭐⭐ |
| **AutoNav** | mode_autonav.cpp | 自动导航：支持 MAVLink offboard 指令 + 探索 | ⭐⭐⭐⭐ 最复杂 |
| **Mecanum** | mode_mecanum.cpp | 麦克纳姆轮底盘：4 轮独立正反转 + 速度混控 | ⭐ 独立于飞行 |

### 7.3 飞行子状态机 `AltHoldModeState`

althold/poshold/autonav 共用同一套高度状态机（ArduPilot 风格）：

```
AltHold_MotorStopped   # 电机停止（未解锁/油门为零）
AltHold_Takeoff        # 起飞中
AltHold_Flying         # 飞行中
AltHold_Landed         # 已着陆
AltHold_Climb          # 爬升（保留）
```

状态判定逻辑（各模式一致）：

```cpp
if (!motors->get_armed())           → AltHold_MotorStopped
else if (takeoff_running() || takeoff_triggered(...) || get_takeoff()) → AltHold_Takeoff
else if (ap->land_complete)         → AltHold_Landed
else                                 → AltHold_Flying
```

### 7.4 子模式切换（CH7 遥控通道）

poshold/autonav 模式通过 **CH7 通道** 切换三个子模式：

| CH7 值 | 子模式 | 说明 |
|---|---|---|
| 0.7~1.0 | 姿态模式 | 只锁姿态，XY 手动 |
| 0.3~0.7 | 位置模式 | 锁位置（定点悬停） |
| <0.3 | 航线/自主模式 | 跟随航线点 或 offboard 指令 |

### 7.5 安全保护机制（分布在模式代码中）

- **低电量返航**：`get_batt_volt() < lowbatt_return_volt` → 强制返航
- **低电量降落**：`get_batt_volt() < lowbatt_land_volt` → 强制降落
- **猛烈撞击检测**：`get_vib_value() > 10.0` → 停桨
- **大倾角保护**：倾角超 60° 持续 5s（或倾倒）→ 停桨
- **定位丢失**：GNSS 丢失 → 强制切回手动姿态模式
- **通信丢失**：地面站/遥控断开 → 自动返航

---

## 8. 配置系统

### 8.1 参数的三层存储

1. **编译期**（`config.h`）：传感器开关、串口分配、PWM 频率等，需重编译
2. **运行期默认**（`common.h` 的 `parameter` 结构体）：PID 增益、速度限制等默认值
3. **持久化**（FRAM / Flash）：`FRAM_Init` + `update_dataflash` 从非易失存储读回，地面站修改后写回

### 8.2 通信协议

- **Mavlink**：与地面站（QGC/Mission Planner）通信，`send_mavlink_*` 系列函数
- **SBUS**：遥控器输入
- **WiFi/ESP**：`use_mlink_esp(MAVLINK_COMM_4)`
- **UWB**：`DEV_COMM` 自定义协议

---

## 9. 学习路线建议

结合官网文档（develop-FanciSwarm-after1120）交叉学习：

| 阶段 | 内容 | 文件 |
|---|---|---|
| 0 | 建立心智模型：程序怎么跑起来 | main.c → freertos.c |
| 1 | 用 demo 上手 API | demo_filter/pid/ekf/uwb/fdcan.cpp |
| 2 | 啃飞行模式状态机 | mode_stabilize → althold → poshold → autonav |
| 3 | 啃核心胶水 | maincontroller.cpp（按功能块读） |
| 4 | 串起来画数据流图 | 对照本文档第 4 节 |
| 5 | 对照 ArduPilot 学算法原理 | 开源 ArduPilot 源码 |

> [!note] 学习方法建议
> 1. 先画"数据流图"再读代码，每段代码都能挂到图上
> 2. 从 400Hz 主循环往回追，比按文件顺序读高效
> 3. 用 IDE 的"跳转定义"：跳不到的（进 .a）就是闭源库，当黑盒
> 4. EKF/PID 数学细节看不到，重点学"怎么调用、传什么参数、输出怎么用"

---

*本文档整理自 FanciSwarm 源码，核心算法库为闭源，学习重点在飞控架构与调用方式。*
