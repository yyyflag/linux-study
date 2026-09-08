# Day 04：perf + CPU 热点分析

## 1. 学习目标

前几天分别解决了：

```text
Day 01：系统资源有没有异常？
Day 02：程序是不是阻塞在 System Call？
Day 03：具体卡在哪个线程、哪个调用栈？
```

今天解决：

> **程序没有卡死，但 CPU 长时间很高，怎么找到到底是哪个线程、哪个函数在吃 CPU？**

建立性能定位主线：

```text
Process
   ↓
Thread
   ↓
Function
```

核心工具：

```bash
top -H -p PID
perf top
perf record
perf report
```

---

## 2. CPU 100% 到底意味着什么？

一个多线程进程可能是：

```text
robot_app
├── Thread 1：UART
├── Thread 2：控制算法
├── Thread 3：日志
└── Thread 4：状态机
```

如果 Thread 2 出现高负载：

```cpp
while (1) {
    calculate();
}
```

可能看到整个进程：

```text
robot_app CPU ≈ 100%
```

但实际可能是：

```text
UART 线程       0.2%
控制线程       99%
日志线程       0.1%
状态机         0.1%
```

所以第一步要从“进程高 CPU”继续定位到“具体线程高 CPU”。

---

## 3. perf 是什么？

可以先把 `perf` 理解成：

> **Linux 自带的性能分析工具，用来观察 CPU 时间主要花在什么函数、调用路径或硬件事件上。**

和前面的工具对比：

```text
strace
→ 看 System Call

gdb
→ 看某一时刻程序停在哪里

perf
→ 看一段时间内 CPU 时间主要花在哪里
```

例如程序运行 10 秒：

```text
foo()      6 秒
bar()      2 秒
kernel     1 秒
others     1 秒
```

那么 perf 可能显示类似：

```text
60% foo
20% bar
10% kernel
10% others
```

这就是 Hotspot（热点）。

性能优化的重要原则：

> **先量化，先找热点，再优化。**

---

## 4. User CPU、System CPU 与 I/O Wait

`top` 中常见：

```text
us
sy
id
wa
```

可以先这样理解：

```text
us = user，用户态 CPU 时间
sy = system，内核态 CPU 时间
id = idle，CPU 空闲
wa = I/O wait，等待 I/O 的时间
```

### us 很高

通常说明大量时间消耗在用户态代码，例如：

```c
for (...) {
    calculate();
}
```

### sy 很高

通常说明大量时间消耗在内核态，例如：

- 大量系统调用
- 驱动处理
- 网络栈
- 上下文切换
- 其它内核逻辑

### wa 很高

说明系统有较明显的 I/O 等待。

注意：

> `wa` 高不代表 CPU 算力一定不够，可能是存储或其它 I/O 路径慢。

---

## 5. 实验一：制造 CPU 高占用并定位线程

终端 A：

```bash
yes > /dev/null
```

终端 B：

```bash
pgrep yes
```

假设 PID 为 12345：

```bash
top -H -p 12345
```

`-H` 表示查看线程。

虽然 `yes` 本身非常简单，但这个命令要记住：

```text
top
→ 哪个进程高

top -H
→ 哪个线程高
```

---

## 6. 实验二：perf top 看实时热点

执行：

```bash
sudo perf top
```

你会看到类似：

```text
Overhead  Shared Object        Symbol
35.2%     libc.so              ...
20.1%     kernel               ...
```

重点关注：

```text
Overhead
```

可以理解为：

> CPU 采样中，这个函数或符号占了多大比例。

目标不是看懂全部符号，而是建立：

```text
top
→ 找进程

top -H
→ 找线程

perf
→ 找函数热点
```

---

## 7. 实验三：离线采样与调用栈

启动 CPU 密集任务：

```bash
yes > /dev/null &
PID=$!
```

采样 10 秒：

```bash
sudo perf record -g -p $PID -- sleep 10
```

然后：

```bash
sudo perf report
```

这里：

```text
-g
```

表示记录调用栈。

这样不仅能看到：

```text
哪个函数占了很多 CPU
```

还能进一步看：

```text
是谁调用了它
它又调用了谁
```

也就是从“函数热点”继续进入“调用路径热点”。

实验结束：

```bash
kill $PID
```

---

## 8. 上下文切换与性能

假设一个 CPU Core 上：

```text
Task A
运行
↓
切换
↓
Task B
运行
↓
切换
↓
Task C
```

这叫 Context Switch。

切换时系统需要保存/恢复执行上下文，例如：

```text
寄存器
程序计数器
栈状态
调度相关状态
```

因此：

> **线程不是越多越好。**

线程过多可能带来：

```text
上下文切换增加
调度开销增加
Cache 局部性下降
锁竞争增加
```

后续会用 `vmstat`、CPU affinity、`taskset`、`nice` 继续深入这一点。

---

## 9. 与嵌入式 / 驱动开发的联系

以后做 Linux Driver，可能遇到：

```text
设备功能正常
↓
驱动也没有崩溃
↓
但 CPU 占用异常高
```

可能原因包括：

```text
中断频率过高
IRQ Handler 太重
轮询代替中断
频繁 memcpy
大量上下文切换
```

所以性能调优不是简单地“把代码写短一点”，而是：

```text
先量化
↓
找热点
↓
判断瓶颈
↓
再优化
```

这正是 Linux 系统和驱动岗位需要的工程思维。

---

## 10. 面试题

### Q1：Linux 程序 CPU 100%，怎么定位？

可以回答：

```text
1. top 确认异常进程
2. top -H -p PID 定位高 CPU 线程
3. perf top 或 perf record 观察热点函数
4. 必要时结合调用栈和调试符号继续定位
5. 判断主要消耗在用户态还是内核态
6. 针对热点进行优化
```

### Q2：perf 和 gdb 有什么区别？

```text
gdb
→ 更偏调试
→ 看某一时刻执行状态
→ 变量、线程、调用栈、断点

perf
→ 更偏性能分析
→ 对一段时间采样
→ 找 CPU 热点和调用路径
```

一句话：

> **GDB 看“现在在哪”，perf 看“时间都去哪了”。**

### Q3：CPU 100% 是否意味着系统一定有性能问题？

不一定。

应该进一步判断：

```text
CPU 使用是否符合设计预期
任务是否按时完成
关键路径延迟是否异常
是否影响其它关键线程
```

嵌入式系统尤其要关注：

> **实时性，而不只是 CPU 利用率。**

---

## 11. 今日总结

建立完整的 CPU 高占用定位链：

```text
系统变慢
   ↓
top
   ↓
哪个进程？
   ↓
top -H
   ↓
哪个线程？
   ↓
perf
   ↓
哪个函数？
   ↓
User / Kernel / I/O？
   ↓
找到真正瓶颈
   ↓
优化
```

核心记忆：

1. 进程 CPU 高时，要继续定位具体线程。
2. `perf` 用来分析一段时间内的 CPU 热点。
3. `us / sy / wa` 可以帮助判断 CPU 时间大致消耗在哪里。
4. 性能优化应该先测量、找热点，再优化。
5. 嵌入式系统不能只看 CPU 利用率，还要关注实时性和调度延迟。

## 下一天

**Day 05：vmstat + Linux 调度基础 + Context Switch**

目标：理解“CPU 明明没有完全跑满，为什么程序仍然可能响应很慢？Linux 到底怎么决定下一个运行谁？”
