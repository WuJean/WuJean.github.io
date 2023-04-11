---
title: ucore-lab0 and 1
date: 2023-04-10 23:24:12
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

为了方便在远程访问，本实验将全程在nuc9上编写

实验环境：

- nuc9幽灵峡谷
- esxi8.0 ubuntu22.04

# 设置实验环境

直接在github拉取代码文件：

```
git clone https://github.com/chyyuu/ucore_lab.git
```

安装硬件模拟器QEMU

```
sudo apt-get install qemu-system
```

# 编程方法和通用数据结构

了解一个操作系统的底层构造，最重要也是最基础的就是了解它的编程方法和通用的数据结构。

## 编程方法

采用了面向对象编程以及方法接口：

```
// pmm_manager is a physical memory management class. A special pmm manager - XXX_pmm_manager
// only needs to implement the methods in pmm_manager class, then XXX_pmm_manager can be used
// by ucore to manage the total physical memory space.
struct pmm_manager {
    // XXX_pmm_manager's name
    const char *name;  
    // initialize internal description&management data structure
    // (free block list, number of free block) of XXX_pmm_manager 
    void (*init)(void); 
    // setup description&management data structcure according to
    // the initial free physical memory space 
    void (*init_memmap)(struct Page *base, size_t n); 
    // allocate >=n pages, depend on the allocation algorithm 
    struct Page *(*alloc_pages)(size_t n);  
    // free >=n pages with "base" addr of Page descriptor structures(memlayout.h)
    void (*free_pages)(struct Page *base, size_t n);   
    // return the number of free pages 
    size_t (*nr_free_pages)(void);                     
    // check the correctness of XXX_pmm_manager
    void (*check)(void);                               
};
```

将函数封装好，我们需要调用时只需要调用对应的函数指针就行。

## 通用数据结构

### 双向链表结构

在uCore内核（从lab2开始）中使用了大量的双向循环链表结构来组织数据，包括空闲内存块列表、内存页链表、进程列表、设备链表、文件系统列表等的数据组织（在[labX/libs/list.h]实现）

 uCore的双向链表结构定义为：

```
struct list_entry {
    struct list_entry *prev, *next;
};
```

需要注意uCore内核的链表节点list_entry没有包含传统的data数据域，，而是在具体的数据结构中包含链表节点。以lab2中的空闲内存块列表为例，空闲块链表的头指针定义（位于lab2/kern/mm/memlayout.h中）为：

```
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

而每一个空闲块链表节点定义（位于lab2/kern/mm/memlayout）为：

```
/* *
 * struct Page - Page descriptor structures. Each Page describes one
 * physical page. In kern/mm/pmm.h, you can find lots of useful functions
 * that convert Page to other data types, such as phyical address.
 * */
