---
tags:
  - PX4
  - HKUST
  - 硬件
  - 原理图
  - STM32H743
  - 传感器
  - 电源
date: 2026-08-20
---

# 硬件解读：NxtPX4-DUAL 飞控板原理图

> 本文解读 `~/桌面/PX4_HKUST/Nxt-FC-Hardware/NxtFC-DUAL/` 的 KiCad 工程（`NxtPX4.pdf` 即 `NxtPX4.kicad_sch` 导出的 A2 图纸），讲清这块 HKUST 自研飞控板的 **电源架构**、**传感器选型**、**外设接口** 与 **关键设计细节**。结论由原理图 + PCB 网络表（`NxtPX4.kicad_pcb`）三方核对得出。软件侧配套看 [[代码解读]]。

---

## 一、整体定位

| 项目 | 内容 |
|------|------|
| 主控 | **STM32H743VIH6**（Cortex-M7，480MHz） |
| 板子名 | NxtPX4-DUAL（rev V1.0.1，`HKUST Khalil`） |
| 定位 | PX4 飞控，**2S–6S** 电池输入，适配 **DJI O3 天空端** |
| `dual` 含义 | **双 IMU**：两片 BMI088（SPI1 + SPI4 各一套，冗余） |
| 图纸形式 | 单张 A2 大图，无分页层级（KiCad Eeschema 生成） |

三句话概括：**一颗 H743 主控 + 双 BMI088 冗余 IMU + 一颗气压计**，电源从 2S–6S 电池降成 5V/9V/3.3V 三轨，接口按 PX4 标准接出一圈（GPS/ESC/UART/PWM/Debug/MicroSD），并专为 DJI O3 做了供电与连接器。

---

## 二、电源架构（核心）

### 2.1 电源树总览

全局电源轨有 7 条：`VCC_BAT` → `VCC_24V` → `System_5V` / `CAM_9V` → `Core_3V3` / `Periph_3V3`，外加 USB 轨 `System_4V5`。

| 转换 | 芯片 | 输入 → 输出 | 用途 |
|------|------|-------------|------|
| Buck | U4 `MP9943GQ-Z` | VCC_24V → **System_5V** | 5V / 3A，供 Core + Periph |
| Buck | U1 `MP9943GQ-Z` | VCC_24V → **CAM_9V** | 9V / 3A（可调 9/10/12V），驱动摄像头 / O3 |
| LDO | U2 `RT9013-33GB` | System_5V → **Core_3V3** | 3.3V / 500mA，MCU + IMU |
| LDO | U3 `RT9013-33GB` | System_5V → **Periph_3V3** | 3.3V / 500mA，TF 卡 / QSPI / OSD |

- 两个 MP9943 都是「2S–6S 输入」的宽压 Buck。**5V 轨**反馈电阻 R11=`7.68kR`（注释 "7.68kR 5V output"）；**CAM_9V 轨**反馈电阻 R16=`2.90k–4.02kR`，靠它调压（注释 "3.54k→10V / 4.02k→9V / 2.9k→12V"）。
- 两个 RT9013 是 3.3V 500mA 的 LDO，分别给「MCU+IMU」和「TF 卡/QSPI/OSD」供电，把模拟噪声敏感的传感器电源与外设电源隔开。

### 2.2 输入保护（BAT power protection）

电池入口（`VCC_BAT`）先过一组保护，再汇成 `VCC_24V`：

| 器件 | 型号 | 作用（原理图注释） |
|------|------|-------------------|
| Q1 | `AP40P05`（P-MOS） | **防反接** |
| D3 | `SMF30CA`（30V TVS） | **冲击保护**（浪涌钳位） |
| D6 | `BZX584C9V1`（9.1V 稳压管） | 栅极钳位 / 稳压 |
| D5、D7 | `DSS34`（3A 肖特基） | 隔离 / 续流 |
| D1 / D2 / D4 | `PESD12VV1BL` / `PESD5V0F1BL` | 接口 ESD |

> 支持 **2S–6S**（约 6–25.2V），标称 `VCC_24V`。注释里 "此处的电流一般不会超过 5A"。

### 2.3 双路供电汇合（电池 vs USB）

