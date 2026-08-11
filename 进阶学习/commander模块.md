# Commander 模块

> 来源：《22-Commander模块（一）》 | PX4 控制系统的核心调度器

---

## 一、一句话理解 Commander

```
Commander = 无人机的大脑中枢

它决定三件事：
  ① 电机能不能转？（解锁状态）
  ② 飞机怎么飞？（飞行模式）
  ③ 传感器不够时怎么办？（故障安全降级）
```

---

## 二、Commander 管理的三种状态

```
┌─────────────────────────────────────────────────────────────┐
│                     Commander 核心职责                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│   │   解锁状态     │    │   飞行模式     │    │   导航状态     │  │
│   │ arming_state  │    │  main_state   │    │  nav_state   │  │
│   ├──────────────┤    ├──────────────┤    ├──────────────┤  │
│   │ 电机能不能转？ │    │ 谁来控制飞机？ │    │ 具体执行什么？  │  │
│   │ Locked/Armed  │    │ 手动/辅助/自主 │    │ 定高/位置/航线 │  │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│          │                   │                   │           │
│          └───────────────────┼───────────────────┘           │
│                              │                               │
│                    最终决定飞控行为                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、解锁状态（Arming State）

### 3.1 六种解锁状态

```
 上电
  │
  ▼
┌──────┐    自检通过    ┌──────────┐    解锁成功    ┌──────────┐
│ INIT │ ─────────────► │ STANDBY  │ ─────────────► │  ARMED   │
│  0   │                │    1     │                │    2     │
└──────┘                └────┬─────┘                └──────────┘
                             │
                             │ 发生错误
                             ▼
                      ┌──────────────┐
                      │ STANDBY_ERROR │
                      │      3        │
                      └──────────────┘

其他状态：
  SHUTDOWN (4)      → 重启
  IN_AIR_RESTORE (5) → 空中恢复（极端情况）
```

| 状态常量 | 值 | 含义 | 电机/舵机 |
|----------|-----|------|-----------|
| `ARMING_STATE_INIT` | 0 | 上电初始化 | ❌ 不响应 |
| `ARMING_STATE_STANDBY` | 1 | 准备就绪 | ❌ 不响应 |
| **`ARMING_STATE_ARMED`** | **2** | **已解锁** | **✅ 响应指令** |
| `ARMING_STATE_STANDBY_ERROR` | 3 | 就绪但有错误 | ❌ 不响应 |
| `ARMING_STATE_SHUTDOWN` | 4 | 重启中 | ❌ 不响应 |
| `ARMING_STATE_IN_AIR_RESTORE` | 5 | 空中恢复 | — |

> 📌 关键：只有 `ARMED(2)` 状态电机才会转！`arm_state` 描述状态，`actuator_armed.armed`（布尔值）才是实际控制变量。

### 3.2 状态转换表（核心逻辑）

```
              当前状态 →
目标状态 ↓    INIT  STANDBY  ARMED  STANDBY_ERR  SHUTDOWN  IN_AIR_RESTORE
INIT           ✅     ✅       ❌       ✅           ❌          ❌
STANDBY        ✅     ✅       ✅       ❌           ❌          ❌
ARMED          ❌     ✅       ✅       ❌           ❌          ✅
STANDBY_ERROR  ✅     ✅       ❌       ✅           ❌          ❌
SHUTDOWN       ✅     ✅       ❌       ✅           ✅          ✅
IN_AIR_RESTORE ❌     ❌       ❌       ❌           ❌          ❌
```

**典型路径示例：**
- ✅ `INIT → STANDBY`：开机自检通过
- ✅ `STANDBY → ARMED`：解锁起飞
- ✅ `ARMED → STANDBY`：上锁落地
- ❌ `INIT → ARMED`：不允许！必须先过自检

### 3.3 解锁转换函数核心流程

```
arming_state_transition()
        │
        ├─ 目标状态 = 当前状态？
        │   └─ YES → 返回 TRANSITION_NOT_CHANGED
        │
        ├─ 查转换表 arming_transitions[new][current]
        │   └─ false → 返回 TRANSITION_DENIED
        │
        ├─ 切换到 ARMED？
        │   └─ YES → 执行飞行前检查（canArm）
        │       └─ 检查失败 → 拒绝
        │
        ├─ HIL 模式？
        │   └─ YES → 特殊处理（强制 lockdown，允许 STANDBY）
        │
        └─ 全部通过 → 返回 TRANSITION_CHANGED
            └─ 记录解锁/上锁原因和时间
