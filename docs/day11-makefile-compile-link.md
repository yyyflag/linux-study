# Day 11：Makefile + Linux C/C++ 编译链接基础

## 1. 学习目标

今天解决三个基础但非常关键的问题：

> **一个 `main.c` 到底怎么变成可执行文件？`.h`、`.o`、`.a`、`.so` 分别是什么？为什么 Makefile 可以只重新编译真正变化的部分？**

建立完整构建流水线：

```text
main.c
  ↓
Preprocess
  ↓
main.i
  ↓
Compile
  ↓
main.s
  ↓
Assemble
  ↓
main.o
  ↓
Link
  ↓
ELF
```

本节重点：

- 预处理、编译、汇编、链接
- `.c / .h / .o / ELF`
- 编译错误与链接错误
- Makefile 依赖关系
- 静态库与动态库基本概念

---

## 2. GCC 四阶段

完整过程可以拆成：

```bash
gcc -E main.c -o main.i
gcc -S main.i -o main.s
gcc -c main.s -o main.o
gcc main.o -o app
```

平时常写：

```bash
gcc main.c -o app
```

只是 GCC 帮你一次完成了所有阶段。

### 预处理 Preprocess

处理：

- `#include`
- `#define`
- 条件编译

### 编译 Compile

把预处理后的 C/C++ 代码转换成汇编。

### 汇编 Assemble

把汇编代码转换成目标文件 `.o`。

### 链接 Link

把多个目标文件和库中的符号连接起来，生成最终 ELF。

---

## 3. 头文件到底干什么？

例如：

```c
#include <stdio.h>
```

发生在预处理阶段。

头文件主要提供：

- 函数声明
- 类型定义
- 宏定义
- 结构体声明

例如：

```c
int add(int a, int b);
```

只是告诉编译器：

> 存在一个叫 `add` 的函数，它的参数和返回值是什么。

真正实现可能在：

```text
math.c
```

因此：

> **“有声明”不等于“链接时一定能找到实现”。**

---

## 4. `.c` 与 `.o` 的关系

假设工程：

```text
main.c
motor.c
uart.c
```

通常分别编译：

```text
main.c  → main.o
motor.c → motor.o
uart.c  → uart.o
```

最后：

```text
main.o
motor.o
uart.o
   ↓
 Linker
   ↓
 robot_app
```

如果只改 `uart.c`，理想情况下：

```text
uart.c
  ↓
uart.o
  ↓
重新链接 app
```

而 `main.o`、`motor.o` 不必重新生成。

这就是增量构建的核心价值。

---

## 5. 编译错误与链接错误

### 编译错误

例如：

```c
int main()
{
    int a =
}
```

语法本身不合法，属于 Compile Error。

### 链接错误

例如：

```c
void motor_init(void);

int main(void)
{
    motor_init();
    return 0;
}
```

执行：

```bash
gcc -c main.c -o main.o
```

可能成功。

但：

```bash
gcc main.o -o app
```

可能出现：

```text
undefined reference to `motor_init'
```

说明链接器找不到 `motor_init` 的真正定义。

如果定义在 `motor.o` 中：

```bash
gcc main.o motor.o -o app
```

即可解析符号。

---

## 6. 实验一：亲手走一遍 GCC 四阶段

创建：

```c
#include <stdio.h>

#define VALUE 100

int main(void)
{
    printf("value=%d\n", VALUE);
    return 0;
}
```

执行：

```bash
gcc -E hello.c -o hello.i
grep "value=" hello.i
```

观察宏已经被展开。

继续：

```bash
gcc -S hello.c -o hello.s
head -40 hello.s
```

再：

```bash
gcc -c hello.c -o hello.o
gcc hello.o -o hello
./hello
```

目标：真正建立：

```text
.c → .i → .s → .o → ELF
```

---

## 7. 实验二：故意制造 Link Error

`main.c`：

```c
void motor_init(void);

