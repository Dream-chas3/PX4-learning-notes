---
tags:
  - PX4
  - uORB
  - C++
  - Publication
  - Subscription
  - SubscriptionData
date: 2026-08-10
---

# uORB C++ 封装 API

## 概述

PX4 提供了 uORB 的 C++ 模板封装类，相比原始 C API 更加类型安全和简洁。主要类包括 `uORB::Publication`（发布）、`uORB::Subscription`（订阅）和 `uORB::SubscriptionData`（订阅+数据缓存）。

## 与 C API 对照

| C API | C++ API | 说明 |
|-------|---------|------|
| `orb_advertise` | `uORB::Publication<T>` | 公告发布 |
| `orb_publish` | `pub.publish(data)` | 发布数据 |
| `orb_subscribe` | `uORB::Subscription` | 订阅 topic |
| `orb_copy` | `uORB::SubscriptionData::get()` | 读取数据 |
| `orb_check_updates` | `sub.updated()` | 检查更新 |

---

## 一、uORB::Publication（发布）

```cpp
#include <uORB/Publication.hpp>
#include <uORB/topics/debug_key_value.h>

// 创建发布对象（构造时自动完成 orb_advertise）
uORB::Publication<debug_key_value_s> dbg_pub{ORB_ID(debug_key_value)};

debug_key_value_s dbg = {};
dbg.timestamp = hrt_absolute_time();
dbg.value = 3.14f;

// 发布数据
dbg_pub.publish(dbg);
```

**优点**：
- 构造函数自动 advertise，析构函数自动清理
- `publish()` 类型安全，编译期校验数据类型
- 不需要手动管理句柄

---

## 二、uORB::Subscription（订阅）

只做订阅检测，数据需要另外拷贝：

```cpp
#include <uORB/Subscription.hpp>

uORB::Subscription dbg_sub{ORB_ID(debug_key_value)};
debug_key_value_s dbg = {};

while (!should_exit()) {
    if (dbg_sub.updated()) {
        dbg_sub.copy(&dbg);      // 拷贝数据到本地
        PX4_INFO("value = %.2f", (double)dbg.value);
    }
    px4_sleep(1);
}
```

| 方法 | 说明 |
|------|------|
| `updated()` | 是否有新数据（非阻塞） |
| `copy(&data)` | 拷贝最新数据 |

---

## 三、uORB::SubscriptionData（订阅+缓存）

同时完成订阅和数据缓存，最便捷：

```cpp
#include <uORB/Subscription.hpp>

// 创建订阅（自动 subscribe）
uORB::SubscriptionData<debug_key_value_s> dbg_sub{ORB_ID(debug_key_value)};

while (!thread_should_exit) {
    if (dbg_sub.update()) {         // 检查 + 自动拷贝
        debug_key_value_s dbg = dbg_sub.get();  // 获取缓存数据
        PX4_INFO("value = %.2f", (double)dbg.value);
    }
    px4_sleep(1);
}
```

| 方法 | 说明 |
|------|------|
| `update()` | 检查更新 + 自动拷贝（返回 true 表示有新数据） |
| `get()` | 获取已缓存的数据副本 |

## 三种订阅方式对比

| | Subscription | SubscriptionData | C API |
|---|---|---|---|
| **订阅** | 构造函数 | 构造函数 | `orb_subscribe` |
| **检测更新** | `updated()` | `update()` | `orb_check_updates` |
| **读数据** | `copy(&buf)` | `get()` | `orb_copy` |
| **数据缓存** | 无（需手动管理） | 内部自动缓存 | 无 |

---

## 四、完整示例：发布 + 订阅

```cpp
// ===== 发送线程 =====
void *thread_loop_send(void *arg)
{
    uORB::Publication<debug_key_value_s> dbg_pub{ORB_ID(debug_key_value)};
    debug_key_value_s dbg = {};
    strncpy(dbg.key, "test", sizeof(dbg.key));
    uint64_t cnt = 0;

    while (!thread_should_exit) {
        dbg.timestamp = hrt_absolute_time();
        dbg.value = (float)cnt;
        dbg_pub.publish(dbg);
        ++cnt;
        px4_sleep(1);
    }
    return nullptr;
}

// ===== 接收线程 =====
void *thread_loop_receive(void *arg)
{
    uORB::SubscriptionData<debug_key_value_s> dbg_sub{ORB_ID(debug_key_value)};

    while (!thread_should_exit) {
        if (dbg_sub.update()) {
            debug_key_value_s dbg = dbg_sub.get();
            PX4_INFO("time is %lu s", dbg.timestamp / 1000000u);
            PX4_INFO("value = %.2f", (double)dbg.value);
        }
        px4_sleep(1);
    }
    return nullptr;
}
```

## 关键知识点

| 概念 | 说明 |
|------|------|
| `uORB::Publication<T>` | C++ 发布者，构造时 advertise |
| `uORB::Subscription` | C++ 订阅者，需手动 copy |
| `uORB::SubscriptionData<T>` | C++ 订阅者，内部缓存 + 自动同步 |
| `updated()` | 非阻塞检查是否有新数据 |
| `update()` | 检查 + 自动拷贝（SubscriptionData 专用） |
| `get()` | 获取缓存副本（SubscriptionData 专用） |
| `copy(&buf)` | 拷贝数据到指定 buffer（Subscription 专用） |
