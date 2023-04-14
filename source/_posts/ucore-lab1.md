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
将qemu动态链接上
sudo ln -s /usr/bin/qemu-system-i386 /usr/bin/qemu
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

#### 解答1

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

我们在执行`make qemu`和`make debug`时候如果不想使用图形化，只需要在后面加上`-nox`参数。

#### 实在不想折腾了 换个desktop保平安

因为你ssh连接的终端会被qemu的进程强制一直刷新占用，其实用gdb远程调试也是可以解决的，难度会相对比较大，这边还是建议新手用图形化界面降低难度。



修改`tools/gdbinit`的内容：

```
file bin/kernel
target remote :1234
set architecture i8086
break kern_init
x /2i $pc
define hook-stop
x/2i $pc
end
```

首先执行`continue`开始调试，之后使用`next`和`si`语句进行单步调试，最后使用`x /2i $pc`指令查看当前位置附近的两条汇编代码。

#### 解答2

现在我们要尝试使用一下启动内核并远程调试：

此时在lab1目录下：

```
qemu -S -s -hda ./bin/ucore.img -monitor stdio     // 启动qemu运行ucore并开启远程调试服务
```

再在lab1目录下执行gdb：

![image-20230412153106199](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230412153106199.png)

首先使用`b *0x7c00`指令在该地址处设下断点，然后使用`c`指令让程序执行至断点位置，最后使用`x /5i $pc`指令查看当前地址附近的 5 条汇编代码。如下图所示测试断点正常。

#### 解答3

修改lab1下的Makefile如下：

![image-20230412153921031](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230412153921031.png)

-d和-D是gcc编译器的选项。

- -d选项用于在编译时生成调试信息。这些调试信息包括汇编代码，源代码和符号表等信息，用于在调试代码时使用。
- -D选项用于定义一个宏。该选项后面必须跟一个宏的名称，该名称将被定义为1。在代码中可以使用该宏名称进行条件编译，例如，如果定义了宏，则可以在代码中包含一些特定的功能，否则不包含。

lab1_asm.log是生成的日志文件名称，其中记录了使用-d选项生成的汇编代码信息。

因此，该命令行参数的含义是：使用gcc编译器生成汇编代码并记录到lab1_asm.log文件中。

我们在Makefile的编译内核选项中加入这个参数，将调试得到的汇编代码放入日记文件中便于查看对比。

执行`make debug`后发现lab1目录下多出了lab1_asm.log文件，将其与bootasm.S和bootblock.asm进行比较后发现长得很像。

#### 解答4



### 练习3 分析bootloader进入保护模式的过程

阅读**小节“保护模式和分段机制”**和lab1/boot/bootasm.S源码

#### 题目

BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

提示：需要阅读**小节“保护模式和分段机制”**和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：

- 为何开启A20，以及如何开启A20
- 如何初始化GDT表
- 如何使能和进入保护模式

#### 保护模式和分段机制

Intel 80386只有在进入保护模式后，才能充分发挥其强大的功能，提供更好的保护机制和更大的寻址空间

1. 实模式

   在bootloader接手BIOS的工作后，当前的PC系统处于实模式（16位模式）运行状态，在这种状态下软件可访问的物理内存空间不能超过1MB，且无法发挥Intel 80386以上级别的32位CPU的4GB内存管理能力。

   实模式将整个物理内存看成分段的区域，程序代码和数据位于不同区域，操作系统和用户程序并没有区别对待，而且每一个指针都是指向实际的物理地址。

   通过修改A20地址线可以完成从实模式到保护模式的转换。

2. 保护模式

   80386的全部32根地址线有效，可寻址高达4G字节的线性地址空间和物理地址空间，可访问64TB（有2^14个段，每个段最大空间为2^32字节）的逻辑地址空间，可采用分段存储管理机制和分页存储管理机制。

3. 分段存储管理机制

   只有在保护模式下才能使用分段存储管理机制。分段机制将内存划分成以起始地址和长度限制这两个二维参数表示的内存块，这些内存块就称之为段（Segment）。

   分段机涉及4个关键内容：逻辑地址、段描述符（描述段的属性）、段描述符表（包含多个段描述符的“数组”）、段选择子（段寄存器，用于定位段描述符表中表项的索引）。

   - 分段地址转换
   - 分段地址转换

