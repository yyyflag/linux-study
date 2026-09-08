# Day 06：CPU Affinity + taskset + nice + Linux 调度策略入门

## 1. 学习目标

今天解决多核 Linux 上的一个关键问题：

> **一个线程到底在哪个 CPU Core 上运行？我能不能限制它？优先级又如何影响 CPU 时间分配？**

建立这一层关系：

```text
Runnable Task
      ↓
Linux Scheduler
      ↓
允许在哪些 CPU 上运行？
      ↓
CPU Affinity
      ↓
多个普通任务如何竞争？
      ↓
nice / CFS
```

本节重点：

- CPU Affinity
- `taskset`
- `nice`
- `SCHED_OTHER`
- 初识 `SCHED_FIFO` / `SCHED_RR`

---

## 2. CPU Affinity 是什么？

假设一个四核 SoC：

```text
CPU0   CPU1   CPU2   CPU3
```

Linux 默认可以让某个普通线程在不同 Core 之间迁移。

设置 CPU Affinity 后，可以限制它允许运行在哪些 Core：

```text
Task A affinity = CPU2

CPU0 ❌
CPU1 ❌
CPU2 ✅
CPU3 ❌
```

注意：

> **CPU Affinity 表示“允许在哪些 CPU 上运行”，不等于独占这些 CPU。**

即使线程只允许在 CPU2 上运行，它仍可能与 CPU2 上的其它 Runnable Task 竞争。

---

## 3. 为什么嵌入式 SoC 关注绑核？

一个典型多核系统可能希望：

```text
CPU0 → 系统服务
CPU1 → 网络
CPU2 → 控制任务
CPU3 → 算法任务
```

如果关键线程频繁在 Core 之间迁移：

```text
CPU2
 ↓
CPU3
 ↓
CPU1
```

可能带来：

- CPU Migration 开销
- Cache Locality 下降
- 实时性抖动增加
- 与非关键任务相互干扰

因此绑核有时可以提升运行行为的可预测性。

但需要记住：

> **绑核 ≠ 实时保证。**

中断、锁竞争、Cache Miss、其它线程仍可能造成延迟。

---

## 4. 实验一：观察任务运行在哪个 CPU

查看逻辑 CPU 数量：

```bash
nproc
```

查看任务最近执行的 CPU：

```bash
ps -eo pid,psr,ni,comm | head
```

其中：

```text
PSR
→ 任务最近/当前执行所在的 CPU 编号
```

例如：

```text
PID   PSR   COMMAND
1250   2    app
1380   5    worker
```

表示这些任务最近分别运行在 CPU2、CPU5。

---

## 5. 实验二：使用 taskset 绑核

启动一个 CPU 密集任务并绑定 CPU0：

```bash
taskset -c 0 yes > /dev/null &
```

获取 PID：

```bash
pgrep -n yes
```

查看 affinity：

```bash
taskset -cp PID
```

可以看到类似：

```text
current affinity list: 0
```

再用：

```bash
top
```

进入后按：

```text
1
```

展开每个 CPU Core 的占用。

如果系统至少有 CPU2，可以动态迁移：

```bash
taskset -cp 2 PID
```

再次观察 CPU0 与 CPU2 负载变化。

实验结束：

```bash
killall yes
```

---

## 6. 实验三：让两个任务竞争同一个 Core

执行：

```bash
taskset -c 0 yes > /dev/null &
taskset -c 0 yes > /dev/null &
```

两个 CPU-bound 任务都只能运行在 CPU0：

```text
CPU0
 ↑
 ├── Task A
 └── Task B
```

虽然两者都处于 Runnable，但同一时刻只能有一个真正 Running。

观察：

```bash
top
```

这正好验证 Day 05 的结论：

> Runnable 不等于 Running。

---

## 7. nice：普通任务的调度权重

重新启动两个任务：

```bash
taskset -c 0 nice -n 0 yes > /dev/null &
taskset -c 0 nice -n 10 yes > /dev/null &
```

再通过：

```bash
top
```

观察两者 CPU 时间分配。

Linux 普通任务的 nice 值通常范围：

```text
-20 ←────────────→ 19
倾向更多 CPU       更“谦让”
```

需要注意：

> nice 不是简单的“固定优先级”，更准确地说，它影响普通调度任务的调度权重和长期 CPU 时间分配倾向。

---

## 8. Linux 普通调度与 RTOS 的区别

FreeRTOS 中可以粗略理解：

```text
最高优先级 Ready Task
        ↓
        CPU
```

Linux 普通任务主要体现公平调度思想：

```text
Task A
Task B
Task C
 ↓
尽量公平地分配 CPU 时间
```

因此：

```text
nice
≠
FreeRTOS 固定任务优先级
```

这是面试里很容易混淆的一点。

---

## 9. 初识 Linux 实时调度

今天只认识三个名字：

```text
SCHED_OTHER
→ 普通调度

SCHED_FIFO
→ 实时固定优先级 FIFO

SCHED_RR
→ 实时固定优先级 + 同优先级轮转
```

### SCHED_FIFO

高优先级实时线程可以持续运行，直到：

- 自己阻塞
- 主动让出 CPU
- 被更高优先级实时线程抢占

### SCHED_RR

与 FIFO 类似，但同优先级实时任务之间进行时间片轮转。

今天不深入内核实现，只建立分类。

---

## 10. 与 SoC 驱动的关系

典型链路：

```text
Driver
  ↓
Interrupt
  ↓
唤醒处理线程
  ↓
线程进入 Runnable
  ↓
Scheduler
  ↓
CPU
```

如果关键线程迟迟拿不到 CPU，就会形成：

> Scheduling Latency

因此后续驱动性能优化可能涉及：

- CPU Affinity
- IRQ Affinity
- Realtime Scheduling
- CPU Isolation

---

## 11. 面试题

### Q1：什么是 CPU Affinity？为什么要设置？

可以回答：

> CPU Affinity 用于限制进程或线程允许运行在哪些 CPU Core 上。合理设置可以减少任务在不同 Core 之间迁移、改善 Cache 局部性，并在嵌入式实时场景中降低与其它负载的相互干扰，提高运行行为的可预测性。但设置 affinity 并不代表任务独占 CPU。

### Q2：nice 越小是不是线程一定越先运行？

不是。

`nice` 主要影响 Linux 普通调度任务的调度权重，而不是 RTOS 式固定优先级抢占语义。

需要更强实时语义时，应进一步考虑：

```text
SCHED_FIFO
SCHED_RR
```

### Q3：为什么绑核有时可以改善实时性？

核心原因包括：

```text
减少 CPU Migration
改善 Cache Locality
减少非关键任务干扰
```

但绑核本身不能保证实时性。

---

## 12. 今日总结

今天建立的关系：

```text
Runnable
   ↓
Scheduler
   ↓
在哪些 CPU 上可以运行？
   ↓
CPU Affinity / taskset

多个普通任务竞争
   ↓
nice
   ↓
影响调度权重
```

核心记忆：

1. Affinity 决定允许在哪些 CPU 上运行，绑核不等于独占核。
2. `taskset` 可以查看和设置 CPU Affinity。
3. `nice` 影响普通任务调度权重，不等于 RTOS 固定优先级。
4. `SCHED_FIFO/RR` 属于 Linux 实时调度策略。

## 下一天

**Day 07：Linux 虚拟内存 + 页表 + Page Fault**

目标：理解用户程序看到的虚拟地址如何通过 MMU 和页表映射到真实物理内存。