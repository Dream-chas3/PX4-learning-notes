---
tags:
  - PX4
  - dataman
  - 飞行任务
  - 航点
  - 持久化存储
date: 2026-08-11
---

# 飞行任务参数存取：dataman 模块

> 来源：《14-飞行任务参数存取》

---

## 一、dataman 是什么

PX4 中有大量需要**持久化保存**的数据（断电不丢失）：航点任务、地理围栏、安全点等。这些数据以文件形式存储在非易失性存储器（SD 卡）中。

```
用户规划任务（QGC）
      │
      ▼
MAVLink 传输到飞控
      │
      ▼
dataman 模块 ──dm_write()──► /fs/microsd/dataman 文件
                              (SD卡持久化存储)
      │
      ▼
应用程序 ──dm_read()──► 读取任务参数
```

> 仿真环境和真实硬件的 dataman 行为**完全一致**（数据存储介质不同：硬盘 vs SD 卡）。

---

## 二、dataman 命令

入口文件：`./src/modules/dataman/dataman.cpp`，主函数：`dataman_main`

```
dataman <命令> [命令参数]
```

| 命令 | 功能 | 参数说明 |
|------|------|----------|
| `start` | 启动模块 | `-f <val>` 指定存储文件，`-r` 使用 RAM 存储；默认 `/fs/microsd/dataman` |
| `stop` | 停止模块 | — |
| `status` | 输出运行状态 | 各接口调用次数等信息 |

> 通常只在启动脚本中使用 `dataman start`，一般不使用 `stop`。

---

## 三、数据结构

### 3.1 任务类型枚举 `dm_item_t`

| 枚举值 | 值 | 说明 |
|--------|-----|------|
| `DM_KEY_SAFE_POINTS` | 0 | 预置安全点 |
| `DM_KEY_FENCE_POINTS` | — | 地理围栏顶点 |
| `DM_KEY_WAYPOINTS_OFFBOARD_0` | — | 航点任务（副本0） |
| `DM_KEY_WAYPOINTS_OFFBOARD_1` | — | 航点任务（副本1） |
| `DM_KEY_MISSION_STATE` | — | 任务状态（航点数量、当前使用的副本） |
| `DM_KEY_COMPAT` | — | 兼容性检查 |
| `DM_KEY_NUM_KEYS` | — | 任务项总数 |

> 每次上传任务时，系统会在 `OFFBOARD_0` 和 `OFFBOARD_1` 之间**切换**写入，防止写入失败时数据丢失。

### 3.2 任务状态结构体 `mission_s`

```cpp
// 对应 DM_KEY_MISSION_STATE
struct mission_s {
    uint64_t timestamp;       // 时间戳
    int32_t current_seq;      // 当前航点序号
    uint16_t count;           // 航点总数
    uint8_t dataman_id;       // 当前使用哪个副本（OFFBOARD_0 或 OFFBOARD_1）
    uint8_t _padding0[1];     // 占位填充
};
```

### 3.3 航点结构体 `mission_item_s`

```cpp
struct mission_item_s {
    double lat;                 // 纬度（度）
    double lon;                 // 经度（度）

    union {                     // 任务参数联合体
        struct {
            union {
                float time_inside;      // 停留时间（秒）
                float circle_radius;    // 圆形区域半径（米）
            };
            float acceptance_radius;    // 完成判定半径（米）
            float loiter_radius;        // 盘旋半径（米）
            float yaw;                  // 目标偏航角（度）
            float altitude;             // 任务高度（米）
            float _lat_float;           // 预留
            float __lon_float;          // 预留
        };
        float params[7];                // 数组形式批量处理
    };

    uint16_t nav_cmd;           // 导航指令类型
    int16_t do_jump_mission_index;  // 跳转目标索引
    uint16_t do_jump_repeat_count;  // 跳转重复次数

    union {
        uint16_t do_jump_current_count; // 当前跳转次数
        uint16_t vertex_count;          // 围栏顶点数
        uint16_t land_precision;        // 着陆精度
    };

    struct {                    // 位域标志
        uint16_t frame : 4,            // 坐标系类型
                 origin : 3,           // 原点参考系
                 loiter_exit_xtrack : 1,
                 force_heading : 1,    // 强制航向
                 altitude_is_relative : 1,  // 高度是否相对值
                 autocontinue : 1,     // 自动继续下一航点
                 vtol_back_transition : 1,
                 _padding0 : 4;
    };
    uint8_t _padding1[2];
};
```

**常用的 `nav_cmd` 导航指令**：

| 指令常量 | 值 | 含义 |
|----------|-----|------|
| `NAV_CMD_WAYPOINT` | 16 | 普通航点 |
| `NAV_CMD_LOITER_TIME_LIMIT` | 19 | 定时盘旋 |
| `NAV_CMD_LAND` | 21 | 自动着陆 |
| `NAV_CMD_TAKEOFF` | 22 | 自动起飞 |
| `NAV_CMD_LOITER_TO_ALT` | 31 | 盘旋改变高度 |

### 3.4 数量获取规则

