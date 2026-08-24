# PX4 入门学习笔记

> 来源：《4-PX4系统介绍》 + 《5-PX4总体框架》 + 《6-PX4系统开发流程》

---

# 一、开发环境搭建

## 1.1 开发平台

| 项目 | 说明 |
|------|------|
| **OS** | Ubuntu LTS 20.04（虚拟机），官方推荐 |
| **为什么不 Windows** | 编译慢，对 JMAVSim/Gazebo 支持不理想 |
| **飞控硬件** | Pixhawk6C mini（CUAV X7+Pro 等均可） |
| **地面站** | QGroundControl（QGC） |
| **编辑器** | VSCode + C/C++ 插件 + CMake 插件 |

## 1.2 源码获取

```bash
# 安装 git
sudo apt install git

# 下载 PX4 v1.14.3 源码
git clone -b v1.14.3 https://github.com/PX4/PX4-Autopilot.git --recursive

# 一键安装编译工具链
cd PX4_1.14.3
./Tools/setup/ubuntu.sh
# 安装完成后重启虚拟机
```

> ⚠️ 从国外下载可能较慢，网络问题导致失败时多执行几次 `ubuntu.sh`

## 1.3 地面站（QGroundControl）六大功能

| 功能 | 说明 |
|------|------|
| **飞行监控** | 实时图形化显示高度、速度、位置、系统状态 |
| **航前检查** | 检测传感器、遥控器、飞行模式等 |
| **飞行计划** | 规划航点、起降方式、任务机动 |
| **参数配置** | 增删改查 PX4 参数（PID 等） |
| **数据记录与分析** | 遥测数据记录、机载日志下载 |
| **飞控调试** | 固件上传、NSH 终端 |

---

# 二、飞控开发模式：前后台 vs RTOS

## 2.1 对比

| | 前后台模式（超循环） | RTOS 模式 |
|------|------|------|
| **架构** | 主循环轮询 + 中断服务 | 多任务并发 + 优先级调度 |
| **实时性** | ❌ 差，任务顺序执行，紧急任务需排队等待 | ✅ 高，高优先级任务可抢占 CPU |
| **移植性** | ❌ 与硬件绑定，移植工作量大 | ✅ 硬件抽象，统一接口，跨平台 |
| **扩展性** | ❌ 任务耦合严重，增加任务管理困难 | ✅ 模块化，任务独立，易于扩展 |
| **适用场景** | 简单飞控程序 | 复杂系统、多人协作、持续维护 |

## 2.2 操作系统四大模块

```
操作系统
├── 任务调度 → 决定哪个任务何时使用 CPU（并发机制）
├── 内存管理 → 分配/回收内存，解决碎片问题
├── 文件系统 → 数据以文件形式组织，提供标准读写接口
└── I/O 系统  → 驱动管理，"一切皆文件"
```

### NuttX 实时操作系统特点

| 方面 | 特点 |
|------|------|
| **任务调度** | 完全抢占式，高优先级任务立即抢占 CPU |
| **内存管理** | 无虚拟内存，避免随机 I/O 阻塞保证实时性 |
| **文件系统** | 类 Linux 虚拟文件系统（VFS），根文件系统为伪文件系统 |
| **Shell** | NSH（NuttX Shell），用法类似 Linux Shell |

> PX4 通过中间转换层提供 `px4_xxx` 标准接口，实现应用程序跨平台兼容。

---

# 三、PX4 总体框架

## 3.1 架构图（文字版）

