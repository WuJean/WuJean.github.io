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

#### 解答

- ucore
  - kernel
  - bootblock

##### ucore

由题目可知我们需要分析ucore.img是如何生成的，我们首先要找到Makefile中有关于ucore生成的部分。

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



##### kernel
