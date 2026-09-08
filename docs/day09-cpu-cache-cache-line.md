# Day 09：CPU Cache + Cache Line + Locality + Cache Miss

## 1. 学习目标

今天进入 CPU Cache 基础，为后续 DMA 与 Cache 一致性打地基。

核心问题：

> **数据已经在 RAM 中，CPU 为什么还需要 Cache？为什么访问同样多的数据，不同访问方式性能可能差很多？**

建立这条硬件路径：

```text
CPU Core
   ↓
L1 Cache
   ↓
L2 Cache
   ↓
L3 Cache（部分 SoC 不一定有）
   ↓
DDR / RAM
```

本节重点：

- CPU Cache
- Cache Line
- Cache Hit / Cache Miss
- 时间局部性
- 空间局部性
- `perf stat` 中的 Cache 事件

---

## 2. 为什么 CPU 需要 Cache？

CPU 执行速度通常比访问 DDR 快得多。

如果每一次：

```c
a = memory[i];
```

都必须等待较慢的主存，CPU 会大量时间停下来等待数据。

因此在 CPU 与 DDR 之间加入更小但更快的 Cache：

```text
CPU
 ↓
Cache
 ↓
RAM
```

Cache 的核心目标：

> **利用程序的局部性，把近期可能继续使用的数据放在离 CPU 更近的位置，减少访问较慢主存的次数。**

---

## 3. Cache Line 是什么？

Cache 通常不会因为程序读取 4 Byte 的 `int` 就只从更低层加载这 4 Byte。

它按一块固定粒度进行缓存和管理，这一块叫：

> **Cache Line**

很多现代 CPU 常见 Cache Line 为 64 Byte，但具体大小取决于架构和 CPU 实现。

例如：

```c
int a[1000];
```

若 Cache Line 为 64 B，一个 `int` 为 4 B，则第一次访问 `a[0]` 时，CPU 可能把附近约 16 个 `int` 所在整条 Cache Line 一起带入 Cache。

于是后续访问：

```text
a[1]
a[2]
a[3]
```

很可能直接 Cache Hit。

因此：

> **Cache 的基本管理粒度通常是 Cache Line，而不是单个变量。**

---

## 4. 时间局部性与空间局部性

### 空间局部性 Spatial Locality

如果刚访问一个地址，很可能马上访问它附近的数据。

例如：

```c
for (int i = 0; i < N; i++)
    sum += a[i];
```

这是连续访问数组，能够充分利用已经加载进来的 Cache Line。

### 时间局部性 Temporal Locality

如果刚访问过一个数据，很可能短时间内再次访问。

例如：

```c
for (...) {
    total += sensor_state;
}
```

`sensor_state` 被频繁复用，第一次进入 Cache 后，后续访问很可能直接命中。

---

## 5. Cache Hit 与 Cache Miss

CPU 请求数据：

```text
L1 有吗？
 ├── 有 → Hit → 直接使用
 └── 没有 → Miss
              ↓
             L2
              ↓
             L3
              ↓
             DDR
```

因此：

```text
Cache Hit
→ 快

Cache Miss
→ 需要访问更低层存储，延迟更高
```

所以性能问题不一定只是“算法算得慢”，也可能是：

> CPU 长时间在等待内存层级返回数据。

---

## 6. 实验一：查看本机 Cache 参数

执行：

```bash
lscpu | grep -i cache
```

再查看 sysfs：

```bash
ls /sys/devices/system/cpu/cpu0/cache/
```

例如：

