---
title: UWB 超宽带定位
tags: [传感器, UWB, 超宽带, 定位, DW1000, 三边定位]
芯片: STM32H743
接口: SPI4
---

# UWB 超宽带定位

> UWB（Ultra-Wideband，超宽带）定位是集群无人机的**室内/高精度定位方案**。相比 GNSS 只能在室外，UWB 在室内也能提供**厘米级**定位。本固件使用 **Decawave DW1000**（现属 Qorvo）芯片。

## 一、原理介绍

### 1.1 什么是 UWB

UWB 是一种**超宽带无线通信技术**，用纳秒级极窄脉冲传输数据，带宽 >500MHz。定位精度远高于 WiFi/蓝牙（米级）和 GNSS（室外米级），可达 **10cm 以内**，且抗多径干扰强。

### 1.2 测距原理：双向测距（TWR）

DW1000 测距用 **TWR（Two-Way Ranging）**，通过记录收发时间戳计算飞行时间：

```
设备A: 发送 Poll（记录 Tsp）
设备B: 收到 Poll（记录 Trp），发送 Response（记录 Tsr）
设备A: 收到 Response（记录 Trr），发送 Final（记录 Tsf）
设备B: 收到 Final（记录 Trf）

飞行时间 ToF = (Ra·Rb - Da·Db) / (Ra + Rb + Da + Db)
其中 Ra=Trr-Tsp, Rb=Trf-Tsr, Da=Tsf-Trp, Db=Tsr-Trp

距离 d = ToF × c
```

这就是 **DS-TWR（Double-Sided TWR）**，通过两次往返消除时钟偏差。

### 1.3 三边定位（Trilateration）

单次测距只能得到距离，要得到**位置**，需要至少 3 个已知坐标的**锚点（Anchor）**：

```
已知锚点 A(x1,y1,z1), B(x2,y2,z2), C(x3,y3,z3)
测得标签到各锚点的距离 d1, d2, d3

解方程组：
(x-x1)²+(y-y1)²+(z-z1)² = d1²
(x-x2)²+(y-y2)²+(z-z2)² = d2²
(x-x3)²+(y-y3)²+(z-z3)² = d3²
```

本固件默认 **4 个锚点**（`ANCHOR_MAX_NUM=4`），`trilateration.h` 提供了 `GetLocation()` 三边定位算法（开源）。

## 二、硬件接口

DW1000 挂在 **SPI4**（外部扩展 SPI）上：

```
PE2 → SPI4_SCK
PE5 → SPI4_MISO
PE6 → SPI4_MOSI
```

