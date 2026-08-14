---
tags:
  - PX4
  - uORB
  - 发布订阅
  - 工作队列
  - SubscriptionCallback
date: 2026-08-11
---

# uORB C++ 封装与工作队列同步

> 来源：《13-uORB常规使用方法》

---

## 一、为什么要有 C++ 封装

uORB 的 C API（`orb_advertise` / `orb_publish` / `orb_check` / `orb_copy`）功能完整但使用繁琐。PX4 提供了 C++ 封装类来简化开发：

| 封装类 | 简化了什么 | 适用场景 |
|--------|-----------|----------|
| `Publication` / `PublicationData` | 合并公告+发布为一个 `publish()` | 发布数据 |
| `Subscription` / `SubscriptionData` | 合并检查+拷贝为一个 `update()` | 轮询接收数据 |
| `SubscriptionCallbackWorkItem` | uORB 数据更新时**自动唤醒**工作队列 | 事件驱动的实时控制 |

```
原始 C API:                    C++ 封装后:

orb_advertise() ─┐
orb_publish()   ─┴──→  dbg_pub.publish(data)   一句搞定！

orb_check()   ─┐
orb_copy()    ─┴──→  dbg_sub.update(&data)     一句搞定！
```

---

## 二、Publication / SubscriptionData 使用

### 2.1 发送端：Publication 类

```cpp
#include <uORB/Publication.hpp>
#include <uORB/topics/debug_key_value.h>

void *thread_loop_send(void *arg) {
    // 构造函数自动处理公告
    // 模板参数 = uORB 主题的数据类型
    uORB::Publication<debug_key_value_s> dbg_pub{ORB_ID(debug_key_value)};

    debug_key_value_s dbg = {};
    strncpy(dbg.key, "test", sizeof(dbg.key));  // 调试标识名（最多10字符）
    uint64_t cnt = 0;

    while (!thread_should_exit) {
        dbg.timestamp = hrt_absolute_time();
        dbg.value = (float)cnt;

        // 首次调用 → 自动公告 + 发布
        // 后续调用 → 仅发布
        dbg_pub.publish(dbg);

        ++cnt;
        px4_sleep(1);
    }
    return nullptr;
}
```

### 2.2 接收端：SubscriptionData 类

```cpp
#include <uORB/Subscription.hpp>
#include <uORB/topics/debug_key_value.h>

void *thread_loop_receive(void *arg) {
    // 构造函数自动订阅
    uORB::SubscriptionData<debug_key_value_s> dbg_sub{ORB_ID(debug_key_value)};

    debug_key_value_s dbg = {};

    while (!thread_should_exit) {
        // update() = orb_check + orb_copy 二合一
        // 有新数据 → 自动拷贝到内部缓冲区 → 返回 true
        // 无新数据 → 返回 false
        if (dbg_sub.update()) {
            // get() 直接返回内部缓冲区的引用（无需 orb_copy）
            dbg = dbg_sub.get();

            PX4_INFO("uorb topic send time is %lu s",
                     dbg.timestamp / 1000000u);
            PX4_INFO("test data is value = %.2f", (double)dbg.value);
        }
        px4_sleep(1);
    }
    return nullptr;
}
```

### 2.3 内置调试主题

PX4 提供了四种用于传递调试数据的 uORB 主题：

| 主题 | 数据类型 | 用途 |
|------|----------|------|
| `debug_key_value` | 名称(key字符串) + 浮点值 | 单个浮点数 |
| `debug_value` | 名称(uint8) + 浮点值 | 单个浮点数 |
| `debug_array` | 名称 + 浮点数组 | 数组数据 |
| `debug_vect` | 名称 + xyz 三维向量 | 向量数据 |

---

## 三、SubscriptionCallbackWorkItem：uORB 驱动的工作队列

### 3.1 问题：轮询 vs 事件驱动

```
轮询模式（前面的示例）：           事件驱动模式（本节）：
                                
while (!exit) {                  uORB 数据更新
    orb_check()  ← 主动检查            │
    if (updated)                      ▼
       处理数据               Run() 被自动调用  ← 不占用 CPU
    px4_sleep() ← 空转浪费
}                               
```

PX4 的姿态控制、位置控制等**核心模块**全部采用事件驱动模式。

### 3.2 实现原理

```
uORB 数据发布流程（深入底层）：
                                     
orb_publish()                         
    │                                 
    ▼                                 
DeviceNode::publish()                 
    │                                 
    ▼                                 
DeviceNode::write()                   
    ├── _generation++ （发布计数器+1）  
    ├── memcpy → _data （数据写入缓冲区）
    ├── 遍历 _callbacks 链表          
    │   └── item->call()              
    │       └── 检查更新              
    │       └── _work_item->ScheduleNow()  ← 唤醒工作队列！
    └── poll_notify(POLLIN)           
```

**关键链路**：
1. `DeviceNode::write()` 中遍历 `_callbacks` 链表
2. 每个 callback 是 `SubscriptionCallbackWorkItem` 实例
3. 其 `call()` 检查更新后调用 `ScheduleNow()` 将 `Run()` 加入工作队列

### 3.3 完整源码示例

#### test_app_main.h

