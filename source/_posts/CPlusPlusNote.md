---
title: C++笔记
date: 2020-05-06 17:56:06
tags: [笔记]
categories: C++
toc: true 
---

此笔记会持续更新，并列出正在阅读的书籍/视频/课程等。

正在阅读以下资料（划线代表已经阅读完毕）

- [CS106L](http://web.stanford.edu/class/cs106l/)的讲义：这是我看到[胡津铭大佬的面试笔记](https://github.com/conanhujinming/tips_for_interview/blob/master/README-zh_CN.md)中提到的一个讲义。和他一样，我也是看C++ Primer看的脑瓜疼，各种语法细节。这本讲义虽然很老，但是整体行文结构是循序渐进的将一些概念引申出来，并且每个概念还会配有一个小而精巧的例子帮助理解。

  <!--more-->

## Stream

> A stream is a channel between a *source* and a *destination* which allows the source to push formatted data to the destination. 

C++ 提供了cin/cout来读取标准输入和写入标准输出，并且在\<fstream\>中提供ifstream和ofstream进行文件的流式读写。

```c++
#include <fstream>
#include <iostream>
int main() {
  std::ifstream input("test.txt");
  if (!input.is_open()) {
    std::cout << "Couldn't open the file test.txt!" << std::endl;
  }
  return 0;
}
```

### Stream manipulator

可以使用流式操作符对流进行操作，常见的流式操作符有：

- `setw`：对输出的宽度进行调整，类似printf("%nd")中的n，如果输出宽度不够会自动补全到指定的宽度
- `left/right`：指定是左对齐/右对齐，也相当于指定补全宽度的方向
- `setfill`：结合`setw`使用，指定补全宽度时使用的字符；指定之后的所有输出都会使用指定的字符

一个利用Stream manipulator输出指定高度的三角形的例子：

```c++
#include <iostream>
#include <fstream>

/**
 * printTriangle(6, '*') can output like this:
 *       *
        ***
       *****
      *******
     *********
    ***********
 *
 */

using namespace std;
void printTriangle(int height, char c) {
    for (int i = 1; i <= height; ++i) {
        cout << setfill(' ') << setw(height - i) << "" << setfill(c) << setw(i - 1) << "";
        cout << setfill(' ') << setw(1) << c;
        cout << setfill(c) << setw(i - 1) << "" << endl;
    }
}
```

### Stream status

当打开文件失败或者输出不正确格式的字符时，Steam会进入错误状态，可以调用`fail`函数判断是否失败，并且使用`clear`函数清除错误状态。

当Stream读取到正确的字符时，会返回一个非0值，当流读取结束时，会返回0。因此可以利用此特性判断是否读取完所有输入字符。

### 直接读取输入的缺点

直接从cin中读取字符有时会因为读取到不正确的分隔符而造成读取结果错误，例如

```c++
string password;    
cout << "Enter administrator password: ";     
cin >> password;     
if(password == "password") { 
  // Use a better password, by the way!         
  cout << "Do you want to erase your hard drive (Y or N)? ";         
  char yesOrNo;         
  cin >> yesOrNo;         
  if(yesOrNo == 'y')             
    EraseHardDrive();     
}
```

如果输入"password y"，则会导致硬盘被清除掉。

### Getline

顾名思义，`getline`函数会读取一整行字符串（直到遇到\n换行符）。但是getline和cin混用的时候要小心，因为cin在每次读取之前是会自动跳过换行符/分隔符的，如果在使用cin直接读取一行字符串之后调用getline，由于cin直接读取在碰到换行符时停了下来，因此getline执行时读到的就是一个空字符串。

### StringStream

StringStream同时具有输入输出功能，输入到StringSteam的任何primitive元素都会变成字符串，同时StringStream也可以输入任何primitive元素。我们可以利用这种特性组装字符串，或者进行字符串格式的校验等。

例如，从标准输入中获得一个整型变量

```c++
#include <string>
#include <sstream>

string GetLine() {
    string line;
    getline(cin, line);
    return line;
}

int GetInteger() {
    while (true) {
        stringstream converter;
        converter << GetLine();

        int result;
        if (converter >> result) {
            char remaining;
            if (converter >> remaining) {
                // 获取到了不是数字的字符
                cout << "Unexpected character: " << remaining << endl;
            } else {
                return result;
            }
        } else {
            cout << "Please enter an integer." << endl;
        }

        cout << "Retry: ";
    }
}
```

## 宏定义

和C相同，C++也有宏定义的功能。宏定义的本质就是将代码中的一些字符串替换成另一种字符串。因此我们可以利用宏定义实现一些神奇的效果

### X Macro trick

这种技巧在定义和使用数据集合的时候比较有用。给定一个枚举类

```c++
enum Color {Red, Green, Blue, Cyan, Magenta, Yellow};
```

如果想实现输入一个枚举类，输出枚举类对应的字符串，我们可能要使用switch来分别输出对应的字符串；那么如果再加一个输入一个枚举类，输出一个相反的颜色，我们又要实现一个函数使用switch来分别输出对应的相反的颜色。如果这样的函数比较多，那之后如果想在枚举类中加入一个新的颜色，就需要分别在这些函数中添加新的分支选项，很容易就会遗漏。

因此我们可以利用宏定义的优势，在一个头文件中定义所有的颜色

```c
// Color.h
DEFINE_COLOR(Red, Cyan)
DEFINE_COLOR(Cyan, Red)
DEFINE_COLOR(Green, Magenta)
DEFINE_COLOR(Magenta, Green)
DEFINE_COLOR(Blue, Yellow)
DEFINE_COLOR(Yellow, Blue)
  
// Color.c
enum Color {
#define DEFINE_COLOR(color, opposite) color,
#include "color.h"
#undef DEFINE_COLOR
};

string ColorToString(Color c) {
    switch(c) {
#define DEFINE_COLOR(color, opposite) case color: return #color;
#include "color.h"
#undef DEFINE_COLOR
        default: return "<unknown>";
    }
}

string GetOppositeColor(Color c) {
    switch (c) {
#define DEFINE_COLOR(color, opposite) case color: return #opposite;
#include "color.h"
#undef DEFINE_COLOR
        default: return "<unknown>";
    }
}
```

需要添加一种颜色的时候，只需要在`Color.h`中使用DEFINE_COLOR新加入一种颜色即可。

### 宏和const的区别

- 宏没有变量类型，const可以指定变量类型，方便在编译的时候发现错误
- 宏可以用于替代任何字符串，而const只能用于声明常量
- 宏是在预处理的时候进行展开，而变量是在编译的时候被定义

### 使用宏定义MAX/MIN

我们通常使用宏定义比较大小的形式为

```c
#define MAX(a, b) ((a) < (b) ? (b) : (a))
```

如果调用宏的时候传入👇这种就会发生错误

```c
MAX(a++, b++)
```

会替换成

```c
(a++) < (b) ? (a++) : b
```

会两次调用a++，从而导致数据出错。正确的方法是👇

```c
#define MAX(a, b) ({__typeof__ (a) _a = (a); \
										__typeof__ (b) _b = (b); \
										_a > _b ? _a : _b; } )
```

通过声明临时变量的方式来避免多次执行表达式。

## Container

### Vector 和 Deque

- Vector的resize函数可以对Vector的**末尾**进行扩展或缩减。可以传入一个初始值，在扩展的时候会使用初始值对扩展后的元素进行初始化

- Deque和Vector类似，但是如果经常的在尾端或头部进行插入的时候，使用Deque会更快。

- 循环向Vector尾部插入的时候，可以先使用reverse函数预留空间，这样就不会频繁的进行空间的重分配，提高效率

  向Deque和Vector分别插入1000个字符串的效率对比👇

  ```
  Insert 1000 strings back with reserved space
  Vector: 0.053 ms
  Deque: 0.135 ms
  Insert 1000 strings back without reserved space
  Vector: 0.108 ms
  Deque: 0.087 ms
  Push 1000 strings front with reserved space
  Vector: 36.761ms
  Deque: 0.11 ms
  Push 1000 strings front without reserved space
  Vector: 36.021ms
  Deque: 0.077 ms
  ```

  可以看到在调用reverse后，Vector的插入效率会提升不少。但向头部插入的时候，Deque的效率碾压Vector。

### Set 和 Map

- Set保证插入的数据不会出现重复数据

  例如判断掷骰子掷多少次能出现两个相同的数

  ```c++
  /* Rolls a six-sided die and returns the number that came up. */     
  int DieRoll() {         
    /* rand() % 6 gives back a value between 0 and 5, inclusive.  Adding one to          
     * this gives us a valid number for a die roll.
     */         
    return (rand() % 6) + 1;     
  }     
  /* Rolls the dice until a number appears twice, then reports the number of die      * rolls.
      */     
  size_t RunProcess() {         
    vector<int> generated;         
    while (true) {             
      /* Roll the die. */             
      int nextValue = DieRoll();             
      /* See if this value has come up before.  If so, return the number of              
       * rolls required.  This is equal to the number of dice that have been              
       * rolled up to this point, plus one for this new roll.
       */             
      for (size_t k = 0; k < generated.size(); ++k)                 
        if (generated[k] == nextValue)                     
          return generated.size() + 1;             
      /* Otherwise, remember this die roll. */             
      generated.push_back(nextValue);         
    }     
  }
  ```

  小tips：从这个程序中可以得到一个有趣的结论：当骰子的面数呈倍数增加的时候，出现两个相同的数的次数却不会成倍增加，而是$\sqrt{n}$，这称之为生日悖论
  - Set能够存储的元素必须保证能够进行比较，因为Set是建立在平衡二叉树上
  - 使用迭代器迭代Set的时候，输出的顺序是按照从小到大的顺序排列的
  - Set的`lower_bound`和`upper_bound`可以返回特殊的迭代器，其中lower_bound返回不小于指定数的迭代器，upper_bound返回大于指定数的迭代器

- Map用于映射对应关系

  - 可以使用`["key"] = value`形式创建和修改map
  - `insert`也可以插入pair，但是不能修改已经存在的pair。如果已经存在，返回的二元组中second会为false
  - 和set一样，迭代的时候map仍然返回的是按照key从小到大排序的顺序

## STL Algorithms

编写代码经常遇到的困难是：

> The  challenge of programming is finding a way to translate a high-level set of commands into a series of low-level instructions that control the machine.

而STL算法函数提供了一种很好的抽象方式，帮助实现上层的逻辑。

在此仅列举一些用到的算法

- accumulate(InputIterator, InputInterator, initial);
- count()
- copy(begin, end, target_iterator)

STL中带if后缀的算法函数都可以传入一个函数参数，算法函数只会对满足这个函数的元素执行相应的操作

- sort(begin, end)
- random_shuffle(begin, end)
- rotate(begin, middle, end)：将middle循环移动到begin位置
- find(begin, end, item)
- binary_search(begin ,end, item)
- ......

### Iterator adaptor

可以将容器或Stream转换为一个iterator，方便访问

- output
  - ostream_iterator\<type\>(target, split)
  - back_insert_iterator\<type\>(target)：在target上执行push_back，也可以调用back_inserter(target)生成iterator
  - inserter(target, location)，对于没有push_back函数的，可以调用此adaptor

- input

  - istream_iterator\<type\>(source)：source可以为空表明结束位置

  - ......

### Removal Algorithms

移除算法并不是真正执行移除操作，而是返回指向要移除的元素的iterator，真正执行删除操作需要调用`erase`函数

### Other

- transform(begin, end, target_begin, func)
- ......