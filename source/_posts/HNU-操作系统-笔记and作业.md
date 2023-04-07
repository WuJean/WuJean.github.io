---
title: HNU-操作系统-笔记and作业
date: 2023-04-03 21:49:36
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover: https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230403215050394.png
---

本笔记开始于操作系统期中考前一周，为了应付莫名奇妙开始的作业，以及整理上课的零碎笔记；这本书的内容安排稀碎，但是课后习题都有配套的代码，以及详细有新意的Readme文档，可以说作业才是这本书的主题内容，希望同学们课可以不认真听，但是课后作业一定要好好学习！

# 开始

先放上两个github的仓库链接，无论你是想要认真学习还是摸鱼通过，这两个仓库都能让你更加轻松快乐的学习

https://github.com/WuJean/ostep-homework.git 官方的作业合集

https://github.com/WuJean/Operating-Systems-Three-Easy-Pieces-NOTES.git 中文答案

我们现在需要将官方的示例代码clone到本地，笔者使用的是本地esxi中的ubuntu22.04环境

```
git clone https://github.com/WuJean/ostep-homework.git
```

在每章中我将采用提问的方式一步步梳理知识点

# 第4章 抽象：进程

操作系统提供了基本的抽象——进程；进程就是运行中的程序；

> 如何提供多cpu的假象来同时运行多个进程？

操作系统通过虚拟化cpu来提供这种假象；

这就是**时分共享**技术

cpu的虚拟化的协调需要：

- 低级机制
- 高级智能

这些策略将在后面一一介绍

## 4.1 抽象：进程

操作系统为正在运行的程序提供的抽象，就是所谓的进程（process）。

- 机器状态的组成部分：

  * 内存

  * 寄存器

  * 持久存储设备

## 4.2 进程API

* 创建
* 销毁
* 等待
* 其他控制
* 状态

## 4.3 进程创建：更多细节

程序如何转化为内存：操作系统如何启动并运行一个程序

* 加载程序到内存中
* 分配堆栈
* 其他初始化
* 跳转到程序入口
* OS移交控制权

## 4.4 进程状态

* 运行
* 就绪
* 阻塞

## 4.5 数据结构

操作系统为了跟踪每个进程的状态使用了进程列表。

对于停止的进程，寄存器上下文将保存其寄存器的内容。当一个进程停止时，它的寄存器将被保存到这个内存位置。  
通过恢复这些寄存器（将它们的值放回实际的物理寄存器中），操作系统可以恢复运行该进程。我们将在后面的章节中更多地了解这种技术，它被称为上下文切换（context switch）。

书中给出了XV6内核中的进程内容，其中proc结构体中有很多进程的信息。

## 问题

这个程序叫 process-run.py, 允许您查看进程状态在 CPU 上运行时的变化情况. 如本章所述，进程可以处于几种不同的状态：

```
  运行(RUNNING) - 进程正在使用CPU
  就绪(READY)   - 进程现在可以使用CPU，但是其他进程正在使用
  等待(WAITING) - 进程正在等待I / O完成(例如，它向磁盘发出请求)
  完成(DONE)    - 过程执行完毕
```

```undefined
Usage: process-run.py [options]

Options:
  -h, --help            查看帮助信息
  -s SEED, --seed=SEED  随机种子
  -l PROCESS_LIST, --processlist=PROCESS_LIST
                        以逗号分隔的要运行的进程列表，格式为X1：Y1，X2：Y2，...，其中X是该进程应运行的指令数，Y是该指令将运行的概率（从0到100）,指令包括使用CPU或进行IO

  -L IO_LENGTH, --iolength=IO_LENGTH
                        IO花费时间
  -S PROCESS_SWITCH_BEHAVIOR, --switch=PROCESS_SWITCH_BEHAVIOR
                        当进程发出IO时,系统的反应: 
                        - SWITCH_ON_IO
                        - SWITCH_ON_END 
  -I IO_DONE_BEHAVIOR, --iodone=IO_DONE_BEHAVIOR
                        IO结束时的行为类型: IO_RUN_LATER/IO_RUN_IMMEDIATE
                        - IO_RUN_IMMEDIATE: 立即切换到这个进程
                        - IO_RUN_LATER: 自然切换到这个进程(例如:取决于进程切换行为)
  -c                    计算结果
  -p, --printstats      打印统计数据； 仅与-c参数一起使用是有效
```

1. `./process-run.py -l 5:100,5:100`​

