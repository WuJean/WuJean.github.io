---
title: ucore-lab2
date: 2023-04-21 21:44:30
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

完成ucore-lab1后总觉得做实验的过程怪怪的，每个练习的关联性并不强，有许多内容需要打开搜索引擎获得答案，难度缺少梯度做起来很难受。直到我某天闲着无聊二刷了lab0和lab1，才发现我所疑惑的一切问题都藏在了参考资料中，可能遇到的坑以及相对应的解决方法，甚至是不同练习之间的联系以及整个设计的框架都在参考资料中有详细的说明。

故我认为ucore的最佳做法是先阅读完实验原理再开始着手做相关练习，不要被内容的顺序所迷惑，这样才能真正有所得。

本周刚刚学完linux的内存管理，并且肝完了小班课的slab原理，顺带了解了一下分段和分页机制，希望在本次的lab2-内存管理模块中能亲手实现内存管理的机制，对内存管理有更深刻的理解。

# 实验目的

- 理解基于段页式内存地址的转换机制
- 理解页表的建立和使用方法
- 理解物理内存的管理方法

# 实验内容

- 了解如何发现系统中的物理内存
- 了解如何建立对物理内存的初步管理
- 了解页表相关的操作

## 练习0 merge

将实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示：可采用diff和patch工具进行半自动的合并（merge）

在ucore/labcodes目录下执行：

```
diff -urN lab2 lab1 > lab1.patch
patch -p1 < lab1.patch
```

# 物理内存管理

首先对实验的执行流程做个介绍，并进一步介绍**如何探测物理内存的大小与布局**，如何**以页为单位来管理计算机系统中的物理内存**，如何**设计物理内存页的分配算法**，最后比较详细地分析了在80386的段页式硬件机制下，ucore操作系统把**段式内存管理的功能弱化**，并实现**以分页为主的页式内存管理的过程**。

## 实验执行流程概述

lab2中内核启动过程：

- 在bootloader中，完成了对物理内存资源的探测工作；
- ucore kernel在后续执行中能够基于bootloader探测出的物理内存情况进行物理内存管理初始化工作；
- bootloader调用位于lab2/kern/init/entry.S中的kern_entry函数设置堆栈，临时建立了一个段映射关系，为之后建立分页机制的过程做准备
- 调用kern_init函数完成一些输出并对lab1实验结果的检查
- 调用pmm_init函数完成物理内存的管理
- 调用pic_init函数和idt_init函数，执行中断和异常相关的初始化工作

lab2中物理内存管理的过程：

1. 初始化物理内存页管理器框架pmm_manager；
2. 建立空闲的page链表，这样就可以分配以页（4KB）为单位的空闲内存了；
3. 检查物理内存页分配算法；
4. 为确保切换到分页机制后，代码能够正常执行，先建立一个临时二级页表；
5. 建立一一映射关系的二级页表；
6. 使能分页机制；
7. 从新设置全局段描述符表；
8. 取消临时二级页表；
9. 检查页表建立是否正确；
10. 通过自映射机制完成页表的打印输出（这部分是扩展知识）

另外，主要注意的相关代码内容包括：

- boot/bootasm.S中**探测内存**部分（从probe_memory到finish_probe的代码）；
- 管理每个物理页的Page数据结构（在**mm/memlayout.h**中），这个数据结构也是**实现连续物理内存分配算法**的关键数据结构，可通过此数据结构来**完成空闲块的链接和信息存储**，而基于这个数据结构的管理**物理页数组起始地址**就是**全局变量pages**，具体初始化此数组的函数位于**page_init函数**中；
- 用于实现连续物理内存分配算法的**物理内存页管理器框架pmm_manager**，这个数据结构定义了实现内存分配算法的关键函数指针，而同学**需要完成这些函数的具体实现**；
- 设定二级页表和建立页表项以完成虚实地址映射关系，这与硬件相关，且用到不少内联函数，源代码相对难懂一些。具体完成页表和**页表项建立**的重要函数是boot_map_segment函数，而get_pte函数是完成**虚实映射关键**的关键。

我们可以看到从内存探测到内存分配、内存管理（页表和虚实映射）都给出了具体实现的函数，接下来的部分将会给出其具体实现的原理，帮助我们更好的理解内存管理的机制。

## 探测系统物理内存布局

获取内存大小的方法由 BIOS 中断调用和直接探测两种：

- BIOS 中断调用方法是一般只能在实模式下完成

  - 基于INT 15h中断（88h e801h e820h）

    NT 15h是一个BIOS中断，用于提供许多与系统硬件和性能相关的服务。在x86架构的计算机中，BIOS中断通常用于执行低级系统任务，例如与键盘、显示器、磁盘驱动器和其他硬件设备的交互。

    INT 15h中断提供了许多不同的功能，其中包括：

    1. 系统内存检测和测试
    2. 键盘输入和输出
    3. 磁盘驱动器控制
    4. 显示器设置和字符生成
    5. 系统时间和日期查询
    6. 打印机操作
    7. 系统电源管理

    这些功能可以通过不同的INT 15h子功能代码来访问。在编写x86汇编语言程序时，可以使用INT 15h中断调用这些子功能来执行各种系统任务。

  - 在本实验中，**我们通过e820h中断获取内存信息**

    INT E820h：是一个32位的扩展BIOS中断，用于获取更为详细的系统内存信息。它提供了有关系统内存布局的详细信息，包括物理内存和设备内存的地址范围、内存类型（例如可用、已保留、ACPI等）等。这些信息对于操作系统和引导加载程序非常重要，可以帮助它们了解系统中哪些内存区域可以使用，哪些必须保留。

- 而直接探测方法必须在保护模式下完成