- 电池路径：`VCC_BAT → VCC_24V → System_5V`（5V）
- USB 路径：Type-C `VBUS`（5V）经肖特基压降 → `System_4V5`（≈4.5V）

两路通过二极管 OR 汇合，**谁电压高谁供电**，因此只插 Type-C 也能给板子供电跑起来（注释 "Type-C power"）。

---

## 三、主控与传感器

### 3.1 主控 STM32H743VIH6

一颗 `STM32H743VIH6`（100 引脚），符号拆成多单元（U11A/B/C）绘制。外接 16MHz 主晶振 Y1（`SG-210STF`）+ 32.768kHz 低速晶振 Y2（`ABS07`，供 RTC）。

### 3.2 双 IMU：BMI088 ×2

U14 / U15 两片 **BMI088**（六轴：加速度计 + 陀螺仪），分别挂 **SPI1** 和 **SPI4**，这就是 `dual` 的来源。

每片 BMI088 用 **2 个片选 + 2 个数据就绪中断**（注释 "ADR for accel data ready / GDR for gyro data ready / ACS for accel cs / GCS for gyro cs"）：

| IMU | SPI | 片选（accel / gyro） | 数据就绪（ADR / GDR） |
|-----|-----|---------------------|----------------------|
| BMI088_0 (U14) | SPI1 | PA2 / PA3 | PA0 / PA1 |
| BMI088_1 (U15) | SPI4 | PC13 / PC2 | PE4 / PE3 |

### 3.3 气压计 SPL06

U13 `SPL06`，挂 **I2C1**（`Baro_I2C1_SDA`=PB7、`Baro_I2C1_SCL`=PB6，数据就绪 `SDO`=PB5）。

地址由 SDO 引脚电平决定（注释原文）：

```
SDO To VSS    → ADDR 0x1110110 (0x76)
SDO To VDDIO  → ADDR 0x1110111 (0x77)
CSB To VDDIO  → 进入 I2C 模式
```

---

## 四、存储

| 器件 | 型号 | 接口 | 说明 |
|------|------|------|------|
| U16 | `W25Q128JVS` | SPI2（QSPI 兼容） | 128Mbit NOR Flash，参数/日志存储 |
| J10 | Micro SD 卡座 | SDMMC1 | 大容量日志 |

> **命名坑**：Flash 的四根网络标号叫 `FRAM_SPI2_*`（历史命名），图纸注释也写 "FRAM / QSPI"。**实际芯片是 W25Q128 NOR Flash**（引脚 `DI(IO0)/DO(IO1)/CLK/CS`，支持 QSPI），板上并没有真正的 FRAM。对应引脚：SCK=PD3、CS=PD4、MOSI=PC3、MISO=PB14。

---

## 五、外设接口（连接器）

| 连接器 | 功能 |
|--------|------|
| J1 | USB Type-C（USB2.0，OTG + 供电） |
| J2 | **DJI O3 天空端**（6pin） |
| J3 | GPS / 磁力计（6pin） |
| J4 | UART2 串口（4pin） |
| J5、J7 | 上位机连接器（4pin ×2） |
| J6 | ESC 电调（8pin） |
| J8 | PWM_AUX 输出（5pin） |
| J9 | 通用（4pin） |
| J10 | Micro SD 卡座 |
| J11 | Debug / SWD（6pin） |
| J12 | AuxGPIO（6pin） |
| H1–H4 | 安装孔 |
| FID1–3 | SMT 定位 Fiducial |

---

## 六、MCU 外设资源占用

由网络标号 + PCB 引脚核对：

