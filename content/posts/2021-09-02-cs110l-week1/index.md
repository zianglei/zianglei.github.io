---
title: "CS110L Week 1"
date: 2021-09-02T12:29:46+08:00
tags:
- Rust
- CS110L
categories: 
- Course 
toc: true
description: 玩转 Valgrind 和 LLVM Sanitizers
---

CS110L 主要关注的是系统编程中的安全问题，使用 Rust 完成一系列实验和实现两个小项目，适合用来熟悉 Rust 的使用

➡️ Week 1 [作业网页](https://reberhardt.com/cs110l/spring-2021/assignments/week-1-exercises/) 

## 程序分析方法

### 静态分析

静态分析通过分析源代码，找到存在问题的部门。但是只能处理比较简单的问题，由于存在“停机问题”，并不能找到所有的问题。

#### 停机问题（halting problem）

停机：某段程序在接受输入后，能在有限长的时间内运行出结果。停机问题表示我们无法通过一个工具判断任意程序的停机问题，总会有一些程序无法判断是否停机。

证明：使用反证法，假设 H 能够判断所有输入是否停机，那么我们构造程序 K，使得 K 输出与 H 相反的结果。那么我们将 K 做为 K 的输入，那么当 K 返回不停机时，证明 H 表明 K 停机；如果 K 返回停机，证明 H 表明 K 不停机，相互矛盾。

这种证明与“理发师悖论”相同。所以我们无法只通过静态分析判断所有的程序代码是否会出现问题。

#### Linter

Linter 使用类似于文本查找的途径寻找常见的错误，常见的 linter 工具有 clang-tidy，通常只用于代码风格的保证。

#### 数据流分析

通过分析程序的执行数据流来检测例如变量未初始化，函数返回未释放内存等错误。

数据流分析会跟随每一个分支，即使这个分支在一些情况下是不可能执行的。而且数据流分析的致命弱点是会产生假阳性错误报告，因此数据流分析要保证有好的信噪比。

#### 前置条件与后置条件

相比与分析函数所使用的所有上下文（不太可能），我们可以单独验证某个函数的前置条件和后置条件（也即是函数执行之后会达到什么效果，返回什么结果），如果所有函数的前置条件和后置条件都满足，就可以将这些函数全部放在一起，只要保证相互条件都满足即可。

在 C 中，编译器并不清楚一个函数的前置条件或者后置条件，也并没有提供语法机制指定函数的条件。如果编译器知道了这些信息，就可以通过这些信息判断代码中函数是否满足这些条件。

### 动态分析

实际运行程序，根据运行结果和现象判断是否出现问题。动态分析需要使用大量不同的测试输入来决定程序是否有问题，但是也很难用测试输入覆盖到所有会出现问题的情况。

#### Valgrind

Valgrind 是动态分析常用工具，其原理是提供一个虚拟化的 CPU（称为 core），然后通过不同的 tool（比如 memcheck）在原始二进制机器吗中插入自身的代码，最终在 core 中执行，采取 JIT 技术动态地将机器记录堆读写的过程，缺点：无法检测栈类型的内存溢出，因为栈本身就是一大块内存，并不需要进行分配操作，且执行效率较低。

#### LLVM

使用多种 sanitizers ，通过编译，在代码中打桩，分析源代码中的内存读写行为来找到各种问题。包括 AddressSanitizer，LeakSanitizer，MemorySanitizer，UndefinedBehaviorSanitizer，ThreadSanitizer，可以分析出非法地址、内存泄漏等行为，在编译的时候通过 -fsanitize 指定要使用的 sanitizer。

例如开启 AddressSanitizer, LeakSanitizer 和 UndefinedBehaviorSanitizer，可以使用 `-fsanitizer=address,leak,undefined` 开启。

但是在很多情况下程序的输入都发生不了访问非法内存/内存泄漏的情况，就无法触发 Sanitizer

##### Address Sanitizer

Address sanitizer 的实现原理可以参考[这篇](https://zhuanlan.zhihu.com/p/37515148)知乎文章，概括来说就是将内存分为两个部分，主内存和影子内存，1 byte 影子内存对应 8 bytes 主内存。影子内存对应主内存不同的情况

- 8 bytes 数据可读写，影子内存 value 为 0
- 8 bytes 数据不可读写，影子内存 value 为负数
- 前 k bytes 可读写，后（8-k）bytes 内存不可读写，影子内存 value 为 k

通过在代码中对内存读写的语句前后进行影子内存检查，即可判断内存是否可读写。

主内存到影子内存的映射使用直接映射的方式，shadow = (Mem >> 3) + 0x7fff8000

#### Fuzzing 模糊测试 

由于动态分析需要依赖大量的测试输入，为了产生不同的输入，我们可以使用 Fuzzing，即模糊测试。

其通过输入的种子生成测试用例，然后通过半随机突变（ semi-random mutation）和随机突变（random mutation）产生更多的自动化测试数据。

常见的 Fuzzing 工具有 AFL 和 libfuzzer

![image-20210902145955669](fuzzing.png)

[此处](http://rk700.github.io/2017/12/28/afl-internals/)对 AFL 原理进行了简单的分析

Clang 自带 libfuzzer，其实现原理也是插桩，通过调用代码中的特定函数，传递产生的半随机字符串。我们需要在代码中插入特定的函数用于接收 libfuzzer 产生的内容，然后调用自己的函数。

```c
int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  // Do something with `data`
  
  return 0;
}
```

在编译时在`-fsanitizer`选项中加入`fuzzer`选项即可。运行之后如果发生内存泄漏，则会将产生内存泄漏的输入字符串写入到临时文件中。