在 bootloader 进入保护模式之前调用这个 BIOS 中断，并且**把 e820 映射结构保存在物理地址0x8000处**。具体实现详见**boot/bootasm.S**。有关探测系统物理内存方法和具体实现的信息参见lab2试验指导的附录A“探测物理内存分布和大小的方法”和附录B“实现物理内存探测”。

### 探测物理内存分布和大小的方法

BIOS中断调用必须在**实模式**下进行，所以在bootloader进入保护模式前完成这部分工作相对比较合适。这些部分由boot/bootasm.S中从**probe_memory处到finish_probe处的代码部分**完成完成。通过BIOS中断获取内存可调用参数为e820h的INT 15h BIOS中断。BIOS通过**系统内存映射地址描述符**（Address Range Descriptor）格式来表示系统物理内存布局，其具体表示如下：

```
Offset  Size    Description
00h    8字节   base address               #系统内存块基地址
08h    8字节   length in bytes            #系统内存大小
10h    4字节   type of address range     #内存类型
```

INT15h BIOS中断的详细调用参数:

```
eax：e820h：INT 15的中断调用参数；
edx：534D4150h (即4个ASCII字符“SMAP”) ，这只是一个签名而已；
ebx：如果是第一次调用或内存区域扫描完毕，则为0。 如果不是，则存放上次调用之后的计数值；
ecx：保存地址范围描述符的内存大小,应该大于等于20字节；
es:di：指向保存地址范围描述符结构的缓冲区，BIOS把信息写入这个结构的起始地址。
```

此中断的返回值为:

```
cflags的CF位：若INT 15中断执行成功，则不置位，否则置位；
eax：534D4150h ('SMAP') ；
es:di：指向保存地址范围描述符的缓冲区,此时缓冲区内的数据已由BIOS填写完毕
ebx：下一个地址范围描述符的计数地址
ecx：返回BIOS往ES:DI处写的地址范围描述符的字节大小
ah：失败时保存出错代码
```

这样，我们通过调用INT 15h BIOS中断，递增di的值（20的倍数），让BIOS帮我们查找出一个一个的**内存布局entry**，并**放入到一个保存地址范围描述符结构的缓冲区中**，供后续的ucore进一步进行物理内存管理。这个缓冲区结构定义在memlayout.h中：

```
struct e820map {
                  int nr_map;
                  struct {
                                    long long addr;
                                    long long size;
                                    long type;
                  } map[E820MAX];
};
```

### 实现物理内存探测

物理内存探测是在bootasm.S中实现的，相关代码很短，如下所示：

```
probe_memory:
//对0x8000处的32位单元清零,即给位于0x8000处的
//struct e820map的成员变量nr_map清零
           movl $0, 0x8000
                  xorl %ebx, %ebx
//表示设置调用INT 15h BIOS中断后，BIOS返回的映射地址描述符的起始地址
                  movw $0x8004, %di
start_probe:
                  movl $0xE820, %eax // INT 15的中断调用参数
//设置地址范围描述符的大小为20字节，其大小等于struct e820map的成员变量map的大小
                  movl $20, %ecx
//设置edx为534D4150h (即4个ASCII字符“SMAP”)，这是一个约定
                  movl $SMAP, %edx
//调用int 0x15中断，要求BIOS返回一个用地址范围描述符表示的内存段信息
                  int $0x15
//如果eflags的CF位为0，则表示还有内存段需要探测
                  jnc cont
//探测有问题，结束探测
                  movw $12345, 0x8000
                  jmp finish_probe
cont:
//设置下一个BIOS返回的映射地址描述符的起始地址
                  addw $20, %di
//递增struct e820map的成员变量nr_map
                  incl 0x8000
//如果INT0x15返回的ebx为零，表示探测结束，否则继续探测
                  cmpl $0, %ebx
                  jnz start_probe
finish_probe:
```

上述代码正常执行完毕后，在0x8000地址处保存了从BIOS中获得的内存分布信息，此信息按照struct e820map的设置来进行填充。这部分信息将在bootloader启动ucore后，由ucore的page_init函数来根据struct e820map的memmap（定义了起始地址为0x8000）来完成对整个机器中的物理内存的总体管理。

## 以页为单位管理物理内存

建立对整个计算机的每一个物理页的属性用结构Page来表示，它包含了映射此物理页的虚拟页个数，描述物理页属性的flags和双向链接各个Page结构的page_link双向链表。

```
struct Page {
    int ref;        // page frame's reference counter
    uint32_t flags; // array of flags that describe the status of the page frame
    unsigned int property;// the num of free block, used in first fit pm manager
    list_entry_t page_link;// free list link
};
```

- ref表示这页被页表的引用记数

- flags表示此物理页的状态标记

  进一步查看kern/mm/memlayout.h中的定义，可以看到：

  ```
  /* Flags describing the status of a page frame */
  #define PG_reserved                 0       // the page descriptor is reserved for kernel or unusable
  #define PG_property                 1       // the member 'property' is valid
  ```

  - bit 1表示此页是否是free的，如果设置为1，表示这页是free的，可以被分配
  - 如果设置为0，表示这页已经被分配出去了，不能被再二次分配

- property用来记录某连续内存空闲块的大小（即地址连续的空闲页的个数）（存在于第一个物理表）

- page_link是便于把多个连续内存空闲块链接在一起的双向链表指针（存在于第一个物理表）

在初始情况下，也许这个物理内存的空闲物理页都是**连续**的，这样就形成了一个大的连续内存空闲块。但随着物理页的分配与释放，这个大的连续内存空闲块会分裂为一系列地址不连续的多个小连续内存空闲块，且每个连续内存空闲块内部的物理页是连续的。那么为了**有效地管理这些小连续内存空闲块**。定义了一个free_area_t数据结构：