| 外设 | 网络标号 | 引脚 | 用途 |
|------|----------|------|------|
| I2C1 | `Baro_I2C1_SDA/SCL` | PB7 / PB6 | 气压计 SPL06 |
| I2C4 | `EX_I2C4_SDA/SCL` | PD13 / PD12 | 外部扩展（GPS/磁力计） |
| SPI1 | `BMI088_0_*` | CS PA3/PA2，DRDY PA1/PA0 | IMU0 |
| SPI4 | `BMI088_1_*` | CS PC2/PC13，DRDY PE3/PE4 | IMU1 |
| SPI2 | `FRAM_SPI2_*` | CS PD4 | W25Q128 Flash |
| SPI3 | `EX_SPI3_*` | — | 外部扩展（预留） |
| SDMMC1 | `SDMMC1_D0–D3/CK/CMD` | — | Micro SD |
| USB OTG FS | `USB_OTG_FS_DP/DM`、`VBUS` | — | Type-C |
| UART/USART | USART1/2/3、UART4/5/6/7/8 | — | 串口（含 SBUS） |
| TIM1/2/3 | `TIM1_CH1–4`、`TIM2_CH3/4`、`TIM3_CH3/4` | — | ESC / PWM 输出（8 路） |
| ADC | `ADC1_INP4_BATT_V`、`ADC2_INP8_BATT_I` | — | 电池电压 / 电流采样 |
| SWD | `SWDIO`、`NRST` | — | 调试 |

---

## 七、关键设计细节

1. **防反接**：P-MOS `AP40P05`（Q1）在电池入口做反接保护。
2. **SBUS 反相**：`SN74LVC1G86DCKR`（U19，异或门）把 SBUS 信号反相后送入 UART5 RX（注释 "If 0R is connected UART5 RX is used as SBUS"，靠 0Ω 电阻切换）。
3. **DJI O3 适配**：J2 专接 O3，`CAM_9V` 为其供电；注释 "板子适配 DJI O3 天空端"、"没有使用，将滤波工作交给 O3"（板载大滤波电容让位，交给 O3 自滤波）。
4. **bootloader 同步按键**：BOOT1（`TD-1183SN-1.5H` 按键）+ 注释 "power up signal to trigger qgroundcontrol bootloader sync"。
5. **电池检测**：`ADC1_INP4_BATT_V` 测电压、`ADC2_INP8_BATT_I` 测电流（注释 "BAT_Voltage_Detection"），经 R13/R17 41.2k、R14 200k 等分压电阻进 ADC。
6. **串口复用提示**：注释 "If want to utilize UART8 in PX4, make UART6 as the debug port"。

---

## 八、硬件 ↔ 软件 BSP 对应

与 [[代码解读]] 里 `boards/hkust/nxt-dual/` 的软件配置一一对照：

| 硬件（原理图） | 软件 BSP |
|----------------|----------|
| SPI1 → BMI088_0 | `spi.cpp`：`initSPIBus(SPI1, {gyr CS PA3 DRDY PA1, acc CS PA2 DRDY PA0})` |
| SPI4 → BMI088_1 | `spi.cpp`：`initSPIBus(SPI4, {gyr CS PC2 DRDY PE3, acc CS PC13 DRDY PE4})` |
| SPI2 → W25Q128（CS PD4） | `spi.cpp`：`initSPIDevice(SPIDEV_FLASH(0), CS PD4)` |
| I2C1 → SPL06 气压计 | `rc.board_sensors`：`spl06 -X -a 0x77 start` |
| I2C1 / I2C4 → 外部 | `i2c.cpp`：`initI2CBusExternal(1)`、`initI2CBusExternal(4)` |
| 16MHz Y1 / 32.768kHz Y2 | 系统时钟（HSE）/ RTC（LSE） |

> **注意**：软件侧 `i2c.cpp` 把 I2C1、I2C4 都声明成 `external`，气压计用 `spl06 -X -a 0x77` 启动（`-X` = 自动扫外部总线），**不绑定具体总线号**，所以驱动能自动在 I2C1 上找到它。物理上气压计在 **I2C1**（PB6/PB7），外部扩展在 **I2C4**（PD12/PD13）。

---

## 九、总结

> **一颗 STM32H743VIH6，双 BMI088 冗余 IMU（SPI1/SPI4），一颗 SPL06 气压计（I2C1），W25Q128 存参数、MicroSD 存日志；电源从 2S–6S 经「防反接 + TVS」保护后，两级降压出 5V / 9V（摄像头）/ 3.3V×2，Type-C 可单独供电；一圈标准 PX4 接口 + 专为 DJI O3 做的供电与连接器。**

对照软件侧结论：硬件原理图上的「总线 + 片选 + 数据就绪」映射，就是 [[代码解读]] 里 `spi.cpp` / `i2c.cpp` 三张配置表的来源；读懂这张原理图，再读那三张表就毫无障碍。
