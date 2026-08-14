---
title: GNSS 定位模块
tags: [传感器, GNSS, GPS, 北斗, RTK, UM482, ublox]
芯片: STM32H743
接口: UART3
---

# GNSS 定位模块

> GNSS（全球导航卫星系统）提供**绝对位置**（经纬度、高度、速度），是定点悬停、航线飞行、返航的基础。本固件支持 **UM482**（和芯星通，双天线 RTK）和 **ublox**（通用 GNSS 模组）。

## 一、原理介绍

### 1.1 卫星定位原理

GNSS 接收机接收多颗卫星的广播信号，通过**测距定位**：

1. 每颗卫星广播自己的位置和精确时间
2. 接收机测出到每颗卫星的**伪距**（信号传播时间 × 光速）
3. 至少 4 颗卫星 → 解算接收机的三维坐标 + 钟差

```
伪距 = sqrt((x-x_s)² + (y-y_s)² + (z-z_s)²) + c·Δt
```

### 1.2 定位精度等级

| 精度等级 | 技术 | 精度 | 说明 |
|---|---|---|---|
| 单点定位 | 纯卫星伪距 | 米级（2-10m） | 常规 GPS |
| **RTK** | 载波相位差分 | **厘米级** | 需要基站，高精度 |
| 伪距差分 | 差分改正 | 亚米级 | 需要基准站 |

**RTK（Real-Time Kinematic）**：基站 + 流动站同时观测，利用载波相位差分消除大气误差，达到厘米级定位。集群无人机编队飞行需要 RTK 精度。

### 1.3 双天线定向

UM482 支持**双天线**，通过两根天线的基线向量计算**航向（heading）**，比磁罗盘更精确（不受磁场干扰）。数据结构的 `baseline_n/e/u`（基线）和 `heading`（定向角）就是双天线输出。

### 1.4 本固件支持的型号

| 型号 | 厂商 | 特点 |
|---|---|---|
| **UM482** | 和芯星通 | 全系统全频点，双天线 RTK 定向定位 |
| **ublox** | u-blox | 通用 GNSS（NEO-M8N 等），UBX 协议 |

## 二、硬件接口

GNSS 挂在 **UART** 上，config.h 里 `COMM_3 = GPS_COMM`：

```
串口3（UART，115200 波特率）──► GNSS 模组
```

config.h 相关配置：

```c
#define COMM_3 GPS_COMM        // 串口3 配置为 GPS 模式
#define COMM_3_BANDRATE 115200 // 波特率
```

> [!note] 波特率注意
> `config.h` 注释明确：最新版本固件的串口波特率需在 APP 上配置，`COMM_x_BANDRATE` 宏可能失效。

## 三、数据结构（开源部分）

`gnss/gps.h` 定义了完整的数据结构（**这是开源的**）：

```c
// 定位结果
typedef struct {
    uint64_t timestamp;
    uint64_t time_utc_usec;      // UTC 时间
    uint16_t year; uint8_t month/day/hour/min/sec;
    int32_t lat;                 // 纬度（1e-7 度）
    int32_t lon;                 // 经度
    int32_t alt;                 // 海拔
    int32_t alt_ellipsoid;       // 椭球高度
    float s_variance_m_s;        // 速度方差
    float eph, epv;              // 水平/垂直精度因子
    float hdop, vdop;            // DOP 值
    float vel_m_s;               // 速度
    float vel_n_m_s, vel_e_m_s, vel_d_m_s;  // NED 速度分量
    uint8_t fix_type;            // 定位类型
    uint8_t heading_status;      // 定向状态（0无效/4固定/5浮动）
    uint8_t satellites_used;     // 使用卫星数
    uint8_t gps_used, bds_used, glo_used;  // 各系统卫星数
    float baseline_n, baseline_e, baseline_u; // 双天线基线
    float heading;               // 双天线定向角（度）
} vehicle_gps_position_s;

// 卫星信息
typedef struct {
    uint8_t count;
    uint8_t svid[20];       // 卫星号
    uint8_t elevation[20];  // 仰角
    uint8_t azimuth[20];    // 方位角
    uint8_t snr[20];        // 信噪比
} satellite_info_s;
```

