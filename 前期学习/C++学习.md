---
tags:
  - C++
  - 编程基础
  - PX4开发
date: 2026-08-11
---

# C++ 快速入门（面向 PX4 开发）

> 零基础友好 | 所有知识点都有 PX4 源码中的实际用法对应

---

## 一、从 C 到 C++ 的关键变化

你之前学的 PX4 代码已经是 C++ 了。C++ 在 C 的基础上主要增加了**类（class）**，把"数据"和"操作数据的函数"打包在一起。

```
C 语言：                        C++：
                                
数据是数据                       class 传感器 {
函数是函数                          float 加速度[3];       ← 数据（成员变量）
二者分离                            void 读取();           ← 函数（成员方法）
                              };  二者打包在一起
```

---

## 二、class（类）—— C++ 最核心的概念

### 2.1 基本语法

```cpp
class TestApp {                    // class 关键字 + 类名
public:                            // 访问控制：public = 外部可访问
    TestApp();                     // 构造函数：创建对象时自动调用
    ~TestApp();                    // 析构函数：销毁对象时自动调用
    void run();                    // 成员函数（方法）

private:                           // private = 只有类内部能访问
    int _task_id;                  // 成员变量（习惯以 _ 开头）
    bool _is_running;
};

// 构造函数实现（类外定义用 :: 指定所属类）
TestApp::TestApp() {
    _task_id = -1;
    _is_running = false;
}

// 使用类
TestApp app;        // 创建对象（栈上）
app.run();          // 调用方法

TestApp *p = new TestApp();  // 创建对象（堆上，用指针）
p->run();                    // 指针调用用 ->
delete p;                    // 释放堆内存
```

### 2.2 你见过哪些类

```cpp
// PX4 中几乎一切都是类：
class Commander { ... };           // Commander 模块
class TestApp { ... };             // 你写的测试程序
class ArmStateMachine { ... };    // 解锁状态机
```

### 2.3 构造函数（Constructor）

构造函数在对象创建时自动执行，用于初始化。

```cpp
// 你见过的：
TestApp::TestApp()
    : px4::WorkItem(MODULE_NAME, px4::wq_configurations::test)
    //  ↑ 初始化列表：初始化父类（见后文继承）
{}

// 等价于：
TestApp::TestApp() {
    // 先自动调用 WorkItem 的构造函数（初始化列表做的）
    // 然后执行本构造函数体
}
```

| 概念 | 说明 |
|------|------|
| `TestApp()` | 构造函数，类名作为函数名，无返回值 |
| `: 成员(值), ...` | 初始化列表，在函数体之前执行 |
| `~TestApp()` | 析构函数，对象销毁时自动调用 |

---

## 三、public / private / protected —— 访问控制

```cpp
class Example {
public:           // 任何人都能访问
    void start();
    int status;

protected:        // 只有本类和子类能访问
    void _internal_check();

private:          // 只有本类能访问
    int _secret_data;
};
```

PX4 典型模式：成员变量几乎全是 `private`，通过 `public` 方法访问——这叫**封装**。

---

## 四、继承（Inheritance）—— 你一直在用

### 4.1 基本概念

继承 = 子类获得父类的所有功能，可以在此基础上扩展。

```
    ┌──────────┐
    │  父类    │    ← 通用功能
    │ (Base)   │
    └────┬─────┘
         │ 继承
    ┌────▼─────┐
    │  子类     │    ← 通用功能 + 特有功能
    │ (Derived)│
    └──────────┘
```

### 4.2 PX4 中的实际例子

```cpp
// 你见过的第一个例子（单继承）：
class TestApp : public ModuleBase<TestApp>   // TestApp 继承 ModuleBase
{
    // TestApp 自动获得 main() / start_command_base / status_command 等方法
    // 你只需要写 run() 即可
};

// 你见过的第二个例子（多继承）：
class TestApp : public ModuleBase<TestApp>, public px4::WorkItem
//                 ↑ 父类1：提供CLI命令管理   ↑ 父类2：提供工作队列调度
{
    void Run() override;  // 重写 WorkItem 的 Run()
};
```

### 4.3 继承的精髓

