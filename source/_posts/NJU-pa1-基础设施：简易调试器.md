---
title: NJU-pa1-基础设施：简易调试器
date: 2023-03-14 20:31:12
updated:
tags: NJU-pa
categories:
keywords:
description:
top_img:
comments:
cover:
---

# 基础设施：简易调试器

| 命令         | 格式          | 使用举例          | 说明                                                         |
| :----------- | :------------ | :---------------- | :----------------------------------------------------------- |
| 帮助(1)      | `help`        | `help`            | 打印命令的帮助信息                                           |
| 继续运行(1)  | `c`           | `c`               | 继续运行被暂停的程序                                         |
| 退出(1)      | `q`           | `q`               | 退出NEMU                                                     |
| 单步执行     | `si [N]`      | `si 10`           | 让程序单步执行`N`条指令后暂停执行, 当`N`没有给出时, 缺省为`1` |
| 打印程序状态 | `info SUBCMD` | `info r` `info w` | 打印寄存器状态 打印监视点信息                                |
| 扫描内存(2)  | `x N EXPR`    | `x 10 $esp`       | 求出表达式`EXPR`的值, 将结果作为起始内存 地址, 以十六进制形式输出连续的`N`个4字节 |
| 表达式求值   | `p EXPR`      | `p $eax + 1`      | 求出表达式`EXPR`的值, `EXPR`支持的 运算请见[调试中的表达式求值](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/1.6.html)小节 |
| 设置监视点   | `w EXPR`      | `w *0x2000`       | 当表达式`EXPR`的值发生变化时, 暂停程序执行                   |
| 删除监视点   | `d N`         | `d 2`             | 删除序号为`N`的监视点                                        |

打开 ~/NJU-pa/ics2022/nemu$ 目录 键入 make run 进入简易调试器 sdb

原项目已经实现了help、c、q等功能 我们需要按照要求完成表格中的其他功能

![image-20230314203148500](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314203148500.png)

我们发现系统已经给出了读取命令的函数

我们发现在cmd_table中有TODO，按照要求先添加相关的命令、命令描述、需要调用的函数。

```
static struct {
  const char *name;
  const char *description;
  int (*handler) (char *);
} cmd_table [] = {
  { "help", "Display information about all supported commands", cmd_help },
  { "c", "Continue the execution of the program", cmd_c },
  { "q", "Exit NEMU", cmd_q },
  /* TODO: Add more commands */
  { "si [N]","Let the program execute N instructions in a single step and then suspend execution",cmd_si}
};
```



## SI [N]

初次上手本类项目毫无头绪，于是按照提示先完成单步执行的si功能。结合前文RTFSC可以看到si只需要通过对readline文本的处理得到命令与执行次数N，再调用c命令中的cpu_exec（）函数即可

参考help命令中的内容，照猫画虎开始写函数：

```
static int cmd_help(char *args) {
  /* extract the first argument */
  char *arg = strtok(NULL, " ");
  int i;

  if (arg == NULL) {
​    /* no argument given */
​    for (i = 0; i < NR_CMD; i ++) {
​      printf("%s - %s\n", cmd_table[i].name, cmd_table[i].description);
​    }
  }
  else {
​    for (i = 0; i < NR_CMD; i ++) {
​      if (strcmp(arg, cmd_table[i].name) == 0) {
​        printf("%s - %s\n", cmd_table[i].name, cmd_table[i].description);
​        return 0;
​      }
​    }
​    printf("Unknown command '%s'\n", arg);
  }
  return 0;
}
```

观察注释中的内容，cmd_help函数分为两个部分：

- /* extract the first argument */ 解析第一个参数

- /* no argument given */ 参数不同给出不同的响应

### strtok函数

C 库函数 char *strtok(char *str, const char *delim) 分解字符串 str 为一组字符串，delim 为分隔符。

- str -- 要被分解成一组小字符串的字符串。

- delim -- 包含分隔符的 C 字符串。

strtok在指定完str字符串后，再次分割只需要使用strtok（NULL，delim）

我们在sdb_mainloop函数中发现本函数先使用 rl_gets() 函数按行读取命令到字符串str中，再对str使用strtok函数，以 “ ” 为分隔符读取到args指针当中，故寻找命令的参数只需要使用：

```
char *arg = strtok(NULL, " ");
```

