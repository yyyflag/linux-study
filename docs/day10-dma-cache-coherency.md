# Day 10：DMA + Cache 一致性

## 1. 学习目标

今天接着 Day 09 的 CPU Cache，正式解决 SoC / Driver 中非常关键的问题：

> **CPU 和 DMA 为什么可能看到不同的数据？Cache Clean / Invalidate 分别解决什么？**

建立完整数据通路：

```text
                 CPU
                  │
              Load/Store
                  ↓
             CPU Cache
                  │
                  ↓
                 DDR
                ↗   ↖
               /     \
             DMA     DMA
             ↑         ↓
           Device    Device
```

最重要的一句话：

> **CPU 通常通过 Cache 访问内存，而 DMA 可以绕开 CPU Cache 直接访问内存。**

本节重点：

- DMA 的作用
- CPU 写 / DMA 读
- DMA 写 / CPU 读
- Cache Clean
- Cache Invalidate
- Coherent / Non-coherent DMA 基本概念

---

## 2. DMA 为什么存在？

如果没有 DMA，大量设备数据搬运可能需要 CPU 反复参与：

```text
Device
  ↓
CPU 读取
  ↓
CPU 写 RAM
  ↓
重复
```

有 DMA 后：

```text
CPU 配置 DMA
      ↓
DMA Controller
      ↓
Device ←→ Memory
```

CPU 可以同时处理其它任务。

因此 DMA 的核心价值可以理解为：

> **让设备与内存之间进行高效数据传输，减少 CPU 逐数据参与搬运。**

常见场景：

```text
UART
SPI
ADC
Camera
Storage
Network
```

---

## 3. 场景一：CPU 写，DMA 读

初始 RAM：

```text
buffer = 100
```

CPU 修改：

```c
buffer = 200;
```

因为 CPU 有 Cache，可能变成：

```text
CPU Cache：200
RAM：      100
```

如果此时 DMA 直接读取 RAM：

```text
DMA → RAM → 100
```

设备拿到的就是旧数据。

解决思路：

> 把 CPU Cache 中的脏数据写回更低层内存。

这个动作通常称为：

```text
Cache Clean / Write-back
```

概念口诀：

```text
CPU 写
  ↓
DMA 读
  ↓
Clean
```

---

## 4. 场景二：DMA 写，CPU 读

初始状态：

```text
CPU Cache：100
RAM：      100
```

DMA 把 RAM 改成：

```text
RAM：200
```

但是 CPU Cache 仍可能是：

```text
100
```

如果 CPU 再次访问且命中 Cache，就可能继续看到旧数据。

解决思路：

> 让 CPU Cache 中对应旧副本失效。

即：

```text
Cache Invalidate
```

概念口诀：

```text
DMA 写
  ↓
CPU 读
  ↓
Invalidate
```

---

## 5. Clean 和 Invalidate 的区别

### Clean

```text
Cache 中脏数据
      ↓
写回更低层内存
```

目标：让 RAM 中获得 CPU 的最新修改。

### Invalidate

```text
Cache 中旧副本
      ↓
标记为无效
```

目标：CPU 下次读取时重新从一致的数据来源获取。

需要强调：

> 上面的“CPU 写→DMA 读用 Clean、DMA 写→CPU 读用 Invalidate”是帮助理解方向的概念模型。真正 Linux Driver 应使用内核 DMA API，让平台根据其一致性模型完成正确同步，而不是自行随意操作 Cache。

---

## 6. 为什么 Cache Line 又变得重要？

假设 Cache Line 为：

```text
64 Byte
```

一条 Line：

```text
0x1000 ─────────────────── 0x103F
```

如果 DMA Buffer 只占其中一部分：

```text
普通变量 A
DMA Buffer
普通变量 B
```

Cache 操作往往以 Cache Line 为粒度，而不是单个变量为粒度。

因此 Driver 中经常还要关注：

```text
Alignment
Cache Line Size
DMA Buffer Boundary
```

这就是为什么 Day 09 必须先理解 Cache Line。

---

## 7. Coherent 与 Non-coherent DMA

### Hardware Coherent 场景

部分平台硬件能够参与维护 CPU Cache 与其它总线 Master 访问的一致性：