```
┌─────────────────────────────────────────────────────┐
│                    中间件层                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ 外部通信  │  │  数据存取  │  │     驱动程序      │  │
│  │ (MAVLink) │  │          │  │ (IMU/GPS/舵机等)  │  │
│  └─────┬─────┘  └────┬─────┘  └────────┬─────────┘  │
│        │              │                 │            │
│        └──────────────┼─────────────────┘            │
│                       ▼                              │
│              ┌────────────────┐                      │
│              │  消息总线 uORB  │  ← 系统核心          │
│              └───────┬────────┘                      │
├──────────────────────┼──────────────────────────────┤
│                      ▼                               │
│  ┌──────────────────────────────────────────────┐   │
│  │              飞行控制栈 (Flight Stack)         │   │
│  │  姿态/位置估计 → 导航控制 → 位置控制 → 姿态控制  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

## 3.2 五大模块

### （1）消息总线 uORB
- **系统核心**，处于架构中心
- **核心思想：去耦合**——所有模块通过 uORB 通信，不直接交互
- 传感器数据 → uORB → 飞控模块；模块间通信也通过 uORB

### （2）外部通信 MAVLink
- 协议：MAVLink V2.0
- 用途：无人机 ↔ 地面站、无人机 ↔ 无人机
- 基于串口通信（也支持网络通信）

### （3）数据存取
| 子模块 | 功能 |
|--------|------|
| **dataman** | 飞行任务参数（航路点、安全点、地理围栏） |
| **param** | 配置参数（飞控参数、传感器校准参数） |
| **logger** | 飞行日志记录，记录 uORB 消息供离线分析 |

### （4）驱动程序
- 位于系统底层，获取外围设备数据
- 重要驱动：IMU、GPS、遥控器输入、舵机输出等
- 主动向 uORB 发布设备数据

### （5）飞行控制栈
- 无人机姿态/位置估计、飞行模式管理、导航和控制
- 支持：固定翼、多旋翼、VTOL 等
- 数据流：**传感器 → 位姿估计 → 导航/位置/姿态控制 → 执行机构**

## 3.3 重要目录结构

| 目录 | 用途 |
|------|------|
| `board/` | 硬件平台配置、编译脚本、NuttX 配置（`defconfig`） |
| `build/` | 编译输出目录（编译后生成），含固件 `.px4` 文件 |
| `cmake/` | CMake 编译规则 |
| **`msg/`** | uORB 消息定义（`.msg` 文件），编译时翻译为 `.h/.cpp` |
| `platforms/` | 平台相关实现，`NuttX/` 子目录 + `common/include/` 通用头文件 |
| **`ROMFS/`** | 启动脚本（`px4fmu_common/init.d/`），`airframe` 文件定义机型 |
| **`src/`** | ⭐ 飞控开发最重要的目录 |
| `src/drivers/` | 传感器驱动代码 |
| `src/examples/` | 示例代码 |
| `src/lib/` | 标准库（坐标转换、矩阵、PID、L1 控制器等） |
| `src/modules/` | 上层应用任务，飞控核心模块 |
| `Tools/` | 工具脚本（固件烧写、仿真等） |

---

# 四、PX4 开发流程：Hello Sky!

## 4.1 六步开发流程

```
编写源码 → 编写 CMakeLists.txt → 注册应用(Kconfig + default.px4board)
→ 编译 → 上传固件 → NSH 终端运行
```

### 第一步：编写源码

```cpp
// ./src/examples/test_app/test_app_main.cpp
#include <px4_log.h>

extern "C" __EXPORT int test_app_main(int argc, char * argv[]);

int test_app_main(int argc, char * argv[])
{
    PX4_INFO("Hello Sky!");
    return 0;
}
```

**要点：**
- 入口函数格式：`<module_name>_main(int argc, char *argv[])`
- 用 `__EXPORT` 声明导出
- 不是从 `main()` 启动（运行在 RTOS 之上）

**日志函数等级：**

| 函数 | 等级 | 用途 |
|------|------|------|
| `PX4_DEBUG()` | DEBUG | 调试信息 |
| `PX4_INFO()` | INFO | 常规信息 |
| `PX4_WARN()` | WARN | 警告 |
| `PX4_ERR()` | ERROR | 错误 |
| `PX4_PANIC()` | PANIC | 严重错误 |

> 日志函数自动换行，无需 `\n`

### 第二步：编写 CMakeLists.txt

```cmake
px4_add_module(
    MODULE examples__test_app        # 双下划线分隔路径
    MAIN test_app                    # 去掉 _main 后缀
    COMPILE_FLAGS
    SRCS
        test_app_main.cpp            # 源文件
    DEPENDS
)
```

### 第三步：注册应用

**Kconfig 文件**（`test_app/Kconfig`）：
```
menuconfig EXAMPLES_TEST_APP
    bool "test_app"
    default n
    -----help----
        Enable support for test_app
```

**default.px4board 末尾添加**（`./boards/px4/fmu-v6c/default.px4board`）：
```
CONFIG_EXAMPLES_TEST_APP=y
```

> 仿真目标对应的注册文件：`./boards/px4/sitl/default.px4board`

### 第四步：编译

```bash
# 查看支持的编译目标
make list_config_targets

# 首次编译前清理
make distclean

# 编译（以 Pixhawk6C mini 为例）
make px4_fmu-v6c

# 遇到奇怪问题时的清理
make clean
```

> CUAV X7 Pro → `make cuav_x7pro`

### 第五步：上传固件

```bash
make px4_fmu-v6c upload
```

也可通过 QGC 地面站上传 `.px4` 固件文件。

### 第六步：NSH 终端运行

QGC → Analyze Tools → NSH 终端：
```
test_app        # 运行程序
test_app start  # 带参数运行
```

---

## 4.2 支持标准命令的完整应用（仿真版）

```cpp
#include <string.h>
#include <px4_log.h>

extern "C" __EXPORT int test_app_main(int argc, char * argv[]);

static int is_running = 0;

