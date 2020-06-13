---
title: MIT6.828 2019学习笔记
date: 2020/05/09 20:24:00
tags:
- 操作系统
categories: 
- 笔记
toc: true
mathjax: true
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

## Lec3 GDB基础操作

讲的比较浅显，略过

## Lec4 系统调用进入和退出

对应的阅读材料是xv6文档的第二章，似乎相比于网上存在的中文翻译版本更加新一点（官网给的是2019/10/27的版本），有时间翻译一下。

从文档中提取出来的一些笔记：

- 操作系统主要实现的是复用（multiplexing），隔离（isolation）和交互（interaction）

  - 复用即是操作系统将底层的物理资源进行抽象，然后给每个用户程序提供相应的功能，避免用户程序直接对底层物理资源操作。这样每个用户程序以为自己占用了所有的物理资源。并且操作系统在各个用户程序之间进行资源的调度

  - 为了隔离用户程序和操作系统，CPU支持硬件级别的隔离，即提供多种操作模式，每种模式有自己的操作权限和等级

    - RISC-V提供三种模式：User mode，Supervisor mode and Machine mode
    - 如果用户程序需要进行一些权限操作（例如读取文件等），RISC-V提供了ecall指令用于从User mode 切换到 Supervisor mode，然后进入到内核中相对应的入口点

    - 关于内核的组织结构，根据内核运行在Supervisor mode的功能多少，分为宏内核（monolithic kernel）和微内核（microkernel），宏内核将内核所有的功能都放在Supervisor mode，这样内核之间的交互就会非常方便，但是会导致内核各功能之间的接口非常复杂，当某一个部分出现bug的时候，整个内核就会崩溃。

      微内核只将给关键的部分放在Supervisor mode里运行，其他的都放在User mode。这样实际的内核部分就会非常精简，但是内核其他功能之间的交互都要通过内核空间的部分进行通信，效率是一个很大的问题。

    - xv6中的最小隔离单位为进程，每个进程有内核栈和用户栈两个部分，这样保证即使用户栈被破坏了内核部分也能正常运行，不影响内核的正常运行。

      - 每个进程都有自己的独立的地址空间（虚拟），这样每个进场就像独自占有整个内存一样，RISC-V使用页表映射虚拟地址和实际的物理地址

        <img src="http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200613152403_image-20200613152106120.png" alt="image-20200613152106120" style="zoom:50%;" />

      - 在xv6中，内核在trapframe中事先放置了以下内容

        ```
        +---------------------+
        |                     |
        |  32 user registers  |
        |                     |
        +---------------------+ <-+ $stvec + 40
        |   kernel hart id    |
        +---------------------+
        |        epc          |
        +---------------------+
        |  usertrap function  |
        +---------------------+
        | kernel stack pointer|
        +---------------------+
        |  (satp) page table  |
        +---------------------+ <-+ $stvec
        ```

        - 内核页表
        - 内核栈指针
        - usertrap()函数地址
        - 预留空间存储用户$PC寄存器值
        - 内核ID
        - **预留空间存储用户的32个寄存器值**

  - 当进程需要与操作系统交互的时候，进程调用系统调用进入到内核，当内核执行完毕后，会使用`sret`指令进行退出内核。

- 从启动到第一个进程创建的过程

  - bootloader将内核拷贝到内存中，然后执行xv6的_entry，设置调用栈，然后进入machine mode
  - 在machine mode，xv6设置一些参数，然后启动时间中断，进入supervisor mode，执行main函数
  - min函数创建了第一个进程执行initcode.S，该进程执行exec替换内存和寄存器，并执行/init，init创建了一个新的终端，并启动shell，至此启动完成

- 跟踪调用系统调用write()的全过程
  - 首先向a7(x17)寄存器写入系统调用编号，SYS_write编号为17，然后在a0，a1，a2...等寄存器写入函数参数
  - 然后调用ecall进入内核
    - ecall执行了三件事情：
      - 跳转到$stvec寄存器地址。\$stvec寄存器在文档中表明为Supervisor trap handler base address，即Supervisor模式下的陷阱处理函数的基地址。  
      - 保存\$pc到$sepc，用于从内核空间返回时使用
      - 切换到supervisor mode
    - 由于现在除了PC其余的都是用户空间的内容，因此还无法执行内核中的函数。需要做的事情
      - 将地址空间切换到内核中
      - 保存所有寄存器的值
        - 如果要将所有寄存器的值存储到trapframe中，需要使用store指令，但是store指令同样需要一个寄存器来表示要存储的目标地址，该用哪个寄存器呢？RISC-V提供了一个`sscratch`寄存器，该寄存器事先被内核设置为trapframe的地址。在store指令之前可以使用`csrrw`指令交换`sscratch`与用户寄存器的值，从而获得trapframe的地址。
        - 将所有寄存器依次保存到trapframe
      - 将\$sp指向内核栈的顶部，即用ld命令从kernel stack pointer位置读取地址到\$sp
      - 读取usertrap函数地址到t0，然后将page table位置的地址存放到satp寄存器，就可以切换到内核的地址空间，进而可以执行内核中的函数。最后跳转到usertrap函数位置
  - 进入内核之后，保存$sepc到trapframe, 读取scause寄存器判断进入trap的原因，跳转到syscall函数处理系统调用
  - 读取保存在trapframe中的用户寄存器，根据系统调用编号调用对应的函数执行系统调用
  - 系统调用完成后，需要从内核返回用户空间
    - RISCV提供了sret指令，但是需要先设置相关寄存器的值
      - sepc：user program counter
      - sstatus：the "previous mode" bit
    - 同时在返回之前我们需要为下一次调用系统调用做准备
      - 设置stvec（用于ecall跳转）
      - 设置内核satp（页表地址）
      - 设置stack pointer
      - 设置usertrap函数地址
      - 设置hartid
    - 最后恢复用户寄存器值，切换到用户空间页表

可以使用gdb在write()函数处打断点（断点地址可以在sh.asm处查看），然后一步一步跟踪来检查相关寄存器的变化。