```cpp
// 父类定义框架，子类填充细节
class ModuleBase {
    // 框架已经写好了 start/stop/status 的逻辑
    // 你只需要重写 Run() 实现业务逻辑
    virtual void Run() = 0;  // 纯虚函数 = 强制子类必须实现
};
```

---

## 五、virtual 和 override —— 多态

### 5.1 含义

| 关键字 | 含义 |
|--------|------|
| `virtual` | 声明此函数可被子类**重写**（覆盖） |
| `override` | 显式标记"我正在重写父类的虚函数"，编译器会帮你检查是否正确 |

### 5.2 PX4 中的例子

```cpp
// 父类 WorkItem 中定义：
class WorkItem {
    virtual void Run() = 0;    // = 0 表示纯虚函数，子类必须实现
};

// 你的子类中：
class TestApp : public WorkItem {
    void Run() override {      // override = "我在重写父类的Run"
        // 你的业务逻辑
        if (vehicle_rates_sub.update(&angular_velocity)) {
            dbg_pub.publish(dbg);
        }
    }
};
```

> 💡 如果不写 `override`，不小心写错函数签名（比如写成 `void run()`），编译器不会报错，但你的函数永远不会被调用。加上 `override` 编译器就会帮你检查。

---

## 六、static —— 静态成员

### 6.1 含义

`static` 成员**属于类本身**，而不是属于某个具体的对象。

```cpp
class TestApp {
    static int _task_id;       // 所有 TestApp 对象共享这一个变量
    static int task_spawn();   // 可以直接 TestApp::task_spawn() 调用
};

// 使用方式：类名::静态方法()
TestApp::task_spawn(argc, argv);   // 不需要对象就能调用
int id = TestApp::_task_id;        // 不需要对象就能访问
```

### 6.2 PX4 中的实际用法

```cpp
// ModuleBase 要求每个模块实现这些静态方法：
static int task_spawn(int argc, char *argv[]);     // 启动任务
static TestApp *instantiate(int argc, char *argv[]); // 创建实例
static int custom_command(int argc, char *argv[]);  // 自定义命令
static int print_usage(const char *reason = nullptr); // 帮助信息
```

> 这些函数被 PX4 框架调用时，还没有 TestApp 对象——所以必须是 `static`。

---

## 七、template（模板）—— 写一次，通用所有类型

### 7.1 含义

模板允许你用**类型参数**写代码，编译时自动生成具体版本。

### 7.2 PX4 中的例子

```cpp
// ① 类模板：ModuleBase<T>
template<class T>                    // T 是占位符，代表"某种类型"
class ModuleBase {
    static int main(int argc, char *argv[]) {
        // 内部用 T::task_spawn()、T::instantiate() 等
        // 当 T = TestApp 时，调用的就是 TestApp::task_spawn()
    }
};

// 使用时：
class TestApp : public ModuleBase<TestApp> {   // T = TestApp
    // ModuleBase 中的所有 T 都被替换为 TestApp
};

// ② 类模板：Publication<T>
uORB::Publication<debug_key_value_s> dbg_pub{ORB_ID(debug_key_value)};
//                ↑ T = debug_key_value_s

// ③ 数组模板
std::vector<std::array<double, 3>> waypoints_;
//           ↑ 3个double的数组  ↑ 动态数组
```

---

## 八、struct vs class

```cpp
// struct：成员默认 public（C 兼容）
struct mission_s {
    uint64_t timestamp;    // 直接访问
    uint16_t count;
};

// class：成员默认 private（C++ 推荐）
class TestApp {
    int _task_id;          // 外部不能直接访问
public:
    void run();            // 通过 public 方法访问
};
```

PX4 中规则：
- **消息/数据结构**用 `struct`（如 `vehicle_status_s`、`mission_item_s`）
- **模块/功能类**用 `class`（如 `Commander`、`TestApp`）

---

## 九、namespace（命名空间）—— 防止名字冲突

```cpp
// PX4 中常见：
uORB::Publication<...>      // Publication 在 uORB 命名空间里
mode_util::getModeRequirements(...)  // 在 mode_util 命名空间里
px4::WorkItem               // WorkItem 在 px4 命名空间里
std::vector                 // vector 在 std 命名空间里

// 相当于给函数/类加了个"姓氏"，避免同名冲突
// 就像：张三::说话()  和  李四::说话()  是两个不同的函数
```

---