SPI4 配置（[spi.c:120-159](Core/Src/spi.c#L120-L159)）：Mode 0（POLARITY_LOW, PHASE_1EDGE），分频 32（96/32=3MHz），DMA 收发。

## 三、数据结构（开源）

`uwb/uwb.h` 定义了 `UWB` 类（**完整的头文件开源**）：

```cpp
class UWB {
public:
    // DW1000 通信配置（EVK1000 默认 mode 3）
    dwt_config_t config = {
        5,              // 信道 5（6489.6 MHz）
        DWT_PRF_64M,    // 脉冲重复频率 64MHz
        DWT_PLEN_1024,  // 前导码长度
        DWT_PAC32,      // 前导码捕获块
        9, 9,           // 收发前导码
        1,              // 非标准 SFD
        DWT_BR_110K,    // 数据率 110kbps
        DWT_PHRMODE_STD,// 物理层头模式
        (1025 + 64 - 32) // SFD 超时
    };

    int Anchordistance[4];    // 到 4 个锚点的距离
    Vector3f uwb_position;    // 解算出的位置
    uint8_t TAG_ID;           // 标签 ID

    // 测距帧（802.15.4e 标准）
    uint8_t tx_poll_msg[8] = {0x41,0x88,0,0x0,0xDE,0x21,0,0};
    uint8_t tx_final_msg[20]= {0x41,0x88,0,0x0,0xDE,0x23,0,...};
    // ...

private:
    // 测距状态机
    typedef enum { idle, receive, poll, resp, final, dis, release, ... } uwb_states;
    vec3d AnchorList[4];       // 锚点坐标
    // 40 位时间戳
    uint64_t poll_rx_ts, resp_tx_ts, final_rx_ts;
    // ...
};
```

> [!tip] DW1000 时间戳
> DW1000 用 **40 位**时间戳，最小单位约 15.65ps（`DWT_TIME_UNITS`）。测距精度取决于时间戳精度，这就是 UWB 能到厘米级的原因。

## 四、驱动代码解读（含开源测距算法）

### 4.1 驱动函数

```c
bool uwb_init(void);        // UWB 初始化（返回 false 则终止 uwbTask）
void uwb_update(void);      // UWB 主循环（测距 + 定位）
void uwb_position_update(void);  // 位置解算
```

### 4.2 调用流程

**初始化**（[freertos.c InitTask](Core/Src/freertos.c#L333-L335)）：

```cpp
if(!uwb_init()){
    osThreadTerminate(uwbTaskHandle);  // 初始化失败则终止 UWB 任务
}
```

**主循环**（[freertos.c UWBTask](Core/Src/freertos.c#L600-L615)）：

```cpp
void UWBTask(void *argument){
    // ...
    for(;;){
        uwb_update();   // 持续测距、定位
    }
}
```

### 4.3 TWR 测距状态机（demo_uwb.cpp 开源）

[demo_uwb.cpp](Maincontroller/demo/demo_uwb.cpp) 完整展示了 **DS-TWR 测距流程**（开源，最有学习价值）：

```cpp
// 测距发送端（Tag 侧）状态机
void uwb_range_tx(void){
    switch(uwb_state){
    case idle:
        dwt_setrxaftertxdelay(POLL_TX_TO_RESP_RX_DLY_UUS);  // 设置响应延迟
        dwt_setrxtimeout(RESP_RX_TIMEOUT_UUS);
        uwb_state = poll;
    case poll:
        // 发送 Poll 帧
        dwt_writetxdata(sizeof(tx_poll_msg), tx_poll_msg, 0);
        dwt_starttx(DWT_START_TX_IMMEDIATE | DWT_RESPONSE_EXPECTED);
        uwb_state = resp;
    case resp:
        // 等待 Response，记录时间戳
        poll_tx_ts_tag = uwb->get_tx_timestamp_u64();
        resp_rx_ts_tag = uwb->get_rx_timestamp_u64();
        // 计算 Final 发送时间，写入时间戳
        uwb_state = final;
    case final:
        // 发送 Final 帧（含 3 个时间戳）
        uwb_state = dis;
    case dis:
        // 接收 Distance 帧，得到距离
        distance_temp = (rx_buffer[6]*100 + rx_buffer[7]);
        uwb_state = idle;
    }
}
```

**接收端（Anchor 侧）** 计算 ToF 的核心公式（[demo_uwb.cpp:430-442](Maincontroller/demo/demo_uwb.cpp#L430-L442)）：

```cpp
Ra = (double)(resp_rx_ts - poll_tx_ts);
Rb = (double)(final_rx_ts_32 - resp_tx_ts_32);
Da = (double)(final_tx_ts - resp_rx_ts);
Db = (double)(resp_tx_ts_32 - poll_rx_ts_32);
tof_dtu = (Ra * Rb - Da * Db) / (Ra + Rb + Da + Db);  // DS-TWR 公式
tof = tof_dtu * DWT_TIME_UNITS;
distance = tof * SPEED_OF_LIGHT;  // 光速换算距离
```

### 4.4 三边定位算法（trilateration.h 开源）

`trilateration.h` 提供定位算法：

```c
int GetLocation(vec3d *best_solution, int use4thAnchor,
                vec3d *anchorArray, int *distanceArray);  // 三边/四边定位
```

锚点坐标在 `common.h` 配置（默认 4 个锚点）：

```c
#define UWB_POS1_X 80.0f   UWB_POS1_Y -80.0f  UWB_POS1_Z 160.0f
#define UWB_POS2_X 0.0f    UWB_POS2_Y 400.0f  UWB_POS2_Z 170.0f
#define UWB_POS3_X 400.0f  UWB_POS3_Y 400.0f  UWB_POS3_Z 170.0f
#define UWB_POS4_X 400.0f  UWB_POS4_Y 0.0f    UWB_POS4_Z 170.0f
```

### 4.5 数据流

```
DW1000 ──► TWR测距 ──► Anchordistance[4] ──► 三边定位 GetLocation() ──► uwb_position ──► 位置估计
```

## 五、UWB 工作模式

`common.h` 定义了 UWB 的多种工作模式：

```c
typedef enum {
    none=0, tag=1, anchor, range, comm_tx, comm_rx
} uwb_modes;
```

- **tag**：标签模式（无人机侧，主动测距定位）
- **anchor**：锚点模式（固定基站，响应测距）
- **comm_tx/comm_rx**：纯通信模式（不用来定位，用于编队间通信）

## 六、要点总结

1. UWB 是**室内/高精度定位**方案，精度厘米级，弥补 GNSS 的室外限制
2. **DS-TWR 双向测距**消除时钟偏差，`demo_uwb.cpp` 的测距状态机是开源的学习范本
3. **三边定位**（trilateration.h）用 3-4 个锚点解算位置
4. 数据率 110kbps、信道 5、PRF 64MHz 是 DW1000 的典型配置
5. UWB 还能用于**编队间通信**（comm_tx/rx 模式），这是集群无人机的特色