int main(void)
{
    motor_init();
    return 0;
}
```

执行：

```bash
gcc -c main.c -o main.o
gcc main.o -o app
```

观察：

```text
undefined reference
```

再创建 `motor.c`：

```c
#include <stdio.h>

void motor_init(void)
{
    printf("motor init\n");
}
```

编译并链接：

```bash
gcc -c motor.c -o motor.o
gcc main.o motor.o -o app
./app
```

目标：区分“声明存在”和“定义参与链接”。

---

## 8. 第一个 Makefile

创建：

```makefile
CC = gcc
CFLAGS = -Wall -O2

app: main.o motor.o
	$(CC) main.o motor.o -o app

main.o: main.c
	$(CC) $(CFLAGS) -c main.c -o main.o

motor.o: motor.c
	$(CC) $(CFLAGS) -c motor.c -o motor.o

clean:
	rm -f *.o app
```

注意命令前使用 Tab。

执行：

```bash
make
```

再次：

```bash
make
```

若文件没有变化，通常不会重复构建。

然后：

```bash
touch motor.c
make
```

观察：

```text
motor.c → motor.o → app
```

而 `main.o` 不必重新编译。

---

## 9. Makefile 最核心的规则

```makefile
target: dependencies
	command
```

例如：

```makefile
app: main.o motor.o
	gcc main.o motor.o -o app
```

可以理解为：

```text
         app
        /   \
   main.o   motor.o
     |         |
   main.c    motor.c
```

Make 维护的是：

> **Dependency Graph（依赖图）**

如果依赖比目标更新，就沿依赖图重新构建。

---

## 10. 静态库与动态库

### 静态库

常见：

```text
libxxx.a
```

链接时需要的代码会被整合进最终程序。

### 动态库

常见：

```text
libxxx.so
```

程序运行时由动态链接机制加载和解析。

查看 `/bin/ls` 的动态依赖：

```bash
ldd /bin/ls
```

这可以和 Day 08 的 `mmap`、动态库映射联系起来。

---

## 11. 与 Linux Kernel Driver 的关系

后续 Kernel Module 会看到：

```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
```

这并不是普通用户态程序的 Makefile，而是调用：

> **Linux Kernel Build System / Kbuild**

嵌入式系统还会涉及：

```text
Bootloader
Kernel
Device Tree
RootFS
Application
```

这些组件都离不开构建系统和交叉编译。

---

## 12. 面试题

### Q1：C 程序从源码到可执行文件经历哪些阶段？

> 预处理、编译、汇编、链接。可以概括为 `.c → .i → .s → .o → ELF`。

### Q2：`undefined reference` 一般属于什么问题？

属于链接阶段问题。常见原因：

- 缺少目标文件 `.o`
- 没有链接对应库
- 库顺序问题
- 条件编译导致实现未生成

### Q3：Makefile 的作用是什么？

> Makefile 描述目标、依赖关系和构建规则；`make` 根据依赖关系和时间戳判断哪些目标需要重新生成，从而实现自动化和增量构建。嵌入式开发中还会用它配置交叉编译器、编译选项和链接选项。

---

## 13. 今日总结

构建主线：

```text
Source Code
   ↓
Preprocess
   ↓
Compile
   ↓
Assemble
   ↓
Object Files
   ↓
Link
   ↓
ELF
```

Makefile 位于整个构建过程上方，负责维护依赖。

核心记忆：

1. 编译和链接不是一回事。
2. 头文件主要提供声明等信息，实现最终仍需参与链接。
3. `undefined reference` 通常是 Link Error。
4. Makefile 的核心是 target、dependencies、command。
5. Driver 开发后续会进入 Kbuild。

## 下一天

**Day 12：Linux 启动流程 + 用户态/内核态 + System Call**

目标：理解 `BootROM → SPL → U-Boot → Kernel → RootFS → systemd → Application`，为 SoC bring-up 和操作系统移植打基础。