```
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
            list_entry_t free_list;                                // the list header
            unsigned int nr_free;                                 // # of free pages in this free list
} free_area_t;
```

- list_entry结构的双向链表指针
- nr_free记录当前空闲页的个数的无符号整型变量

有了这两个数据结构，ucore就可以管理起来整个以页为单位的物理内存空间。接下来需要解决两个问题：

- 管理页级物理内存空间所需的Page结构的内存空间从哪里开始，占多大空间？ 

- 空闲内存空间的起始地址在哪里？

根据bootloader给出的内存布局信息找出**最大的物理内存地址maxpa**（定义在page_init函数中的局部变量），由于x86的起始物理内存地址为0，所以可以得知**需要管理的物理页个数**为

```
npage = maxpa / PGSIZE
```

这样，我们就可以预估出**管理页级物理内存空间**所需的Page结构的内存空间所需的内存大小为：

```
sizeof(struct Page) * npage)
```

为了简化起见，从地址0到地址pages+ sizeof(struct Page) * npage)结束的物理内存空间设定为已占用物理内存空间（起始0~640KB的空间是空闲的），地址pages+ sizeof(struct Page) * npage)以上的空间为空闲物理内存空间，这时的空闲空间起始地址为

```
uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);
```

其实实验二在内存分配和释放方面最主要的作用是建立了一个**物理内存页管理器框架**，这实际上是一个函数指针列表，定义如下：

```
struct pmm_manager {
            const char *name; //物理内存页管理器的名字
            void (*init)(void); //初始化内存管理器
            void (*init_memmap)(struct Page *base, size_t n); //初始化管理空闲内存页的数据结构
            struct Page *(*alloc_pages)(size_t n); //分配n个物理内存页
            void (*free_pages)(struct Page *base, size_t n); //释放n个物理内存页
            size_t (*nr_free_pages)(void); //返回当前剩余的空闲页数
            void (*check)(void); //用于检测分配/释放实现是否正确的辅助函数
};
```

## 物理内存页分配算法实现

### 使用双向链表实现

lab2的第一部分是完成first_fit的分配算法。原理FirstFit内存分配算法上很简单，但要在ucore中实现，需要充分了解和利用ucore已有的数据结构和相关操作、关键的一些全局变量等。

```
kern_init --> pmm_init-->page_init-->init_memmap--> pmm_manager-->init_memmap
```

1. `kern_init()`：是操作系统内核的入口函数，用来初始化整个内核和启动操作系统的各个子系统。
2. `pmm_init()`：是物理内存管理（Physical Memory Manager，PMM）子系统的初始化函数，用来初始化PMM的数据结构和其他必要的设置。
3. `page_init()`：用来初始化操作系统中的页面（page）数据结构，其中包括可用页面的列表和映射到物理内存的页面。
4. `init_memmap()`：用来初始化物理内存映射表，它把操作系统中所有可用的物理内存区域映射到相应的页面。
5. `pmm_manager()`：是物理内存管理器的方法，用来初始化物理内存管理器所维护的内存映射表。
6. `init_memmap()` 是一个函数，用于初始化初始化内存映射。

default_init_memmap需要根据page_init函数中传递过来的参数（某个连续地址的空闲块的起始页，页个数）来建立一个连续内存空闲块的双向链表。这里有一个假定page_init函数是按地址**从小到大的顺序**传来的**连续内存空闲块**的。链表头是**free_area.free_list**，链表项是Page数据结构的base->page_link。这样我们就依靠Page数据结构中的成员变量page_link形成了连续内存空闲块列表。

### 设计实现

default_init_memmap可大致实现如下：

```
default_init_memmap(struct Page *base, size_t n) {
    struct Page *p = base;
    for (; p != base + n; p ++) {
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```

以首Page为起始地址，遍历该链表修改每个页的标志位，最后将此内存的信息写入第一个页中。

如果要分配一个页，那要考虑哪些呢？

### 考虑实现default_alloc_pages函数

如果要分配一个页，那要考虑哪些呢？

firstfit需要从空闲链表头开始查找最小的地址，通过list_next找到下一个空闲块元素，通过le2page宏可以更加链表元素获得对应的Page指针p。通过p->property可以了解此空闲块的大小。如果>=n，这就找到了！如果<n，则list_next，继续查找。直到list_next== &free_list，这表示找完了一遍了。找到后，就要从新组织空闲块，然后把找到的page返回。所以default_alloc_pages可大致实现如下：

```
static struct Page *
default_alloc_pages(size_t n) {
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            list_add(&free_list, &(p->page_link));
        }
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}
```

default_free_pages函数的实现其实是default_alloc_pages的逆过程，不过需要考虑空闲块的合并问题。

- 需要注意实现时尽量多考虑一些边界情况，这样确保软件的鲁棒性
- 加一些 assert函数，在有错误出现时，能够迅速发现
- 通过le2page宏可以更加链表元素获得对应的Page指针p

## 实现分页机制

由于ucore OS是基于80386 CPU实现的，所以CPU在进入保护模式后，就直接使能了段机制，并使得ucore OS需要在段机制的基础上建立页机制。

### 段页式管理基本概念

在 ucore 中段式管理只起到了一个过渡作用，它将逻辑地址不加转换直接映射成线性地址

![img](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image006-20230422175131923-20230422175137780.png)

页式管理将线性地址分成三部分（图中的 Linear Address 的 Directory 部分、 Table 部分和 Offset 部分）。

