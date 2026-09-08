# Day 05：vmstat + Linux 调度基础 + Context Switch

## 1. 学习目标

今天解决一个比“CPU 是否跑满”更底层的问题：

> **CPU 明明没有 100%，为什么关键线程仍然可能响应很慢？Linux 到底怎么决定下一个运行谁？**

建立这一层调度视角：

```text
多个线程都想运行
        ↓
Linux Scheduler
        ↓
谁能获得 CPU？
        ↓
Running / Runnable / Sleeping
        ↓
Context Switch
```

本节重点：

- 理解 `Running`、`Runnable`、`Sleeping`
- 理解上下文切换（Context Switch）
- 使用 `vmstat` 观察调度压力
- 理解 CPU 利用率与调度延迟不是一回事

---

## 2. Runnable 不等于 Running

假设只有一个 CPU Core，同时有 3 个线程：

```text
Task A  Ready
Task B  Ready
Task C  Ready
```

三个线程都已经具备运行条件，但同一时刻 CPU 只能真正执行一个。

因此：

```text
Runnable
→ 已经准备好运行，但可能还在等待 CPU

Running
→ 当前真正占用 CPU 执行

Sleeping
→ 因 I/O、锁、定时器等条件尚未满足而等待
```

这也是为什么系统总 CPU 使用率不高，并不能保证某个关键线程一定及时获得 CPU。

---

## 3. 什么是 Context Switch？

假设 CPU 当前运行线程 A，随后切换到线程 B：

```text
Thread A
   ↓
保存执行现场
   ↓
Scheduler
   ↓
恢复 Thread B 的执行现场
   ↓
Thread B
```

需要保存或恢复的信息包括：

- CPU 寄存器
- 程序计数器 PC
- 栈指针 SP
- 调度相关状态
- 与执行上下文有关的其它状态

这就是 **Context Switch（上下文切换）**。

上下文切换本身是多任务系统必需的，但过多切换会带来额外成本，例如：

```text
调度开销
Cache 局部性下降
锁竞争
CPU 有效业务时间下降
```

因此：

> **线程越多，并不一定性能越高。**

---

## 4. vmstat：快速观察整个系统

执行：

```bash
vmstat 1
```

常见输出：

```text
procs --------memory-------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free ...       si so     bi bo      in   cs   us sy id wa
```

今天重点关注：

```text
r   正在运行或等待 CPU 的任务数量
b   处于不可中断睡眠的任务数量
cs  每秒上下文切换次数
us  用户态 CPU 时间比例
sy  内核态 CPU 时间比例
id  CPU 空闲比例
wa  I/O wait
```

### 如何理解 r？

假设机器有 4 个 CPU Core：

```text
r ≈ 1~3
→ CPU 竞争通常不算明显

r 长期远大于 4
→ 有较多 Runnable Task 在等待 CPU
```

此时就算某个关键线程计算量不大，也可能因为调度竞争而产生延迟。

---

## 5. 实验一：观察正常系统

执行：

```bash
vmstat 1
```

观察约 20 秒，重点盯：

```text
r
cs
us
sy
id
wa
```

再打开浏览器、执行编译或运行其它程序，观察各字段变化。

目标：建立直觉。

```text
系统空闲
→ id 较高
→ r 较低

CPU繁忙
→ us/sy 上升
→ r 可能增加
```

注意：`vmstat` 第一行通常是系统启动以来的平均值，实时观察时重点看后续刷新行。

---

## 6. 实验二：人为制造 CPU 竞争

查看 CPU 数量：

```bash
nproc
```

假设机器有 8 个逻辑 CPU，制造 12 个 CPU 密集任务：

```bash
for i in {1..12}; do yes > /dev/null & done
```

然后：

```bash
vmstat 1
```

重点看：

```text
r
us
id
cs
```

这时可以理解为：

```text
8 个 CPU
12 个 Runnable 任务
↓
部分任务只能等待 CPU
```

实验结束：

```bash
killall yes
```

---

## 7. 实验三：观察进程和线程

执行：

```bash
ps -eLf | head
```

重点认识：

```text
PID   进程 ID
LWP   Linux 线程 ID
```

一个多线程进程可能表现为：

```text
PID    LWP
1000   1000
1000   1001
1000   1002
```

说明：

```text
Process 1000
├── Thread 1000
├── Thread 1001
└── Thread 1002
```

与之前学过的：

```bash
top -H -p PID
```

联系起来理解：Linux 的调度问题最终通常要落到具体线程上。

---

## 8. 初识 nice

查看普通任务的 nice 值：

```bash
ps -eo pid,ni,pri,comm | head
```

Linux 普通任务 nice 值通常范围：

```text
-20  ←────────→  19
倾向获得更多CPU   更“谦让”
```

例如：

```bash
nice -n 10 ./my_program
```

表示让这个普通任务更加“谦让”。

注意：

> `nice` 主要影响普通调度任务的权重，并不是实时任务优先级。

---

## 9. 与嵌入式 SoC 的关系

典型设备处理链：

```text
Device
  ↓
IRQ
  ↓
Driver
  ↓
唤醒处理线程
  ↓
线程等待 Scheduler
  ↓
CPU 执行
```

即使设备数据已经到达，如果处理线程迟迟得不到 CPU，仍然会产生：

> **Scheduling Latency（调度延迟）**

因此嵌入式实时性不能只看平均 CPU 使用率。

---

## 10. 面试题

### Q1：进程和线程有什么区别？

可以回答：

> 进程拥有独立的虚拟地址空间和资源集合；同一进程中的线程通常共享地址空间、全局变量和打开的文件等资源，但每个线程拥有独立的执行上下文，例如寄存器和线程栈。Linux 调度器实际调度的是可运行任务，因此同一进程的不同线程可以分别运行在不同 CPU Core 上。

### Q2：CPU 使用率只有 50%，关键线程为什么仍可能延迟？

可能原因包括：

- 与其它任务竞争同一个 CPU Core
- 等待 mutex
- 等待 I/O
- 被更高优先级任务抢占
- 中断/软中断占用 CPU
- 调度策略不合理

因此：

> CPU 总利用率不能直接代表实时性。

### Q3：线程是不是越多性能越高？

不是。

线程过多可能导致：

```text
上下文切换增加
调度开销增加
Cache 局部性下降
锁竞争增加
内存占用增加
```

线程数量应结合 CPU Core 数、任务性质、I/O 比例和实时性要求综合设计。

---

## 11. 今日总结

今天建立的核心关系：

```text
Runnable Tasks
      ↓
Linux Scheduler
      ↓
选择一个任务运行
      ↓
Context Switch
      ↓
CPU
```

核心记忆：

1. Runnable 不等于 Running。
2. Context Switch 是多任务必须机制，但过多会产生成本。
3. `vmstat` 的 `r` 和 `cs` 是观察调度压力的重要入口。
4. CPU 利用率描述“CPU 有多忙”，调度研究“谁什么时候能使用 CPU”。

## 下一天

**Day 06：CPU Affinity + taskset + nice + Linux 调度策略入门**

目标：理解多核 CPU 上任务到底在哪个 Core 运行，以及如何限制它的运行 CPU。