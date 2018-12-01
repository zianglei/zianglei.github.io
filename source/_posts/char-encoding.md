---
title: 基础知识 | 字符编码
date: 2018-12-01 00:09:45
tags: 
  - 总结
categories:
  - 码农学习日常
toc: true
---

编程中经常会涉及到字符编码的知识，容易混淆，在这里总结一下。

<!--more-->

## 编码的作用

计算机处理都是使用二进制编码进行处理的，所以在处理字符的时候需要将字符进行**编码**，映射成二进制序列，然后才能被计算机处理和传输。下面介绍一些常用的字符编码。

## ASCII

ASCII（美国信息交换标准码）是我们接触最多的字符编码。它是随着计算机诞生而产生的，所有只用十进制的**0-128**表示一些字符，其中也包括大小写字母。这种编码一直用到现在，之后新产生的编码都兼容ASCII码。

打印出所有 ASCII 字符的C程序：

```c
#include <stdio.h>

int main()
{
    int i = 0;
    for(i = 0; i < 128; i++)
    {
        printf("%d. %c\n", i, i);
    }
    return 0;
}
```

在打印出来之后，有些字符会无法显示，是因为 ASCII 包含了一些控制符等无法显示的字符，例如退格等。ASCII 包含的所有字符可以查看[维基百科](https://en.wikipedia.org/wiki/ASCII)。

## Unicode

随着计算机的发展和普及，ASCII 编码已经不能满足表示所有字符的需求，Unicode 这时候就诞生了，其作用就是用一套编码来表示所有文字，使计算机能够支持多语言环境。Unicode 说是编码其实是一种字符集，包含了所有的字符。

Unicode 一共定义了1114112个码位（code point)（从0x000000到0x10FFFF），表示方法为用“U+”或者"\u"后跟一个十六进制数。这么多字符基本上可以包含世界上所有的字符了。但是它并没有规定计算机如何存储这些字符，并且还存在很多问题，比如：

> 这里就有两个严重的问题，第一个问题是，如何才能区别 Unicode 和 ASCII ？计算机怎么知道三个字节表示一个符号，而不是分别表示三个符号呢？第二个问题是，我们已经知道，英文字母只用一个字节表示就够了，如果 Unicode 统一规定，每个符号用三个或四个字节表示，那么每个英文字母前都必然有二到三个字节是`0`，这对于存储来说是极大的浪费，文本文件的大小会因此大出二三倍，这是无法接受的。

因此Unicode 定义了两种映射方式，其中一种叫做 **Unicode Transformation Format**，即 UTF，衍生出来的编码方式就是我们常见的 UTF-8、UTF-16、UTF-32 等等，这些编码名称里面的数字代表用多少位表示 Unicode 中的码位。

### 大小端模式

关于 Unicode 编码的直接存储，有两种模式，一种是小端模式（Little Endian) ，一种是大端模式（Big Endian），例如汉字`李`的 Unicode 码是`U+673E`，使用小端模式（字节的高位存储在内存的高位）存储为`3E 67`，使用大端模式（字节的高位存储在内存的低位）为`67 3E`。

那如何知道文件是使用大端模式还是小端模式呢，Unicode 规定每个文件的第一个字符用来表示编码顺序，如果是 `FE FF`，表示使用大端模式，如果是`FF FE`，表示使用小端模式。