ucore 的页式管理通过一个**二级的页表**实现。一级页表的**起始物理地址存放在 cr3 寄存器中**，这个地址必须是一个**页对齐**的地址，也就是低 12 位必须为 0。目前，ucore 用boot_cr3（mm/pmm.c）记录这个值。

> 低12位必须为0是因为在x86架构中，一个页面的大小是2的幂次方，即4KB（2的12次方），如果一个页面的起始地址不是页面大小的整数倍，那么这个页面就会跨越两个页面，这会导致内存访问出错。

### 建立段页式管理中需要考虑的关键问题

为了实现分页机制，需要建立好虚拟内存和物理内存的页映射关系，即正确建立二级页表。此过程涉及硬件细节，不同的地址映射关系组合，相对比较复杂。总体而言，我们需要思考如下问题：

- 如何在建立页表的过程中维护全局段描述符表（GDT）和页表的关系，确保ucore能够在各个时间段上都能正常寻址？
- 对于哪些物理内存空间需要建立页映射关系？
- 具体的页映射关系是什么？
- 页目录表的起始地址设置在哪里？
- 页表的起始地址设置在哪里，需要多大空间？
- 如何设置页目录表项的内容？
- 如何设置页表项的内容？

### 系统执行中地址映射的四个阶段

为了建立正确的地址映射关系，**ld在链接阶段**生成了ucore OS执行代码的虚拟地址，而bootloader与ucore OS协同工作，通过在运行时对地址映射的一系列“腾挪转移”，从计算机加电，启动段式管理机制，启动段页式管理机制，在段页式管理机制下运行这整个过程中，虚地址到物理地址的映射产生了多次变化，实现了最终的段**页式映射关系**：

```
 virt addr = linear addr = phy addr + 0xC0000000
```

在lab1中：

```
ENTRY(kern_init)

SECTIONS {
            /* Load the kernel at this address: "." means the current address */
            . = 0x100000;

            .text : {
                       *(.text .stub .text.* .gnu.linkonce.t.*)
            }
```

这意味着在lab1中通过ld工具形成的ucore的起始虚拟地址从**0x100000**开始，注意：这个地址是虚拟地址。但由于lab1中建立的**段地址映射关系为对等关系**，所以**ucore的物理地址也是0x100000**，而ucore的入口函数kern_init的起始地址。所以在lab1中虚拟地址，线性地址以及物理地址之间的映射关系如下：

```
 lab1： virt addr = linear addr = phy addr
```

在lab2中：

```
ENTRY(kern_entry)

SECTIONS {
            /* Load the kernel at this address: "." means the current address */
            . = 0xC0100000;

            .text : {
                        *(.text .stub .text.* .gnu.linkonce.t.*)
            }
```

这意味着lab2中通过ld工具形成的ucore的起始虚拟地址**从0xC0100000开始**，注意：这个地址也是虚拟地址。入**口函数为kern_entry函数**（在kern/init/entry.S中）。这与lab1有很大差别。但其实在lab1和lab2中，bootloader把ucore都放在了起始物理地址为0x100000的物理内存空间。

### 第一阶段--bootloader阶段

从bootloader的start函数（在boot/bootasm.S中）到执行ucore kernel的kern_entry函数之前，其虚拟地址，线性地址以及物理地址之间的映射关系与lab1的一样，即：

```
 lab2 stage 1： virt addr = linear addr = phy addr
```

### 第二阶段

从kern_entry函数开始，到执行enable_page函数（在kern/mm/pmm.c中）之前再次更新了段映射，还没有启动页映射机制。由于gcc编译出的**虚拟起始地址从0xC0100000开始**，ucore被bootloader放置在从**物理地址0x100000处开始**的物理内存中。

当kern_entry函数完成新的段映射关系后，且ucore在没有建立好页映射机制前，CPU按照ucore中的虚拟地址执行，能够被分段机制映射到正确的物理地址上，确保ucore运行正确。这时的虚拟地址，线性地址以及物理地址之间的映射关系为：

```
 lab2 stage 2： virt addr - 0xC0000000 = linear addr = phy addr
```

注意此时CPU在寻址时还是只采用了分段机制。最后后并使能分页映射机制（请查看lab2/kern/mm/pmm.c中的enable_paging函数），一旦执行完enable_paging函数中的**加载cr0指令**（即让CPU使能分页机制），则接下来的访问是基于段页式的映射关系了。（进入了**保护模式**？）

### 第三阶段

从enable_page函数开始，到执行gdt_init函数（在kern/mm/pmm.c中）之前，启动了页映射机制，但没有第三次更新段映射。这时的虚拟地址，线性地址以及物理地址之间的映射关系比较微妙：

```
 lab2 stage 3:  virt addr - 0xC0000000 = linear addr  = phy addr + 0xC0000000 
 # 物理地址在0~4MB之外的三者映射关系
                virt addr - 0xC0000000 = linear addr  = phy addr # 物理地址在0~4MB之内的三者映射关系
```

请注意`pmm_init`函数中的一条语句：

```
 boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)];		//KERNBASE以上的虚拟地址对应的物理地址在0-4MB之间
```

就是用来建立物理地址在0~4MB之内的三个地址间的临时映射关系`virt addr - 0xC0000000 = linear addr = phy addr`。

> 物理内存的前4MB是被预留出来用于存放一些重要的系统信息和硬件设备的显存等。操作系统在启动阶段需要对这4MB以内的物理内存进行特殊处理，以保证系统能够正常运行。
>
> 因此，操作系统在启动阶段需要特别处理0~4MB的物理内存，建立临时的页表映射，使得内核代码和数据可以被正确地加载和执行。而超过4MB的物理内存则需要建立另外的映射关系，以便用户程序可以正常地访问和使用。

### 第四阶段

