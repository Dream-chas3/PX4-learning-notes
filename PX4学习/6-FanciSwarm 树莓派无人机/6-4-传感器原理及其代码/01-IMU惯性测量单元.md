---
title: IMU 惯性测量单元
tags: [传感器, IMU, 陀螺仪, 加速度计, MPU6000, MPU9250, ICM20608]
芯片: STM32H743
接口: SPI1
---

# IMU 惯性测量单元

> IMU（Inertial Measurement Unit）是飞控**最核心的传感器**，提供三轴加速度和三轴角速度，是姿态解算（AHRS）的唯一高频输入。

## 一、原理介绍

### 1.1 什么是 IMU

IMU 由 **加速度计（Accelerometer）** + **陀螺仪（Gyroscope）** 组成（9 轴 IMU 再叠加磁力计）：

| 元件 | 测量量 | 物理原理 | 作用 |
|---|---|---|---|
| 加速度计 | 比力（重力+运动加速度）m/s² | MEMS 电容式：质量块在加速度下位移 → 电容变化 → 电压 | 估计水平姿态（roll/pitch）、重力方向 |
| 陀螺仪 | 角速度 °/s | 科里奥利效应：振动质量块在旋转时产生横向力 | 积分得到角度，短期姿态估计 |

> [!note] 互补关系（飞控经典思路）
> - **陀螺仪**：短期准、长期漂移（积分误差累积）
> - **加速度计**：长期准（重力方向稳定）、短期噪声大（振动敏感）
> - 两者通过 **滤波融合**（本固件用 EKF 四元数）取长补短。

### 1.2 本固件支持的 IMU 型号

| 型号 | 厂商 | 轴数 | 说明 |
|---|---|---|---|
| **MPU6000** | InvenSense（TDK） | 6 轴（acc+gyro） | 经典飞控 IMU，SPI 接口 |
| **MPU9250** | InvenSense（TDK） | 9 轴（acc+gyro+mag） | MPU6000 + 内置磁力计 |
| **ICM20608** | InvenSense（TDK） | 6 轴 | MPU6000 的升级版，性能更好 |
| **L3GD20** | ST | 3 轴（仅陀螺仪） | 独立陀螺仪（备用/冗余） |

## 二、硬件接口

IMU 挂在 **SPI1** 上（[Core/Src/spi.c](Core/Src/spi.c)）：

```
PA5  → SPI1_SCK   （时钟）
PA6  → SPI1_MISO  （主入从出）
PA7  → SPI1_MOSI  （主出从入）
MPU_CS → 片选（软件控制，见 hal.h 的 MPU_CS_H/L 宏）
```