sdb_mainloop函数如下：

```
void sdb_mainloop()
{
  if (is_batch_mode)
  {
​    cmd_c(NULL);
​    return;
  }

  for (char *str; (str = rl_gets()) != NULL;)
  {
​    char *str_end = str + strlen(str);
​    /* extract the first token as the command */
​    char *cmd = strtok(str, " ");
​    if (cmd == NULL)
​    {
​      continue;
​    }

​    /* treat the remaining string as the arguments,
​     \* which may need further parsing
​     */
​    char *args = cmd + strlen(cmd) + 1;
​    if (args >= str_end)
​    {
​      args = NULL;
​    }

\#ifdef CONFIG_DEVICE
​    extern void sdl_clear_event_queue();
​    sdl_clear_event_queue();
\#endif


​    int i;
​    for (i = 0; i < NR_CMD; i++)
​    {
​      if (strcmp(cmd, cmd_table[i].name) == 0)
​      {
​        if (cmd_table[i].handler(args) < 0)
​        {
​          return;
​        }
​        break;
​      }
​    }

​    if (i == NR_CMD)
​    {
​      printf("Unknown command '%s'\n", cmd);
​    }
  }
}
```

### sscanf函数

C 库函数 int sscanf(const char *str, const char *format, ...) 从字符串读取格式化输入。

- str -- 这是 C 字符串，是函数检索数据的源。

- format -- 这是 C 字符串，包含了以下各项中的一个或多个：空格字符、非空格字符 和 format 说明符。 format 说明符形式为 [=%[*][width][modifiers]type=]，具体讲解如下：

| 参数      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| *         | 这是一个可选的星号，表示数据是从流 stream 中读取的，但是可以被忽视，即它不存储在对应的参数中。 |
| width     | 这指定了在当前读取操作中读取的最大字符数。                   |
| modifiers | 为对应的附加参数所指向的数据指定一个不同于整型（针对 d、i 和 n）、无符号整型（针对 o、u 和 x）或浮点型（针对 e、f 和 g）的大小： h ：短整型（针对 d、i 和 n），或无符号短整型（针对 o、u 和 x） l ：长整型（针对 d、i 和 n），或无符号长整型（针对 o、u 和 x），或双精度型（针对 e、f 和 g） L ：长双精度型（针对 e、f 和 g） |
| type      | 一个字符，指定了要被读取的数据类型以及数据读取方式。具体参见下一个表格。 |

sscanf 类型说明符：

| 类型      | 合格的输入                                                   | 参数的类型     |
| --------- | ------------------------------------------------------------ | -------------- |
| c         | 单个字符：读取下一个字符。如果指定了一个不为 1 的宽度 width，函数会读取 width 个字符，并通过参数传递，把它们存储在数组中连续位置。在末尾不会追加空字符。 | char *         |
| d         | 十进制整数：数字前面的 + 或 - 号是可选的。                   | int *          |
| e,E,f,g,G | 浮点数：包含了一个小数点、一个可选的前置符号 + 或 -、一个可选的后置字符 e 或 E，以及一个十进制数字。两个有效的实例 -732.103 和 7.12e4 | float *        |
| o         | 八进制整数。                                                 | int *          |
| s         | 字符串。这将读取连续字符，直到遇到一个空格字符（空格字符可以是空白、换行和制表符）。 | char *         |
| u         | 无符号的十进制整数。                                         | unsigned int * |
| x,X       | 十六进制整数。                                               | int *          |

我们暂时只需要将读取的字符串转化为int类型

```
sscanf(arg,"%d",&N);
```

最后按照逻辑写出si的完整代码如下：

```
static int cmd_si(char *args)
{
  int N = 1;
  /* extract the first argument */
  char *arg = strtok(NULL, " ");
  char *arg2 = strtok(NULL," ");    //find more argument
  if(arg2!=NULL){
​    printf("Too much args \n");  
  }
  else if(arg==NULL){
​    cpu_exec(N);
  }
  else{
​    sscanf(arg,"%d",&N);
​    if(N<0||N>9){
​      printf("please enter number from 1~9. default:1\n");
​      return 0;
​    }
​    cpu_exec(N);
  }
  return 0;
}
```

![image-20230314204601531](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314204601531.png)