从gdt_init函数开始，第三次更新了段映射，形成了新的段页式映射机制，并且取消了临时映射关系，即执行语句“boot_pgdir[0] **=** 0**;**”把boot_pgdir[0]的第一个页目录表项（0~4MB）清零来取消临时的页映射关系。这时形成了我们期望的虚拟地址，线性地址以及物理地址之间的映射关系：

```
 lab2 stage 4： virt addr = linear addr = phy addr + 0xC0000000
```

> 在操作系统启动阶段，物理内存的前4MB通常会被保留用于存储一些系统信息和硬件设备的显存等。因此，在建立临时映射关系的时候，操作系统需要将这些区域映射到虚拟地址空间的KERNBASE以上的地址，以便内核代码和数据可以被正确加载和执行。
>
> 然而，在建立新的段页式映射机制之后，这个临时映射关系就不再需要了，操作系统会将boot_pgdir[0]的第一个页目录表项（0~4MB）清零，从而取消这个映射关系。

## 链接地址/虚地址/物理地址/加载地址以及edata/end/text的含义

### 链接脚本简介

ucore kernel各个部分由组成kernel的各个.o或.a文件构成，且**各个部分在内存中地址位置**由ld工具根据kernel.ld链接脚本（linker script）来设定。ld工具**使用命令-T指定链接脚本**。链接脚本主要用于规定如何把输入文件（各个.o或.a文件）内的section放入输出文件（lab2/bin/kernel，即**ELF格式的ucore内核**）内， 并**控制输出文件内各部分在程序地址空间内的布局**。下面简单分析一下/lab2/tools/kernel.ld，来了解一下ucore内核的地址布局情况。kernel.ld的内容如下所示：

```
/* Simple linker script for the ucore kernel.
   See the GNU ld 'info' manual ("info ld") to learn the syntax. */

OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(kern_entry)

SECTIONS {
    /* Load the kernel at this address: "." means the current address */
    . = 0xC0100000;

    .text : {
        *(.text .stub .text.* .gnu.linkonce.t.*)
    }

    PROVIDE(etext = .); /* Define the 'etext' symbol to this value */

    .rodata : {
        *(.rodata .rodata.* .gnu.linkonce.r.*)
    }

    /* Include debugging information in kernel memory */
    .stab : {
        PROVIDE(__STAB_BEGIN__ = .);
        *(.stab);
        PROVIDE(__STAB_END__ = .);
        BYTE(0)     /* Force the linker to allocate space
                   for this section */
    }

    .stabstr : {
        PROVIDE(__STABSTR_BEGIN__ = .);
        *(.stabstr);
        PROVIDE(__STABSTR_END__ = .);
        BYTE(0)     /* Force the linker to allocate space
                   for this section */
    }

    /* Adjust the address for the data segment to the next page */
    . = ALIGN(0x1000);

    /* The data segment */
    .data : {
        *(.data)
    }

    PROVIDE(edata = .);

    .bss : {
        *(.bss)
    }

    PROVIDE(end = .);

    /DISCARD/ : {
        *(.eh_frame .note.GNU-stack)
    }
}
```

其实从链接脚本的内容，可以大致猜出它指定告诉链接器的各种信息：

- 内核加载地址：0xC0100000
- 入口（起始代码）地址： ENTRY(kern_entry)
- cpu机器类型：i386

其最主要的信息是告诉链接器各输入文件的各section应该怎么组合：应该从哪个地址开始放，各个section以什么顺序放，分别怎么对齐等等，最终组成输出文件的各section.

### 链接地址/加载地址/虚地址/物理地址

```
/* *
 * Virtual memory map:                                          Permissions
 *                                                              kernel/user
 *
 *     4G ------------------> +---------------------------------+
 *                            |                                 |
 *                            |         Empty Memory (*)        |
 *                            |                                 |
 *                            +---------------------------------+ 0xFB000000
 *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE
 *     VPT -----------------> +---------------------------------+ 0xFAC00000
 *                            |        Invalid Memory (*)       | --/--
 *     KERNTOP -------------> +---------------------------------+ 0xF8000000
 *                            |                                 |
 *                            |    Remapped Physical Memory     | RW/-- KMEMSIZE
 *                            |                                 |
 *     KERNBASE ------------> +---------------------------------+ 0xC0000000
 *                            |                                 |
 *                            |                                 |
 *                            |                                 |
 *                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.
 *
 * */
```

bootloader把ucore kernel加载到内存时，采用的是**加载地址（load addr）**，这是由于ucore还没有运行，即还没有启动页表映射，导致这时采用的寻址方式是段寻址方式，用的是boot loader在初始化阶段设置的段映射关系，其映射关系（可参看bootasm.S的末尾处有关段描述符表的内容）是：

```
linear addr = phy addr = virtual addr
```

简言之： OS的链接地址（link addr） 在tools/kernel.ld中**设置好了**，是一个虚地址（virtual addr）；而ucore kernel的加载地址（load addr）在boot loader中的bootmain函数中指定，是一个**物理地址**。

### edata/end/text的含义

在基于ELF执行文件格式的代码中，存在一些对代码和数据的表述，基本概念如下：

- BSS段（bss segment）：指用来存放程序中未初始化的全局变量的内存区域。BSS是英文Block Started by Symbol的简称。BSS段属于静态内存分配。
- 数据段（data segment）：指用来存放程序中已初始化的全局变量的一块内存区域。数据段属于静态内存分配。
- 代码段（code segment/text segment）：指用来存放程序执行代码的一块内存区域。这部分区域的大小在程序运行前就已经确定，并且内存区域通常属于只读, 某些架构也允许代码段为可写，即允许修改程序。在代码段中，也有可能包含一些只读的常数变量，例如字符串常量等。