```

---

## 四、飞行模式（Flight Mode）

### 4.1 模式分类一览

```
飞行模式（24种）
├── 手动类 ── 用户摇杆直接控制
│   ├── MANUAL      → 摇杆直通舵机/电机
│   ├── STABILIZED  → 摇杆→姿态角指令（飞控辅助稳定）
│   └── ACRO        → 摇杆→角速度指令（特技飞行）
│
├── 辅助类 ── 用户控制 + 飞控辅助
│   ├── ALTCTL      → 定高飞行（水平位置可能漂移）
│   └── POSCTL      → 位置保持（飞控自动抗风）
│
└── 自主类 ── 飞控完全自主
    ├── AUTO_MISSION  → 执行预设航线
    ├── AUTO_LOITER   → 当前点盘旋
    ├── AUTO_RTL      → 自动返航
    ├── AUTO_TAKEOFF  → 自动起飞
    ├── AUTO_LAND     → 自动降落
    ├── OFFBOARD      → 机载电脑控制
    ├── ORBIT         → 环绕飞行
    ├── AUTO_FOLLOW_TARGET → 目标跟随
    ├── AUTO_PRECLAND → 精准着陆
    └── AUTO_VTOL_TAKEOFF → VTOL 起飞
```

### 4.2 各模式对比速查

| 模式 | 常量 | 水平控制 | 垂直控制 | 需要传感器 |
|------|------|----------|----------|-----------|
| MANUAL | `MAIN_STATE_MANUAL` | 摇杆直通 | 摇杆直通 | — |
| STABILIZED | `MAIN_STATE_STAB` | 姿态角指令 | 油门直通 | 角速度+姿态 |
| ACRO | `MAIN_STATE_ACRO` | 角速度指令 | 油门直通 | 角速度 |
| **ALTCTL** | `MAIN_STATE_ALTCTL` | 姿态角指令 | **自动定高** | +本地高度 |
| **POSCTL** | `MAIN_STATE_POSCTL` | **位置保持** | **自动定高** | +本地位置 |
| **AUTO_MISSION** | — | **全自主** | **全自主** | +全球位置+任务 |
| AUTO_RTL | — | **全自主** | **全自主** | +全球位置+home点 |
| OFFBOARD | — | 机载电脑 | 机载电脑 | +offboard信号 |

---

## 五、模式切换全流程（七阶段核心流程）

> 这是 Commander 最重要的工作流，分 7 个阶段逐环推进：

```
┌──────────────────────────────────────────────────────────────────┐
│                         Commander::run() 主循环                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [阶段1] 触发请求                                                │
│    地面站(MAVLink) ──► vehicle_command ──► handle_command()      │
│    遥控器(RC)      ──► action_request  ──► executeActionRequest()│
│         │                                                        │
│         ▼                                                        │
│  [阶段2] 更新用户意图 UserModeIntention::change()                 │
│    ├─ 未解锁 → 总是允许切换                                       │
│    ├─ 已解锁 → 调用 canRun() 检查传感器是否满足                   │
│    └─ 失败时尝试回退（如 POSCTL → ALTCTL）                        │
│         │                                                        │
│         ▼                                                        │
│  [阶段3] 健康检查 HealthAndArmingChecks::update() (@10Hz)         │
│    遍历 30+ 检查器：加速计/空速/位置/模式需求/电池/...            │
│    └─ getModeRequirements() 验证每种模式需要的传感器              │
│         │                                                        │
│         ▼                                                        │
│  [阶段4] 故障安全验证 handleModeIntentionAndFailsafe()            │
│    ├─ RC信号丢失？→ 返航/降落                                     │
│    ├─ 地面站失联？→ 返航/降落                                     │
│    ├─ 电池低电压？→ 降落                                          │
│    ├─ 传感器降级？→ 模式回退（POSCTL→ALTCTL）                     │
│    └─ 任务结束？→ 盘旋/返航                                       │
│         │                                                        │
│         ▼                                                        │
│  [阶段5] 解锁状态交互 ArmStateMachine::arming_state_transition()  │
│    ├─ 模式需解锁？→ 执行预飞行检查 → 解锁                          │
│    ├─ Failsafe触发Disarm？→ 上锁                                  │
│    └─ HIL模式特殊处理                                             │
│         │                                                        │
│         ▼                                                        │
│  [阶段6] 设置控制标志 updateControlMode()                         │
│    根据最终 nav_state 设置 flag_control_* 系列标志位              │
│    如 POSCTL：手动+速率+姿态+高度+爬升率+位置+速度 全开           │
│         │                                                        │
│         ▼                                                        │
│  [阶段7] 后处理与循环                                             │
│    发布 vehicle_status / vehicle_control_mode / actuator_armed   │
│    蜂鸣/LED 反馈 → 睡眠等待 → 回到阶段1                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.1 阶段 1：触发请求