- 考虑了输入参数超过1的的情况

- 考虑了输入过大的情况

- numu对执行的命令数量有设置 故执行的命令过多会退出程序

## info 

> 打印寄存器就更简单了. 不过既然寄存器的结构是ISA相关的, 我们希望能为简易调试器屏蔽ISA的差异. 框架代码已经为大家准备了如下的API:
>
> ```
> // nemu/src/isa/$ISA/reg.c
> void isa_reg_display(void);
> ```
>
> 执行info r之后, 就调用isa_reg_display(), 在里面直接通过printf()输出所有寄存器的值即可.

我们只需要在cmd_info函数中调用isa_reg_display函数并补全isa_reg_display()函数

```
const char *regs[] = {
  "$0", "ra", "sp", "gp", "tp", "t0", "t1", "t2",
  "s0", "s1", "a0", "a1", "a2", "a3", "a4", "a5",
  "a6", "a7", "s2", "s3", "s4", "s5", "s6", "s7",
  "s8", "s9", "s10", "s11", "t3", "t4", "t5", "t6"
};
```

在reg.c中发现寄存器是以char*[]形式存储的，参考GDB的info输出，可以写出输出函数：

```
void isa_reg_display() {
  for(int i=0;i<32;i++){
​    printf("%s            %d\n",regs[i],*regs[i]);
  }
}
```

cmd_info函数如下，该函数预留了info w命令查看监视点的接口：

```
static int cmd_info(char *args){
  char *arg = strtok(NULL, " ");
  char *arg2 = strtok(NULL," ");    //find more argument
  if(arg2!=NULL){
    printf("Too much args \n");  
  }
  else if(*arg=='r'){
    isa_reg_display();
  }
  else if(*arg=='w'){  
  }
  return 0; 
}
```

使用结果如下，info r查看所有32个寄存器的名称及对应的值：

![image-20230314204829876](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314204829876.png)

## x N EXPR

> 内存通过在nemu/src/memory/paddr.c中定义的大数组pmem来模拟. 在客户程序运行的过程中, 总是使用vaddr_read()和vaddr_write() (在nemu/src/memory/vaddr.c中定义)来访问模拟的内存. 
>
> 扫描内存的实现也不难, 对命令进行解析之后, 先求出表达式的值. 但你还没有实现表达式求值的功能, 现在可以先实现一个简单的版本: 规定表达式EXPR中只能是一个十六进制数, 例如
>
> x 10 0x80000000

在nemu/src/memory/vaddr.c中找到vaddr_read()函数：

```
word_t vaddr_read(vaddr_t addr, int len) {
  return paddr_read(addr, len);
}
```

可以看到这个函数的参数完美符合我们输入的参数，现在只需要完成保证输入的地址是一个十六进制的数，再调用函数即可。

### 如何判断是否为十六进制？

在 C 语言中 isdigit函数用于判断一个 字符是否为十进制数字，而 isxdigit 函数用于判断一个数字是否为十六进制数字，如果是则返回非零值，否则，返回 0。

```
#include <ctype.h>    //判断十六进制
   if(isxdigit(arg[0])){
  }
  else{
    printf("EXPR need 16\n");
    return 0;
  }
```

完整代码如下：

```
static int cmd_x(char *args){
  char *arg = strtok(NULL, " ");
  int N;
  if(arg!=NULL){
    sscanf(arg,"%d",&N);
  }
  else{
    printf("arg1 need num\n");
    return 0;
  }
  char *arg2 = strtok(NULL," ");
  if(arg2==NULL){
    printf("Need addr\n");
    return 0;
  }
  if(isxdigit(arg2[0])){
  }
  else{
    printf("EXPR need 16\n");
    return 0;
  }
  int addr;
  sscanf(arg2,"%x",&addr);
  char *arg3 = strtok(NULL," ");    //find more argument
  if(arg3!=NULL){
    printf("Too much args \n");  
  }
  else if(arg!=NULL && arg2!=NULL){
    printf("addr = %x\n",addr+N*4);
    printf("%u\n",vaddr_read(addr+N*4,4));
  }
  return 0;
}
```

![image-20230314205053981](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314205053981.png)

### 改进版本 info

寄存器为8位，查阅资料后发现在 

```
#include "local-include/reg.h"
```

中有查看寄存器相关的函数：

