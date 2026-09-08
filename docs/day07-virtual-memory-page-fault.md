# Day 07：Linux 虚拟内存 + 页表 + Page Fault

## 1. 学习目标

今天从 CPU 调度切换到 Linux 内存体系，解决三个核心问题：

> **程序中的地址到底是什么？为什么 VIRT 很大却不等于占用大量 RAM？Linux 如何把虚拟地址转换为真实物理内存？**

建立这一条主线：

```text
Application
   ↓
Virtual Address
   ↓
MMU
   ↓
Page Table
   ↓
Physical Page
   ↓
RAM
```

本节重点：

- Virtual Address / Physical Address
- MMU
- Page / Page Table
- VIRT / RSS
- Page Fault
- Minor / Major Page Fault

---

## 2. 程序看到的地址通常不是物理地址

例如：

```c
int a = 10;
printf("%p\n", &a);
```

打印出的 `0x7f...` 一类地址通常是当前进程的：

> **Virtual Address（虚拟地址）**

CPU 真正访问 RAM 时，会经历：

```text
CPU
 │ Virtual Address
 ↓
MMU
 │ 查 Page Table
 ↓
Physical Address
 ↓
RAM
```

MMU（Memory Management Unit）负责地址翻译，Page Table 记录虚拟页到物理页之间的映射关系。

---

## 3. 为什么需要虚拟内存？

没有虚拟内存时，不同程序如果直接操作物理地址，很容易互相破坏数据。

有了虚拟内存：

```text
Process A
Virtual 0x1000
      ↓
Physical 0xA000

Process B
Virtual 0x1000
      ↓
Physical 0xF000
```

两个进程可以看到相同的虚拟地址，但通过各自页表映射到不同的物理内存。

主要价值之一：

> **进程地址空间隔离。**

这也是为什么普通用户程序不能随意把某个 SoC 物理寄存器地址当普通指针直接访问。

---

## 4. Page 是什么？

Linux 不会按单个字节维护虚拟地址映射，而是按固定大小的 Page（页）管理。

很多 ARM/Linux 系统常见页大小：

```text
4 KB
```

概念上：

```text
Virtual Address
┌──────────────┬───────────┐
│ Virtual Page │  Offset   │
└──────────────┴───────────┘
        ↓
     Page Table
        ↓
┌──────────────┬───────────┐
│Physical Page │  Offset   │
└──────────────┴───────────┘
```

页内 offset 保持不变，页表负责找到对应的物理页。

---

## 5. VIRT 和 RSS

查看：

```bash
ps -eo pid,vsz,rss,comm | head
```

其中：

```text
VSZ / VIRT
→ 进程虚拟地址空间大小

RSS / RES
→ 当前驻留在物理内存中的页面规模
```

因此：

> **VIRT 大不等于真实 RAM 占用大。**

VIRT 中可能包含：

- 尚未真正分配物理页的地址范围
- 动态库映射
- `mmap` 区域
- 保留地址空间

---

## 6. Page Fault 是什么？

假设程序：

```c
char *p = malloc(100 * 1024 * 1024);
```

申请了 100 MB 虚拟空间，但 Linux 不一定立即给所有页面分配真实物理内存。

当第一次真正访问：

```c
p[0] = 1;
```

MMU 查页表后发现当前映射无法直接完成访问，于是触发：

> **Page Fault**

Kernel 处理流程可以粗略理解为：

```text
CPU访问虚拟地址
      ↓
页表不能直接完成映射
      ↓
Page Fault
      ↓
进入 Kernel
      ↓
判断访问是否合法
      ↓
分配/准备物理页
      ↓
更新页表
      ↓
重新执行原访问
```

因此：

> **Page Fault 本身并不等于程序崩溃。**

---

## 7. Minor 与 Major Page Fault

### Minor Page Fault

需要建立或修正页面映射，但不需要从磁盘读取所需数据。

### Major Page Fault

需要从存储设备读取页面内容：

```text
CPU访问
 ↓
页面当前不在 RAM
 ↓
需要 Storage I/O
 ↓
加载页面
 ↓
建立映射
```

Major Fault 的延迟通常明显更高，因此实时系统尤其关注关键路径上的不可预测缺页。

---

## 8. 实验一：观察进程内存布局

找一个 PID：

```bash
ps -eo pid,vsz,rss,comm | head
```

然后：

```bash
pmap -x PID
```

观察不同虚拟地址区间以及 RSS。

目标：理解一个进程的地址空间由多种区域组成，例如：

```text
代码
动态库
heap
stack
mmap 区域
```

---

## 9. 实验二：直接查看 /proc/<PID>/maps

执行：

```bash
cat /proc/$$/maps
```

你可能看到：

```text
r-xp
rw-p
[heap]
[stack]
libc.so
```

常见权限：

```text
r = readable
w = writable
x = executable
```

例如代码段往往可读、可执行，而栈通常可读写但不可执行。

这也说明虚拟内存不仅用于地址翻译，还承担内存保护作用。

---

## 10. 实验三：观察 Page Fault

执行：

```bash
/usr/bin/time -v python3 - <<'PY'
size = 200 * 1024 * 1024
x = bytearray(size)

for i in range(0, size, 4096):
    x[i] = 1
PY
```

重点观察：

```text
Minor page faults
Major page faults
Maximum resident set size
```

每隔 4096 Byte 访问一次，是为了对应典型 4 KB Page 建立直觉。

---

## 11. 与 Linux Driver 的关系

后续驱动会遇到：

```text
User Virtual Address
        ↓
Page Table
        ↓
Physical Memory
        ↑
       DMA
        ↑
      Device
```

还会接触：

```text
mmap()
ioremap()
kmalloc()
vmalloc()
dma_alloc_coherent()
```

这些 API 的前提都是先分清：

> 虚拟地址、物理地址和设备访问地址不是同一个概念。

---

## 12. 面试题

### Q1：VIRT 很大是不是说明程序占用了很多 RAM？

不是。

VIRT 表示虚拟地址空间，实际驻留物理内存更应结合 RSS/RES 判断。

### Q2：Page Fault 是不是程序异常？

不是。

Page Fault 是 CPU 发现当前页表映射不能直接完成访问时进入 Kernel 的机制。合法访问可以由 Kernel 建立映射后继续执行；非法访问如果无法处理，才可能最终导致 `SIGSEGV`。

因此：

> Page Fault ≠ Segmentation Fault。

### Q3：为什么驱动开发必须区分虚拟地址和物理地址？

因为 CPU 软件通常通过虚拟地址访问内存，而硬件寄存器、DMA 等涉及物理地址或总线地址。驱动必须建立正确映射，否则可能导致设备无法访问 Buffer、数据错误甚至 Kernel Crash。

---

## 13. 今日总结

核心链路：

```text
Virtual Address
      ↓
     MMU
      ↓
 Page Table
      ↓
Physical Page
      ↓
     RAM
```

核心记忆：

1. 用户程序中的地址通常是虚拟地址。
2. VIRT 大不等于实际 RAM 占用大，RSS 更接近驻留物理内存。
3. Linux 以 Page 为基本单位管理虚拟内存。
4. Page Fault 是正常虚拟内存机制的一部分。

## 下一天

**Day 08：read/write + mmap + Page Cache**

目标：把文件 I/O 与虚拟内存连接起来，理解数据从存储设备进入应用程序的路径。