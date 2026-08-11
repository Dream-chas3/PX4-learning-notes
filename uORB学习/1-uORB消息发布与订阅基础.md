---
tags:
  - PX4
  - uORB
  - 发布订阅
  - orb_advertise
  - orb_subscribe
date: 2026-08-10
---

# uORB 消息发布与订阅基础

## 概述

uORB（micro Object Request Broker）是 PX4 的**异步发布-订阅消息总线**。各模块通过它交换数据而不需要直接依赖对方。一个模块发布消息到某个 topic，其他模块订阅该 topic 接收消息。

## 核心概念

```
┌──────────┐     publish(topic, data)     ┌──────────────┐
│  发布者A  │ ───────────────────────────→ │              │
└──────────┘                              │   uORB 总线   │
                                          │              │
┌──────────┐     subscribe(topic)         │              │
│  订阅者B  │ ←─────────────────────────── │              │
└──────────┘                              └──────────────┘
```

- **Topic**：消息主题，由 `.msg` 文件定义数据结构
- **orb_advertise**：公告一个 topic，获得发布句柄
- **orb_publish**：发布数据到 topic
- **orb_subscribe**：订阅一个 topic，获得订阅句柄
- **orb_copy**：从订阅句柄读取最新数据

---

## 一、定义消息：.msg 文件

以 `debug_key_value.msg` 为例：

```
uint64 timestamp	# time since system start (microseconds)
char[10] key		# 最大10个字符的键名
float value		# 浮点值
```

构建系统自动生成对应的 C/C++ 结构体：

```cpp
struct debug_key_value_s {
    uint64_t timestamp;
    float value;
    char key[10];
    uint8_t _padding0[2];
};
```

---

## 二、发布端代码

```cpp
#include <uORB/topics/debug_key_value.h>

// 发送线程：每秒发布一次
void *thread_loop_send(void *arg)
{
    // ① 公告 topic —— 只需执行一次
    orb_advert_t test_pub = orb_advertise(ORB_ID(debug_key_value), nullptr);

    debug_key_value_s test_data = {};
    uint64_t cnt = 0;

    while (!thread_should_exit) {
        test_data.timestamp = hrt_absolute_time();
        test_data.value = (float)cnt;
        strncpy(test_data.key, "test", sizeof(test_data.key));

        // ② 发布数据
        orb_publish(ORB_ID(debug_key_value), test_pub, &test_data);

        ++cnt;
        px4_sleep(1);  // 1Hz 频率
    }
    return nullptr;
}
```

### 关键 API

| API | 说明 |
|-----|------|
| `orb_advertise(ORB_ID(topic), data)` | 公告 topic，返回发布句柄 |
| `orb_publish(ORB_ID(topic), handle, &data)` | 发布一条消息 |
| `orb_advertise_queue(..., queue_size)` | 公告 topic 并指定队列深度 |

---

## 三、订阅端代码

```cpp
// 接收线程：每秒检查一次
void *thread_loop_receive(void *arg)
{
    // ① 订阅 topic —— 只需执行一次
    int test_sub = orb_subscribe(ORB_ID(debug_key_value));

    debug_key_value_s test_data = {};
    bool updated = false;

    while (!thread_should_exit) {
        // ② 检查是否有新数据
        orb_check_updates(test_sub, &updated);

        if (updated) {
            // ③ 拷贝最新数据
            orb_copy(ORB_ID(debug_key_value), test_sub, &test_data);

            PX4_INFO("time is %lu s", test_data.timestamp / 1000000u);
            PX4_INFO("value = %.2f", (double)test_data.value);
        }
        px4_sleep(1);
    }
    return nullptr;
}
```

### 关键 API

| API | 说明 |
|-----|------|
| `orb_subscribe(ORB_ID(topic))` | 订阅 topic，返回句柄 |
| `orb_check_updates(handle, &updated)` | 检查是否有新数据 |
| `orb_copy(ORB_ID(topic), handle, &data)` | 拷贝最新数据到本地 |

---

## 四、完整执行流程

```
发送线程                           uORB 总线                   接收线程
────────                          ────────                     ────────
orb_advertise ──→ 注册发布者
orb_publish   ──→ [msg#1]  →→→→→→→→→→→→→→→→→→→→ orb_subscribe ──→ 注册订阅者
px4_sleep(1)                                orb_check_updates ──→ 无更新
orb_publish   ──→ [msg#2]  →→→→→→→→→→→→→→→→→→→→ orb_check_updates ──→ 有更新!
px4_sleep(1)                                   orb_copy ──→ 拿到 msg#2
```

---

## 五、队列深度

默认队列深度为 1（只保留最新消息）。若发布频率高而订阅端处理慢，消息可能丢失：

```cpp
// 设置队列深度为 8
orb_advert_t pub = orb_advertise_queue(ORB_ID(topic), nullptr, 8);
```

```
queue_size=1:  ┌─────────┐
               │ msg #1  │ ← msg #2 覆盖
               └─────────┘

queue_size=4:  ┌───┬───┬───┬───┐
               │ 1 │ 2 │ 3 │ 4 │
               └───┴───┴───┴───┘
```

## 常用命令

```bash
listener debug_key_value    # 查看 topic 最新数据
uorb top                    # 查看所有 topic 发布频率
```

## 关键知识点

| 概念 | 说明 |
|------|------|
| `orb_advertise` | 公告发布，返回发布句柄 |
| `orb_subscribe` | 订阅 topic，返回订阅句柄 |
| `orb_publish` | 发布数据 |
| `orb_copy` | 拷贝最新数据到本地 buffer |
| `orb_check_updates` | 检查是否有新数据（非阻塞） |
| `ORB_ID(topic)` | 获取 topic 的元数据指针 |
| `orb_advertise_queue` | 带队列深度的公告 |