##### 段描述符

每个段由如下三个参数进行定义：

- 段基地址(Base Address)
- 段界限(Limit)
- 段属性(Attributes)

#### 解答1

A20线是一根用于控制地址总线的信号线，在早期的x86架构中，由于历史原因和硬件限制，A20线的状态是不稳定的，这会导致地址总线上的某些位不能被正确地设置或读取，从而影响系统的稳定性和可靠性。因此，为了解决这个问题，操作系统需要开启A20线，以确保地址总线的正确性和稳定性。

具体来说，开启A20线可以使得CPU在保护模式下访问超过1MB的内存地址空间，这是因为在实模式下，CPU只能访问1MB以下的内存，而在保护模式下，CPU可以访问超过1MB的内存地址空间，这需要地址总线上的A20线处于稳定状态。

利用键盘控制器（8042芯片）：在早期的PC机中，键盘控制器也被用作A20信号线的控制器。通过向键盘控制器发送特定的指令，可以开启或关闭A20线。具体实现方法可以参考Intel手册或者各种开源操作系统的实现代码。

##### 具体操作

分析`boot/bootasm.S`中开启A20总线的相关代码：

```
 # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

这段代码实现了开启A20地址线的过程，主要包括以下步骤：

1. 等待8042芯片的输入缓冲区空闲，以便发送下一条指令。
2. 向8042芯片发送0xd1指令，表示要对8042芯片的P2端口进行写操作。
3. 再次等待8042芯片的输入缓冲区空闲。
4. 向8042芯片的P2端口发送0xdf指令，表示要设置A20地址线的第21位。
5. 完成A20地址线的开启。

8042芯片的命令/状态端口是一个8位寄存器，地址为0x64，它包含了8042芯片的状态信息以及接收指令的功能。具体来说，它包含以下位：

- Bit 0（Output Buffer Status，输出缓冲区状态）：当该位为0时，表示输出缓冲区中没有数据；当该位为1时，表示输出缓冲区中有数据。
- Bit 1（Input Buffer Status，输入缓冲区状态）：当该位为0时，表示输入缓冲区空闲；当该位为1时，表示输入缓冲区正在被使用。
- Bit 2（System Flag，系统标志）：该位用于控制系统级别，不在本次讨论范围内。
- Bit 3（Command/Data，指令/数据）：当该位为0时，表示将发送的是命令；当该位为1时，表示将发送的是数据。
- Bit 4（Keyboard Locked，键盘锁定）：该位用于表示键盘是否被锁定，不在本次讨论范围内。
- Bit 5（Auxiliary Output Buffer Status，辅助输出缓冲区状态）：该位用于表示辅助输出缓冲区中是否有数据，不在本次讨论范围内。
- Bit 6（Timeout，超时）：当该位为1时，表示接收到的数据已超时。
- Bit 7（Parity Error，奇偶校验错误）：当该位为1时，表示接收到的数据存在奇偶校验错误。

在这段代码中，使用了0x64端口的第0位和第1位进行等待和状态检查。

#### 解答2

在x86架构的计算机系统中，GDT（全局描述符表）是一个系统级别的数据结构，用于存储操作系统和应用程序的内存分段信息。GDT表的初始化是操作系统引导过程中的一个重要步骤。

以下是GDT表的初始化步骤：

1. 定义一个包含GDT表项的结构体，每个表项描述了一个内存段的起始地址、大小、特权级等信息。
2. 为结构体中每个表项设置对应的标识符（段选择符）。
3. 将结构体的基地址和大小（表项数目）存储在GDTR（全局描述符表寄存器）中。
4. 禁用CPU的保护模式，并使用LGDT（装载全局描述符表寄存器）指令加载GDT表到GDTR寄存器中。
5. 重新启用CPU的保护模式，此时GDT表已经初始化完成。

##### 具体操作

```
# Bootstrap GDT
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