int test_app_main(int argc, char * argv[])
{
    if (argc <= 1 || strcmp(argv[1], "-h") == 0
        || strcmp(argv[1], "help") == 0)
        return print_usage();          // 无参数 → 打印用法

    if (strcmp(argv[1], "start") == 0)
        return start_command();        // start → 启动

    if (strcmp(argv[1], "status") == 0)
        return status_command();       // status → 查看状态

    if (strcmp(argv[1], "stop") == 0)
        return stop_command();         // stop → 停止

    return custom_command(argc-1, argv+1);  // 未知命令 → 报错
}
```

**标准命令模式**：`start` / `status` / `stop` / `help`（参考 `ModuleBase` 类）

---

# 五、NSH 常用命令

## 5.1 命令格式

```
命令名 参数1 参数2 ... 参数n
```

| 命令 | 功能 |
|------|------|
| `help` | 列出所有命令（上半部分：NuttX 内置命令；下半部分：PX4 应用） |
| `ls` | 查看文件和目录 |
| `cd` | 切换目录 |
| `pwd` | 显示当前路径 |
| `echo` | 打印文本 |
| `free` | 查看内存使用（无参数，与 Linux 不同） |
| `ver all` | 查看软硬件版本信息 |
| `top` | 任务管理器（动态显示任务状态，回车退出） |
| `top once` | 一次性显示任务信息（不持续占用终端） |

## 5.2 free 命令输出

| 字段 | 含义 |
|------|------|
| `total` | 内存总容量（字节） |
| `used` | 已使用内存量，持续增长可能表示内存泄漏 |
| `free` | 当前剩余内存量 |
| `largest` | 最大连续内存块，过小则无法分配大块内存 |

## 5.3 top 命令输出

| 字段 | 含义 |
|------|------|
| PID | 任务 ID |
| COMMAND | 程序名 |
| CPU (ms) / CPU (%) | CPU 占用时间/利用率 |
| USED / STACK | 已用/分配的堆栈大小（字节） |
| PRIO (BASE) | 优先级 |
| STATE | 程序状态 |
| FD | 文件句柄数 |

## 5.4 重要目录（NSH 中）

| 目录 | 用途 |
|------|------|
| `/dev/` | 设备驱动节点（`tty*` = 串口） |
| `/etc/` | 配置文件 + NSH 脚本（`init.d/` 为启动脚本） |
| `/fs/microsd/` | SD 卡挂载点（默认工作目录），`dataman` 航点文件、`log/` 日志目录 |
| `/obj/` | uORB 底层实现相关 |
| `/proc/` | 运行时系统信息（数字子目录 = 运行中的任务，`meminfo` = 内存信息） |

---

# 六、仿真（SITL）

## 6.1 两种仿真模式

| 模式 | 说明 |
|------|------|
| **SITL**（软件在环） | 飞控程序在桌面计算机运行，通过网络连接飞行动力学模型 |
| **HITL**（硬件在环） | 飞控程序在真实飞控硬件运行，屏蔽传感器测量 |

## 6.2 仿真环境对比

| 仿真器 | 推荐程度 | 特点 |
|--------|----------|------|
| **Gazebo** | ⭐最推荐 | 3D 仿真，支持避障/视觉/多飞行器，常与 ROS 搭配 |
| **Gazebo Classic** | 非常推荐 | 功能强大，支持多类型载具 |
| **jMAVSim** | 快速四旋翼 | 简单易用，仅支持四旋翼，适合算法测试 |
| FlightGear | 特定场景 | 高真实感，可模拟天气（雷暴/雨雪等） |
| JSBSim | 特定场景 | 基于风洞数据的高级飞行动力学 |
| AirSim | 特定场景 | 基于虚幻引擎，物理+视觉逼真，资源要求高 |
| **SIH** | 特定场景 | HITL 替代方案，C++ 模块直接集成在固件中 |

## 6.3 启动仿真

```bash
# 设置起始位置（可写入 ~/.profile 避免重复设置）
export PX4_HOME_LAT=34.4384
export PX4_HOME_LON=108.7404

# jMAVSim（四旋翼）
make px4_sitl jmavsim

# Gazebo 无界面模式（轻量，适合虚拟机）
HEADLESS=1 make px4_sitl gazebo_<vehicle-model>

# vehicle-model: plane | standard_vtol | rover | (空=四旋翼)
```

## 6.4 启动脚本

| 目标 | 主启动脚本路径 |
|------|---------------|
| **仿真** | `./ROMFS/px4fmu_common/init.d-posix/rcS` |
| **硬件** | `./ROMFS/px4fmu_common/init.d/rcS` |

在启动脚本末尾添加 `test_app start` 可实现应用程序开机自启。

> SITL 仿真固件基于 POSIX 标准，不同于 NuttX 但使用相同 PX4 接口，源码兼容。

---

# 七、核心要点总结

1. **PX4 = RTOS（NuttX）+ 中间件 + 飞行控制栈**，采用模块化架构
2. **uORB 是系统核心**：所有模块通过消息总线去耦合通信
3. **开发流程**：源码 → CMakeLists.txt → Kconfig 注册 → 编译 → 上传 → NSH 运行
4. **应用入口**：`<name>_main(argc, argv)`，由操作系统调用，非传统 `main()`
5. **编译仿真目标前用 SITL**：快速调试，无需硬件
6. **GAzebo** 是最推荐的仿真器，虚拟机可用 `HEADLESS=1` 无界面模式
7. **启动脚本** 实现应用程序开机自动启动