```
static inline int check_reg_idx(int idx) {
  IFDEF(CONFIG_RT_CHECK, assert(idx >= 0 && idx < 32));
  return idx;
}
#define gpr(idx) cpu.gpr[check_reg_idx(idx)]
static inline const char* reg_name(int idx, int width) {
  extern const char* regs[];
  return regs[check_reg_idx(idx)];
}
```

一番探索后终于找到寄存器的定义位置，可以看到系统使用了gpr定义了32个寄存器，还有起始地址pc

![image-20230314205235233](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230314205235233.png)

```
check_reg_idx(int idx)
```

该函数接受一个下标地址，通过判断下标是否合法来保证查看寄存器值得时候不会越界

```
#define gpr(idx) cpu.gpr[check_reg_idx(idx)]
```

我们需要调用cpu.gpr直接访问结构体中gpr数组中得内容，还可以使用

```
static inline const char* reg_name(int idx, int width)
```

来获取寄存器得名称，输入参数为寄存器得下标以及一个置空参数width

为了保证可读性，每行显示4个寄存器的值

```
void isa_reg_display() {
  int j=0;
  for(int i=0;i<32;i++){
    printf("%-3s : 0x%08x | ",reg_name(i,0),cpu.gpr[i]);
    j++;
    if(j%4==0) printf("\n");
  }
  printf("$pc : 0x%08x \n",cpu.pc);
}
```

### 改进版本 x N EXPR

查看别人的博客后发现我对这个功能的理解出错 正确的做法应该是以4个字节为间隔遍历该内存下的数据，但大体上的代码没用大问题，效果如下图所示：

![img](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FPXm0zsbKQeeDoXO9vWXw%2Fuploads%2FFFbU6d1UNeZk1HGjRdDo%2Fimage.png?alt=media&token=31312f6d-9b54-4f46-87f7-36d79cc26c6d)

修改后的代码如下：

```
static int cmd_x(char *args){
  char *arg = strtok(NULL, " ");
  int N;
  if(arg!=NULL){
    // sscanf(arg,"%d",&N);
    N = atoi(arg);  //str2int
  }
  else{
    printf("arg1 need num\n");
    return 0;
  }
  char *arg2 = strtok(NULL," ");
  if(arg2==NULL){
    printf("Need addr\n");
    return 0;
  }
  if(isxdigit(arg2[0])){
  }
  else{
    printf("EXPR need 16\n");
    return 0;
  }
  // sscanf(arg2,"%x",&addr);
  char *str;    //provide endptr for strtol
  vaddr_t addr =  strtol( arg2,&str,16 );
  char *arg3 = strtok(NULL," ");    //find more argument
  if(arg3!=NULL){
    printf("Too much args \n");  
  }
  else if(arg!=NULL && arg2!=NULL){
    for(int i=0;i<N;i++){
      // printf("addr = %x\n",addr+N*4);
      // printf("0x%02x ",vaddr_read(addr+i*4,4));
      uint32_t data = vaddr_read(addr + i * 4,4);
      printf("0x%08x  " , addr + i * 4 );
        for(int j =0 ; j < 4 ; j++){
            printf("0x%02x " , data & 0xff);
            data = data >> 8 ;
        }
        printf("\n");
    }
  }
  return 0;
}
```

### 字符串于数字转化函数 atoi()   strtol()

### atoi()

C 库函数 int atoi(const char *str) 把参数 str 所指向的字符串转换为一个整数（类型为 int 型）

```
int atoi(const char *str)
```

- str -- 要转换为整数的字符串。

该函数返回转换后的长整数，如果没有执行有效的转换，则返回零。

### strtol()

字符串转化为各种进制

C 库函数 char *strncpy(char *dest, const char *src, size_t n) 把 src 所指向的字符串复制到 dest，最多复制 n 个字符。当 src 的长度小于 n 时，dest 的剩余部分将用空字节填充。

- str -- 要转换为长整数的字符串。
- endptr -- 对类型为 char* 的对象的引用，其值由函数设置为 str 中数值后的下一个字符。
- base -- 基数，必须介于 2 和 36（包含）之间，或者是特殊值 0。

该函数返回转换后的长整数，如果没有执行有效的转换，则返回一个零值。

### 输出函数 printf() 及一些常用的输出设置
