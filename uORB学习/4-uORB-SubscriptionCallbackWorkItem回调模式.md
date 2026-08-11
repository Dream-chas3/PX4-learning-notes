---
tags:
  - PX4
  - uORB
  - SubscriptionCallback
  - WorkItem
  - 回调
date: 2026-08-10
---

# uORB SubscriptionCallbackWorkItem 回调模式

## 概述

前面几种方式都需要**主动轮询**（`updated()` + `px4_sleep`）。PX4 还提供了**回调模式**：当订阅的 topic 有新数据时，自动触发 `Run()` 方法。这种方式通过 `uORB::SubscriptionCallbackWorkItem` 实现，结合 WorkItem 工作队列调度。

## 架构

```
vehicle_angular_velocity 发布新数据
        │
        ▼
SubscriptionCallbackWorkItem 自动检测
        │
        ├─ 触发 WorkItem::Run()  ← 你的回调函数
        │       │
        │       ├─ 读取 angular_velocity
        │       ├─ 转换为 debug_vect
        │       └─ 发布 debug_vect
        │
        └─ 等待下一次数据到达
```

## 头文件

```cpp
#include <px4_platform_common/px4_work_queue/WorkItem.hpp>
#include <uORB/SubscriptionCallback.hpp>

class TestApp : public ModuleBase<TestApp>, public px4::WorkItem {
public:
    TestApp();
    void Run() override;   // ← 回调入口：数据到达时自动调用
    void init();

private:
    // 订阅回调对象：绑定 this 和 topic
    uORB::SubscriptionCallbackWorkItem vehicle_rates_sub{
        this,                                    // 关联的 WorkItem（即自己）
        ORB_ID(vehicle_angular_velocity)         // 订阅的 topic
    };

    uORB::Publication<debug_vect_s> dbg_pub{ORB_ID(debug_vect)};
    vehicle_angular_velocity_s angular_velocity{};
    debug_vect_s dbg{};
};
```

### 关键：`this` 参数

```cpp
uORB::SubscriptionCallbackWorkItem vehicle_rates_sub{this, ORB_ID(...)};
//                                                    ↑
//                    把当前 TestApp 实例传给回调框架
//                    当 vehicle_angular_velocity 数据到达时，
//                    框架调用 this->Run()
```

## 构造函数

```cpp
TestApp::TestApp() : px4::WorkItem(MODULE_NAME, px4::wq_configurations::test1) {}
//                            ↑
//              注册到 test1 工作队列（WorkItem 必需）
```

## init()：注册回调

```cpp
void TestApp::init()
{
    vehicle_rates_sub.registerCallback();
    // 调用后，当 vehicle_angular_velocity 有新数据时，
    // 自动触发 Run() 方法
}
```

## Run()：数据到达时的回调

```cpp
void TestApp::Run()
{
    if (should_exit()) {
        vehicle_rates_sub.unregisterCallback();
        exit_and_cleanup();
        return;
    }

    // update() 自动拷贝新数据到 angular_velocity
    if (vehicle_rates_sub.update(&angular_velocity)) {
        // 转换数据格式
        dbg.x = angular_velocity.xyz[0];
        dbg.y = angular_velocity.xyz[1];
        dbg.z = angular_velocity.xyz[2];

        // 发布到 debug_vect topic
        dbg_pub.publish(dbg);
    }
}
```

## run()：保持任务存活

```cpp
void TestApp::run()
{
    init();  // 注册回调

    while (!should_exit()) {
        px4_usleep(100000);  // 主线程只需保持存活，回调在另一个上下文执行
    }

    vehicle_rates_sub.unregisterCallback();
}
```

## 完整数据流

```
[DDS/驱动发布 vehicle_angular_velocity]
        │
        ▼
SubscriptionCallbackWorkItem 检测到更新
        │
        ▼
WorkItem::Run() 被调度执行
        │
        ├─ update(&angular_velocity) → 获取最新角速度
        ├─ dbg.x = angular_velocity.xyz[0]  → 转换
        ├─ dbg.y = angular_velocity.xyz[1]
        ├─ dbg.z = angular_velocity.xyz[2]
        └─ dbg_pub.publish(dbg) → 发布 debug_vect
        │
        ▼
listener debug_vect 可查看实时数据
```

## 四种 uORB 模式对比

| 模式 | 触发方式 | 适用场景 |
|------|---------|---------|
| C API 轮询 | `orb_check_updates` + `sleep` | 简单场景 |
| C++ Publication/Subscription | `updated()` + `sleep` | 类型安全 |
| SubscriptionInterval | 固定间隔自动对齐 | 定频采集 |
| **SubscriptionCallbackWorkItem** | **数据到达自动回调** | **响应式处理、生产级模块** |

## 关键知识点

| 概念 | 说明 |
|------|------|
| `SubscriptionCallbackWorkItem` | 结合订阅和 WorkItem 的回调类 |
| `this` 参数 | 将当前对象绑定为回调目标 |
| `registerCallback()` | 激活回调（调用后 Run() 才会被触发） |
| `unregisterCallback()` | 注销回调（退出前必须调用） |
| `Run()` override | 回调入口，WorkItem 纯虚函数 |
| `update(&data)` | 拷贝数据到指定变量，返回 true 表示有新数据 |
