# Day 01：Linux 系统观测基础——top / ps / free / /proc

## 1. 学习目标

今天解决最基础的问题：**一个 Linux 程序出现“卡、慢、占资源”时，第一步应该看什么？**

建立第一层诊断链：

```text
程序有问题
   ↓
进程还活着吗？
   ↓
CPU 是否异常？
   ↓
内存是否异常？
   ↓
哪个线程在工作？
   ↓
进一步 strace / gdb / perf
```

本节重点：

- `top`：实时观察 CPU、内存、进程/线程状态
- `ps`：精确查看指定进程
- `free`：理解 Linux 内存使用
- `/proc`：理解很多系统工具的数据来源

---

## 2. 进程、线程与 CPU

Linux 中运行的程序以进程形式存在，一个进程内部可以有多个线程。

例如一个机器人程序可能是：

```text
robot_node
├── 主线程
├── UART 接收线程
├── ROS executor 线程
├── 日志线程
└── 算法工作线程
```

因此“某个进程 CPU 80%”并不意味着所有线程都很忙，可能只是某一个线程出现了死循环或高负载。

### top 中重点关注的字段

```text
PID     进程 ID
%CPU    CPU 占用
%MEM    内存占用比例
RES     当前驻留在物理内存中的大小
VIRT    虚拟地址空间大小
S       进程状态
TIME+   累计 CPU 时间
```

其中一个常见误区：

> **VIRT 很大，不等于程序真的占用了这么多 RAM。**

真正更接近当前物理内存占用的是 `RES / RSS`。

### 常见进程状态

```text
R  Running / Runnable
S  可中断睡眠，最常见
D  不可中断睡眠，常见于等待 I/O
Z  Zombie，僵尸进程
T  Stopped
```

如果系统 CPU 不高，但出现大量 D 状态任务，就应该开始考虑 I/O、驱动或底层设备等待问题。

---

## 3. Linux 内存：free 为什么经常很低？

执行：

```bash
free -h
```

你会看到类似字段：

```text
total   used   free   shared   buff/cache   available
```

不要看到 `free` 很低就认为系统快没内存了。

Linux 会主动利用空闲 RAM 做缓存，例如：

- Page Cache
- Buffer
- 文件缓存

因此更值得重点关注：

```text
available
```

它比单独看 `free` 更能反映系统当前还能提供多少可用内存。

---

## 4. /proc：Linux 系统信息的重要入口

Linux 内核会通过 `/proc` 暴露大量运行时信息。

常见文件：

```text
/proc/cpuinfo
/proc/meminfo
/proc/<PID>/status
/proc/<PID>/maps
/proc/<PID>/task/
```

例如：

```bash
cat /proc/1234/status
```

可以查看：

```text
State
Threads
VmSize
VmRSS
```

可以把很多 Linux 工具理解成：

> 帮你整理 `/proc` 中的数据并以更友好的方式展示。

---

## 5. 实验一：使用 top 观察系统

执行：

```bash
top
```

进入后按：

```text
1
```

观察每一个 CPU Core 的占用。

再按：

```text
H
```

切换到线程视图。

目标：理解“进程整体负载”和“某一个线程负载”并不是一回事。

---

## 6. 实验二：制造 CPU 高占用

终端 A：

```bash
yes > /dev/null
```

终端 B：

```bash
top
```

找到 `yes` 对应的 PID，然后：

```bash
ps -p PID -o pid,ppid,stat,%cpu,%mem,cmd
```

结束进程：

```bash
kill PID
```

形成最基本的诊断链：

```text
发现 CPU 异常
↓
找到 PID
↓
确认进程
↓
下一步找线程 / 调用栈 / 热点
```

---

## 7. 实验三：直接读取 /proc

先执行：

```bash
echo $$
```

`$$` 是当前 Shell 的 PID。

然后：

```bash
cat /proc/$$/status
```

重点找：

```text
State
Threads
VmSize
VmRSS
```

再执行：

```bash
cat /proc/meminfo | head
```

目标：理解 `top`、`free` 等工具背后的系统信息从哪里来。

---

## 8. 面试题

### Q1：一个 Linux 进程 VIRT 显示 5 GB，是不是说明它占用了 5 GB RAM？

不是。

`VIRT` 表示虚拟地址空间，可能包含：

- 动态库
- `mmap` 映射
- 尚未真正分配物理页的地址范围

实际驻留在物理内存中的大小更接近 `RES / RSS`。

### Q2：Linux 程序 CPU 100%，你怎么定位？

可以按以下顺序：

```text
1. top 确认异常进程
2. top -H / ps 看具体线程
3. 确定异常线程 TID
4. gdb 查看线程调用栈
5. perf 分析热点函数
6. 如果怀疑系统调用阻塞，再使用 strace
```

### Q3：Linux 的 free 内存只剩 200 MB，是不是一定内存不足？

不一定。

应该综合看：

```text
available
swap
进程 RSS
OOM 日志
```

Linux 会主动把空闲 RAM 用作缓存，因此不能只看 `free`。

---

## 9. 今日总结

今天建立的是最基础的一层：

```text
Linux 程序异常
      ↓
top
判断 CPU / Memory / Process
      ↓
ps / top -H
定位到进程或线程
      ↓
/proc
理解系统真实信息
      ↓
下一层工具
strace / gdb / perf
```

核心记忆：

1. `VIRT` 大不等于实际占用 RAM 大。
2. `RES / RSS` 更接近当前驻留物理内存。
3. Linux `free` 很低不一定是内存不足，要看 `available`。
4. `/proc` 是理解 Linux 运行状态的重要入口。

## 下一天

**Day 02：strace + System Call + 阻塞问题定位**

目标：解决“程序没崩、CPU 也不高，但就是卡住了，到底在等什么？”
