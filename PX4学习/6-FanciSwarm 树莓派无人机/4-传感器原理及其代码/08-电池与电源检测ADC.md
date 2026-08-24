---
title: 电池与电源检测（ADC）
tags: [传感器, ADC, 电压检测, 电流检测, 电池, 低电量保护]
芯片: STM32H743
接口: ADC1（内部）
---

# 电池与电源检测（ADC）

> ADC（模数转换器）检测电池**电压**和**电流**，是飞控**电源管理**的核心。低电量保护（返航/降落）、剩余电量估算都依赖它。这不是"外部传感器"，而是 STM32 内部的模拟量采集。

## 一、原理介绍

### 1.1 ADC 原理

ADC 把连续的模拟电压转换成数字量：

```
数字值 = (模拟电压 / 参考电压) × (2^分辨率 - 1)
```

STM32H743 的 ADC 是 **16 位**逐次逼近型（SAR）ADC，转换精度高。

### 1.2 电压/电流检测电路

电池电压通常高于 ADC 量程（3.3V），需要**分压电阻**：

```
电池电压 → 分压电阻（如 1:10）→ ADC 引脚 → 数字值 × 分压比 = 实际电压
```

电流通过**采样电阻**（shunt resistor）检测：电流流过小电阻产生压降，测压降 ÷ 阻值 = 电流。

## 二、硬件接口

ADC1 配置了 **4 个通道**，DMA 循环采集（[Core/Src/adc.c](Core/Src/adc.c)）：

| ADC 通道 | 引脚 | 检测对象 | 说明 |
|---|---|---|---|
| CH8 | PC5 | `VDD_5V_SENS` | 板载 5V 供电电压 |
| CH14 | PA2 | `BAT_V_SENS` | 电池电压（分压后） |
| CH15 | PA3 | `BAT_C_SENS` | 电池电流（采样电阻） |
| CH18 | PA4 | `P_SENS` | 外部模拟输入（备用） |

ADC1 配置（[adc.c:47-66](Core/Src/adc.c#L47-L66)）：

| 参数 | 值 | 含义 |
|---|---|---|
| Resolution | 16bit | 16 位分辨率 |
| ScanConvMode | ENABLE | 扫描模式（多通道） |
| ContinuousConvMode | ENABLE | 连续转换 |
| NbrOfConversion | 4 | 4 个通道 |
| ConversionDataManagement | DMA_CIRCULAR | DMA 循环搬运 |
| ClockPrescaler | DIV256 | 降低 ADC 时钟保证精度 |

> [!tip] DMA 循环采集
> ADC 以 DMA 循环模式自动采集 4 个通道，结果写入内存缓冲区，CPU 不干预。这是 STM32 多通道 ADC 的标准做法。

## 三、驱动函数（hal.h 声明）

```c
void adc_init(void);           // ADC 初始化
void adc_update(void);         // 刷新全部 ADC 数据
float get_batt_volt(void);     // 电池电压 V
float get_batt_current(void);  // 电池电流 A
float get_5v_in(void);         // 板载 5V 电压（0~6.6V）
float get_pressure(void);      // 外部模拟输入（0~6.6V）
```

## 四、调用流程

**初始化**（[freertos.c InitTask](Core/Src/freertos.c#L317)）：

```cpp
adc_init();
```

**数据采集**（[freertos.c Loop200hzTask](Core/Src/freertos.c#L385-L405)，200Hz）：

```cpp
void Loop200hzTask(void *argument){
    // ...
    adc_update();   // 200Hz 刷新电压电流
}
```

> [!note] 硬件层面的 ADC 初始化
> `MX_ADC1_Init()`（[adc.c](Core/Src/adc.c)）由 CubeMX 生成，在 main.c 里调用，做底层寄存器配置 + 校准（`HAL_ADCEx_Calibration_Start`）。`adc_init()` 是应用层二次初始化（分压比换算等，闭源）。

## 五、在飞控中的应用——低电量保护

电池电压在飞行模式中被大量用于安全判断：

```cpp
// 低电量 → 强制返航（mode_poshold.cpp）
if(get_batt_volt() < param->lowbatt_return_volt.value){
    execute_return = true;
}
// 电量过低 → 强制降落
if(get_batt_volt() < param->lowbatt_land_volt.value){
    execute_land = true;
}
```

相关参数（`common.h`）：

```c
#define LOWBATT_RETURN_VOLT 0.0f   // 低电量返航电压阈值
#define LOWBATT_LAND_VOLT   6.6f   // 低电量降落电压阈值（默认 6.6V）
#define VOLTAGE_GAIN 1.0f          // 电压增益（校准）
#define CURRENT_GAIN 1.0f          // 电流增益（校准）
```

## 六、要点总结

1. ADC 检测电压/电流，是**低电量安全保护**的基础
2. 16 位 + DMA 循环采集 4 通道，是 STM32 多通道 ADC 的标准配置
3. 电压分压、电流采样电阻的**增益校准**（VOLTAGE_GAIN/CURRENT_GAIN）决定读数准确性
4. `get_batt_volt()` 贯穿所有飞行模式的安全判断