## 自映射机制（太牛了）

如果我们这时需要按虚拟地址的地址顺序显示整个页目录表和页表的内容，则要查找页目录表的页目录表项内容，根据页目录表项内容找到页表的物理地址，再转换成对应的虚地址，然后访问页表的虚地址，搜索整个页表的每个页目录项。这样过程比较繁琐。

我们需要有一个简洁的方法来实现这个查找。ucore做了一个很巧妙的地址自映射设计，把页目录表和页表放在一个连续的4MB虚拟地址空间中，并设置页目录表自身的虚地址<-->物理地址映射关系。这样在已知页目录表起始虚地址的情况下，通过连续扫描这特定的4MB虚拟地址空间，就很容易访问每个页目录表项和页表项内容。

具体而言，ucore是这样设计的，首先设置了一个常量（memlayout.h）：

VPT=0xFAC00000， 这个地址的二进制表示为：

1111 1010 1100 0000 0000 0000 0000 0000

高10位为1111 1010 11，即10进制的1003，中间10位为0，低12位也为0。在pmm.c中有两个全局初始化变量

```
pte_t * const vpt = (pte_t *)VPT;
pde_t * const vpd = (pde_t *)PGADDR(PDX(VPT), PDX(VPT), 0);
```

并在pmm_init函数执行了如下语句：

boot_pgdir[PDX(VPT)] = PADDR(boot_pgdir) | PTE_P | PTE_W;

```
（）1111 1010 11｜11 1110 1011 ｜ 0000 0000 0000

（） 	页目录表   ｜11 1110 1011  ｜ 0000 0000 0000

									页目录表项		 ｜    12位的寻址（4k）
```

这些变量和语句有何特殊含义呢？其实vpd变量的值就是页目录表的起始虚地址**0xFAFEB000**，且它的高10位和中10位是相等的，都是10进制的1003。当执行了上述语句，就确保了vpd变量的值就是页目录表的起始虚地址，且vpt是页目录表中第一个目录表项指向的页表的起始虚地址。此时描述内核虚拟空间的页目录表的虚地址为0xFAFEB000，大小为4KB。页表的理论连续虚拟地址空间**0xFAC00000~0xFB000000**，大小为4MB。因为这个连续地址空间的大小为4MB，可有1M个PTE，即**可映射4GB的地址空间**。

> 为什么是0xFAC00000~0xFB000000呢？
>
> 页目录表的起始虚地址为0xFAFEB000，页表的起始虚地址为0xFAC00000，因此0xFAC00000这个虚拟地址的高10位对应页目录表中的索引为1003，中10位对应页表中的索引为0，低12位为0。因此，在页目录表中找到索引为1003的目录表项，其中存储的指针为0xFAC00000，指向页表的起始虚地址。而在该页表中，索引为0的页表项中存储的物理地址为0x00000000，即将0xFAC00000这个虚拟地址映射到了物理地址0x00000000。类似地，该页表中的其它页表项也会将相应的虚拟地址映射到对应的物理地址。因此，这个4MB的连续地址空间可以被映射到物理地址0x00000000~0x04000000，即可以被寻址。

# 练习

## 练习1 实现 first-fit 连续物理内存分配算法

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default_pmm.c中的default_init，default_init_memmap，default_alloc_pages， default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。

## 解答

###  Prepare

```
be familiar to the struct list in list.h.
You should know howto USE: 
list_init, list_add(list_add_after), list_add_before, list_del, list_next, list_prev
you can find some MACRO: le2page (in memlayout.h)
```

### default_pmm.c

```
* (2) default_init: you can reuse the  demo default_init fun to init the free_list and set nr_free to 0.
*     free_list is used to record the free mem blocks. nr_free is the total number for free mem blocks.
```

调用list_init函数建立物理内存页表，页表数为0

```
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

### default_init_memmap

```
* (3) default_init_memmap:  CALL GRAPH: kern_init --> pmm_init-->page_init-->init_memmap--> pmm_manager->init_memmap
 *              This fun is used to init a free block (with parameter: addr_base, page_number).
 *              First you should init each page (in memlayout.h) in this free block, include:
 *                  p->flags should be set bit PG_property (means this page is valid. In pmm_init fun (in pmm.c),
 *                  the bit PG_reserved is setted in p->flags)
 *                  if this page  is free and is not the first page of free block, p->property should be set to 0.
 *                  if this page  is free and is the first page of free block, p->property should be set to total num of block.
 *                  p->ref should be 0, because now p is free and no reference.
 *                  We can use p->page_link to link this page to free_list, (such as: list_add_before(&free_list, &(p->page_link)); )
 *              Finally, we should sum the number of free mem block: nr_free+=n
