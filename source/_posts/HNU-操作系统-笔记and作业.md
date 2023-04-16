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

# 第六章

## 

## 问题

`gettimeofday()`​ 函数用于获取当前的时间和时区信息，其函数原型如下：

```undefined
#include <sys/time.h>

int gettimeofday(struct timeval *tv, struct timezone *tz);
```

其中，`tv`​ 是指向 `timeval`​ 结构体的指针，用于返回当前时间，`tz`​ 是指向 `timezone`​ 结构体的指针，用于返回当前时区信息。如果不需要时区信息，可以将 `tz`​ 设为 `NULL`​。

函数返回值为 0 表示成功，-1 表示失败并设置 `errno`​ 错误码。

### 时钟的精确度

```undefined
#include<stdio.h>
#include<sys/time.h>
int main()
{
    struct timeval start,end;
    for (int i = 0; i < 10; i++){
        gettimeofday(&start,NULL);
        gettimeofday(&end,NULL);
        printf("Begin: %d us,Time used: %d us,End: %d us \n",start.tv_usec,end.tv_usec-start.tv_usec,end.tv_usec);
    }
    return 0;
}
```

```undefined
Begin: 785855 us,Time used: 1 us,End: 785856 us 
Begin: 785945 us,Time used: 0 us,End: 785945 us 
Begin: 785950 us,Time used: 0 us,End: 785950 us 
Begin: 785954 us,Time used: 0 us,End: 785954 us 
Begin: 785957 us,Time used: 0 us,End: 785957 us 
Begin: 785960 us,Time used: 1 us,End: 785961 us 
Begin: 785964 us,Time used: 0 us,End: 785964 us 
Begin: 785968 us,Time used: 0 us,End: 785968 us 
Begin: 785971 us,Time used: 0 us,End: 785971 us 
Begin: 785974 us,Time used: 0 us,End: 785974 us
```

十次连续调用下函数耗时少，精度高，可以用来测量

#### CPU亲和性

在Linux系统中，可以使用sched_setaffinity()函数来设置线程的CPU亲和性，具体使用方法如下：

```undefined
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);

// 参数说明：
// pid：要设置CPU亲和性的线程的ID，可以使用0表示当前线程
// cpusetsize：mask参数的大小，一般使用sizeof(cpu_set_t)获取
// mask：指向cpu_set_t类型的指针，表示要设置的CPU亲和性

// 返回值：
// 成功：0
// 失败：-1，错误代码存储在errno中
```

其中，cpu_set_t类型是一个位掩码，每一位表示一个CPU核心，可以使用以下函数进行设置和清除：

```undefined
#include <sched.h>

void CPU_ZERO(cpu_set_t *set);    // 清空所有CPU核心
void CPU_SET(int cpu, cpu_set_t *set);    // 将指定CPU核心加入亲和性集合
void CPU_CLR(int cpu, cpu_set_t *set);    // 将指定CPU核心从亲和性集合中删除
int CPU_ISSET(int cpu, const cpu_set_t *set);    // 判断指定CPU核心是否在亲和性集合中
```

改进版本（单核心运行）

```undefined
#include<sched.h>
//使该程序在单核CPU上运行
void set_cpu(int cpu_number)
{
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(cpu_number, &mask);
    if (sched_setaffinity(0, sizeof(mask), &mask) == -1)
    {
        printf("warning: could not set CPU affinity, continuing...\n");
    }
}
int main()
{
    set_cpu(0);
    struct timeval start,end;
    for (int i = 0; i < 10; i++){
        gettimeofday(&start,NULL);
        gettimeofday(&end,NULL);
        printf("Begin: %ld us,Time used: %ld us,End: %ld us \n",start.tv_usec,end.tv_usec-start.tv_usec,end.tv_usec);
    }
    return 0;
}
"main.cpp" 27L, 682C
```

### 测量系统调用的上下文切换成本

# 第七章 进程调度：介绍

操作系统调度程序采用的上层策略

* 工作负载假设

  * 任务同时到达
  * 只使用cpu
  * 一旦开始保持运行到完成
  * 任务运行相同时间
  * 工作时间已知
* 先进先出 (First In First Out，FIFO) 调度算法：

  - 原理：按照进程到达的顺序依次分配CPU时间片，即先到达的进程先分配CPU时间，后到达的进程后分配CPU时间。
  - 特点：适用于长作业，但容易产生“饥饿”现象，即长时间等待CPU的进程可能一直无法被执行。
* 非抢占式短作业优先 (Shortest Job First，SJF) 调度算法：
  - 原理：按照进程的执行时间长度进行调度，即先执行执行时间短的进程，后执行执行时间长的进程。
  - 特点：可以保证平均等待时间最短，但无法预测进程执行时间，可能会出现长时间等待的情况。
* 抢占式短作业优先 (Shortest Remaining Time First，SRTF) 调度算法：
  - 原理：按照进程的剩余执行时间长度进行调度，当有新进程到达时，如果该进程的剩余执行时间比当前正在执行的进程的剩余执行时间短，则抢占CPU资源执行该进程，否则等待。
  - 特点：可以保证最短等待时间和最短周转时间，但需要不断更新进程的剩余执行时间，增加了调度的开销。