| 触发方式 | 数据来源 | 处理函数 | uORB 主题 |
|----------|----------|----------|-----------|
| 地面站 | MAVLink 指令 | `handle_command()` | `vehicle_command` |
| 遥控器 | RC 开关/手势 | `executeActionRequest()` | `action_request` |

**地面站命令解析**：`VEHICLE_CMD_DO_SET_MODE` 通过 `param2`（主模式）+ `param3`（子模式）组合成目标导航状态。

**主模式映射**：
| param2 | 含义 |
|--------|------|
| `PX4_CUSTOM_MAIN_MODE_MANUAL (1)` | 手动 |
| `PX4_CUSTOM_MAIN_MODE_ALTCTL (2)` | 定高 |
| `PX4_CUSTOM_MAIN_MODE_POSCTL (3)` | 位置保持 |
| `PX4_CUSTOM_MAIN_MODE_AUTO (4)` | 自主（需配合子模式） |
| `PX4_CUSTOM_MAIN_MODE_ACRO (5)` | 特技 |
| `PX4_CUSTOM_MAIN_MODE_OFFBOARD (6)` | 离线 |
| `PX4_CUSTOM_MAIN_MODE_STABILIZED (7)` | 增稳 |

**自主子模式**（主模式=AUTO 时生效）：
| param3 | 含义 |
|--------|------|
| 1 | 准备 |
| 2 | 自动起飞 |
| 3 | 盘旋 |
| 4 | 航线 |
| 5 | 返航 |
| 6 | 降落 |
| 8 | 目标跟随 |
| 9 | 精准着陆 |

### 5.2 阶段 2：用户意图变更

```
UserModeIntention::change(desired_nav_state)
         │
         ├─ 未解锁？ → always_allow = true（随便切）
         │
         ├─ 已解锁？ → canRun(nav_state) 检查传感器
         │    └─ 失败 → 尝试回退（POSCTL→ALTCTL）
         │
         └─ 成功 → _user_intented_nav_state = 目标模式
                  _had_mode_change = true
```

### 5.3 阶段 3：健康检查

每 **100ms (10Hz)** 执行一次，遍历 30+ 个检查器：

```
检查器列表（部分）：
├── 加速计检查
├── 空速检查
├── 气压计检查
├── 电池检查
├── GPS 检查
├── 位置估计器检查
├── 模式需求检查（ModeChecks）← 关键！
├── 遥控器检查
├── 电源检查
└── ...
```

**模式需求检查**：每个导航状态都定义了一个**位掩码**，表示必需的传感器：

| 需求标志位 | 含义 | 缺失导致的问题 |
|-----------|------|--------------|
| `mode_req_angular_velocity` | 需要角速度 | 几乎所有模式都需要 |
| `mode_req_attitude` | 需要姿态 | 姿态估计失败则无法自动飞行 |
| `mode_req_local_alt` | 需要本地高度 | 无法定高 |
| `mode_req_local_position` | 需要本地位置 | 无法位置保持 |
| `mode_req_global_position` | 需要全球位置(GPS) | 无法执行航线 |
| `mode_req_mission` | 需要任务数据 | 无法执行航线任务 |
| `mode_req_home_position` | 需要 home 点 | 无法返航 |
| `mode_req_offboard_signal` | 需要机载电脑信号 | 无法 offboard |
| `mode_req_manual_control` | 需要遥控器 | 手动模式不可用 |
| `mode_req_prevent_arming` | 禁止解锁 | 降落/终止/返航时不可解锁 |