## 十、指针和引用（&）

### 10.1 指针（你已熟悉）

```cpp
int x = 10;
int *p = &x;     // p 指向 x
*p = 20;         // 通过指针修改 x
p->something;    // 指针访问成员用 ->
```

### 10.2 引用（C++ 新特性）

```cpp
int x = 10;
int &ref = x;    // ref 是 x 的别名
ref = 20;        // 等价于 x = 20，不需要解引用
```

### 10.3 PX4 中的典型用法

```cpp
// 函数参数用引用避免拷贝大对象
void getModeRequirements(uint8_t vehicle_type, failsafe_flags_s &flags)
//                                               ↑ 引用传递，直接修改外部变量

// const 引用：只读不修改
void checkAndReport(const Context &context, Report &reporter);
//                   ↑ 只读               ↑ 可修改

// 指针：用于可选参数
static int print_usage(const char *reason = nullptr);
//                                      ↑ nullptr = 空指针，"没有值"
```

---

## 十一、union（联合体）—— 同一块内存，多种解读

```cpp
// PX4 中你见过的：
struct mission_item_s {
    union {                         // 同一块内存
        struct {
            float time_inside;      // 当它是"停留时间"
            float circle_radius;    // 当它是"盘旋半径"
        };
        float params[7];            // 当它是"参数数组"
    };
};
```

> 节省内存：time_inside 和 params[0] 占据**同一块内存地址**。

---

## 十二、位域（Bit Field）—— 省内存的结构体

```cpp
// PX4 中你见过的：
struct {                          // 总共只占 16 位 = 2 字节
    uint16_t frame : 4,           // 占 4 位（0-15）
             origin : 3,          // 占 3 位（0-7）
             loiter_exit_xtrack : 1,  // 占 1 位（0/1）
             force_heading : 1,       // 占 1 位
             altitude_is_relative : 1,// 占 1 位
             autocontinue : 1,        // 占 1 位
             vtol_back_transition : 1,// 占 1 位
             _padding0 : 4;           // 占 4 位（填充）
};
```

---

## 十三、const / constexpr —— 常量与编译期常量

| 关键字 | 含义 | 示例 |
|--------|------|------|
| `const` | 运行时常量，赋值后不可修改 | `const int x = 5;` |
| `constexpr` | 编译时常量，编译时就确定值 | `static constexpr int task_id_is_work_queue = -2;` |

```cpp
// PX4 中的实际用法：
static constexpr bool arming_transitions[6][6] = { ... };
// 这个数组在编译时就已经算好，运行时直接读取，零开销
```

---

## 十四、enum（枚举）—— 给整数起名字

```cpp
// C 风格枚举（PX4 中最常见）：
enum PX4_CUSTOM_MAIN_MODE {
    PX4_CUSTOM_MAIN_MODE_MANUAL = 1,   // 对应整数 1
    PX4_CUSTOM_MAIN_MODE_ALTCTL,       // 自动 = 2
    PX4_CUSTOM_MAIN_MODE_POSCTL,       // 自动 = 3
    PX4_CUSTOM_MAIN_MODE_AUTO,         // = 4
};

// 使用：
if (custom_main_mode == PX4_CUSTOM_MAIN_MODE_AUTO) { ... }
```

---

## 十五、auto —— 自动类型推导

```cpp
// 不用写又长又臭的类型名
auto &last_waypoint = waypoints_.back();  // 编译器自动推导类型
// 等价于：
// std::array<double, 3> &last_waypoint = waypoints_.back();

// PX4 中常见：
for (auto item : _callbacks) {   // 自动推导 item 的类型
    item->call();
}
```

---

## 十六、nullptr —— C++ 的空指针

```cpp
// C 语言用 NULL（本质是 0）
int *p = NULL;     // C 风格

// C++ 用 nullptr（有类型安全）
int *p = nullptr;  // C++ 风格
```

---

## 十七、extern "C" —— 兼容 C 语言

```cpp
// 你见过的每一行：
extern "C" __EXPORT int test_app_main(int argc, char *argv[]);

// 含义：这个函数用 C 的命名规则编译（C 和 C++ 的函数命名规则不同）
// 不加这个的话，C 代码调用不到这个函数
```

---

## 十八、std:: 标准库速查