```

此函数是用来初始化内存映射表的，结合我们了解的前置知识可以知道该函数的设计有两个要点：

- 该映射表中的元素是Page，我们需要注意初始化Page中的几个参数。

  ```
  struct Page {
      int ref;        // page frame's reference counter
      uint32_t flags; // array of flags that describe the status of the page frame
      unsigned int property;// the num of free block, used in first fit pm manager
      list_entry_t page_link;// free list link
  };
  ```

  - ref表示这页被页表的引用记数

  - flags表示此物理页的状态标记

    进一步查看kern/mm/memlayout.h中的定义，可以看到：

    ```
    /* Flags describing the status of a page frame */
    #define PG_reserved                 0       // the page descriptor is reserved for kernel or unusable
    #define PG_property                 1       // the member 'property' is valid
    ```

    - bit 1表示此页是否是free的，如果设置为1，表示这页是free的，可以被分配
    - 如果设置为0，表示这页已经被分配出去了，不能被再二次分配

- 注意初始化第一个Page中独占的两个全局参数：

  - property用来记录某连续内存空闲块的大小（即地址连续的空闲页的个数）（存在于第一个物理表）
  - page_link是便于把多个连续内存空闲块链接在一起的双向链表指针（存在于第一个物理表）

故我们可以写出以下的函数：

```
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```

其中根据实验指导书的设计要求，对多种情况做出了判断，便于我们测试函数模块。

在该函数中我们接受Page的基址，以及一个需要分配的页数，通过循环的方式进行遍历，调用`PageReserved(p)`查看该块 是非 被占用，若 没被占用则使用`p->flags = p->property = 0`对每个块进行初始化标记为已被使用，再使用`set_page_ref(p, 0)`添加引用计数。

完成对要求页数的初始化后，对第一个页进行设置：设置该页表中空闲块的数量`base->property = n;`以及将该页表加入到页目录中，最终完成了一个页表的初始化。

### default_alloc_pages

```
default_alloc_pages: search find a first free block (block size >=n) in free list and reszie the free block, return the addr
 *              of malloced block.
 *              (4.1) So you should search freelist like this:
 *                       list_entry_t le = &free_list;
 *                       while((le=list_next(le)) != &free_list) {
 *                       ....
 *                 (4.1.1) In while loop, get the struct page and check the p->property (record the num of free block) >=n?
 *                       struct Page *p = le2page(le, page_link);
 *                       if(p->property >= n){ ...
 *                 (4.1.2) If we find this p, then it' means we find a free block(block size >=n), and the first n pages can be malloced.
 *                     Some flag bits of this page should be setted: PG_reserved =1, PG_property =0
 *                     unlink the pages from free_list
 *                     (4.1.2.1) If (p->property >n), we should re-caluclate number of the the rest of this free block,
 *                           (such as: le2page(le,page_link))->property = p->property - n;)
 *                 (4.1.3)  re-caluclate nr_free (number of the the rest of all free block)
 *                 (4.1.4)  return p
 *                 (4.2) If we can not find a free block (block size >=n), then return NULL
```

其实这些函数都在实验指导书中给出了具体的实现原理了

```
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {		//大于可用空闲块
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {		//还未找到链表尾
        struct Page *p = le2page(le, page_link);		//获取当前指针
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;			//减去用掉的链表数
            SetPageProperty(p);
            ## list_add(&free_list, &(p->page_link));	//错误！
           +list_add_before(&free_list, &(p->page_link));
    }
    	 +list_del(&(page->page_link));		//删除tmp指针
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}
```

为什么用before呢？我们知道题目要求我们的页表项的地址要从小到大排列，由双向链表的性质可以得出以下的图：

![IMG_4C134E314195-1](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/IMG_4C134E314195-1.jpeg)

整个函数的过程用图可以表示为：

![IMG_A98781407AB6-1](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/IMG_A98781407AB6-1.jpeg)

### default_free_pages

```
* (5) default_free_pages: relink the pages into  free list, maybe merge small free blocks into big free blocks.
 *               (5.1) according the base addr of withdrawed blocks, search free list, find the correct position
 *                     (from low to high addr), and insert the pages. (may use list_next, le2page, list_add_before)
 *               (5.2) reset the fields of pages, such as p->ref, p->flags (PageProperty)
 *               (5.3) try to merge low addr or high addr blocks. Notice: should change some pages's p->property correctly.
 */
```

就如实验指导书中所说：default_free_pages函数的实现其实是default_alloc_pages的逆过程，不过需要考虑空闲块的合并问题。

```
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));		//页面还未被使用&&页引用为0
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;				//将第一个空闲页面的属性设置为要释放的页面数量
    SetPageProperty(base);
    list_entry_t *le = list_next(&free_list);			
    while (le != &free_list) {			//遍历页表
        p = le2page(le, page_link);
        le = list_next(le);
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        else if (p + p->property == base) {		//合并页表的过程
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
    list_add(&free_list, &(base->page_link));			//错误！
    le = &free_list;
		while ((le = list_next(le)) != &free_list){
    	p = le2page(le, page_link);
    	if (base + base->property <= p) {
        	assert(base + base->property != p);
        	break;
    		}
			}
		list_add_before(le, &(base->page_link));
}
```

该函数会将第一个空闲页面的属性设置为要释放的页面数量，并将其标记为可用空闲页面。然后，它遍历所有的空闲页面，将与要释放页面相邻的页面合并成一个大页面，并从空闲页面列表中删除旧的页面。

> 引用计数是用于跟踪页面是否被占用的技术，当页面被占用时，引用计数会增加。而空闲页数量是指当前系统中可用的未被占用的物理页面的数量。
>
> 在引用计数的实现中，当一个进程需要访问某个页面时，它会增加该页面的引用计数，表示该页面正在被使用。当该进程完成使用后，它会将该页面的引用计数减少，如果该页面的引用计数为0，表示该页面可以被释放并添加到空闲页列表中，从而增加空闲页数量。



### 可能的优化

在使用 first-fit 算法分配内存后，可能会留下一些非常小的空闲块。为了避免这种情况，可以在分配内存时，如果剩余空闲块的大小小于一个阈值，可以将该块合并到前一个或后一个空闲块中，从而减少碎片化。

## 练习2：实现寻找虚拟地址对应的页表项

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全get_pte函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解get_pte函数中的注释。get_pte函数的调用关系图如下所示：

![img](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image001.png)

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

- 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。
- 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

### 解答

```
     * Some Useful MACROs and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   PDX(la) = the index of page directory entry of VIRTUAL ADDRESS la.
     *   KADDR(pa) : takes a physical address and returns the corresponding kernel virtual address.
     *   set_page_ref(page,1) : means the page be referenced by one time
     *   page2pa(page): get the physical address of memory which this (struct Page *) page  manages
     *   struct Page * alloc_page() : allocation a page
     *   memset(void *s, char c, size_t n) : sets the first n bytes of the memory area pointed by s
     *                                       to the specified value c.
     * DEFINEs:
     *   PTE_P           0x001                   // page table/directory entry flags bit : Present
     *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
     *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
     */
```

```
		pde_t *pdep = &pgdir[PDX(la)];

    // 如果该项不可用
    if (!(*pdep & PTE_P))
    {
        struct Page *page;

        // 如果分配页面失败或者不允许分配，则返回NULL
        if (!create || (page = alloc_page()) == NULL)
            return NULL;

        // 否则进行分配
        // 设置该物理页面的引用次数为1
        set_page_ref(page, 1);

        // 获取当前物理页面所管理的物理地址
        uintptr_t pa = page2pa(page);

        // 清空该物理页面的数据（需要注意使用的是虚拟地址）
        // KADDR(pa) : 获取物理地址pa对应的内核虚拟地址
        memset(KADDR(pa), 0, PGSIZE);

        // 将新分配的页表设置权限后填入页目录项中
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
     		return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
```

### 解答2

如果 ucore 执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

**答：**

- 将触发`页访问异常`的**虚地址**保存到`cr2`寄存器中
- 设置**错误代码**，触发 14 号中断，也就是`缺页错误`
- 抛出`Page Fault`异常，将外存的数据读到内存中（应该是读存在硬盘上的虚拟内存分页文件）
- 进行`上下文切换`，退出中断，返回到中断前的状态

## 练习3

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构 Page 做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。补全 page_remove_pte 函数。

## 解答

```
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep)
{
    /* LAB2 EXERCISE 3: YOUR CODE
    *
    * Please check if ptep is valid, and tlb must be manually updated if mapping is updated
    *
    * Maybe you want help comment, BELOW comments can help you finish the code
    *
    * Some Useful MACROs and DEFINEs, you can use them in below implementation.
    * MACROs or Functions:
    *   struct Page *page pte2page(*ptep): get the according page from the value of a ptep
    *   free_page : free a page
    *   page_ref_dec(page) : decrease page->ref. NOTICE: ff page->ref == 0 , then this page should be free.
    *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : Invalidate a TLB entry, but only if the page tables being
    *                        edited are the ones currently in use by the processor.
    * DEFINEs:
    *   PTE_P           0x001                   // page table/directory entry flags bit : Present
    */

    // 如果传入的页表项是可用的
    if (*ptep & PTE_P)
    {
        // 获取该页表项所对应的地址
        struct Page *page = pte2page(*ptep);

        // 如果该页的引用次数在减1后为0，表明仅被我们当前引用了1次，可以释放
        if (page_ref_dec(page) == 0)
            // 释放当前页
            free_page(page);

        // 二级页表表项清零
        *ptep = 0;

        // 刷新TLB中该页的缓存使其无效（当且仅当正在编辑的页表是处理器当前正在使用的页表时）
        tlb_invalidate(pgdir, la);
    }
}
```

>         TLB(Translation Lookaside Buffer)转换检测缓冲区是一个内存管理单元,是用于改进虚拟地址到物理地址转换速度的缓存。
>         其中每一行都保存着一个由单个PTE(Page Table Entry,页表项)组成的块。
>         如果没有TLB，则每次取数据都需要两次访问内存。

![](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230423150431884.png)

### 解答2

数据结构 Page 的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是什么？

当页目录项或页表项有效时，`Page`数组中的项与页目录项或页表项**存在对应关系**。实际上，**每个页目录项记录一个页表的信息，每个页表项记录一个物理页的信息。**页目录表中存放着数个页表项，这些页表项中存放了某个二级页表所在物理页的信息，包括该二级页表的**物理地址**，但使用**线性地址**的头部`PDX(Page Directory Index)`来索引页目录表。而页表（二级页表）与页目录（一级页表）具有类似的特性，页表中的页表项指向所管理的物理页的**物理地址**，使用**线性地址**的中部`PTX(Page Table Index)`来索引页表。页目录项保存的物理页面地址（即某个页表），以及页表项保存的物理页面地址都对应于`Page`数组中的某一页。

### 解答3

如果希望虚拟地址与物理地址相等，则需要如何修改 lab2 才能完成此事？

- 先将`tools/kernel.ld`中的`第10行`的`内核加载地址`从`0xC0100000`修改为`0x00100000`

```none
// 代码第10行 修改前：
. = 0xC0100000;

// 代码第10行 修改后:
. = 0x00100000;
```

- 再将`kern/mm/memlayout.h`中的`第56行`的`内核偏移地址`从`0xC0000000`修改为`0x00000000`

```c
// 代码第56行 修改前:
#define KERNBASE            0xC0000000

// 代码第56行 修改后:
#define KERNBASE            0x00000000
```

- **关闭页机制**，这一步是为了保证运行时不报错。将`kern/init/entry.S`中`开启页机制`的那一段代码删除即可。否则在分页模式，取消掉 `boot_pgdir[0]` 的页表下 `kern_init` 会被置于一个根本无法访问到的地址。

```asm
# 代码第14-17行 修改前：
movl %cr0, %eax
orl $(CR0_PE | CR0_PG | CR0_AM | CR0_WP | CR0_NE | CR0_TS | CR0_EM | CR0_MP), %eax
andl $~(CR0_TS | CR0_EM), %eax
movl %eax, %cr0

# 代码第14-17行 修改后：
# 直接删除第14-17行代码
```

​	
