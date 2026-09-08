# Day 13：Linux Kernel Module——.ko、insmod/rmmod、printk/dmesg

## 1. 学习目标

今天正式跨入 Linux Kernel Driver 的入口。

前面一直在 User Space 使用 Linux；今天第一次回答：

> **Linux Driver 代码怎样进入 Kernel Space 并运行？**

建立这一条主线：

```text
hello.c
   ↓
Kernel Build System
   ↓
hello.ko
   ↓
insmod
   ↓
Linux Kernel
   ↓
module_init()
   ↓
printk()
   ↓
dmesg

卸载：
rmmod
   ↓
module_exit()
```

本节重点：

- Kernel Module
- `.ko`
- `insmod / rmmod / lsmod / modinfo`
- `module_init / module_exit`
- `printk / dmesg`
- Kernel Module 与普通 Application 的区别

---

## 2. 什么是 Kernel Module？

Linux Kernel 本身包含大量子系统和 Driver。

如果所有 Driver 都必须永久编译进 Kernel，那么修改一个 Driver 往往意味着重新编译并重新启动整个 Kernel。

Linux 因此支持：

> **Loadable Kernel Module，可动态加载内核模块。**

模块文件常见后缀：

```text
xxx.ko
```

例如：

```bash
sudo insmod hello.ko
```

可以把模块加载进正在运行的 Kernel。

卸载：

```bash
sudo rmmod hello
```

因此先建立一个最基础区别：

```text
普通程序
app
 ↓
User Space

Kernel Module
driver.ko
 ↓
Kernel Space
```

`.ko` 可以先理解为 Kernel Object。

---

## 3. Kernel Module 为什么没有 main()？

普通程序：

```c
int main(void)
{
    return 0;
}
```

运行过程：

```text
Shell
 ↓
./app
 ↓
main()
 ↓
User Space
```

Kernel Module 不是普通用户进程，它由 Kernel 加载，因此没有普通意义上的 `main()`。

模块通过：

```c
module_init(...)
module_exit(...)
```

指定加载和卸载入口。

例如：

```text
insmod hello.ko
        ↓
hello_init()

rmmod hello
        ↓
hello_exit()
```

---

## 4. 第一个 Kernel Module

示例：

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "hello: module loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("yyyflag");
MODULE_DESCRIPTION("My first Linux kernel module");
```

今天先不展开这些宏的内部实现，只理解模块生命周期。

---

## 5. 为什么 Kernel 里不用 printf()？

`printf()` 属于用户空间 libc 接口体系。

Kernel 并不是普通用户程序，因此不能按照用户态程序方式直接使用 libc。

Kernel 中常使用：

```c
printk()
```

或更常见的日志宏：

```c
pr_info()
pr_err()
pr_warn()
```

日志进入 Kernel Log Buffer，可以通过：

```bash
dmesg
```

查看。

对比：

```text
User Space
printf()
   ↓
stdout / terminal

Kernel Space
printk() / pr_info()
   ↓
Kernel Log
   ↓
dmesg
```

---

## 6. 实验一：查看当前已加载模块

执行：

```bash
lsmod | head -20
```

再看：

```bash
cat /proc/modules | head
```

两者信息存在明显对应关系。

这也把 Day 01 学过的 `/proc` 再次连接回来。

`lsmod` 可以理解为：

> 查看当前 Kernel 已加载模块的友好工具。

---

## 7. 实验二：查看真实模块信息

先用：

```bash
lsmod | head
```

选择一个实际存在的模块，例如系统中如果有：

```text
i2c_dev
```

执行：

```bash
modinfo i2c_dev
```

常见字段：

```text
filename
license
description
author
depends
name
vermagic
```

重点认识：

```text
depends
vermagic
```

`vermagic` 与 Kernel 版本和构建环境兼容性有关。

因此：

> **一个 `.ko` 编译成功，不代表一定能加载到任意 Linux Kernel。**

模块必须针对兼容的 Kernel Build 环境构建。

---

## 8. 实验三：真正编译 hello.ko

先查看当前 Kernel：

```bash
uname -r
```

检查 Kernel Build 目录：

```bash
ls /lib/modules/$(uname -r)/build
```

创建 `hello.c` 后，再创建 Makefile：

```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

执行：

```bash
make
```

可能生成：

```text
hello.o
hello.mod.c
hello.mod.o
Module.symvers
modules.order
hello.ko
```

最关键目标：

```text
hello.c
  ↓
Kernel Build System
  ↓
hello.ko
```

---

## 9. 加载和卸载模块

如果系统允许加载自定义模块：

```bash
sudo insmod hello.ko
```

查看日志：

```bash
dmesg | tail
```

应该能看到类似：

```text
hello: module loaded
```

查看模块：

```bash
lsmod | grep hello
```

卸载：

