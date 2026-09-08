# Day 08：Linux 文件 I/O——read/write、mmap 与 Page Cache

## 1. 学习目标

今天把昨天的虚拟内存和 Linux 文件 I/O 连起来，解决三个问题：

> **read() 一个文件时数据经过哪里？Page Cache 是干什么的？mmap() 为什么可以像访问内存一样访问文件？**

先建立两条数据路径。

普通 `read()`：

```text
Storage
  ↓
Linux Kernel
  ↓
Page Cache
  ↓
copy_to_user()
  ↓
User Buffer
  ↓
Application
```

`mmap()`：

```text
Storage
  ↓
Page Cache
  ↕
映射到进程虚拟地址空间
  ↓
Application
```

本节重点：

- Buffered I/O
- Page Cache
- `read/write`
- `mmap`
- `iostat`
- 文件 I/O 与虚拟内存的关系

---

## 2. read() 到底做了什么？

例如：

```c
char buf[4096];
int fd = open("data.bin", O_RDONLY);
read(fd, buf, 4096);
```

程序只看到 `open()` 和 `read()`，但普通 buffered I/O 下大致会经历：

```text
Application
   ↓ read()
System Call
   ↓
Kernel / VFS
   ↓
Page Cache
```

Kernel 首先判断：

```text
需要的数据已经在 Page Cache 中吗？
```

如果在：

```text
Page Cache
   ↓
复制到 User Buffer
```

如果不在：

```text
Storage
   ↓
Page Cache
   ↓
复制到 User Buffer
```

因此 Linux Driver 中经常会遇到：

```text
copy_to_user()
copy_from_user()
```

它们用于在 Kernel Space 与 User Space 之间安全传递数据。

---

## 3. Page Cache 是什么？

可以先理解为：

> **Linux 使用 RAM 缓存文件内容的重要机制。**

第一次读取一个较大文件：

```text
SSD / eMMC
    ↓
Page Cache
    ↓
Application
```

第二次读取同一文件时，如果页面仍在 Cache 中：

```text
Page Cache
    ↓
Application
```

就可能减少重新访问较慢存储设备的次数。

这也解释了 `free -h` 中大量 `buff/cache` 并不天然代表内存浪费。

---

## 4. 为什么 free 很低不一定内存不足？

假设：

```text
RAM = 16 GB
Application = 5 GB
Page Cache = 8 GB
完全空闲 = 2 GB
```

如果只看：

```text
free = 2 GB
```

容易误以为内存不足。

但很多 Page Cache 可以在应用真正需要内存时被 Kernel 回收。

因此前面 Day 01 提到的：

```text
available
```

比单独看 `free` 更有参考意义。

---

## 5. mmap() 是什么？

`mmap()` 的核心思想是：

> **把文件对应的页面映射进进程虚拟地址空间。**

例如映射后：

```c
char x = p[100];
```

程序像访问普通内存一样访问文件内容。

实际链路：

```text
Virtual Address
      ↓
     MMU
      ↓
 Page Table
      ↓
Page Cache 中对应文件页
```

因此：

```text
File
 ↕
Virtual Memory
```

通过映射连接起来。

---

## 6. mmap 并不意味着整个文件立刻读进 RAM

假设：

```c
mmap(... 1GB ...)
```

映射一个 1 GB 文件，不代表 `mmap()` 调用瞬间就从磁盘读取 1 GB 数据。

第一次访问某个尚未驻留页面时，可能发生：

```text
CPU访问映射虚拟地址
        ↓
页表无法直接完成访问
        ↓
Page Fault
        ↓
Kernel读取文件页
        ↓
进入 Page Cache
        ↓
建立 Page Table 映射
        ↓
程序继续执行
```

因此 Day 07 的 Page Fault 与今天的 mmap 是同一个虚拟内存体系的一部分。

---

## 7. 实验一：观察 Page Cache

先：

```bash
free -h
```

记录：

```text
buff/cache
available
```

创建一个 512 MB 文件：

```bash
dd if=/dev/zero of=/tmp/testfile bs=1M count=512
```

