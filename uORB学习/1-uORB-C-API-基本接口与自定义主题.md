---
tags:
  - PX4
  - uORB
  - 发布订阅
  - C-API
  - 自定义主题
date: 2026-08-11
---

# uORB C API：基本接口与自定义主题

> 来源：《12-uORB基本接口使用方法》

---

## 一、uORB 是什么

uORB（Micro Object Request Broker）是 PX4 的**核心数据交互模块**，负责上百个独立任务模块之间的数据传输。

```
┌──────────────────────────────────────────────────────────┐
│                    uORB 核心设计理念                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   发布者（Publisher）         订阅者（Subscriber）         │
│   ┌──────────┐              ┌──────────┐                │
│   │ 传感器驱动 │              │ 飞控算法  │                │
│   │          │    uORB      │          │                │
│   │ 只管发数据 │ ═══ 主题 ═══＞ │ 只管取数据 │                │
│   │          │  (Topic)     │          │                │
│   └──────────┘              └──────────┘                │
│                                                          │
│   特性：                                                  │
│   ✅ 异步通信：发送方和接收方互不依赖                       │
│   ✅ 多对多：同一主题可被多个模块发布/订阅                   │
│   ✅ 只保留最新数据：旧数据被新数据覆盖                      │
│   ✅ 双向灵活：同一模块可同时发布和订阅多个主题               │
└──────────────────────────────────────────────────────────┘
```

### 对比传统方式

| 传统方式 | uORB 方式 |
|----------|-----------|
| 模块直接耦合，修改一处影响全局 | 模块解耦，逻辑清晰 |
| 扩展功能需大改系统 | 更换传感器/主板，应用代码不变 |
| 多人协作容易冲突 | 通过主题接口隔离，独立开发 |

### 底层实现：虚拟文件 + 主题

每条 uORB 消息对应 `/obj/主题名` 的虚拟设备文件。多个任务同时打开同一虚拟文件，通过读写驱动文件实现数据共享。每个主题仅包含**一种**消息类型。

---

## 二、uORB 调试命令

### 2.1 系统命令

| 命令 | 环境 | 功能 |
|------|------|------|
| `ls /obj` | 仅 NSH | 列出所有 uORB 主题（每个主题对应一个虚拟文件） |
| `listener <topic> [n]` | NSH/PXH | 监听主题数据，打印最新 n 条（默认1条） |

> NSH = NuttX Shell（系统级），PXH = PX4 Shell（应用级）。`ls` 是系统级文件操作，PXH 不支持访问 `/obj` 这类 NuttX 虚拟文件路径。

### 2.2 模块命令（`uorb <cmd>`）

| 命令 | 功能 | 说明 |
|------|------|------|
| `uorb status` | 查看所有主题摘要 | 显示名称、实例数、订阅数、队列长度、消息大小、文件路径 |
| `uorb top` | 动态查看主题信息 | 比 status 多显示**消息速率(RATE)** |
| `uorb top -1` | 同上，但仅打印一次 | `-a` 打印所有主题，`topic1 topic2...` 过滤指定主题 |
| `uorb start` | 启动 uORB 模块 | 通常写入启动脚本 |

---

## 三、基本 C API（五大接口）

### 3.1 API 总览

```
发布者流程：                         订阅者流程：
                                    
orb_advertise()  ← 公告主题（仅一次）   orb_subscribe()  ← 订阅主题（仅一次）
       │                                     │
       ▼                                     ▼
orb_publish()   ← 发布数据（每次）       orb_check()     ← 检查是否更新（每次）
                                            │
                                            ▼
                                       orb_copy()      ← 拷贝数据到本地（有更新时）
```

### 3.2 接口详解

#### （1）公告主题

```c
orb_advert_t orb_advertise(const struct orb_metadata *meta, const void *data);
```

| 项目 | 说明 |
|------|------|
| `meta` | 主题元数据句柄，由 `ORB_ID(主题名)` 生成 |
| `data` | 初始数据指针（可为 `NULL`） |
| **返回值** | `orb_advert_t` 公告句柄，唯一标识该公告实例 |

> 同一主题可公告多个实例（如左、右电机分别公告 `motor_status`），句柄用于区分。

#### （2）发布数据

```c
int orb_publish(const struct orb_metadata *meta, orb_advert_t handle, const void *data);
```

| 项目 | 说明 |
|------|------|
| `meta` | 必须与 `orb_advertise` 的 meta 一致 |
| `handle` | 公告句柄，明确"往哪个实例写" |
| `data` | 待发布数据指针 |
| **返回值** | 0 成功，非零为错误码 |

> 可在中断服务函数中调用（不经过操作系统）。

#### （3）订阅主题

```c
int orb_subscribe(const struct orb_metadata *meta);
```

即使主题尚未被公告，订阅也会成功（但暂时获取不到数据）。每个订阅者获得独立句柄，互不干扰。

#### （4）检查更新

```c
int orb_check(int handle, bool *updated);
```

| 项目 | 说明 |
|------|------|
| `updated` | 输出参数，`true` = 有新数据，`false` = 未更新 |

#### （5）拷贝数据

```c
int orb_copy(const struct orb_metadata *meta, int handle, void *buffer);
```