[维基百科链接](https://en.wikipedia.org/wiki/Unicode)

## UTF-8

上面提到，UTF-8 是用8位（即一个字节）表示 Unicode 的码位，但是很明显8位是不够的，所以 UTF-8 是一种变长编码（最长为4个字节），编码规则如下：

| 字节数 | 第一个码点 | 最后一个码点 |  字节1   |  字节2   |  字节3   |  字节4   |
| :----: | :--------: | :----------: | :------: | :------: | :------: | :------: |
|   1    |   U+0000   |    U+007F    | 0xxxxxxx |          |          |          |
|   2    |   U+0080   |    U+07FF    | 110xxxxx | 10xxxxxx |          |          |
|   3    |   U+0800   |    U+FFFF    | 1110xxxx | 10xxxxxx | 10xxxxxx |          |
|   4    |  U+10000   |   U+1FFFFF   | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |

可以看到 UTF-8 将 Unicode 的所有码点划分为了四块，并用不同的字节长度来表示。其规定：当字节的开头是0时，表示U+0000到U+007F的字符，即 ASCII 码对应的字符；当字节的开头是110的时候，其表示加上后面的字节，两个字节一起表示一个字符。

举例：`A`在 Unicode 里为`U+0041`，二进制为 `00000000 01000001`，根据上表得知使用一个字节来表示，然后从二进制的最后一位开始，替换上表的x，替换完成为 `01000001`，即`A`的 UTF-8 编码为`0x41`；

举例：汉字`李`在 Unicode 里的码位为`U+673E`，二进制为`0110 0111 0011 1110`，根据上表得知使用三个字节来表示（所有的汉字基本上都是用三个字节来表示），然后从二进制的最后一位开始，替换上表的x，替换完成为`11100110  10011100  10111110`，即`李`的 UTF-8 编码为 `0xE6 0x9C 0xBE`

Unicode 码转换成 UTF-8 的 C 代码如下：

```c
// Unicode 转 UTF8
// 需要保证char* utf8c至少有4字节的空间
// 返回值：返回编号后所占的字节数，如果出错返回-1
// 在此使用的是小端排序
int unicodeToUTF8(unsigned long unicode, char* utf8c)
{
    if (unicode <= 0x007F)
    {   // 10xxxxxx
        *utf8c = (char)(unicode & 0x7F);
        return 1;
    }
    if (unicode <= 0x07FF)
    {	// 110xxxxx 10xxxxxx
        *utf8c = (char)((unicode >> 6 & 0x1F) | 0xC0);
        *(utf8c + 1) = (char)((unicode & 0x3F) | 0x80);
        return 2;
    }
    if (unicode <= 0xFFFF)
    { // 1110xxxx 10xxxxxx 10xxxxxx
        *utf8c = (char)((unicode >> 12 & 0x000F) | 0x00E0);
        *(utf8c + 1) = (char)((unicode >> 6 & 0x003F) | 0x0080);
        *(utf8c + 2) = (char)((unicode & 0x003F) | 0x0080);
        return 3;
    }
    if (unicode <= 0x1FFFFF)
    {
        // 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
        *utf8c = (char)((unicode >> 18 & 0x07) | 0xF0 );
        *(utf8c + 1) = (char)((unicode >> 12 & 0x3F) | 0x80);
        *(utf8c + 2) = (char)((unicode >> 6 & 0x3F) | 0x80);
        *(utf8c + 3) = (char)((unicode & 0x3F) | 0x80);
        return 4;
    }
    return -1;
}
```

UTF-8 转换成 Unicode 的 C 代码如下：

```c
// 将 UTF-8 编码转换成 Unicode
// @utf8c: 需要转换的utf8编码的字符指针
// @Return: 返回转换后的 Unicode 码位
//
long utf8ToUnicode(unsigned char* utf8c)
{
    // 判断 utf8 编码的长度
    assert(utf8c != NULL);
    int size = 0;
    if ((*utf8c & 0x80) == 0x00) size = 1;
    else if ((*utf8c & 0xE0) == 0xC0 && (*(utf8c + 1) & 0xC0) == 0x80) 
        size = 2;
    else if ((*utf8c & 0xF0) == 0xE0 && (*(utf8c + 1) & 0xC0) == 0x80 
             && (*(utf8c + 2) & 0xC0) == 0x80) 
        size = 3;
    else if ((*utf8c & 0xF8) == 0xF0 && (*(utf8c + 1) & 0xC0) == 0x80 
             && (*(utf8c + 2) & 0xC0) == 0x80 && (*(utf8c + 3) & 0xC0) == 0x80) 
        size = 4;
    else return -1;

    if (size == 1) return *utf8c & 0x7F;
    if (size == 2) return ((*utf8c & 0x1F) << 6 )| (*(utf8c + 1) & 0x3F);
    if (size == 3) 
        return (*utf8c & 0x0F) << 12 | ((*(utf8c + 1) & 0x3F) << 6) | (*(utf8c + 2) & 0x3F);
    return (*utf8c & 0x07) << 18 | ((*(utf8c + 1) & 0x3F) << 12) | ((*(utf8c + 2) & 0x3F) << 6) | (*(utf8c + 3) & 0x3F);
}
```

[维基百科链接](https://en.wikipedia.org/wiki/UTF-8)

## GB2312

GB2312 是由中国发布的一个简体中文字符集，基本满足了汉字的计算机处理需求，但是一些罕用字和繁体字还没有包含在里面。GB2312 把汉字进行了分区处理，每个区含有 94 个汉字/符号，一共有 94 个区，每个字符用其所在的区和位来表示。

GB2312 的编码方法如下：

**每个汉字及符号通过两个字节来表示**，第一个字节（称为高位字节）范围为 **0xA1-0xF7**，即字符的区号加上 0xA0，第二个字节（称为低位字节）范围为 **0xA1-0xFE**，即 1-94 加上 0xA0， 由于一级汉字从 16 区开始，到87区结束（包括87区），所以汉字区的“高位字节”范围为 **0xB0-0xF7**, 低位字节的范围为 **0xA1-0xFE**。

[维基百科链接](https://en.wikipedia.org/wiki/GB_2312)

## GBK

`GBK` 是 Windows 系统使用的汉字编码符，其起源是因为 `GB2312` 含有一些未收录的字符，因此 `GBK` 利用 `GB2312` 未使用的编码区间，对 `GB2312` 进行了扩展。

`GBK` 的编码方式包括一字节和双字节两种：

- 一字节范围为`00-7F`，与 ASCII 保持一致
- 双字节的第一字节范围为`81-FE`，第二字节一部分在`40-7E`，另一部分在`80-FE`

`GBK` 完全兼容 `GB2312`，[维基百科链接](https://en.wikipedia.org/wiki/GBK_(character_encoding))

### GBK 与 Unicode 的映射关系

由于 `GBK` 和 `Unicode` 并没有直接的对应关系，我们在转换的时候需要使用映射表来进行转换。我们可以在网上找到对应的映射表来进行转换，也可以使用 `libiconv` 库来进行转换。

[libiconv ](https://www.gnu.org/software/libiconv/)是一个专门用于字符编码转换的一个库，其支持很多种编码方式（具体请查看官方文档）。在 Ubuntu 上默认就已经安装了这个库，下面是一个示例 C 程序：

```c
#include <iconv.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    iconv_t fd = iconv_open("UTF-8", "GBK");
    if (fd == 0) return -1;
    size_t inLen = 10;
    size_t outLen = 255;

    char* inbuf = (char*)malloc(sizeof(char)* inLen);
    char* outbuf = (char*)malloc(sizeof(char) * outLen);
    bzero(outbuf, outLen * sizeof(char));
    // iconv函数的第二个参数和第四个参数需要传入指向输入缓存和输出缓存的指针（二级指针）
    char *in = inbuf;
    char *out = outbuf;

    scanf("%s", inbuf);
    iconv(fd, &in, &inLen, &out, &outLen);

    printf("%s\n", outbuf);

    free(inbuf);
    free(outbuf);
    iconv_close(fd);
}
```
