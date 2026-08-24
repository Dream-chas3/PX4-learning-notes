---
tags:
  - PX4
  - Mavlink
  - 外部通信
  - MAVLink流
  - 自定义消息
date: 2026-08-11
---

# 外部通信：Mavlink 自定义消息发送

## 一、MAVLink 概述

MAVLink（Micro Air Vehicle Link）是 PX4 与外部设备（地面站 QGC、其他无人机等）通信使用的**二进制通信协议**。它定义了消息的帧格式，将飞控的状态数据（姿态、位置、速度等）打包成标准格式发送，同时接收地面站下发的控制指令。

### 两层架构

```
┌─────────────────────────────────────────────┐
│  上层 Mavlink 模块                           │
│  ./src/modules/mavlink/                     │
│  - 提供 mavlink start/stop/status 命令       │
│  - 管理 Mavlink 实例（通道）                  │
│  - 管理 Mavlink 流（数据发送调度）             │
│  - 从 uORB 获取数据，填充 Mavlink 消息        │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│  底层 Mavlink 库                             │
│  ./src/modules/mavlink/mavlink/ (git子模块)  │
│  编译时生成到 ./build/XXX/mavlink/            │
│  - 消息打包/解包                              │
│  - 底层传输协议                               │
└─────────────────────────────────────────────┘
```

### 核心概念

| 概念 | 说明 |
|------|------|
| **Mavlink 消息** | 传输基本单元，每条有唯一消息 ID（如 HEARTBEAT=0） |
| **Mavlink 实例（通道）** | 绑定到串口/USB/网络的独立通信通道，最多 4 个 |
| **Mavlink 流** | 上层概念，将消息与发送频率绑定，如 `ATTITUDE @ 50Hz` |
| **流模式** | 预定义的流集合：normal、onboard、custom 等 |

---

## 二、数据流全景

```
┌──────────────┐     uORB        ┌──────────────┐     MAVLink      ┌──────────┐
│  test_app    │ ──────────────→ │ Mavlink 流    │ ──────────────→ │  QGC     │
│  (发布端)    │  test_topic_x   │ TEST_MESSAGE  │  TEST_MESSAGE   │ (地面站) │
│              │                 │ (订阅+封装)   │  id=200         │          │
└──────────────┘                 └──────────────┘                 └──────────┘
      ↑                               ↑                              ↑
  px4_task_spawn_cmd            MavlinkStream                   mavlink status
  pthread_send                  ::send()                        streams
```

完整路径：**uORB 主题 → Mavlink 流 → Mavlink 消息帧 → 物理通道 → 地面站**

---

## 三、发送自定义 Mavlink 消息：三步法

### 第 1 步：定义 Mavlink 消息（XML）

在 `./src/modules/mavlink/mavlink/message_definitions/v1.0/development.xml` 中添加：

```xml
<message id="200" name="TEST_MESSAGE">
    <description>This is a test message</description>
    <field type="uint64_t" name="data">uint64_t</field>
</message>
```

- `id="200"`：消息 ID，用户自定义范围 163~229，需确保全系统唯一
- `name="TEST_MESSAGE"`：消息名，编译后生成 `mavlink_test_message_t` 结构体 + `MAVLINK_MSG_ID_TEST_MESSAGE` 宏
- `<field>`：消息字段，支持 `uint8_t`~`uint64_t`、`int`、`float`、数组等

> 编译时 `mavlinkgenerate.py` 自动生成 `mavlink_msg_test_message.h`

---

### 第 2 步：定义 + 注册 MAVLink 流

#### 2.1 创建流头文件 `TEST_MESSAGE.hpp`

目录：`./src/modules/mavlink/streams/TEST_MESSAGE.hpp`

```cpp
#ifndef TEST_MESSAGE_HPP          // ⚠️ 头文件保护宏名
#define TEST_MESSAGE_HPP

#include "uORB/topics/test_topic.h"

class MavlinkStreamTestMessage : public MavlinkStream
{
public:
    // ===== 框架接口（必须实现）=====

    // 工厂方法：框架用它创建流实例
    static MavlinkStream *new_instance(Mavlink *mavlink)
    {
        return new MavlinkStreamTestMessage(mavlink);
    }

    // 流名称（对应 mavlink stream -s 的参数、configure_stream_local 的参数）
    static constexpr const char *get_name_static() { return "TEST_MESSAGE"; }

    // 消息 ID（由 XML 定义自动生成）
    static constexpr uint16_t get_id_static() { return MAVLINK_MSG_ID_TEST_MESSAGE; }

    const char *get_name() const override { return get_name_static(); }
    uint16_t get_id() override { return get_id_static(); }

    // 消息大小：有发布者时返回有效大小，否则返回 0（流不会发送）
    unsigned get_size() override
    {
        return _test_sub.advertised()
            ? (MAVLINK_MSG_ID_TEST_MESSAGE_LEN + MAVLINK_NUM_NON_PAYLOAD_BYTES)
            : 0;
    }

private:
    explicit MavlinkStreamTestMessage(Mavlink *mavlink) : MavlinkStream(mavlink) {}

    // 订阅 uORB 主题（发布者 → uORB → 这里 → MAVLink → 地面站）
    uORB::Subscription _test_sub{ORB_ID(test_topic_x)};

    // ===== 核心：发送逻辑 =====
    bool send() override
    {
        test_topic_s test_data;

        if (_test_sub.update(&test_data)) {           // 检查 uORB 更新
            mavlink_test_message_t msg{};              // 创建 MAVLink 消息
            msg.data = test_data.timestamp;            // uORB 数据 → MAVLink 字段
            mavlink_msg_test_message_send_struct(      // 通过当前通道发送
                _mavlink->get_channel(), &msg);
            return true;
        }
        return false;
    }
};

#endif
```