struct Page {
    atomic_t ref;          // page frame's reference counter
    ……
    list_entry_t page_link;         // free list link
};
```

这样以free_area_t结构的数据为双向循环链表的链表头指针，以Page结构的数据为双向循环链表的链表节点，就可以形成一个完整的双向循环链表。

这种通用的双向循环链表结构避免了为每个特定数据结构类型定义针对这个数据结构的特定链表的麻烦，而可以让所有的特定数据结构共享通用的链表操作函数。

### 对链表的操作

- 初始化 `list_init(list_entry_t *elm)`

- 插入`__list_add(list_entry_t *elm, list_entry_t *prev, list_entry_t *next)`

- 删除`list_del(list_entry_t *listelm)`

- 访问链表节点所在的宿主数据结构

  Linux为此提供了针对数据结构XXX的le2XXX(le, member)的宏，其中le，即list entry的简称，是指向数据结构XXX中list_entry_t成员变量的指针，也就是存储在双向循环链表中的节点地址值， member则是XXX数据类型中包含的链表节点的成员变量。例如，我们要遍历访问空闲块链表中所有节点所在的基于Page数据结构的变量，则可以采用如下编程方式（基于lab2/kern/mm/default_pmm.c）：

  ```
  //free_area是空闲块管理结构，free_area.free_list是空闲块链表头
  free_area_t free_area;
  list_entry_t * le = &free_area.free_list;  //le是空闲块链表头指针
  while((le=list_next(le)) != &free_area.free_list) { //从第一个节点开始遍历
      struct Page *p = le2page(le, page_link); //获取节点所在基于Page数据结构的变量
      ……
  }
  ```

这部分内容适合二刷，第一次做只需要稍微了解一下就行

# Lab 1 系统软件启动过程

lab1中包含一个bootloader和一个OS。这个bootloader可以切换到X86保护模式，能够读磁盘并加载ELF执行文件格式，并显示字符。而这l为了实现lab1的目标，lab1提供了6个基本练习和1个扩展练习，要求完成实验报告。

对实验报告的要求：

- 基于markdown格式来完成，以文本方式为主。
- 填写各个基本练习中要求完成的报告内容
- 完成实验后，请分析ucore_lab中提供的参考答案，并请在实验报告中说明你的实现与参考答案的区别
- 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）
- 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

b1中的OS只是一个可以处理时钟中断和显示字符的幼儿园级别OS。

## 练习

在github中clone来的代码中没有源代码仓库，找了一圈后发现源代码在master分支中。运行：

```
git checkout master
```

来到主分支，即可查看源代码开始实验。

```
cd labcodes/lab1/
```

前往lab1的源代码目录。

### 练习1 理解通过make生成执行文件的过程

#### 题目

列出本实验各练习中对应的OS原理的知识点，并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。

在此练习中，大家需要通过静态分析代码来了解：

1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

#### 解答1

- ucore
  - kernel
  - bootblock

##### ucore

由题目可知我们需要分析ucore.img是如何生成的，我们首先要找到Makefile中有关于ucore生成的部分。

##### 编译用的函数

```
listf_cc = $(call listf,$(1),$(CTYPE))

# for cc
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))
```

`$(call listf,$(1),$(CTYPE))`函数会返回指定目录下的所有符合指定文件类型的文件列表，`$(1)`和`$(CTYPE)`是该函数的参数。

`$(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))`函数用于将指定目录下的源文件编译成目标文件，并添加到编译对象列表中，`$(1)`、`$(CC)`、`$(CFLAGS)`、`$(3)`、`$(2)`和`$(4)`是该函数的参数。

`$(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))`函数用于创建目标文件，并链接所有的目标文件成为可执行文件，`$(1)`、`$(2)`、`$(3)`、`$(CC)`和`$(CFLAGS)`是该函数的参数。

我们顺藤摸瓜找到这些编译的参数：

```
HOSTCC		:= gcc
HOSTCFLAGS	:= -g -Wall -O2

CC		:= $(GCCPREFIX)gcc
CFLAGS	:= -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
CTYPE	:= c S
```

`HOSTCC`是用于编译主机上的程序的C编译器，这里指定为`gcc`。

`HOSTCFLAGS`是用于编译主机上的程序的编译选项，包括`-g`（产生调试信息）、`-Wall`（启用警告信息）、`-O2`（开启优化）等。

`CC`是用于编译目标程序的C编译器，这里使用了`$(GCCPREFIX)gcc`的形式，表示使用了一个前缀`GCCPREFIX`来指定编译器。

`CFLAGS`是用于编译目标程序的编译选项，包括`-fno-builtin`（禁用内建函数）、`-Wall`（启用警告信息）、`-ggdb`（产生调试信息）、`-m32`（生成32位代码）等。

`$(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)`用于判断编译器是否支持`-fno-stack-protector`选项，如果支持，则添加该选项。

`CTYPE`用于指定可以编译的文件类型，包括`.c`和`.S`（汇编语言）。

查看Makefile中的内容：

```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

代码中的第一行定义了一个名为UCOREIMG的变量，用于表示ucore.img目标文件的路径。

接下来的代码块是一个规则，用于生成ucore.img文件。该规则有两个依赖项，即kernel和bootblock文件。它执行以下操作：

1. 使用dd命令将/dev/zero的内容写入ucore.img文件中，总共写入10000个块，每个块的大小由系统决定。
2. 使用dd命令将bootblock文件内容复制到ucore.img文件的开头，覆盖掉前面写入的0。
3. 使用dd命令将kernel文件内容追加到ucore.img文件中，从第二个块开始，保留前面写入的0。

最后，代码使用make函数create_target创建一个名为ucore.img的目标文件。该函数还会自动将目标文件添加到要构建的目标列表中。