将 uORB 中最新的数据拷贝到本地 buffer。

---

## 四、定义自定义 uORB 主题

### 4.1 创建 .msg 文件

在 `./msg/` 目录下创建 `test_topic.msg`：

```
uint64 timestamp          # 必须包含：时间戳（日志模块需要）
uint64 data               # 实际数据字段
uint8 TEST_CONST_DATA = 0  # 常量字段（不可修改）
# TOPICS test_topic test_topic_x  # 可选：创建多个主题实例
```

**语法规则**：
- 变量无需分号
- 类型为 `uint64`（非 `uint64_t`），变量名不可大写开头
- `# TOPICS` 声明同类型的多个独立主题，第一个必须与文件名同名

### 4.2 注册到编译

将 `test_topic.msg` 添加到 `./msg/CMakeLists.txt` 的 set 列表中。编译后自动生成：
- `struct test_topic_s` — 消息结构体
- `ORB_ID(test_topic)` / `ORB_ID(test_topic_x)` — 主题元数据

---

## 五、完整示例：双线程 uORB 发布/订阅

### 5.1 架构

```
test_app start
    │
    ▼
┌─────────────────────────────────────────────┐
│  PX4 Task "test_app" (主线程：等待退出)       │
│                                             │
│  ┌──────────────────┐  ┌──────────────────┐ │
│  │ thread_loop_send │  │thread_loop_recv  │ │
│  │                  │  │                  │ │
│  │ orb_advertise    │  │ orb_subscribe    │ │
│  │ orb_publish(1Hz)─┼──┼→ uORB ─→ orb_check│ │
│  │                  │  │ orb_copy + 打印   │ │
│  └──────────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────┘
```

### 5.2 发送线程

```cpp
void *thread_loop_send(void *arg) {
    test_topic_s test_data = {};

    // ① 公告主题（仅一次）
    orb_advert_t test_pub = orb_advertise(ORB_ID(test_topic_x), &test_data);

    uint64_t cnt = 0;
    while (!thread_should_exit) {
        // ② 填充数据
        test_data.timestamp = hrt_absolute_time();  // 微秒时间戳
        test_data.data = cnt;

        // ③ 发布到 uORB 总线
        orb_publish(ORB_ID(test_topic_x), test_pub, &test_data);

        ++cnt;
        px4_sleep(1);  // 1Hz 频率
    }
    return nullptr;
}
```

### 5.3 接收线程

```cpp
void *thread_loop_receive(void *arg) {
    // ① 订阅主题（仅一次）
    int test_sub = orb_subscribe(ORB_ID(test_topic_x));

    bool updated;
    test_topic_s test_data = {};

    while (!thread_should_exit) {
        // ② 检查是否有新数据
        orb_check(test_sub, &updated);

        if (updated) {
            // ③ 拷贝最新数据到本地
            orb_copy(ORB_ID(test_topic_x), test_sub, &test_data);

            // ④ 打印（时间戳除以 1,000,000 转为秒）
            PX4_INFO("uorb topic send time is %lu s",
                     test_data.timestamp / 1000000u);
            PX4_INFO("test data is cnt = %lu", test_data.data);
        }
        px4_sleep(1);
    }
    return nullptr;
}
```

### 5.4 主线程：创建线程 + 等待退出

```cpp
void TestApp::run() {
    PX4_INFO("start!");

    // 配置线程属性（栈大小 + 优先级）
    pthread_attr_t pth_attr;
    pthread_attr_init(&pth_attr);
    pthread_attr_setstacksize(&pth_attr, 64);

    struct sched_param param;
    pthread_attr_getschedparam(&pth_attr, &param);
    param.sched_priority = SCHED_PRIORITY_MIN + 5;
    pthread_attr_setschedparam(&pth_attr, &param);

    // 创建发送和接收线程
    pthread_t pth_send, pth_receive;
    pthread_create(&pth_send,    &pth_attr, thread_loop_send,    nullptr);
    pthread_create(&pth_receive, &pth_attr, thread_loop_receive, nullptr);
    pthread_attr_destroy(&pth_attr);

    // 主线程等待退出信号
    while (!should_exit()) {
        px4_sleep(1);
    }

    // 优雅关闭
    thread_should_exit = true;
    pthread_join(pth_send, nullptr);
    pthread_join(pth_receive, nullptr);
}
```

### 5.5 执行时序

```
时间 →

主线程:    创建线程 → 销毁属性 ── while(sleep) ──→ 通知退出 → join等待 → 结束

pth_send:             开始 ──→ publish + sleep (1Hz循环) ──→ 检测标志 → 退出

pth_receive:          开始 ──→ check + print (1Hz循环) ──→ 检测标志 → 退出
```

---

## 六、关键要点

| 要点 | 说明 |
|------|------|
| **异步解耦** | 发布者不关心谁来接收，订阅者不关心谁来发送 |
| **只保留最新** | 新数据直接覆盖旧数据，确保实时性 |
| **五步 API** | 公告→发布；订阅→检查→拷贝 |
| **.msg 文件** | 定义消息结构，`# TOPICS` 创建多实例 |
| **orb_publish 可中断调用** | 不经过操作系统，中断安全 |
| **timestamp 必含** | 每个 topic 必须有 `uint64 timestamp` |