## 四、驱动代码解读（含开源解析）

### 4.1 驱动函数

```c
bool Gnss_Init(GnssType type);      // 初始化（type: UBLOX/UM482）
void gnss_update(void);              // 更新（循环调用）
void get_gnss_data(uint8_t buf);     // 逐字节喂给解析器
bool get_gnss_state(void);           // 获取定位状态
void ParseUM482(uint8_t b);          // UM482 协议解析（开源！）
```

### 4.2 调用流程

**初始化 + 数据采集**（[freertos.c GnssTask](Core/Src/freertos.c#L558-L591)，50Hz）：

```cpp
void GnssTask(void *argument){
    // 根据 config 选择 GNSS 挂在哪个串口
    if(COMM_3==GPS_COMM) set_gnss_comm(gnss_comm3);
    // ...
    Gnss_Init(UM482);        // 初始化 UM482（会阻塞 10s 等待锁定）
    // ...
    for(;;){
        // LED 指示定位状态
        FMU_LED5_Control(get_gnss_state());
        gnss_update();       // 解析并更新 GNSS 数据
        osDelay(20);         // 50Hz
    }
}
```

### 4.3 UM482 协议解析（开源状态机）

`gnss/um482.h` 定义了完整的**字节流解析状态机**（这是开源的驱动解析逻辑）：

```c
typedef enum {
    UM482_IDLE = 0,        // 空闲
    UM482_Header1_24,      // 帧头
    UM482_Header2_24,
    UM482_Header3_24,
    UM482_Gnss_4,          // 消息类型
    UM482_Length_1,        // 长度
    UM482_UTC_6,           // UTC 时间（年月日时分秒）
    UM482_RTK_Status_1,    // RTK 状态（0无效/1单点/2伪距差分/4固定解/5浮动解）
    UM482_Heading_Status_1,// 定向状态
    UM482_GPS_Num_1,       // GPS 卫星数
    UM482_BDS_Num_1,       // 北斗卫星数
    // ... 逐字段解析 ...
    UM482_Lat_8,           // 纬度（8字节）
    UM482_Lon_8,           // 经度
    UM482_Alt_8,           // 高度
    // ...
    UM482_CRC_4            // CRC 校验
} UM482_State;
```

> [!tip] 状态机解析思路
> `ParseUM482(uint8_t b)` 逐字节喂数据，状态机按枚举顺序前进：找帧头 → 读长度 → 按字段长度逐个取数据 → CRC 校验。这是串口协议解析的经典实现。ublox 同理有 `ublox.h`（UBX 协议，578 行）。

### 4.4 数据流

```
GnssTask ──► gnss_update() ──► gps_position ──► EKF_GNSS ──► 位置(xy)
```

在 [Loop400hzTask](Core/Src/freertos.c#L371)：

```cpp
ekf_gnss_xy();   // GNSS XY 融合
```

EKF_GNSS 对象（[maincontroller.cpp](Maincontroller/src/maincontroller.cpp#L117)）：

```cpp
EKF_GNSS *ekf_gnss = new EKF_GNSS(_dt, 0.0016, 0.0016, 0.0016, 0.0016, 0.000016, ...);
```

## 五、在飞行控制中的作用

- **定点悬停（PosHold）**：GNSS 提供 XY 位置，`get_gnss_state()` 判断定位是否有效
- **返航（Return）**：记录起飞点 `gnss_origin_pos`，返航时飞回
- **航线（Mission）**：`gnss_point_prt` 航点数组，逐点飞行
- **定位丢失保护**：`!get_gnss_state()` 时强制切回手动姿态模式

## 六、要点总结

1. GNSS 提供绝对位置，RTK 达到厘米级（集群编队必需）
2. UM482 双天线还能定向（heading），比磁罗盘更准
3. **协议解析（UM482/ublox）是开源的**，可深入学习串口状态机解析
4. 数据经 `EKF_GNSS` 与 IMU 融合，得到平滑的 XY 位置
5. 定位状态（fix_type/RTK status）决定飞控能否进入自主模式