##### kernel

```
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

我们可以看到要生成kernel目标需要: kernel.ld $(KOBJS)等依赖

KOBJS在目录kern中，通过统一的脚本来编译

```
# -------------------------------------------------------------------
# kernel

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)
```

- KINCLUDE 变量包含了一系列内核代码路径，这些路径被用作编译时的 include 路径。
- KSRCDIR 变量包含了一系列内核代码路径，这些路径被用来构建 kernel 和 libs 目录的源代码列表。
- KCFLAGS 变量被设置为 KINCLUDE 的每个路径前添加 -I 的结果，这样编译器在编译时就能正确地搜索这些路径。
- $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS)) 调用了两个函数来构建内核代码的编译列表。
- 最后，KOBJS 变量被设置为 $(call read_packet,kernel libs) 的结果

##### bootblock

```
# -------------------------------------------------------------------

# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# -------------------------------------------------------------------
```

我们在boot文件夹中找到两个需要生成的文件 bootasm bootmain，并使用默认的编译函数来编译。

#### 解答2

一个被系统认为是符合规范的硬盘主引导扇区（MBR，Master Boot Record）的特征如下：

1. 大小为512字节。
2. 前446字节包含引导程序，最后两个字节为0x55和0xAA（分别是MBR的标志位）。
3. 后64字节分成4个16字节的分区表项，每个分区表项描述了一个分区的起始位置、大小和类型。其中，分区类型标识了分区的文件系统类型，例如0x07表示NTFS文件系统，0x83表示Linux文件系统等。
4. MBR必须位于硬盘的第一个扇区，即逻辑块地址（LBA）为0的位置。

这些特征符合了BIOS和操作系统对MBR的规范要求，使得系统可以正确地引导硬盘上的操作系统。

### 练习2 使用qemu执行并调试lab1中的软件

#### 题目

为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
2. 在初始化位置0x7c00设置实地址断点,测试断点正常。
3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

#### 解答

按照提示使用单步跟踪：

1. 修改 lab1/tools/gdbinit

```
set architecture i8086
target remote :1234
```

2. 在 lab1目录下，执行

```
make debug
```

出现如下报错：

```
wjd@wjd:~/ucore_lab/labcodes/lab1$ make debug
WARNING: Image format was not specified for 'bin/ucore.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
gtk initialization failed
/bin/sh: 1: gnome-terminal: not found
make: *** [Makefile:222: debug] Error 127
```

`/bin/sh: 1: gnome-terminal: not found`表示找不到Makefile中设置的默认终端，Gnome Terminal 是 Linux 操作系统下的一种终端仿真器，它是 Gnome 桌面环境的一部分。

我们使用的是非图形化界面，故修改Makefile中的终端类型：

```
## TERMINAL        :=gnome-terminal
TERMINAL        :=bash
```

如果您在运行QEMU命令时遇到“gtk initialization failed”错误消息，这通常是由于您没有在终端中启用X11转发（X11 forwarding）引起的。X11转发是一种将X Window System的GUI应用程序显示在远程计算机上的技术，而QEMU需要X11转发来显示图形界面。

翻来覆去最终在Makefile里面找到了非图形化的编译选项

```
.PHONY: qemu qemu-nox debug debug-nox
qemu-mon: $(UCOREIMG)
        $(V)$(QEMU)  -no-reboot -monitor stdio -hda $< -serial null
qemu: $(UCOREIMG)
        $(V)$(QEMU) -no-reboot -parallel stdio -hda $< -serial null
log: $(UCOREIMG)
        $(V)$(QEMU) -no-reboot -d int,cpu_reset  -D q.log -parallel stdio -hda $< -serial null
qemu-nox: $(UCOREIMG)
        $(V)$(QEMU)   -no-reboot -serial mon:stdio -hda $< -nographic
#TERMINAL        :=gnome-terminal
TERMINAL        :=bash
debug: $(UCOREIMG)
        $(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
        $(V)sleep 2
        $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
        
debug-nox: $(UCOREIMG)
        $(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
        $(V)sleep 2
        $(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
```

我们在执行`make qemu`和`make debug`时候如果不想使用图形化，只需要在后面加上`-nox`参数：