```bash
sudo rmmod hello
```

再次：

```bash
dmesg | tail
```

观察：

```text
hello: module unloaded
```

如果在 WSL、容器或公司受限环境，加载自定义 Kernel Module 可能被禁止。此时完成编译和观察已有模块即可，不必为了实验修改安全策略。

---

## 10. insmod 和 modprobe

### insmod

```bash
insmod xxx.ko
```

直接加载指定模块文件。

### modprobe

```bash
modprobe xxx
```

属于更高层的模块管理工具，可以结合系统模块数据库和依赖关系完成加载。

例如 A 依赖 B：

```text
A.ko
 ↓
B.ko
```

`modprobe` 更适合处理日常系统模块管理和依赖关系。

---

## 11. module_init() 和 probe() 不是一回事

这是驱动学习中很容易混淆的一点。

今天的：

```c
module_init(hello_init);
```

表示：

> 模块被加载时执行的模块级初始化入口。

以后学习 `platform_driver` 时还会遇到：

```c
probe()
```

典型流程将是：

```text
insmod xxx.ko
      ↓
module_init()
      ↓
注册 platform_driver
      ↓
Driver Core 匹配 Device
      ↓
probe()
      ↓
初始化具体硬件
```

所以：

```text
module_init
→ 模块级初始化

probe
→ 设备匹配成功后的设备初始化
```

---

## 12. 为什么 Kernel Driver 的 Bug 更危险？

普通 User Space 程序非法访问：

```c
int *p = NULL;
*p = 10;
```

通常导致：

```text
SIGSEGV
↓
该进程退出
```

但 Kernel Module 运行在 Kernel Space。

如果发生严重非法访问，可能造成：

```text
Kernel Oops
Kernel Panic
整个系统异常
```

因为 Driver 与 Kernel 处于高权限内核环境。

因此 Driver 开发特别强调：

- 边界检查
- 资源管理
- 错误路径
- 并发安全
- 生命周期管理

---

## 13. Kernel Module 的第一层调试工具

现阶段可以使用：

```text
printk / pr_info / pr_err
      ↓
dmesg

lsmod
→ 模块是否加载

modinfo
→ 模块信息

/proc /sys
→ 查看运行状态
```

后续还会继续补：

```text
dynamic_debug
ftrace
tracepoints
perf
kgdb
```

所以前面学过的 `perf`、`/proc`、`dmesg` 后面都会重新出现。

---

## 14. 面试题

### Q1：Kernel Module 和普通 Application 有什么区别？

可以回答：

> Application 运行在 User Space，通过 System Call 请求 Kernel 服务；Kernel Module 被动态加载到 Kernel Space，可直接使用内核接口并实现驱动等功能。普通应用从 `main()` 开始，而 Kernel Module 通过 `module_init()` 和 `module_exit()` 指定加载与卸载入口。由于模块运行在 Kernel Space，严重错误可能影响整个系统。

### Q2：insmod 和 modprobe 有什么区别？

```text
insmod
→ 直接加载指定 .ko

modprobe
→ 更高层模块管理工具
→ 可结合模块依赖等信息加载
```

### Q3：Kernel Driver 怎么调试？

当前阶段可以回答：

```text
printk / pr_* + dmesg
lsmod
modinfo
/proc
/sys
```

进一步还能使用 dynamic debug、ftrace、tracepoint、perf、kgdb 等工具。

---

## 15. 今日总结

现在 Linux 软件层级已经可以画成：

```text
Application
    ↓
User Space
    ↓
open / read / write / ioctl
    ↓
System Call
══════════════════════════
Kernel Space
    ↓
Linux Kernel
    ↓
Driver Framework
    ↓
Kernel Module (.ko)
    ↓
Driver
    ↓
Hardware
```

模块生命周期：

```text
make
 ↓
xxx.ko
 ↓
insmod
 ↓
module_init()
 ↓
运行
 ↓
rmmod
 ↓
module_exit()
```

核心记忆：

1. `.ko` 是 Linux 可加载 Kernel Module。
2. Kernel Module 没有普通程序的 `main()`。
3. `module_init()` / `module_exit()` 管理模块加载卸载入口。
4. Kernel 中使用 `printk/pr_*` 输出日志，并通过 `dmesg` 查看。
5. `module_init()` 与后续设备驱动中的 `probe()` 不是一个概念。

## 下一天

**Day 14：Character Device Driver + file_operations + open/read/write/ioctl**

目标：回答一个真正的 Linux Driver 核心问题：

```text
Application
fd = open("/dev/my_device")
read(fd, buf, 100)

        ↓

Linux 怎么知道这个 read()
应该调用我的 Driver 中哪个函数？
```

下一课会把：

```text
fd
System Call
VFS
/dev
Kernel Module
file_operations
```

第一次完整连接起来。