该条命令运行两个进程，我们指定的参数为"5：100"，这意味着程序应该由 5 条指令组成，并且每条指令为 CPU 指令的机会为 100％。

CPU的利用率应为百分之百

```undefined
$ python3 process-run.py -l 5:100,5:100 -c -p
Time        PID: 0        PID: 1           CPU           IOs
  1        RUN:cpu         READY             1
  2        RUN:cpu         READY             1
  3        RUN:cpu         READY             1
  4        RUN:cpu         READY             1
  5        RUN:cpu         READY             1
  6           DONE       RUN:cpu             1
  7           DONE       RUN:cpu             1
  8           DONE       RUN:cpu             1
  9           DONE       RUN:cpu             1
 10           DONE       RUN:cpu             1

Stats: Total Time 10
Stats: CPU Busy 10 (100.00%)
Stats: IO Busy  0 (0.00%)
```

2. `python3 process-run.py -l 4:100,1:0`​

先执行cpu指令，再执行io指令，最后cpu发出io_down指令，共需要11个时钟周期。

```undefined
$ python3 process-run.py -l 4:100,1:0 -c -p
Time        PID: 0        PID: 1           CPU           IOs
  1        RUN:cpu         READY             1
  2        RUN:cpu         READY             1
  3        RUN:cpu         READY             1
  4        RUN:cpu         READY             1
  5           DONE        RUN:io             1
  6           DONE       BLOCKED                           1
  7           DONE       BLOCKED                           1
  8           DONE       BLOCKED                           1
  9           DONE       BLOCKED                           1
 10           DONE       BLOCKED                           1
 11*          DONE   RUN:io_done             1

Stats: Total Time 11
Stats: CPU Busy 6 (54.55%)
Stats: IO Busy  5 (45.45%)
```

可以注意到cpu的利用率很低。

3. `python3 process-run.py -l 1:0，4:100`​

先发出io请求，cpu执行指令，io和cpu同时运行。

```undefined
$ python3 process-run.py -l 1:0,4:100 -c -p
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED       RUN:cpu             1             1
  3        BLOCKED       RUN:cpu             1             1
  4        BLOCKED       RUN:cpu             1             1
  5        BLOCKED       RUN:cpu             1             1
  6        BLOCKED          DONE                           1
  7*   RUN:io_done          DONE             1

Stats: Total Time 7
Stats: CPU Busy 6 (85.71%)
Stats: IO Busy  5 (71.43%)
```

可以看到由于io和cpu可以同时执行，io发起的时机对cpu的利用率有很大的影响。

4. `python3 process-run.py -l 1:0,4:100 -c -S SWITCH_ON_END -p`

将标志设置为 `SWITCH_ON_END`​，在进程进行 I/O 操作时，系统将不会切换到另一个进程，而是等待进程完成。

```undefined
$ python3 process-run.py -l 1:0,4:100 -c -S SWITCH_ON_END -p
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED         READY                           1
  3        BLOCKED         READY                           1
  4        BLOCKED         READY                           1
  5        BLOCKED         READY                           1
  6        BLOCKED         READY                           1
  7*   RUN:io_done         READY             1
  8           DONE       RUN:cpu             1
  9           DONE       RUN:cpu             1
 10           DONE       RUN:cpu             1
 11           DONE       RUN:cpu             1

Stats: Total Time 11
Stats: CPU Busy 6 (54.55%)
Stats: IO Busy  5 (45.45%)
```

这样会导致cpu的利用率下降。

5. `python3 process-run.py -l 1:0,4:100 -c -S SWITCH_ON_IO`​

`SWITCH_ON_IO`​在等待 I/O 时切换到另一个进程。这种策略可以在io时候很好的利用cpu。

```undefined
$ python3 process-run.py -l 1:0,4:100 -c -S SWITCH_ON_IO -p
Time        PID: 0        PID: 1           CPU           IOs
  1         RUN:io         READY             1
  2        BLOCKED       RUN:cpu             1             1
  3        BLOCKED       RUN:cpu             1             1
  4        BLOCKED       RUN:cpu             1             1
  5        BLOCKED       RUN:cpu             1             1
  6        BLOCKED          DONE                           1
  7*   RUN:io_done          DONE             1

Stats: Total Time 7
Stats: CPU Busy 6 (85.71%)
Stats: IO Busy  5 (71.43%)
```

可以看到cpu的利用率提高了。

6. `python3 process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p`​

