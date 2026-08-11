---
tags:
  - PX4
  - uORB
  - SubscriptionInterval
  - 时序同步
date: 2026-08-10
---

# uORB 时序不匹配问题与 SubscriptionInterval

## 问题场景

发送线程每 1 秒发布一次，接收线程也每 1 秒用 `px4_sleep(1)` 检查一次。但实际输出间隔不稳定：

```
INFO  uorb topic send time is 17 s  ← 间隔 1s
INFO  uorb topic send time is 18 s  ← 间隔 1s
INFO  uorb topic send time is 20 s  ← 间隔 2s！跳过了 19s 的数据
INFO  uorb topic send time is 22 s  ← 间隔 2s！
```

## 根因分析

```
发送线程 (1Hz)              接收线程 (~1.01Hz)
──────────                   ────────────────
t=17s: publish msg#17
                             t=17.0s: 检测到，打印 #17
                             px4_sleep(1) → 实际睡了 1.01s
t=18s: publish msg#18
                             t=18.1s: 检测到，打印 #18
t=19s: publish msg#19
                             (还在 sleep... 错过了!)
t=19.5s: publish msg#20
                             t=19.6s: 检测到，打印 #20
                             msg#19 永远丢失
```

`px4_sleep(1)` 保证**至少**睡 1 秒，加上 `PX4_INFO` 打印耗时，每轮实际略超 1 秒。微小误差累积后，接收线程会越过一次发布，直接读到下一条消息。

## 解决方案：uORB::SubscriptionInterval

用高精度定时器（hrt）替代 `px4_sleep`，每次间隔严格对齐：

```cpp
#include <uORB/SubscriptionInterval.hpp>

// 构造函数参数：ORB_ID + 间隔时间（微秒）
uORB::SubscriptionInterval dbg_sub{ORB_ID(debug_key_value), 1000000};
//                                                             ↑ 1秒 = 1,000,000 微秒

debug_key_value_s dbg = {};
while (!thread_should_exit) {
    if (dbg_sub.updated()) {       // 每 1 秒严格触发一次
        dbg_sub.copy(&dbg);
        PX4_INFO("value = %.2f", (double)dbg.value);
    }
}
```

## 对比

```
px4_sleep(1) 方式:               SubscriptionInterval 方式:
t=17.0: 检测 → sleep(1.01)       t=17.0: 检测 → hrt 等待 1.000s
t=18.1: 检测 → sleep(1.02)       t=18.0: 检测 → hrt 等待 1.000s  ✓
t=19.1: 跳过!                     t=19.0: 检测 → 不会漂移!
t=19.6: 跳过!                     t=20.0: 检测
```

## 三种订阅类对比

| 类 | 更新方式 | 适用场景 |
|----|---------|---------|
| `uORB::Subscription` | 手动 `updated()` + 自定节奏 | 灵活节奏 |
| `uORB::SubscriptionData` | 手动 `update()` + 自定节奏 | 灵活节奏 + 自动缓存 |
| `uORB::SubscriptionInterval` | 固定间隔自动对齐 | 需要精确定频 |

## 关键知识点

| 概念 | 说明 |
|------|------|
| `px4_sleep` 漂移 | `sleep(N)` 保证至少 N 秒，实际偏大 |
| `SubscriptionInterval` | 基于 hrt 的定频订阅，不漂移 |
| 构造函数参数 | `(ORB_ID(topic), interval_us)` 微秒级 |