SPI1 配置（[spi.c:47-60](Core/Src/spi.c#L47-L60)）：

| 参数 | 值 | 含义 |
|---|---|---|
| Mode | Master | 主机模式 |
| DataSize | 8bit | 8 位数据 |
| CLKPolarity | LOW | 空闲低电平（Mode 0） |
| CLKPhase | 1EDGE | 第一边沿采样 |
| BaudRatePrescaler | 64 | 96MHz/64 = 1.5MHz |
| DMA | RX+TX | 收发用 DMA |

> [!tip] 片选信号
> 每个 SPI 设备（IMU 用 MPU_CS、气压计用 BARO_CS）通过独立的 GPIO 片选引脚区分，宏定义在 `hal.h`：
> ```c
> #define MPU_CS_H  HAL_GPIO_WritePin(MPU_CS_GPIO_Port, MPU_CS_Pin, GPIO_PIN_SET)
> #define MPU_CS_L  HAL_GPIO_WritePin(MPU_CS_GPIO_Port, MPU_CS_Pin, GPIO_PIN_RESET)
> ```

## 三、数据结构

`hal.h` 中定义了各 IMU 的数据结构（**这是能解读到的驱动接口，寄存器读取实现在闭源 .a 中**）：

```c
typedef struct{
    Vector3_Int16 acc_raw;   // 原始加速度计读数（寄存器原始值）
    Vector3_Int16 gyro_raw;  // 原始陀螺仪读数
    Vector3_Float accf;      // 换算后加速度 m/s²
    Vector3_Float gyrof;     // 换算后角速度 °/s
    int16_t temp;            // 温度
}MPU6000_Data;

typedef struct{              // MPU9250 比 MPU6000 多磁力计
    // ...同上...
    Vector3_Int16 mag_raw;
    Vector3_Float magf;      // 磁场 uT
}MPU9250_Data;

typedef struct{              // ICM20608 结构同 MPU9250
    // acc_raw/gyro_raw/mag_raw/accf/gyrof/magf/temp
}ICM20608_Data;

typedef struct{              // L3GD20 只有陀螺仪
    Vector3_Int16 gyro_raw;
    Vector3_Float gyrof;     // °/s
    int16_t temp;
}L3GD20_Data;
```

> 注意：原始值（`acc_raw`）→ 物理值（`accf`）的换算（量程、灵敏度、单位换算）发生在闭源驱动里。数据结构的注释已经标明了物理单位。

## 四、驱动代码解读

### 4.1 驱动函数（hal.h 声明）

```c
void IMU_Init(void);      // IMU 初始化（上电配置寄存器、量程、滤波）
void IMU_Get_Data(void);  // 读取 IMU 数据（触发 SPI 读取 + 单位换算）
```

### 4.2 调用流程

**初始化**（[freertos.c InitTask](Core/Src/freertos.c#L309-L347)）：

```cpp
void InitTask(void *argument){
    // ... 其他初始化 ...
    IMU_Init();     // 1. IMU 初始化（含自检/校准）
    MAG_Init();     // 磁罗盘初始化
    // ...
}
```

**数据采集**（[freertos.c IMUTask](Core/Src/freertos.c#L624-L640)，500Hz）：

```cpp
void IMUTask(void *argument){
    // 每 2ms 执行一次（500Hz）
    vTaskDelayUntil(&PreviousWakeTime, TimeIncrement);  // 2ms
    IMU_Get_Data();    // 读 IMU 原始数据
    BARO_Get_Date();   // 顺带读气压计
}
```

**数据消费**（[freertos.c Loop400hzTask](Core/Src/freertos.c#L356-L377)，400Hz）：

```cpp
void Loop400hzTask(void *argument){
    // ...
    ahrs_update();     // 姿态解算：IMU 数据 → EKF → 四元数姿态
    // ...
}
```

### 4.3 数据流

```
IMUTask(500Hz)                Loop400hzTask(400Hz)
IMU_Get_Data()  ──► mpu6000_data ──► 滤波(_accel_filter/_gyro_filter)
                                     ──► ahrs_update() ──► 姿态(roll/pitch/yaw)
```

关键滤波参数（[maincontroller.cpp](Maincontroller/src/maincontroller.cpp)）：

```cpp
static float accel_filt_hz = 10;   // 加速度低通滤波截止频率
static float gyro_filt_hz  = 20;   // 陀螺仪低通滤波截止频率
static LowPassFilter2pVector3f _accel_filter, _gyro_filter;  // 二阶低通
```

## 五、校准

IMU 数据使用前必须校准（[maincontroller.cpp](Maincontroller/src/maincontroller.cpp) 的校准流程）：

- **加速度计校准**：`AccelCalibrator`（`accelCalibrator` 对象）——6 面法或最小二乘，求零偏 + 比例 + 轴间耦合（`accel_offsets`/`accel_diagonals`/`accel_offdiagonals` 参数）
- **陀螺仪校准**：上电静置求零偏（`gyro_offset`），由 `GYRO_INIT_MAX_DIFF_DPS 0.1`（common.h）限制初始漂移阈值

校准参数存储在 `common.h` 的 `parameter` 结构体里，可在地面站调整。

## 六、要点总结

1. IMU 是姿态控制的唯一高频数据源，**400Hz 主循环直接依赖它**
2. 加速度计 + 陀螺仪互补融合（EKF），解决"漂移 vs 噪声"矛盾
3. 驱动实现闭源，但**数据结构（单位）+ 调用时序（500Hz 采集、400Hz 消费）+ 总线配置（SPI1）** 都可见
4. 校准（零偏/比例/耦合）是 IMU 能否正常工作的前提