利用`-I IO_RUN_LATER`​，当 I/O 完成时，发出它的进程不一定马上运行。

```undefined
$ python3 process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p
Time        PID: 0        PID: 1        PID: 2        PID: 3           CPU           IOs
  1         RUN:io         READY         READY         READY             1
  2        BLOCKED       RUN:cpu         READY         READY             1             1
  3        BLOCKED       RUN:cpu         READY         READY             1             1
  4        BLOCKED       RUN:cpu         READY         READY             1             1
  5        BLOCKED       RUN:cpu         READY         READY             1             1
  6        BLOCKED       RUN:cpu         READY         READY             1             1
  7*         READY          DONE       RUN:cpu         READY             1
  8          READY          DONE       RUN:cpu         READY             1
  9          READY          DONE       RUN:cpu         READY             1
 10          READY          DONE       RUN:cpu         READY             1
 11          READY          DONE       RUN:cpu         READY             1
 12          READY          DONE          DONE       RUN:cpu             1
 13          READY          DONE          DONE       RUN:cpu             1
 14          READY          DONE          DONE       RUN:cpu             1
 15          READY          DONE          DONE       RUN:cpu             1
 16          READY          DONE          DONE       RUN:cpu             1
 17    RUN:io_done          DONE          DONE          DONE             1
 18         RUN:io          DONE          DONE          DONE             1
 19        BLOCKED          DONE          DONE          DONE                           1
 20        BLOCKED          DONE          DONE          DONE                           1
 21        BLOCKED          DONE          DONE          DONE                           1
 22        BLOCKED          DONE          DONE          DONE                           1
 23        BLOCKED          DONE          DONE          DONE                           1
 24*   RUN:io_done          DONE          DONE          DONE             1
 25         RUN:io          DONE          DONE          DONE             1
 26        BLOCKED          DONE          DONE          DONE                           1
 27        BLOCKED          DONE          DONE          DONE                           1
 28        BLOCKED          DONE          DONE          DONE                           1
 29        BLOCKED          DONE          DONE          DONE                           1
 30        BLOCKED          DONE          DONE          DONE                           1
 31*   RUN:io_done          DONE          DONE          DONE             1

Stats: Total Time 31
Stats: CPU Busy 21 (67.74%)
Stats: IO Busy  15 (48.39%)
```

我们发现在第7个时钟周期时，进程1发出的io已经结束，进程进入ready状态；但是由于`-I IO_RUN_LATER`​的设置，系统选择先运行进程3，导致io阻塞了一段很长的时间。同时，由于进程在等待自己的io完成，浪费了很多的cpu时间。

系统的资源没有被合理利用。

7. `python3 process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_IMMEDIATE -c -p`​

`-I IO_RUN_IMMEDIATE`​ 设置，该设置立即运行发出I/O 的进程。

```undefined
$ python3 process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_IMMEDIATE -c -p
Time        PID: 0        PID: 1        PID: 2        PID: 3           CPU           IOs
  1         RUN:io         READY         READY         READY             1
  2        BLOCKED       RUN:cpu         READY         READY             1             1
  3        BLOCKED       RUN:cpu         READY         READY             1             1
  4        BLOCKED       RUN:cpu         READY         READY             1             1
  5        BLOCKED       RUN:cpu         READY         READY             1             1
  6        BLOCKED       RUN:cpu         READY         READY             1             1
  7*   RUN:io_done          DONE         READY         READY             1
  8         RUN:io          DONE         READY         READY             1
  9        BLOCKED          DONE       RUN:cpu         READY             1             1
 10        BLOCKED          DONE       RUN:cpu         READY             1             1
 11        BLOCKED          DONE       RUN:cpu         READY             1             1
 12        BLOCKED          DONE       RUN:cpu         READY             1             1
 13        BLOCKED          DONE       RUN:cpu         READY             1             1
 14*   RUN:io_done          DONE          DONE         READY             1
 15         RUN:io          DONE          DONE         READY             1
 16        BLOCKED          DONE          DONE       RUN:cpu             1             1
 17        BLOCKED          DONE          DONE       RUN:cpu             1             1
 18        BLOCKED          DONE          DONE       RUN:cpu             1             1
 19        BLOCKED          DONE          DONE       RUN:cpu             1             1
 20        BLOCKED          DONE          DONE       RUN:cpu             1             1
 21*   RUN:io_done          DONE          DONE          DONE             1

Stats: Total Time 21
Stats: CPU Busy 21 (100.00%)
Stats: IO Busy  15 (71.43%)
```

