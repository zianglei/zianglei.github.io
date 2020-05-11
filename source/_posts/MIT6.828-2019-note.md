---
title: MIT6.828 2019学习笔记
date: 2020/05/09 20:24:00
tags:
- 操作系统
categories: 
- 笔记
toc: true
---

## Lec1 系统调用

- fork和exec分开的原因

  分开之后可以在exec执行之前进行一些操作，例如关闭FD等，从而实现重定向、管道等功能。

## Lec2 动态内存分配

程序内存布局

```
  [text |  data | heap ->  ...    <- stack]
  0                                       top of address space
```

kcalloc的实现：每次调用会按照地址排序分配地址最小、大小为PG_SIZE的区块

在free的时候为了知道这个区块多大，需要在分配的空间前记录该区块的大小信息和下一个区块的位置。

malloc函数的目标：

- 快速分配和释放内存
- 小的内存开销
- 良好的内存位置
- 好的扩展性（扩展到多线程/多核）

讲义提供的[malloc.c](https://pdos.csail.mit.edu/6.828/2019/lec/malloc.c)展现了三种malloc和free实现的方式：

- K&R malloc/free

  这是在C语言经典教材K&R实现的内存分配和释放方式，其策略为first-fit，即分配第一个满足需求的内存区块，在free的时候合并区块。

  malloc 流程

  ```flow
  F=>operation: 计算所需要的块大小，一块是sizeof(Header)大小
  计算方式为(bytes + sizeof(Header) - 1)/sizeof(Header) + 1
  A=>condition: freep == NULL?
  B=>operation: freep = &base, p = &base, 
  base.size =0, base.ptr = &base, 
  prevp = &base;
  C=>condition: p = prevp->ptr, 
  判断是否有区块满足需要
  D=>operation: 该区块大小正好满足需求
  从列表中删除该区块；
  若大于需求，则从尾部分配空间
  E=>operation: freep == p, 
  没有区块满足需求，调用sbrk申请空间
  并将新空间加入分配列表
  F->A
  A(yes)->B
  A(no)->C
  B->C
  C(yes)->D
  C(no)->E
  ```

  free 流程

  ```flow
  A=>operation: 找到要释放的区块的Header
  B=>operation: 从freep开始，判断区块的位置
  指针p须指向一个区块，保证释放的区块在p指向的区块地址范围内，
  或者p指向空闲区块的最后一个，
  且要释放的区块在p的后面或者第一个空闲区块的前面
  C=>operation: 判断区块是否能够与p指向的区块或p->ptr指向的区块合并
  如果不能合并，就加入到块列表中
  D=>operation: 将freep指向p，即释放区块的上一个区块
  A->B->C->D
  ```

  效率和开销：申请10000个16bytes内存，消耗67008usec，比较慢，并且其开销与实际申请内存有关，例如如果一直申请16bytes，那么申请内存开销就为50%

- Region allocator

  这种分配策略是一次分配一大块内存（在程序中是100*4096），然后每次malloc时，就从这一大块内存中取一小块。最后释放的时候整个Region全部释放。这种分配方法要求应用程序需要的内存不能超过Region的大小。

  - 效率和开销

    效率最高，申请10000个16bytes内存，消耗116us，很小的开销（只用维护一个结构体记录Region的首尾和现在的位置即可）。

- Buddy allocator

  参考[Wiki](https://en.wikipedia.org/wiki/Buddy_memory_allocation)

  - 实现原理：将空间分成不同大小的区块，每种区块的大小都是2的倍数

    设区块种类为n，则第k种区块的大小为$2^{k}$，个数为$2^{n-1-k}$个

  - 初始化的时候会创建一个bd_size数组，长度为n，分别存放不同种类的区块的分配情况，使用bit vector表示区块是否被分配以及是否被分割，并用一个双向循环链表链接各个free的区块。各个区块的链表刚开始都指向同一大块区域。

  - malloc

    - 根据分配的空间大小判断其属于哪种区块，然后判断链表是否为空，不为空则代表有free区块可以用，否则向更大的区块种类中寻找，直到找到free区块，或者分配失败。
    - 找到free区块之后，先在表示区块被分配的bit vector中标记此区块被分配，然后一直split该区块，直到达到满足需要分配的空间大小的最小等级。
    - 返回区块

  - free

    - 根据地址判断对应区块对应的等级（通过判断该区块在等级中是否标记为被分割）
    - 清除分配标志，找到buddy所在区块的索引和地址，如果buddy已经free了，那么就将buddy从链表中删除（因为要合并），然后将高一等级中merge后的区块的split标志位删除掉，表示合并，然后将合并后的区块放入此等级的链表中，表示可用。

  - 效率和开销
    - 申请10000个16bytes内存，消耗2071us，开销就是维护不同等级的链表。比K&R实现的malloc效率高的原因是不用遍历整个链表找到合适的free区块，只用在当前等级的链表pop一个区块。

- System allocator

  - 调用系统提供的malloc/free申请10000个16bytes，消耗1270us，还是系统实现的更强，但也更复杂，有机会再具体分析。