这段代码是通过使用汇编语言中的宏定义，定义了三个段描述符并存放在gdt数组中，然后将gdt数组的地址和大小存放在gdtdesc数组中。具体解释如下：

- .p2align 2：这个指令告诉汇编器要将下一个指令的地址对齐到4字节的边界。
- gdt：这是一个标签，定义了一个包含3个段描述符的数组。
- SEG_NULLASM：这个宏定义创建一个值为0的空段描述符。在GDT表中，第一个段描述符必须是空的，因此使用该宏定义将第一个元素设置为NULL段描述符。
- SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)：这个宏定义创建一个代码段描述符，包括代码段基地址为0，段限长为0xffffffff，访问权限设置为可读可执行（STA_X|STA_R）。
- SEG_ASM(STA_W, 0x0, 0xffffffff)：这个宏定义创建一个数据段描述符，包括数据段基地址为0，段限长为0xffffffff，访问权限设置为可写（STA_W）。
- gdtdesc：这是一个标签，定义了一个包含两个字（word）和一个双字（long）的数组，其中第一个字表示GDT表的大小，第二个双字表示GDT表的地址。
- .word 0x17：这个字表示GDT表的大小（3 * 8 - 1），其中3表示GDT表中包含3个段描述符，8表示每个段描述符的大小。
- .long gdt：这个双字表示GDT表的地址，指向gdt数组的首地址。

因此，这段代码初始化了一个包含3个段描述符的GDT表，其中第一个段描述符为NULL段描述符，第二个段描述符为可执行的代码段描述符，第三个段描述符为可写的数据段描述符。同时，gdtdesc数组存储了GDT表的大小和地址信息，便于将该GDT表加载到处理器的GDTR寄存器中。

#### 解答3

使能和进入保护模式需要进行以下步骤：

1. 初始化GDT表：定义GDT表中的段描述符，包括空段描述符和代码和数据段描述符。这些描述符规定了内存中不同段的起始地址、大小和访问权限等属性。
2. 加载GDT表：使用LGDT指令将GDT表的地址和大小（即gdtdesc中的值）加载到GDTR寄存器中。
3. 设置CR0寄存器：将CR0寄存器的第0位设置为1，开启保护模式。同时，将CR0寄存器的第16位（即WP位）设置为1，启用写保护模式。
4. 进入保护模式：使用汇编指令jmp指令跳转到保护模式代码段的起始地址。此时处理器将处于保护模式，操作系统可以运行在受保护的地址空间内。

##### 具体操作

```
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain

    # If bootmain returns (it shouldn't), loop.
spin:
    jmp spin
```

这段代码使能和进入保护模式的具体步骤如下：

1. 加载GDT表：首先通过 `lgdt` 指令加载 GDT 描述符（`gdtdesc`）中的 GDT 表，告诉处理器要使用哪些段描述符以及每个段描述符的位置和属性。
2. 打开保护模式：将 `cr0` 控制寄存器的 `PE` 位设置为1，以使能保护模式。
3. 切换到保护模式：使用 `ljmp` 指令跳转到 `protcseg` 标号所在地址，跳转的目的是为了将处理器切换到 32 位代码段（`PROT_MODE_CSEG`）。
4. 设置段寄存器：切换到保护模式后，需要重新设置段寄存器，以便使用新的段描述符表。首先将数据段选择符（`PROT_MODE_DSEG`）存储到 `ax` 寄存器中，然后分别将其存储到 `ds`、`es`、`fs`、`gs`、`ss` 寄存器中。
5. 设置栈指针：在 16 位实模式下，栈指针一般设置在内存的顶端（0x9ffff），而在保护模式下，需要根据实际情况设置栈指针。这里将栈指针 `esp` 设置为 `start` 标号所在地址（0x7c00），栈区间为从0到0x7c00的地址空间。
6. 调用 C 函数：设置好栈指针后，调用 `bootmain` 函数，进入内核启动过程。
7. 循环等待：如果 `bootmain` 函数返回，说明出现了错误，跳转到 `spin` 标号所在地址，进行循环等待。