```bash
cat /sys/devices/system/cpu/cpu0/cache/index0/level
cat /sys/devices/system/cpu/cpu0/cache/index0/type
cat /sys/devices/system/cpu/cpu0/cache/index0/size
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

可能看到类似：

```text
1
Data
32K
64
```

代表某级 Data Cache 容量和 Cache Line 大小。

目标：把 Cache Line 从书本概念变成真实硬件参数。

---

## 7. 实验二：连续访问和跨步访问

创建：

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N (64 * 1024 * 1024)

int main(void)
{
    char *a = malloc(N);
    if (!a) return 1;

    for (int i = 0; i < N; i++)
        a[i] = 1;

    volatile long sum = 0;
    clock_t start = clock();

    for (int i = 0; i < N; i++)
        sum += a[i];

    clock_t end = clock();

    printf("sum=%ld time=%f\n",
           sum,
           (double)(end - start) / CLOCKS_PER_SEC);

    free(a);
    return 0;
}
```

编译：

```bash
gcc -O2 cache_test.c -o cache_test
./cache_test
```

再把：

```c
i++
```

改成更大的跨步访问，例如：

```c
i += 64
```

注意：不能只比较总时间，因为访问次数已经不同。真正要理解的是：

> 跨步访问时，一整条 Cache Line 可能只使用极少的数据，空间局部性变差。

---

## 8. 实验三：使用 perf 观察 Cache 事件

执行：

```bash
perf stat -e cache-references,cache-misses ./cache_test
```

观察：

```text
cache-references
cache-misses
```

概念上的 Miss Rate：

```text
cache-misses / cache-references
```

不同 CPU、虚拟机、WSL 或权限配置可能不支持全部硬件计数器，不需要为环境限制纠结。

这里的目标是把 Day 04 的 `perf` 和 CPU Cache 联系起来：

> `perf` 不仅能找热点函数，也可以观察部分硬件性能事件。

---

## 9. 与 ARM SoC 的关系

一个典型 ARM SoC 数据路径：

```text
CPU Cortex-A
   ↓
L1 Cache
   ↓
L2 Cache
   ↓
DDR
```

设备数据路径可能是：

```text
Camera / Sensor
      ↓
     DMA
      ↓
     DDR
      ↓
     CPU
```

这马上带来一个关键问题：

```text
RAM 中数据已经被 DMA 改了
但 CPU Cache 里可能还有旧副本
```

或者反过来：

```text
CPU 改了 Cache
但新数据还没写回 RAM
DMA 读取 RAM 时可能拿到旧值
```

这就是下一课的 Cache Coherency 问题。

---

## 10. 面试题

### Q1：为什么 CPU 需要 Cache？

可以回答：

> CPU 执行速度与主存访问速度存在明显差距，因此在 CPU 与主存之间引入容量较小但速度更快的 Cache，利用时间局部性和空间局部性缓存近期或邻近的数据和指令，从而减少访问主存的次数，提高整体执行效率。

### Q2：什么是 Cache Line？

> Cache Line 是 CPU Cache 与更低层存储层级之间缓存和管理数据的基本粒度之一。发生 Cache Miss 时，CPU 通常加载包含目标地址的一整条 Cache Line，而不是只读取程序请求的几个字节。

### Q3：为什么 DMA 可能产生 Cache 一致性问题？

> DMA 可以绕过 CPU 的正常 load/store Cache 路径直接访问内存，因此 CPU Cache 中的数据副本与 RAM 中被 DMA 访问的数据可能出现不一致，需要硬件一致性机制或软件同步操作来保证双方看到正确数据。

---

## 11. 今日总结

核心链路：

```text
CPU
 ↓
L1 / L2 / L3 Cache
 ↓
DDR
```

核心记忆：

1. Cache 的根本原因是 CPU 与主存存在速度差。
2. Cache 通常以 Cache Line 为基本管理粒度。
3. 连续访问数组通常具有更好的空间局部性。
4. Cache Miss 会增加访问更低层存储的开销。
5. CPU 不一定每次都直接读写 RAM，这正是 DMA 一致性问题的基础。

## 下一天

**Day 10：DMA + Cache 一致性**

目标：理解 CPU 写/DMA 读、DMA 写/CPU 读时为什么会看到旧数据，以及 Clean / Invalidate 分别解决什么问题。