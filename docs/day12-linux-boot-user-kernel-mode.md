# Day 12：Linux 启动流程 + 用户态/内核态 + System Call

## 1. 学习目标

今天把前面零散的 Linux 知识放进真正的嵌入式 Linux 系统中，重点回答两个问题：

> **ARM SoC 从上电到应用程序运行经历什么？应用执行 `read()` 时为什么会从用户态进入内核态？**

先建立完整启动链：

```text
Power On
   ↓
BootROM
   ↓
SPL
   ↓
U-Boot
   ↓
Linux Kernel
   ↓
Device Tree
   ↓
RootFS
   ↓
init / systemd
   ↓
Application
```

本节重点：

- BootROM
- SPL
- U-Boot
- Kernel
- Device Tree
- RootFS
- User Space / Kernel Space
- System Call
- Bring-up 基本排障思维

---

## 2. BootROM：SoC 上电后的最早阶段

BootROM 是：

> **固化在 SoC 芯片内部的一小段启动代码。**

CPU Reset 后，根据芯片设计进入 BootROM。

BootROM 的典型任务之一：

```text
系统刚上电
   ↓
读取 Boot Strap / Fuse 等配置
   ↓
判断启动介质
   ↓
SD / eMMC / SPI Flash / USB ...
   ↓
加载下一阶段启动代码
```

因此 BootROM 并不是 Linux 文件系统里的某个文件，而是芯片内部早期启动逻辑的一部分。

---

## 3. 为什么需要 SPL？

完整 U-Boot 通常比较大，但系统刚上电时 DDR 可能尚未初始化。

此时 SoC 可使用的可能只是较小的片内 SRAM。

因此常见启动链：

```text
BootROM
   ↓
加载较小的 SPL
   ↓
SPL 初始化 DDR 等关键硬件
   ↓
DDR 可用
   ↓
加载完整 U-Boot
```

SPL：

> **Secondary Program Loader**

可以先理解为一个早期、精简的加载阶段，其中非常重要的任务之一是让后续完整 Bootloader 获得可用的 DDR 环境。

不同 SoC 启动链可能不同，但这一思路很典型。

---

## 4. U-Boot 在做什么？

U-Boot 运行后可以进行更多初始化，并为 Linux Kernel 建立启动环境。

常见职责：

```text
初始化必要硬件
   ↓
设置 bootargs / bootcmd
   ↓
找到 Kernel Image
   ↓
找到 Device Tree
   ↓
加载到 RAM
   ↓
向 Kernel 传递启动参数
   ↓
跳转到 Kernel
```

常见启动参数：

```text
console=...
root=...
```

例如：

```text
console=ttyS0,115200
root=/dev/mmcblk0p2
```

可以理解为告诉 Kernel：

- 控制台在哪里
- 根文件系统在哪里

---

## 5. Device Tree 的作用

ARM SoC 板上可能存在：

```text
UART
I2C
SPI
GPIO
Timer
Interrupt Controller
```

Linux Kernel 需要知道这块具体板子有什么硬件，以及资源在哪里。

Device Tree 用来描述：

> **硬件拓扑和资源信息。**

以后会看到类似：

```dts
uart0: serial@12340000 {
    compatible = "vendor,uart";
    reg = <0x12340000 0x1000>;
    interrupts = <...>;
};
```

今天不学 DTS 语法，只先理解：

```text
Device Tree
   ↓
告诉 Kernel
   ↓
设备是什么
寄存器范围在哪
中断号是什么
设备之间如何连接
```

后续 `platform_driver + probe()` 会与 Device Tree 真正连接起来。

---

## 6. Kernel 接管后发生什么？

U-Boot 跳转到 Kernel 后，Kernel 会逐步建立完整操作系统环境，例如：

```text
CPU 初始化
   ↓
内存管理
   ↓
中断系统
   ↓
Scheduler
   ↓
Driver / 各内核子系统
   ↓
挂载 RootFS
   ↓
启动第一个用户空间进程
```

前面学过的：

```text
Page Table
Scheduler
Virtual Memory
Driver
```

本来就是 Kernel 在运行期提供的核心机制。

---

## 7. RootFS 是什么？

RootFS：

> **Root File System，根文件系统。**

典型内容：

```text
/
├── bin
├── dev
├── etc
├── lib
├── proc
├── sys
├── usr
└── ...
```

Kernel 和 RootFS 不是一回事。

### Kernel 提供

- 进程与调度
- 虚拟内存
- 系统调用
- 文件系统机制
- 驱动
- 中断管理

### RootFS 提供

- `/bin/bash`
- 动态库
- 配置文件
- 系统工具
- 用户程序

Kernel 挂载 RootFS 后，才能继续进入完整用户空间环境。

---

## 8. init / systemd

Kernel 完成基础初始化并挂载 RootFS 后，需要启动第一个用户空间进程。

现代 Linux 中通常是：

```text
systemd
```

然后：

```text
systemd
   ↓
系统 Service
   ↓
network / ssh / daemon ...
   ↓
Application
```

到这里 Linux 系统才进入日常运行状态。

---

## 9. 用户态和内核态

普通程序中的计算：

```c
int a = 1 + 2;
```