```cpp
#pragma once
#include <px4_platform_common/module.h>
#include <px4_platform_common/px4_work_queue/WorkItem.hpp>
#include <uORB/SubscriptionCallback.hpp>
#include <uORB/Publication.hpp>
#include <uORB/topics/vehicle_angular_velocity.h>
#include <uORB/topics/debug_vect.h>

// 双重继承：
// ModuleBase → CLI 命令管理（start/stop/status）
// WorkItem  → 工作队列调度（Run 方法在线程池执行）
class TestApp : public ModuleBase<TestApp>, public px4::WorkItem
{
public:
    TestApp();
    virtual ~TestApp() = default;

    static int task_spawn(int argc, char *argv[]);
    static TestApp *instantiate(int argc, char *argv[]);
    static int custom_command(int argc, char *argv[]);
    static int print_usage(const char *reason = nullptr);

    void Run() override;   // ← 当 uORB 数据更新时被自动调用
    void init();           // ← 注册 uORB 回调

private:
    // 核心成员：this = 当前 WorkItem，数据更新时唤醒本对象的 Run()
    uORB::SubscriptionCallbackWorkItem vehicle_rates_sub{
        this, ORB_ID(vehicle_angular_velocity)};

    uORB::Publication<debug_vect_s> dbg_pub{ORB_ID(debug_vect)};

    vehicle_angular_velocity_s angular_velocity{};
    debug_vect_s dbg{};
};
```

#### test_app_main.cpp

```cpp
#include "test_app_main.h"

TestApp::TestApp()
    : px4::WorkItem(MODULE_NAME, px4::wq_configurations::test)
{}

// ── 初始化：注册 uORB 回调 ──
void TestApp::init()
{
    // 将本对象注册为 vehicle_angular_velocity 主题的监听者
    // 之后每次该主题数据更新，都会自动调用 this->Run()
    vehicle_rates_sub.registerCallback();
}

// ── 由 uORB 数据更新触发执行 ──
void TestApp::Run()
{
    strncpy(dbg.name, "test", sizeof(dbg.name));

    // 检查退出信号
    if (should_exit()) {
        vehicle_rates_sub.unregisterCallback();  // 注销回调
        exit_and_cleanup();
        return;
    }

    // update() = orb_check + orb_copy 合并
    if (vehicle_rates_sub.update(&angular_velocity)) {
        // 将角速度数据转发到调试主题
        dbg.x = angular_velocity.xyz[0];  // Roll 角速度
        dbg.y = angular_velocity.xyz[1];  // Pitch 角速度
        dbg.z = angular_velocity.xyz[2];  // Yaw 角速度
        dbg_pub.publish(dbg);
    }
}

// ── 任务创建（关键区别：使用工作队列而非独立线程）──
int TestApp::task_spawn(int argc, char *argv[])
{
    TestApp *instance = instantiate(argc - 1, argv + 1);
    _object.store(instance);

    if (instance) {
        _task_id = task_id_is_work_queue;  // ← 标记为工作队列任务
        instance->init();                   // ← 注册 uORB 回调
        return PX4_OK;
    } else {
        PX4_ERR("failed to instantiate object");
        _task_id = -1;
        return PX4_ERROR;
    }
}

// 其余函数与之前相同（instantiate / custom_command / print_usage / test_app_main）
```

### 3.4 与轮询模式的关键区别

| | 轮询模式（C API / Publication+SubscriptionData） | 事件驱动（SubscriptionCallbackWorkItem） |
|------|------|------|
| **触发方式** | 定时轮询检查 `orb_check` | uORB 数据更新自动唤醒 |
| **CPU 占用** | 持续占用 | 仅数据更新时执行 |
| **继承类** | 仅 `ModuleBase` | `ModuleBase` + `WorkItem` |
| **init() 内容** | `ScheduleOnInterval(...)` 定时调度 | `registerCallback()` 注册回调 |
| **task_spawn** | `px4_task_spawn_cmd` 创建独立任务 | `task_id_is_work_queue` 工作队列 |
| **适用场景** | 低频、非实时 | **高频、实时控制**（姿态、位置等核心模块） |

---

## 四、快速记忆：两种开发模式对比

```
模式一：pthread 轮询（第12篇）        模式二：工作队列回调（第13篇）

init(): 无特殊初始化               init():
                                      registerCallback()
Run(): 不使用
                                      ↓
run(): while(!exit) {              Run(): 收到新数据时被自动调用
    sleep(1);                            ↓
}                                     处理数据

task_spawn:                        task_spawn:
    px4_task_spawn_cmd(...)            _task_id = task_id_is_work_queue
```

---

## 五、关键要点

| 要点 | 说明 |
|------|------|
| **Publication** | `publish(data)` = 公告 + 发布，首次自动公告 |
| **SubscriptionData** | `update()` = check + copy，`get()` = 获取内部缓存引用 |
| **SubscriptionCallbackWorkItem** | uORB 数据更新 → 自动 `ScheduleNow()` → `Run()` 被调用 |
| **registerCallback()** | 向 DeviceNode 的 `_callbacks` 链表注册监听 |
| **PX4 核心模块** | 全部使用 SubscriptionCallbackWorkItem 模式 |
| **run_trampoline** | 已在 `module.h` 实现，用户代码无需重复 |
