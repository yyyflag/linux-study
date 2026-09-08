# Day 03：gdb + 多线程程序卡死定位

## 1. 学习目标

前一天通过 `strace` 可以知道程序是不是阻塞在某个 System Call 上。

今天进一步解决：

> **已经知道程序卡住了，怎么找到到底是哪一个线程、哪个函数、哪一条调用路径？**

今天重点掌握：

```text
attach 到正在运行的进程
↓
查看所有线程
↓
切换线程
↓
查看调用栈
```

核心命令：

```bash
gdb -p PID
```

进入 GDB 后：

```gdb
info threads
thread N
bt
thread apply all bt
```

---

## 2. 什么是调用栈？

假设程序执行路径是：

```text
main()
  ↓
robot_run()
  ↓
uart_receive()
  ↓
read()
```

如果线程当前停在 `read()`，调用栈可能类似：

```text
#0 read()
#1 uart_receive()
#2 robot_run()
#3 main()
```

`bt` 即 backtrace，用来查看当前线程的调用栈。

它回答的是：

> **程序是沿着哪条函数调用路径走到当前位置的？**

因此 `strace` 和 `gdb` 是互补的：

```text
strace
→ 卡在 futex/read/poll

gdb
→ 哪个线程？哪个函数？哪条调用路径？
```

---

## 3. 为什么多线程程序更难排查？

假设：

```text
Thread A：串口接收
Thread B：数据解析
Thread C：控制计算
Thread D：日志
```

如果：

```text
Thread A 拿了 mutex
↓
A 又卡在其它地方
↓
Thread B 等 mutex
↓
Thread C 又等待 B 的结果
```

就可能出现：

```text
CPU 不高
程序不崩
线程都存在
业务却完全停止
```

这时仅靠 `strace` 可能只能看到很多：

```text
futex(...)
```

但 `gdb` 可以进一步查看所有线程当前的调用栈。

---

## 4. 实验一：attach 到正在运行的进程

终端 A：

```bash
sleep 1000
```

终端 B：

```bash
pgrep sleep
```

假设 PID 为 12345：

```bash
gdb -p 12345
```

进入 GDB：

```gdb
bt
```

通常可以看到程序停在类似：

```text
nanosleep
sleep
main
```

这说明程序并没有“卡死”，而是在正常睡眠。

退出前：

```gdb
detach
quit
```

需要建立的概念：

> GDB 不只是分析崩溃，也可以在线 attach 一个仍在运行但行为异常的进程。

---

## 5. 实验二：查看所有线程

可以临时创建一个多线程程序：

```bash
python3 - <<'PY'
import threading
import time

def worker():
    while True:
        time.sleep(10)

for _ in range(3):
    threading.Thread(target=worker).start()

while True:
    time.sleep(10)
PY
```

找到 PID：

```bash
pgrep -n python3
```

attach：

```bash
gdb -p PID
```

查看线程：

```gdb
info threads
```

切换到某个线程：

```gdb
thread 2
bt
```

再切换：

```gdb
thread 3
bt
```

目标：形成下面的认识：

> 一个进程不是只有一条执行路径，多线程程序必须逐线程观察。

---

## 6. 实验三：一次性查看所有线程调用栈

执行：

```gdb
thread apply all bt
```

这是多线程卡死排查非常重要的一条命令。

假设输出：

```text
Thread 1
#0 futex_wait
#1 pthread_mutex_lock
#2 process_data

Thread 2
#0 read
#1 uart_rx_thread

Thread 3
#0 epoll_wait
#1 event_loop
```

可以快速判断：

```text
Thread 1
→ 在等锁

Thread 2
→ 在等串口数据

Thread 3
→ 在正常等待事件
```

这比只看“程序卡住了”要有价值很多。

---

## 7. Deadlock：死锁

经典死锁：

```text
Thread A              Thread B

lock(A)               lock(B)
   ↓                      ↓
lock(B)               lock(A)
   ↓                      ↓
等待                     等待
```

也就是：

```text
A 等 B
B 等 A
```

永远无法推进。

死锁常见特征：

```text
CPU 很低
进程还活着
线程也存在
业务完全停滞
```

这和 CPU 死循环完全不同。

### 常见避免方法

```text
统一加锁顺序
减少锁嵌套
缩小临界区
必要时使用 trylock / timeout
避免持锁期间执行阻塞 I/O
```

其中非常重要的是：

> **统一锁顺序。**

例如整个系统都规定“先锁 A，再锁 B”，就不要有另一处代码反过来“先锁 B，再锁 A”。

---

## 8. 面试题

### Q1：Linux 多线程程序卡死，你怎么排查？

推荐回答：

```text
1. top / ps 看进程和线程状态
2. CPU 很低时用 strace 看是否阻塞在 System Call
3. 如果看到 futex 或怀疑线程同步问题
4. gdb -p PID attach
5. info threads 查看所有线程
6. thread apply all bt 查看全部调用栈
7. 找出持锁线程和等待线程
8. 结合代码分析锁顺序或条件变量逻辑
```

面试官真正关心的是：

> 有没有“从现象 → 缩小范围 → 定位具体代码”的逻辑。

### Q2：`bt` 和 `thread apply all bt` 有什么区别？

```text
bt
→ 当前线程调用栈

thread apply all bt
→ 所有线程调用栈
```

多线程程序卡死时，后者通常更有价值。

### Q3：如何避免死锁？

核心方法：

```text
统一锁顺序
减少锁嵌套
缩小临界区
使用 trylock / timeout
避免持锁做阻塞 I/O
```

---

## 9. 今日总结

把前三天串起来：

```text
程序异常
   ↓
top / ps / free
   ↓
CPU 高？
 ┌───────┴────────┐
是                否
↓                 ↓
找热点线程       是否卡住？
↓                 ↓
perf             strace
                  ↓
            read / poll / futex
                  ↓
                 gdb
                  ↓
        info threads / all bt
                  ↓
           定位具体代码
```

核心记忆：

1. `gdb -p PID` 可以 attach 到正在运行的进程。
2. `info threads` 查看线程。
3. `bt` 看当前线程调用栈。
4. `thread apply all bt` 看所有线程调用栈。
5. 多线程卡死时，要重点关注锁、条件变量和死锁。

## 下一天

**Day 04：perf + CPU 热点函数定位**

目标：解决“程序没有卡死，但是 CPU 长期很高，到底是哪一个线程、哪个函数最耗 CPU？”