```text
CPU Cache
   ↕
Memory
   ↕
DMA
```

硬件帮助保证一定范围的一致性。

### Non-coherent 场景

如果硬件路径不自动保持一致：

```text
CPU Cache
    X
DMA / Memory
```

则软件必须按照平台和 Linux DMA API 的规则进行同步。

在 Linux Driver 中以后会接触：

```text
dma_alloc_coherent()
dma_map_single()
dma_sync_single_for_cpu()
dma_sync_single_for_device()
```

今天只认识这些名字，不背 API 细节。

---

## 8. 实验一：再次确认 Cache Line 大小

执行：

```bash
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

假设输出：

```text
64
```

自己画：

```text
0x1000 ────────── 0x103F
       64 Bytes

0x1040 ────────── 0x107F
       64 Bytes
```

思考：

如果 DMA Buffer 从 `0x1020` 开始，它实际上与其它数据共享了 `0x1000~0x103F` 这条 Cache Line。

---

## 9. 实验二：观察 Linux 系统中的 DMA 信息

执行：

```bash
grep -i dma /proc/interrupts
```

再看：

```bash
dmesg | grep -i dma | head -30
```

不同平台输出差异很大。普通 PC 上不一定有直观 DMA Controller 条目，ARM SoC / 开发板通常更容易看到相关信息。

目标：意识到 DMA 是系统真实硬件路径的一部分，而不仅是 MCU HAL 中的一个 API 名字。

---

## 10. 实验三：两个数据一致性思维题

### Case A

```text
CPU Cache = 0xAA
RAM       = 0x00

DMA 准备读取 RAM
```

DMA 可能读到：

```text
0x00
```

需要：

```text
Clean
↓
RAM = 0xAA
↓
DMA 读取
```

### Case B

```text
CPU Cache = 0x11
RAM       = 0x11

DMA 把 RAM 改成 0x99
```

CPU 可能仍读到：

```text
0x11
```

需要：

```text
Invalidate
↓
CPU重新取数
↓
0x99
```

如果能不看答案自己讲清楚这两个 Case，今天就达标。

---

## 11. 面试题

### Q1：DMA 为什么会产生 Cache 一致性问题？

可以回答：

> CPU 对内存的访问通常经过 Cache，而 DMA 可以直接访问内存，因此同一块 Buffer 可能同时存在 RAM 中的数据和 CPU Cache 中的副本。如果 CPU 修改尚未写回 RAM，DMA 可能读到旧数据；如果 DMA 修改了 RAM，而 CPU Cache 仍保留旧副本，CPU 又可能读到旧数据，所以需要硬件一致性机制或软件同步。

### Q2：Cache Clean 和 Invalidate 有什么区别？

```text
Clean
→ 把 Cache 中脏数据写回内存

Invalidate
→ 让 Cache 中对应副本失效
```

概念上：

```text
CPU 写 → DMA 读：重点考虑 Clean
DMA 写 → CPU 读：重点考虑 Invalidate
```

### Q3：用了 DMA 为什么 CPU 占用会下降？

> 因为 CPU 不再逐数据完成设备与内存之间的搬运，而主要负责配置 DMA、提交 Buffer、处理完成事件或中断、管理同步等工作。真正的大块数据传输由 DMA 控制器完成。

---

## 12. 今日总结

今天把前几天的内存知识第一次串成 SoC 数据通路：

```text
Application
    ↓
Virtual Address
    ↓
MMU / Page Table
    ↓
Physical Memory
    ↙          ↘
 CPU           DMA
  ↓             ↓
Cache         Device
  └────一致性────┘
```

核心记忆：

1. DMA 减少 CPU 对大块数据搬运的参与。
2. CPU Cache 与 DMA 都可能访问同一块 RAM，因此可能看到不同版本的数据。
3. Clean 将 CPU 的新数据写回内存。
4. Invalidate 让 CPU 的旧 Cache 副本失效。
5. 真正 Linux Driver 中应使用 Linux DMA API，而不是自行随意操作 Cache。

## 下一天

**Day 11：Makefile + Linux C/C++ 编译链接基础**

目标：理解 `.c → .i → .s → .o → ELF`，以及 Makefile 为什么是 Linux Driver 和嵌入式系统构建的基础。