我们可以看到cpu利用率达到了百分百，进程在等待io时切换到其他进程，io完成后马上io_done并释放io，大大提高了系统资源的利用效率。

8. 随机生成+调参

# 第五章 插叙：进程API

## 5.1 fork系统调用

`fork()` 是一个在 Unix 系统中常用的系统调用，它用于创建一个新的进程。在 `fork()` 调用之前，进程只有一个，而在 `fork()` 调用之后，进程会被分成两个进程：原始进程和新进程。这两个进程将运行相同的程序，但是它们各自有不同的进程 ID，也就是 PID。

`fork()` 调用的基本语法是：

```
#include <unistd.h>

pid_t fork(void);
```

`fork()` 调用返回两次，一次在父进程中，一次在子进程中。在父进程中，返回的值是新进程的 PID，而在子进程中，返回的值是 0。这样，程序可以根据返回值判断它是在父进程中还是在子进程中。

在 `fork()` 调用之后，父进程和子进程将各自拥有自己的地址空间、文件描述符等资源，但是它们之间共享代码段、数据段和堆栈段等只读的资源。这种机制使得程序可以并发地执行，也为进程间通信提供了便利。

## 5.2 wait系统调用

`wait()` 是 Unix 系统中常用的系统调用之一，用于父进程等待子进程结束并获取子进程的状态信息。当父进程调用 `wait()` 函数时，如果它的任何一个子进程已经结束并且还没有被等待，则 `wait()` 函数会阻塞父进程，直到其中一个子进程结束并返回其退出状态信息。

`wait()` 调用的基本语法如下：

```
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *status);
```

其中 `status` 参数是一个指向 `int` 类型的指针，用于存储子进程的退出状态信息。如果子进程正常结束，`status` 将包含子进程的退出状态码；如果子进程被信号终止，`status` 将包含一个相应的信号值。

`wait()` 函数返回结束的子进程的进程 ID，如果没有子进程结束，则返回 -1。如果父进程同时有多个子进程在运行，则 `wait()` 函数只会等待其中一个子进程结束，而其他子进程继续运行。

除了 `wait()` 函数外，还有一些类似的系统调用，如 `waitpid()` 和 `wait3()`，它们可以更精细地控制子进程的等待和获取子进程的状态信息。

## 5.3 exec系统调用

`exec()` 是 Unix 系统中常用的系统调用之一，用于将当前进程替换为一个新的进程。通过调用 `exec()` 函数，可以让当前进程执行一个全新的程序文件，而无需创建一个新的进程。

`exec()` 系列函数的基本语法如下：

```
#include <unistd.h>

int execvp(const char *file, char *const argv[]);
```

其中，`file` 参数是要执行的程序文件的路径名，`argv` 参数是一个指向参数字符串的指针数组，它是以 NULL 结尾的。

`exec()` 函数会将当前进程的代码、数据和堆栈等全部替换为新程序的代码、数据和堆栈等，然后开始执行新程序。新程序接收的命令行参数是由 `argv` 参数指定的。

由于 `exec()` 函数会替换当前进程的代码和数据，因此调用 `exec()` 后，原来的进程 ID、环境变量、文件描述符等信息都会被新程序取代。如果 `exec()` 函数调用成功，它将不会返回，而是直接开始执行新程序。

在 Unix 系统中，常见的 `exec()` 系列函数有 `execlp()`、`execvp()`、`execle()`、`execv()` 等。这些函数的区别在于参数传递方式和搜索可执行文件的方式等。

## 5.4 为什么这样设计API

看不懂

## 5.5 其他API

后面会学到

## 文档

```
cd /workspace/ostep-homework/cpu-api
```

### fork.py

可以通过`fork.py`查看fork后的进程树

### generator.py

可以通过命令生成fork和wait的c语言文件并显示执行的结果，用来理解wait的含义

## 问题

1. 编写一个调用 fork()的程序。在调用之前,让主进程访问一个变量(例如 x)并将其值设置为某个值(例如 100)。子进程中的变量有什么值?当子进程和父进程都改变 x 的值时,变量会发生什么?

```undefined
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int x = 100;
    pid_t pid = fork();

    if (pid < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (pid == 0) {
        printf("Child process: x = %d\n", x); // 子进程访问 x 的值
        x = 200; // 子进程修改 x 的值
        printf("Child process: x = %d\n", x); // 子进程再次访问 x 的值
    } else {
        printf("Parent process: x = %d\n", x); // 父进程访问 x 的值
        x = 300; // 父进程修改 x 的值
        printf("Parent process: x = %d\n", x); // 父进程再次访问 x 的值
    }

    return 0;
}
```