| 任务类型 | 获取数量的方式 |
|----------|---------------|
| 航点 | `DM_KEY_MISSION_STATE` → `mission_s.count` |
| 地理围栏顶点 | `DM_KEY_FENCE_POINTS` + 索引0 → `mission_stats_entry_s.num_items` |
| 安全点 | `DM_KEY_SAFE_POINTS` + 索引0 → `mission_stats_entry_s.num_items` |

---

## 四、dataman API

```c
#include <dataman/dataman.h>
```

| API | 功能 | 返回值 |
|-----|------|--------|
| `dm_read(item, index, buffer, buflen)` | 读取任务参数 | 实际读取字节数 |
| `dm_write(item, index, buffer, buflen)` | 写入任务参数 | 实际写入字节数 |
| `dm_clear(item)` | 清除该类型所有数据 | 0 成功 |

---

## 五、完整示例：读取飞行任务

### 5.1 示例代码

```cpp
#include "test_app_main.h"
#include <dataman/dataman.h>

void TestApp::run()
{
    // ===== 1. 读取航点任务 =====
    struct mission_s mission;
    dm_read(DM_KEY_MISSION_STATE, 0, &mission, sizeof(mission_s));
    PX4_INFO("Mission has %u WPs", mission.count);

    struct mission_item_s item[mission.count];
    for (unsigned i = 0; i < mission.count; ++i) {
        // 使用 dataman_id 确定当前有效的副本
        dm_read((dm_item_t)mission.dataman_id, i, &item[i],
                sizeof(struct mission_item_s));
        PX4_INFO("mission #%d cmd ID is %d", i, item[i].nav_cmd);
    }

    // ===== 2. 读取地理围栏 =====
    struct mission_stats_entry_s stats;
    dm_read(DM_KEY_FENCE_POINTS, 0, &stats, sizeof(mission_stats_entry_s));
    PX4_INFO("Geofence has %u points", stats.num_items);

    struct mission_fence_point_s fence_point;
    for (unsigned i = 1; i <= stats.num_items; ++i) {  // 索引从 1 开始！
        dm_read(DM_KEY_FENCE_POINTS, i, &fence_point,
                sizeof(mission_fence_point_s));
        PX4_INFO("Geofence point #%d: (%f, %f)",
                 i, fence_point.lon, fence_point.lat);
    }

    // ===== 3. 读取安全点 =====
    dm_read(DM_KEY_SAFE_POINTS, 0, &stats, sizeof(mission_stats_entry_s));
    PX4_INFO("Safepoints has %u points", stats.num_items);

    struct mission_safe_point_s safe_point;
    for (unsigned i = 1; i <= stats.num_items; ++i) {
        dm_read(DM_KEY_SAFE_POINTS, i, &safe_point,
                sizeof(mission_safe_point_s));
        PX4_INFO("Safepoint #%d: (%f, %f)",
                 i, safe_point.lon, safe_point.lat);
    }
}
```

### 5.2 执行流程

```
1. QGC 规划任务（航点+围栏+安全点）
      │
2. Upload Required → MAVLink 上传
      │
3. PX4 内部调用 dm_write() 持久化
      │
4. 运行: test_app start
      │
5. dm_read() 读取 → 打印验证
```

### 5.3 关键注意事项

| 注意点 | 说明 |
|--------|------|
| **索引从 1 开始** | 地理围栏和安全点：索引 0 是统计信息，实际数据从 1 开始 |
| **航点副本切换** | `mission.dataman_id` 指示当前有效副本，读取航点时必须用它 |
| **统计信息类型不同** | 索引 0 返回 `mission_stats_entry_s`，索引 1+ 返回 `mission_fence_point_s` / `mission_safe_point_s` |
| **仿真与真机一致** | 仿真环境数据写硬盘，真机数据写 SD 卡，API 完全一致 |

---

## 六、与 dataman 模块的交互流程

```
┌──────────┐     MAVLink      ┌──────────┐    dm_write()    ┌──────────────┐
│   QGC    │ ───────────────► │   PX4    │ ──────────────►  │  SD卡 dataman │
│ 地面站   │   上传任务参数     │ Commander│    持久化存储     │     文件      │
└──────────┘                  └──────────┘                  └──────┬───────┘
                                                                   │
                                                          dm_read()│
                                                                   ▼
                                                           ┌──────────────┐
                                                           │  应用程序模块  │
                                                           │ (Navigator等) │
                                                           └──────────────┘
```

---

## 七、关键要点

| 要点 | 说明 |
|------|------|
| **dataman = 任务参数的持久化存取层** | 航点、围栏、安全点 |
| **三个核心 API** | `dm_read` / `dm_write` / `dm_clear` |
| **副本切换** | 系统维护两个航点副本防止写入失败 |
| **索引规则** | 航点从 0 开始；围栏/安全点从 1 开始（0 是统计信息） |
| **nav_cmd** | 每个航点有独立的导航指令（起飞/降落/盘旋等） |
| **位域标志** | `altitude_is_relative`、`autocontinue`、`frame` 等控制在 2 字节内 |