* 优先级 (Priority) 调度算法：
  - 原理：为每个进程分配一个优先级，按照优先级的高低进行调度，优先级高的进程先执行。
  - 特点：可以满足不同进程的不同需求，但容易出现“饥饿”现象，即优先级低的进程可能无法被执行。
* 最高相应比优先 (Highest Response Ratio Next，HRRN) 调度算法：
  - 原理：综合考虑进程的等待时间和执行时间，按照最高响应比进行调度，最高响应比的计算公式为：响应比 = (等待时间 + 执行时间) / 执行时间。
  - 特点：可以有效避免长作业等待时间过长的问题，但需要实时更新进程的等待时间和执行时间。

计算周转时间的公式为：周转时间 = 完成时间 - 到达时间，其中完成时间为进程结束时的时间。

* 响应时间=首次运行-到达时间
* 轮转（RR）

  * 在一个时间片内运行一个工作然后切换到队列中的下一个任务
  * 时间片长度要考虑上下文切换的成本（权衡长度摊销成本）
* 重叠可以提高利用率（第五章作业）

* 结合I/O

  * 保证交互运行
  * 将子工作视为独立的工作
* 当无法预测工作的长度该怎么办？

  * 使用多级反馈队列（下一章）

## 问题

```undefined
./scheduler -h

Usage: scheduler.py [options]

Options:
  -h, --help            show this help message and exit
  -s SEED, --seed=SEED  the random seed
  -j JOBS, --jobs=JOBS  number of jobs in the system
  -l JLIST, --jlist=JLIST
                        instead of random jobs, provide a comma-separated list
                        of run times
  -m MAXLEN, --maxlen=MAXLEN
                        max length of job
  -p POLICY, --policy=POLICY
                        sched policy to use: SJF, FIFO, RR
  -q QUANTUM, --quantum=QUANTUM
                        length of time slice for RR policy
  -c                    compute answers for me
```

1. ```undefined
   ./scheduler.py -p FIFO -j 3 -l 200,200,200 -c
   Average -- Response: 200.00  Turnaround 400.00  Wait 200.00
   ./scheduler.py -p SJF -j 3 -l 200,200,200 -c
   Average -- Response: 200.00  Turnaround 400.00  Wait 200.00
   ```

2. ```undefined
   ./scheduler.py -p FIFO -j 3 -l 100,200,300 -c
    Average -- Response: 133.33  Turnaround 333.33  Wait 133.33
   ./scheduler.py -p SJF -j 3 -l 100,200,300 -c
    Average -- Response: 133.33  Turnaround 333.33  Wait 133.33
   ```

3. ```undefined
   ./scheduler.py -p RR -j 3 -l 200,200,200 -q 1 -c
   Average -- Response: 1.00  Turnaround 599.00  Wait 399.00
   ```

添加 -q 1 选项，设置时间片为1，但是这个响应时间怪怪的

好吧我懂了 响应时间指的是cpu第一次开始处理某个任务的时间

4. 1. 当作业长度相同时，SJF调度算法的顺序与FIFO调度算法的顺序相同。因此，当所有作业的长度相同时，SJF调度算法将提供与FIFO调度算法相同的周转时间。
    2. 当所有作业的到达时间相同时，SJF调度算法将执行与FIFO调度算法相同的调度顺序。在这种情况下，SJF调度算法将提供与FIFO调度算法相同的周转时间。

5. 1. 当所有作业的长度相同时，SJF调度算法的顺序与RR调度算法的顺序相同。因此，在这种情况下，它们提供相同的响应时间。
    2. 当所有进程的可用时间都相同时，并且RR算法的量子长度等于或大于所有进程的执行时间时，SJF算法和RR算法提供相同的响应时间。在这种情况下，SJF算法会立即选择可用时间最短的进程，并将其执行完毕。RR算法则会在相同的时间段内运行所有进程一次，因此如果所有进程的执行时间都不超过量子长度，则它们的响应时间将相同。

6. 可能会增加，SJF算法可能不适合在具有变化的作业长度的系统中使用，因为它无法保证所有作业的响应时间都很短。
7. 可能会增加或保持不变

# 第八章 调度：多级反馈队列（MLFQ）

操作系统通常不知道工作要运行多久，而这又是SJF（或 STCF）等算法所必需的。MLFQ 希望给交互用户（如用户坐在屏幕前，等着进程结束）很好的交互体验，因此需要降低响应时间。

## 8.1 MLFQ：基本规则

- 规则 1：如果 A 的优先级 > B 的优先级，运行 A（不运行 B）。

- 规则 2：如果 A 的优先级 = B 的优先级，轮转运行A 和 B。

这会导致低优先级的进程永远无法执行，该怎么办呢？

## 8.2 尝试 1：如何改变优先级