运行结果：

```
Parent process: x = 100
Parent process: x = 300
Child process: x = 100
Child process: x = 200
```

当主进程访问变量 `x`​ 时，其值设置为 `100`​。当 `fork()`​ 调用创建子进程时，子进程将获得父进程的副本，因此 `x`​ 的值将是 `100`​。在子进程和父进程中都修改 `x`​ 的值时，每个进程都将修改其副本的值，因此父进程和子进程的 `x`​ 变量值是不同的。例如，子进程将 `x`​ 设置为 `200`​，而父进程将 `x`​ 设置为 `300`​，因此子进程中的 `x`​ 的值为 `200`​，而父进程中的 `x`​ 的值为 `300`​。

2. 编写一个打开文件的程序(使用 open 系统调用),然后调用 fork 创建一个新进程。子进程和父进程都可以访问 open()返回的文件描述符吗?当它们并发(即同时)写入文件时,会发生什么?

```undefined
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    int fd;
    pid_t pid;
    char *file_path = "/path/to/file.txt";

    fd = open(file_path, O_WRONLY | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    pid = fork();
    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // Child process
        char *child_message = "This is the child process writing to the file.\n";
        write(fd, child_message, strlen(child_message));
        exit(EXIT_SUCCESS);
    } else {
        // Parent process
        char *parent_message = "This is the parent process writing to the file.\n";
        write(fd, parent_message, strlen(parent_message));
        exit(EXIT_SUCCESS);
    }
}
```

在这个程序中，父进程和子进程都可以访问 `open`​ 返回的文件描述符 `fd`​。当它们并发（即同时）写入文件时，可能会发生数据交错的情况，也就是说，父进程和子进程的输出会混合在一起，而不是按照它们写入的顺序分开。这是因为在多进程环境下，操作系统会在进程之间切换，从而导致输出的顺序不确定。如果需要保证输出的顺序，需要使用进程间通信机制，如管道或共享内存等。

但我的运行结果一直是父进程先输出

这是因为在 fork() 调用后，父进程和子进程共享相同的文件描述符，即使它们有不同的进程 ID。当父进程和子进程同时写入文件时，由于操作系统的调度方式和速度的不同，很难预测哪个进程会首先写入文件。但是，无论哪个进程首先写入，它们写入的内容都会被保留，因为它们写入的文件描述符相同，不会发生竞争或冲突。因此，在这个程序中，父进程和子进程都会写入文件，并且它们写入的顺序不能保证，但是文件中应该包含两个消息。

3. 使用 fork()编写另一个程序。子进程应打印“hello”,父进程应打印“goodbye”你应该尝试确保子进程始终先打印。你能否不在父进程调用 wait()而做到这一点呢?

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(){
    char *child_message = "hello\n";
    char *parent_message = "goodbye\n";
    pid_t pid = fork();

    if(pid<0){
        fprintf(stderr,"fork failed\n");
        exit(1);
    }else if(pid==0){
        printf("%s",child_message);
    }else{
        //pid_t wait_pid = wait();
        sleep(1);
        printf("%s",parent_message);
    }

    return 0;
}
```

我们可以使用sleep来让父进程休眠；如果我们不调用 `wait()` 或 `waitpid()` 函数来等待子进程结束，子进程可能会成为“僵尸进程”。

4. 编写一个调用 fork()的程序,然后调用某种形式的 exec()来运行程序"/bin/ls"看看是否可以尝试 exec 的所有变体,包括 execl()、 execle()、 execlp()、 execv()、 execvp()和 execve(),为什么同样的基本调用会有这么多变种？



5. 现在编写一个程序，在父进程中使用 wait(),等待子进程完成。wait()返回什么？如果你在子进程中使用 wait()会发生什么？



6. 对前一个程序稍作修改，这次使用 waitpid()而不是 wait().什么时候 waitpid()会有用？



6. 编写一个创建子进程的程序，然后在子进程中关闭标准输出（STDOUT_FILENO).如果子进程在关闭描述符后调用 printf() 打印输出，会发生什么？



6. 编写一个程序，创建两个子进程，并使用 pipe()系统调用，将一个子进程的标准输出连接到另一个子进程的标准输入。
