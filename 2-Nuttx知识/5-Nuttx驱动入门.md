---
tags:
  - Nuttx
  - PX4
  - 驱动
  - 字符设备
  - CDev
date: 2026-08-06
---

# Nuttx 驱动入门：字符设备（Character Device）

## 概述

在 Nuttx/PX4 中，**字符设备（Character Device, cdev）** 是最常见的驱动形式。驱动注册后会在 `/dev/` 下创建一个设备文件，应用程序通过标准的文件操作（`open`/`read`/`write`/`close`）来访问硬件，就像读写普通文件一样。

## 核心思想：一切皆文件

```
应用程序                   内核/驱动
─────────                ─────────────
px4_open("/dev/xxx")  →  CDev 基类 → TestCdev（你的驱动）
px4_read(fd, buf, n)  →  重写的 read()
px4_write(fd, buf, n) →  重写的 write()
px4_close(fd)         →  重写的 close()
```

## 头文件

```cpp
#include <lib/cdev/CDev.hpp>

class TestCdev : public cdev::CDev
{
public:
    TestCdev(const char *devname);
    virtual ~TestCdev();

    int open(cdev::file_t *filp) override;
    int close(cdev::file_t *filp) override;
    ssize_t read(cdev::file_t *filp, char *buffer, size_t buflen) override;
    ssize_t write(cdev::file_t *filp, const char *buffer, size_t buflen) override;
};
```

## 四个虚函数

| 虚函数 | 触发时机 | 返回值 |
|--------|---------|--------|
| `open(filp)` | 应用调用 `px4_open("/dev/xxx")` | 0=成功, 负值=错误 |
| `close(filp)` | 应用调用 `px4_close(fd)` | 0=成功, 负值=错误 |
| `read(filp, buffer, buflen)` | 应用调用 `px4_read(fd, ...)` | 实际读取字节数 |
| `write(filp, buffer, buflen)` | 应用调用 `px4_write(fd, ...)` | 实际写入字节数 |

> `buffer` 永远属于应用层：`read` 时你往 buffer 写数据，`write` 时你从 buffer 读数据。

## 构造函数：注册设备路径

```cpp
TestCdev::TestCdev(const char *path) : CDev(path) {}
//                                       ↑
//                          基类 CDev 根据路径在 /dev/ 下创建设备节点
```

## read() 实现

```cpp
ssize_t TestCdev::read(cdev::file_t *filp, char *buffer, size_t buflen)
{
    char rbuf[] = "This is a write test!";
    memcpy(buffer, rbuf, strlen(rbuf));    // 数据从驱动 → 应用 buffer
    return strlen(rbuf);
}
```

## write() 实现

```cpp
ssize_t TestCdev::write(cdev::file_t *filp, const char *buffer, size_t buflen)
{
    char wbuf[50];
    memcpy(wbuf, buffer, buflen);   // 数据从应用 buffer → 驱动
    return buflen;
}
```

> ⚠️ 注意不要 `buflen + 1`，会导致缓冲区溢出。

## 驱动注册：两步完成

```cpp
int test_cdev_main(int argc, char *argv[])
{
    TestCdev *cdev = new TestCdev("/dev/testCDev");  // ① 创建对象
    cdev->init();                                     // ② 注册设备节点
    return PX4_OK;
}
```

## 完整调用链路

```
[启动] test_cdev → new + init → /dev/testCDev 就绪

[运行] test_app start:
    px4_open("/dev/testCDev")  →  TestCdev::open()
    px4_read(fd, buf, 50)      →  TestCdev::read()
    px4_write(fd, buf, 50)     →  TestCdev::write()
    px4_close(fd)              →  TestCdev::close()
```

## 关键知识点

| 概念 | 说明 |
|------|------|
| **字符设备（cdev）** | 以字节流方式访问的设备，通过 `/dev/xxx` 路径访问 |
| **CDev 基类** | PX4 提供的字符设备框架 |
| **`cdev::file_t`** | 文件句柄结构体，每次 `open` 创建一个实例 |
| **设备注册** | `new` + `init()` 两步完成 |
| **`buflen`** | 应用层缓冲区的最大可用字节数，驱动不能超出 |

## 相关笔记

- [[4-队列]] — PX4 工作队列
- [[3-线程]] — pthread 线程与条件变量
- [[2-任务的同步和互斥]] — 信号量实现互斥
