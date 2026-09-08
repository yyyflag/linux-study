# Day 02：strace + System Call + 阻塞问题定位

## 1. 学习目标

今天解决的问题是：

> **程序没有崩溃，CPU 占用也不高，但业务就是不往下走，到底卡在哪里？**

建立第二层诊断链：

```text
程序卡住
   ↓
CPU 又不高
   ↓
怀疑在等待某个资源
   ↓
strace
   ↓
看最后停在哪个 System Call
   ↓
判断：I/O？锁？网络？设备？定时等待？
```

核心工具：

```bash
strace
```

---

## 2. 什么是 System Call？

假设用户程序执行：

```c
read(fd, buf, 100);
```

真正读取文件、串口、Socket 等资源，需要 Linux Kernel 完成。

可以把调用路径理解成：

```text
Application
   ↓
libc
   ↓
System Call
   ↓
Linux Kernel
   ↓
Driver / File / Network / Hardware
```

常见系统调用：

```text
read()
write()
openat()
close()
poll()
select()
epoll_wait()
futex()
nanosleep()
ioctl()
```

`strace` 的核心作用就是：

> **观察程序和 Linux Kernel 之间正在发生什么交互。**

---

## 3. 如何根据 System Call 判断“程序在等什么”？

### 卡在 read()

```text
read(3, ...
```

通常说明线程正在等待 fd=3 的数据。

可能对应：

- 串口
- Socket
- Pipe
- 普通文件
- 设备节点

不能只看到 `read()` 就断言“串口坏了”，还要继续确认这个 fd 对应什么。

### 卡在 poll / select / epoll_wait

```text
poll(...)
epoll_wait(...)
```

通常说明线程在等待一个或多个 fd 变为就绪状态。

### 卡在 futex

```text
futex(... FUTEX_WAIT ...)
```

常常和多线程同步相关，例如：

- `pthread_mutex`
- `std::mutex`
- `condition_variable`

看到 `futex` 时，不应该直接认为内核有问题，而应该考虑线程是不是在等待锁或条件变量。

---

## 4. 与嵌入式项目的联系

假设 Linux 上有一个 UART 线程：

```cpp
while (running) {
    read(serial_fd, buffer, sizeof(buffer));
    parse_frame(buffer);
}
```

如果串口配置为阻塞模式，而 STM32 突然停止发送数据，线程可能长时间停在：

```text
read()
```

此时常见现象：

```text
CPU ≈ 0%
进程仍存在
程序没崩
业务却不再推进
```

使用：

```bash
strace -p PID
```

如果看到：

```text
read(5,
```

长时间没有返回，就已经确定：

> 当前线程正在等待 fd=5 的读取操作完成。

接下来再确认 fd=5 到底是什么。

---

## 5. 实验一：观察 cat 阻塞在 read

终端 A：

```bash
cat
```

先不要输入任何内容。

终端 B：

```bash
ps aux | grep '[c]at'
```

找到 PID 后：

```bash
strace -p PID
```

通常会看到程序停在类似：

```text
read(0,
```

这里：

```text
0 = stdin
1 = stdout
2 = stderr
```

说明 `cat` 并没有崩溃，而是在正常等待标准输入。

回到终端 A 输入：

```text
hello
```

此时 `read()` 返回，随后还能观察到 `write()`。

核心结论：

> **Blocking 不等于 Crash。**

---

## 6. 实验二：观察一个程序进行了哪些系统调用

执行：

```bash
strace ls
```

先不用理解全部输出，只寻找：

```text
openat
read
write
close
mmap
```

再执行：

```bash
strace -c ls
```

`-c` 会给出系统调用统计摘要。

可以把它理解成：

> 这个程序与内核交互时，各类 System Call 的“调用账单”。

---

## 7. 实验三：查看一个 fd 到底对应什么

假设某程序 PID 为 1234，`strace` 显示：

```text
read(7,
```

执行：

```bash
ls -l /proc/1234/fd/7
```

可能看到：

```text
/proc/1234/fd/7 -> /dev/ttyUSB0
```

此时才能进一步确认：

> 程序正在等待 `/dev/ttyUSB0` 的输入。

这体现了一个很重要的排障原则：

```text
先观察事实
↓
缩小范围
↓
再判断根因
```

而不是一看到 `read` 就直接猜硬件坏了。

---

## 8. futex 与线程同步

C++ 中：

```cpp
std::mutex
std::condition_variable
```

pthread 中：

```c
pthread_mutex_lock()
```

在 `strace` 中经常不会直接显示成这些用户态函数名，而会看到：

```text
futex(...)
```

可以粗略理解成：

```text
Thread A 拿到 mutex
       ↓
Thread B 尝试拿 mutex
       ↓
拿不到
       ↓
futex 等待
       ↓
Thread B 睡眠
```

这和 RTOS 中“任务因为资源不可用而进入阻塞状态”的思想是相通的。

---

## 9. 面试题

### Q1：进程还存在、CPU 接近 0%，但程序完全没有响应，你怎么定位？

推荐回答顺序：

```text
1. top / ps 确认进程和线程状态
2. strace -p PID 看是否阻塞在系统调用
3. 如果是 read/poll/epoll_wait，检查对应 I/O
4. 如果大量停在 futex，考虑锁竞争或线程同步问题
5. 进一步用 gdb 查看各线程调用栈
```

### Q2：看到 read(7, ...) 长时间不返回，说明什么？

只能确定：

> 当前线程正在等待 fd=7 的读取操作完成。

还不能直接判断是串口、Socket 还是其它设备。

可以进一步：

```bash
ls -l /proc/PID/fd/7
```

确认 fd 的真实对象。

### Q3：strace 和 gdb 有什么区别？

```text
strace
→ 看程序与 Linux Kernel 的交互
→ System Call 层面

gdb
→ 看程序内部执行状态
→ 函数、线程、变量、调用栈
```

一句话记忆：

> `strace` 看“你在等内核干什么”，`gdb` 看“你的代码现在执行到哪里”。

---

## 10. 今日总结

形成第二层排障图：

```text
程序异常
   │
   ├── CPU 很高
   │      ↓
   │   top -H
   │      ↓
   │   perf / gdb
   │
   └── CPU 不高，但卡住
          ↓
       strace
          ↓
    ┌─────┼─────┐
    ↓     ↓     ↓
  read   poll  futex
    ↓     ↓     ↓
   I/O   等事件  线程同步
```

核心记忆：

1. `strace` 观察的是 System Call。
2. CPU 不高但程序卡住时，要重点考虑阻塞等待。
3. `read/poll/epoll_wait/futex` 是非常重要的排障信号。
4. 看到系统调用后，还要继续确认 fd、线程和具体代码路径。

## 下一天

**Day 03：gdb + 多线程程序卡死定位**

目标：当 `strace` 已经说明程序卡在锁或某个等待点后，进一步找到“到底是哪一个线程、哪个函数、哪条调用路径”。