### 5.4 阶段 4：故障安全（Failsafe）

> 这是 Commander 最"智能"的部分——传感器出问题时自动降级或返航。

```
Failsafe 检查清单：
┌──────────────────────┬──────────────────────┐
│      检测条件          │      触发动作          │
├──────────────────────┼──────────────────────┤
│ RC 遥控信号丢失        │  返航(RTL) 或 降落    │
│ 地面站通信丢失         │  返航(RTL) 或 降落    │
│ 电池低电压             │  降落                 │
│ 传感器精度不足         │  模式降级              │
│        POSCTL→ALTCTL  │  (位置保持→定高)      │
│        无法定高时→手动 │                       │
│ 任务完成               │  盘旋等待              │
│ 飞行超时/逆风过大      │  返航                  │
│ VTOL 过渡失败          │  紧急降落              │
└──────────────────────┴──────────────────────┘
```

**模式回退机制示例**：
```
用户请求 POSCTL
  → GPS 精度不够
  → Failsafe 覆盖为 ALTCTL（定高模式）
  → 如果高度传感器也故障
  → 降级为 STABILIZED（增稳模式）
```

### 5.5 阶段 5：解锁状态交互

ArmStateMachine 确保模式切换与解锁/上锁正确配合。例如：
- 自主航线模式需要先解锁（ARMED）才能执行
- Failsafe 触发 Disarm 时会自动上锁
- 记录解锁/上锁的原因和时间

### 5.6 阶段 6：控制标志发布

根据最终导航状态，设置各控制层级的标志位：

```cpp
// 以 POSCTL 为例
flag_control_manual_enabled    = true  // 手动输入使能
flag_control_rates_enabled     = true  // 角速率控制
flag_control_attitude_enabled  = true  // 姿态控制
flag_control_altitude_enabled  = true  // 高度控制
flag_control_climb_rate_enabled= true  // 爬升率控制
flag_control_position_enabled  = true  // 位置控制
flag_control_velocity_enabled  = true  // 速度控制
```

> 下游模块（PositionControl、AttitudeControl）根据这些标志位决定是否执行控制逻辑。

### 5.7 阶段 7：发布与反馈

每 **500ms（2Hz）** 或状态变更时立即发布：

| 发布内容 | uORB 主题 | 消费者 |
|----------|-----------|--------|
| 解锁状态 | `actuator_armed` | 输出模块 |
| 控制模式 | `vehicle_control_mode` | 控制模块 |
| 飞行器状态 | `vehicle_status` | 所有模块 |
| 故障检测 | `failure_detector_status` | 日志/地面站 |

---

## 六、关键源码文件速查

| 文件 | 功能 |
|------|------|
| `src/modules/commander/Commander.cpp` | 主循环 `run()`，七阶段流程 |
| `Arming/ArmStateMachine.cpp` | 解锁状态机，`arming_transitions[][]` 转换表 |
| `UserModeIntention.cpp` | 用户模式意图管理，`change()` 函数 |
| `ModeUtil/mode_requirements.cpp` | 每种导航模式的传感器需求定义 |
| `HealthAndArmingChecks.cpp` | 健康检查框架，30+ 检查器遍历 |
| `failsafe.cpp` | 故障安全逻辑，`CHECK_FAILSAFE` 宏 |
| `control_mode.cpp` | `getVehicleControlMode()` 设置控制标志 |

---

## 七、核心要点总结

| # | 要点 |
|---|------|
| 1 | Commander 是 **PX4 的大脑**，管理三种状态：解锁、飞行模式、导航状态 |
| 2 | **只有 ARMED 状态电机才转**，其他状态都是锁定的 |
| 3 | 飞行模式分三大类：**手动（3种）、辅助（2种）、自主（10+种）** |
| 4 | 模式切换走**七阶段流程**：触发 → 意图 → 健康检查 → Failsafe → 解锁 → 控制标志 → 发布 |
| 5 | **Failsafe 是关键**：传感器出问题时自动降级模式或触发返航/降落 |
| 6 | 每种模式需要的传感器不同：手动几乎不需要，自主航线需要 GPS+位置+任务全套 |
| 7 | 未解锁时模式可以随便切，**已解锁后的切换必须通过健康检查** |