通常直接在 User Mode 执行，不需要每条指令进入 Kernel。

但是：

```c
read(fd, buf, 100);
```

需要 Kernel 提供文件、设备等受保护资源的服务。

链路：

```text
Application
   ↓ read()
System Call
   ↓
进入 Kernel Mode
   ↓
Linux Kernel
   ↓
VFS / Driver
   ↓
File / Hardware
   ↓
返回 User Mode
```

因此可以把 System Call 理解为：

> **用户程序受控地请求 Kernel 服务的接口。**

---

## 10. 为什么需要 User / Kernel 权限隔离？

如果普通应用可以随意：

```text
修改页表
关闭中断
访问任意硬件
修改其它进程内存
```

一个普通 Bug 就可能直接破坏整个系统。

因此现代 CPU 和 Linux 使用不同权限级别与虚拟内存保护：

```text
User Space
低权限
Application

──────── 权限边界 ────────

Kernel Space
高权限
Kernel / Driver
```

用户程序需要通过 System Call 进入受控的内核服务路径。

---

## 11. 实验一：重新理解 strace

执行：

```bash
strace cat /etc/hostname
```

观察：

```text
openat(...)
read(...)
write(...)
close(...)
```

Day 02 中只是把 `strace` 当成阻塞定位工具；今天进一步理解：

> **strace 观察的正是 User Space 与 Kernel Space 之间的 System Call 交互。**

---

## 12. 实验二：查看 Kernel 启动参数

执行：

```bash
cat /proc/cmdline
```

可能看到：

```text
BOOT_IMAGE=...
root=...
console=...
quiet
```

嵌入式板上常见：

```text
console=ttyS0,115200
root=/dev/mmcblk0p2
```

这些参数通常由 Bootloader 为 Kernel 准备或传递。

---

## 13. 实验三：查看 Kernel 启动日志

执行：

```bash
dmesg | head -80
```

如果权限受限：

```bash
sudo dmesg | head -80
```

重点寻找：

```text
Linux version
CPU
Memory
Kernel command line
filesystem
driver
```

目标：把 `dmesg` 看成 Kernel 启动和设备初始化过程留下的运行记录。

---

## 14. Bring-up 基本排障思路

面试场景：

> **一块新 ARM SoC 板第一次上电，串口完全没有输出，怎么排查？**

不要一上来就检查 Linux Application，因为系统可能根本没有运行到 Kernel。

应该按启动链逐级确认：

```text
Power
  ↓
Clock
  ↓
Reset
  ↓
Boot Strap
  ↓
BootROM
  ↓
SPL
  ↓
DDR
  ↓
U-Boot
  ↓
Kernel
```

如果连 U-Boot 日志都没有，可能考虑：

- 供电是否正常
- Clock 是否工作
- Reset 是否释放
- Boot Mode 是否正确
- 启动介质是否可访问
- SPL 是否成功执行
- UART Pinmux / Clock / 波特率是否正确
- DDR 初始化是否卡死

这时示波器、逻辑分析仪、JTAG、串口工具都会真正进入 bring-up 流程。

---

## 15. 面试题

### Q1：ARM Linux 从上电到 Application 运行大致经历什么？

可以回答：

> SoC 上电复位后先执行 BootROM，根据启动配置加载下一阶段代码；典型平台会进入 SPL 完成 DDR 等关键硬件初始化，再加载完整 U-Boot。U-Boot 准备 Kernel、Device Tree 和启动参数后跳转 Kernel。Kernel 初始化 CPU、内存、中断、调度器和驱动等子系统，挂载 RootFS，随后启动 init/systemd，最终运行用户空间 Service 和 Application。

### Q2：为什么用户态不能直接访问硬件？

> Linux 利用 CPU 权限级别和虚拟内存隔离用户空间与内核空间。普通程序权限较低，不能任意执行特权操作或访问硬件，需要通过 System Call 请求 Kernel，再由内核子系统或 Driver 完成实际操作，从而保证系统安全性和稳定性。

### Q3：Bootloader 和 Kernel 有什么区别？

> Bootloader 主要负责建立 Kernel 运行前的基本环境、必要硬件初始化、加载 Kernel/Device Tree、准备启动参数并交出控制权；Kernel 接管后负责进程调度、内存管理、中断、文件系统、设备驱动和 System Call 等完整操作系统功能。

---

## 16. 今日总结

完整启动链：

```text
Power On
  ↓
BootROM
  ↓
SPL / DDR Init
  ↓
U-Boot
  ↓
Kernel + Device Tree
  ↓
RootFS
  ↓
init / systemd
  ↓
Application
  ↓
System Call
  ↓
Kernel / Driver
  ↓
Hardware
```

核心记忆：

1. BootROM 负责最早期启动并加载下一阶段。
2. SPL 常承担 DDR 等关键早期初始化。
3. U-Boot 为 Kernel 准备镜像、Device Tree 和启动参数。
4. Kernel 挂载 RootFS 后进入用户空间。
5. Application 通过 System Call 受控请求 Kernel 服务。

## 下一天

**Day 13：Linux Kernel Module——.ko、insmod/rmmod、printk/dmesg**

目标：第一次让自己的 C 代码真正进入 Kernel Space 执行。