读取：

```bash
cat /tmp/testfile > /dev/null
```

再执行：

```bash
free -h
```

观察 `buff/cache` 的变化。

实际变化值受系统当前状态影响，不要求精确增加 512 MB。

结束：

```bash
rm /tmp/testfile
```

---

## 8. 实验二：用 strace 看文件 I/O

执行：

```bash
strace -e openat,read,write,close cat /etc/hostname
```

你会看到：

```text
openat(...)
read(...)
write(...)
close(...)
```

把 Day 02 的 `strace` 和今天联系起来：

```text
cat
 ↓
read()
 ↓
System Call
 ↓
Kernel
 ↓
Page Cache / Filesystem
```

注意工具边界：

> `strace` 能看到发生了 `read()`，但不会直接告诉你这次读取是 Page Cache Hit 还是 Miss。

---

## 9. 实验三：观察 mmap

执行：

```bash
strace -e mmap,munmap ls >/dev/null
```

会看到多个 `mmap()` 调用，其中很多用于映射动态库。

再结合：

```bash
cat /proc/$$/maps
```

可以认识到：

> mmap 不只是“读取大文件的 API”，Linux 程序本身在动态库装载等场景中也大量使用内存映射。

---

## 10. 初识 iostat

执行：

```bash
iostat -xz 1
```

`iostat` 更偏向观察块设备 I/O 状态。

与之前工具形成排障链：

```text
程序很慢
  ↓
top
  ↓
CPU 不高
  ↓
strace
  ↓
大量 read/write
  ↓
iostat
  ↓
检查 Storage I/O
```

如果系统未安装 `iostat`，Ubuntu 通常由 `sysstat` 包提供。

---

## 11. 与 Linux Driver 的关系

后续驱动中常见数据路径：

```text
Device
  ↓
Kernel Buffer
  ↓ copy_to_user()
User Buffer
```

对于高吞吐设备，又可能通过 `mmap()` 减少重复拷贝：

```text
Device
  ↓
DMA
  ↓
Physical RAM
  ↓
Kernel
  ↓ mmap
User Virtual Address
  ↓
Application
```

因此理解 `read/write` 和 `mmap` 是后续理解零拷贝和 DMA Buffer 的基础。

---

## 12. 面试题

### Q1：read() 和 mmap() 有什么区别？

可以回答：

> 普通 buffered `read()` 通过系统调用读取文件数据，文件页通常进入 Page Cache，然后数据被复制到用户提供的 Buffer；`mmap()` 则将文件页面映射到进程虚拟地址空间，后续程序可通过普通内存访问方式访问映射区域，页面通常按需通过 Page Fault 建立和填充。

### Q2：Page Cache 有什么作用？

> 使用 RAM 缓存文件内容，减少重复访问较慢存储设备，提高文件 I/O 性能；很多文件缓存页在内存紧张时可以被 Kernel 回收。

### Q3：为什么不能把用户态指针直接拿到 Kernel Driver 里随便解引用？

因为用户地址属于 User Virtual Address Space，Kernel 需要安全处理地址合法性和缺页等问题。传统驱动接口通常通过：

```text
copy_from_user()
copy_to_user()
```

进行用户态和内核态数据传输。

---

## 13. 今日总结

今天把文件 I/O 和虚拟内存连接起来：

```text
Application
   │
   ├── read() ─→ Kernel ─→ Page Cache
   │
   └── mmap() ─→ Virtual Address ─→ Page Table ─→ Page Cache
```

核心记忆：

1. 普通 `read()` 常经过 Page Cache，并复制到用户 Buffer。
2. Page Cache 使用 RAM 缓存文件内容，提高 I/O 性能。
3. `mmap()` 将文件页映射到进程虚拟地址空间。
4. `mmap()` 不等于一次性把整个文件读入 RAM。

## 下一天

**Day 09：CPU Cache + Cache Line + Locality + Cache Miss**

目标：理解 CPU 为什么需要 Cache，以及数据访问方式为什么会显著影响性能。