**流类的生命周期**：

```
Mavlink 启动 → 创建实例 → 遍历 streams_list → new_instance() → 加入发送循环
                             ↓
                    每帧轮询 send()
                    有新 uORB 数据 → 打包 → 发送
                    无数据 → 跳过
```

#### 2.2 注册流到 `mavlink_messages.cpp`

```cpp
// ① 添加 include（按字母顺序）
# include "streams/TEST_MESSAGE.hpp"

// ② 在 StreamListItem 数组 streams_list 末尾添加：
#if defined(TEST_MESSAGE_HPP)
    create_stream_list_item<MavlinkStreamTestMessage>()
#endif
```

> ⚠️ **关键细节**：`TEST_MESSAGE.hpp` 的头文件保护宏名必须与 `#if defined(...)` 一致（都是 `TEST_MESSAGE_HPP`），否则流永远不会被注册。列表最后一项不加 `,` 逗号。

---

### 第 3 步：配置流发送速率

在 `mavlink_main.cpp` → `configure_streams_to_default()` 函数中，根据工作模式添加：

```cpp
case MAVLINK_MODE_ONBOARD:   // SITL 仿真默认此模式
    // ... 其他流配置 ...
    configure_stream_local("TEST_MESSAGE", unlimited_rate);  // 不限速
    break;
```

| 模式 | 适用场景 |
|------|---------|
| `MAVLINK_MODE_NORMAL` | 飞控硬件默认模式 |
| `MAVLINK_MODE_ONBOARD` | SITL 仿真默认模式 |
| `MAVLINK_MODE_CUSTOM` | 自定义模式 |

---

## 四、应用端：uORB 数据源

`test_app` 模块在独立的 PX4 任务中创建两个 pthread 线程，以 1Hz 频率向 `test_topic_x` 发布数据：

```cpp
// 发送线程
void *thread_loop_send(void *arg) {
    test_topic_s test_data = {};
    orb_advert_t test_pub = orb_advertise(ORB_ID(test_topic_x), &test_data);

    uint64_t cnt = 0;
    while (!thread_should_exit) {
        test_data.timestamp = hrt_absolute_time();
        test_data.data = cnt;
        orb_publish(ORB_ID(test_topic_x), test_pub, &test_data);
        ++cnt;
        px4_sleep(1);   // 1Hz
    }
    return nullptr;
}
```

### uORB 消息定义（test_topic.msg）

```c
uint64 timestamp    # 微秒时间戳
uint64 data         # 自增计数器
# TOPICS test_topic test_topic_x   ← 创建第二个 topic 实例（测试用）
```

---

## 五、完整数据链路

```
══════════════════════════════════════════════════════════
                    运行时数据流
══════════════════════════════════════════════════════════

① test_app start
   → px4_task_spawn_cmd → run_trampoline → run()
   → pthread_send 启动
   → 每秒 orb_publish(test_topic_x, {timestamp, cnt})

② Mavlink 模块（已启动，运行在 MAVLINK_MODE_ONBOARD）
   → 流调度器每秒轮询 TEST_MESSAGE 流
   → send() → _test_sub.update(&test_data)
   → 有新 uORB 数据 → 填充 mavlink_test_message_t
   → mavlink_msg_test_message_send_struct() 发送

③ 地面站 QGC
   → 接收 MAVLink 消息帧
   → 解析 TEST_MESSAGE (id=200)
   → 显示/记录数据
```

---

## 六、验证命令

```bash
# 启动应用（uORB 数据源）
test_app start

# 查看 Mavlink 实例和流的运行状态
mavlink status streams
# 应在 instance #1 下看到：
#   TEST_MESSAGE     unlimited_rate (unlimited_rate)  [Message Size]

# 在 QGC 中查看
# Widgets → MAVLink Inspector → 选择 TEST_MESSAGE
```

---

## 七、踩过的坑

| 坑 | 现象 | 原因 | 解决 |
|-----|------|------|------|
| 头文件保护宏名不一致 | `mavlink status streams` 看不到 TEST_MESSAGE | `#ifndef TEST_MESSAGE_H` vs `#if defined(TEST_MESSAGE_HPP)` | 统一为 `TEST_MESSAGE_HPP` |
| 缺 `#include` | 同上 | 只在 arrays 里加了 `#if defined`，但没 include 头文件 | 添加 `#include "streams/TEST_MESSAGE.hpp"` |
| 列表最后一项有逗号 | 编译报错 `expected '}'` | 前面一项添加了 `,` 但忘了移到最后一项 | 最后一项不加逗号 |
| 没在 `configure_streams_to_default` 配置 | 流存在但从不发送 | 流注册了但没配速率 | 添加 `configure_stream_local("TEST_MESSAGE", rate)` |

## 关键知识总结

| 层次 | 关键技术 |
|------|---------|
| uORB 层 | `.msg` 文件定义 → `# TOPICS` 多实例 → `orb_advertise` + `orb_publish` |
| MAVLink 消息层 | `.xml` 定义消息 → 编译生成 `mavlink_msg_*.h` → `MAVLINK_MSG_ID_*` 宏 |
| MAVLink 流层 | `MavlinkStream` 子类 → `get_name_static`/`get_id_static`/`send` → `new_instance` |
| 注册层 | `#include` 流头文件 → `#if defined(...)` → `create_stream_list_item` |
| 配置层 | `configure_streams_to_default` → `configure_stream_local("流名", 速率)` |

## 相关笔记

- [[4-4-飞行日志]] — uORB 发布订阅 + pthread 双线程
- [[3-1-uORB-C-API-基本接口与自定义主题]] — uORB C API 详解
- [[3-2-uORB-C++封装与工作队列同步]] — uORB C++ 封装
- [[4-1-配置参数存取]] — param 系统