PX4 中常用的 `std::` 容器：

| 容器 | 用途 | PX4 中的例子 |
|------|------|-------------|
| `std::vector<T>` | 动态数组 | `std::vector<std::array<double,3>> waypoints_` |
| `std::array<T, N>` | 固定大小数组 | `std::array<double,3>` = 3 个 double |
| `std::sqrt()` | 平方根 | `std::sqrt(dx*dx + dy*dy + dz*dz)` |
| `std::strcmp()` | 字符串比较 | `strcmp(argv[1], "start")` |

---

## 十九、PX4 源码中的 C++ 模式速查

### 19.1 ModuleBase 的标准模板

```cpp
// .hpp 文件
#pragma once                          // 防止重复包含
#include <px4_platform_common/module.h>

class 我的模块 : public ModuleBase<我的模块> {
public:
    我的模块();
    ~我的模块() = default;             // 默认析构函数

    static int task_spawn(int argc, char *argv[]);
    static 我的模块 *instantiate(int argc, char *argv[]);
    static int custom_command(int argc, char *argv[]);
    static int print_usage(const char *reason = nullptr);

    void run() override;              // override = 重写父类

private:
    int _内部变量 = 0;
};
```

### 19.2 ModuleBase + WorkItem 模板

```cpp
class 我的模块 : public ModuleBase<我的模块>, public px4::WorkItem {
public:
    我的模块() : px4::WorkItem(MODULE_NAME, px4::wq_configurations::test) {}
    void Run() override;     // 注意大写 R，这是 WorkItem 的要求
    void init();

private:
    uORB::SubscriptionCallbackWorkItem _订阅者{this, ORB_ID(主题名)};
};
```

### 19.3 PX4 中的命名惯例

| 惯例 | 示例 | 说明 |
|------|------|------|
| 类名大写开头 | `Commander`, `TestApp` | PascalCase |
| 成员变量 `_` 前缀 | `_task_id`, `_is_running` | 一眼看出是成员变量 |
| 结构体 `_s` 后缀 | `vehicle_status_s`, `mission_s` | s = struct |
| 枚举大写 + 下划线 | `ARMING_STATE_ARMED` | 常量 |
| 函数名小写 + 下划线 | `task_spawn()`, `orb_publish()` | snake_case |

---

## 二十、从你的学习路径回顾 C++

| 你见过的 PX4 代码 | 涉及的 C++ 知识点 |
|-------------------|-------------------|
| `class TestApp : public ModuleBase<TestApp>` | class、public、继承、template |
| `void Run() override` | 虚函数、override |
| `static int task_spawn(...)` | static 成员 |
| `private: int _task_id;` | 访问控制、命名惯例 |
| `px4::WorkItem(...)` | namespace、构造函数、初始化列表 |
| `uORB::Publication<debug_vect_s>` | namespace、template |
| `strncpy(dbg.key, "test", sizeof(...))` | C 标准库函数 |
| `extern "C" __EXPORT int test_app_main(...)` | extern "C"、链接属性 |
| `static constexpr bool arming_transitions[][]` | constexpr、编译期常量 |
| `union { struct { ... }; float params[7]; }` | union、匿名嵌套 |
| `uint16_t frame : 4, origin : 3;` | 位域 |
| `for (auto item : _callbacks)` | auto、范围 for |
| `instance->init()` | 指针、箭头运算符 |
| `std::vector<std::array<double,3>>` | STL 容器 |

---

## 二十一、核心要点

| # | 要点 |
|---|------|
| 1 | `class` = 数据 + 方法的打包体，C++ 相比于 C 最大的变化 |
| 2 | `public` 继承 = "是一个"，TestApp **是一个** ModuleBase |
| 3 | `virtual` + `override` = 父类定义框架，子类填充细节（多态） |
| 4 | `static` = 属于类本身，`TestApp::task_spawn()` 无需对象就能调用 |
| 5 | `template<T>` = 编译时自动生成代码，`ModuleBase<TestApp>` 就是你的专属版本 |
| 6 | PX4 模块 = `ModuleBase` 提供 CLI + `WorkItem` 提供调度 + 你的 `Run()` |
| 7 | 成员变量加 `_` 前缀、结构体加 `_s` 后缀，是 PX4 的命名惯例 |