既有运行时间很短、频繁放弃 CPU 的交互型工作，也有需要很多 CPU 时间、响应时间却不重要的长时间计算密集型工作。

- 规则 3：工作进入系统时，放在最高优先级（最上层队列）。
- 规则 4a：工作用完整个时间片后，降低其优先级（移入下一个队列）。

算法的一个主要目标：如果不知道工作是短工作还是长工作，那么就在开始的时候假设其是短工作，并赋予最高优先级。如果确实是短工作，则很快会执行完毕，否则将被慢慢移入低优先级队列，而这时该工作也被认为是长工作了。

- 规则 4b：如果工作在其时间片以内主动释放 CPU，则优先级不变。

针对IO的输入输出体验。

但如果有程序主动定时放弃CPU来愚弄CPU呢，该怎么修改规则？

## 8.3 尝试 2：提升优先级 

- 规则 5：经过一段时间 S，就将系统中所有工作重新加入最高优先级队列

使得进程不会饿死。

## 8.4 尝试 3：更好的计时方式 

- 规则 4：一旦工作用完了其在某一层中的时间配额（无论中间主动放弃了多少次CPU），就降低其优先级（移入低一级队列）。

这样就解决了进程的优先级的问题，我们现在剩下的问题就是如何优化这个调度方案。

## 8.5 MLFQ 调优及其他问题 

调整时间片。

## 作业

## 问题

# 第九章 调度：比例份额

一个不同类型的调度程序——比例份额（proportional-share）调度程序，有时也称为公平份额（fair-share）调度程序。比例份额算法基于一个简单的想法：调度程序的最终目标，是确保每个工作获得一定比例的 CPU 时间，而不是优化周转时间和响应时间。

彩票机制：

* 彩票货币
* 彩票转让（客户端给服务端）
* 彩票通胀

实现：遍历累加判断大小

例如，假设有三个进程A、B和C，它们分别获得了10、20和30张彩票，总共有60张彩票。当CPU时间片结束时，调度程序会随机抽取一张彩票，如果被选中的彩票属于进程B，则进程B将获得下一个时间片。

彩票调度算法的优点是可以实现公平分配CPU时间片的目标，具有一定的随机性和不可预测性，避免了一些恶意进程长期占用CPU时间的问题。但是，由于彩票调度算法需要维护彩票池和进程的彩票数，因此需要较大的系统开销，且不能保证所有进程都能获得足够的CPU时间。

步长调度：选择累积行程最短的任务执行

在步长调度算法中，每个任务都有一个步长值，用于计算任务的累积行程。步长值通常根据任务的优先级和执行时间来确定。步长值较小的任务优先级较高，可以更快地执行。

## 作业

## 问题

# 第十章 多处理器调度（高级）

增加了更多的 CPU 并没有让这类程序运行得更快，为了解决这个问题，不得不重写这些应用程序，使之能并行（parallel）执行，也许使用多线程（thread）。多线程需要使用策略来管理：

与单CPU的区别：

* 硬件缓存（cache）的使用

  * 时间局部性
  * 空间局部性
* 多处理器共享数据的方式

**问题：缓存一致性**

解决方法：

* 硬件角度

  * 通过监控内存访问
  * 总线窥探
* 程序角度

  * 保证原子操作
  * 加锁

会带来性能上的问题，随着cpu数量的增加。访问同步共享的数据结构会变慢。

## 缓存亲和度

程序运行在一个cpu上时，会在缓存中维护许多状态，下次该进程运行在相同的cpu上时运行得更快。

多处理器调度应考虑这种缓存亲和性，并尽可能将进程保持在同一个cpu上。

## 单队列多处理器调度（SQMS）

* 优点：

  * 简单
  * 将原有的策略用于多个cpu
* 问题：

  * 缺乏可拓展性（需要加锁）
  * 性能损失
  * 工作不断在cpu间转移，违背缓存亲和性
* 解决：

  * 引入亲和性机制
  * 机制变复杂
  * 牺牲其他工作的亲和度来实现负载均衡

## 多队列多处理器调度（MQMS）

将工作放入某个调度队列中

![image](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230403083446-3027434.png)​

* 优点：

  * 避免了单队列中数据共享和同步带来的问题
  * 具有可拓展性：队列数量随着cpu增加而增加
  * 具有缓存亲和性
* 问题：

  * 无法实现负载均衡（利用多cpu）
* 解决：

  * 让工作移动（迁移）
  * 工作窃取（偷看其他队列的负载）
* 需要找到合适的阈值，不然会带来较高的系统开销

#### Linux多处理器调度

不同的系统使用不同的调度策略

# 第十一章 CPU虚拟化总结

操作系统是一个拼接起来的怪物，这个怪物的每一部分都有许多实现方式，他们有一个共同的美好的哲学：高性能、可靠性但是一个美丽的解决方案最终还是少数，我们不得不做出妥协，不同的方案在特定的情况下效率更高，这就是计算机的工程。

# 第十二章 内存虚拟化

虚拟化 = 易用

隔离与保护

# 第十三章 抽象：地址空间

