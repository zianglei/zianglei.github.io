---
title: Wireshark | 编写自己的协议解析器（一）
date: 2018-12-04 09:26:12
tags:
 - Wireshark
categories:
 - 学习笔记
toc: true
---

最近的一门课的课设要求使用规定的一个协议对以太网分组进行更改，由于更改后的分组用 Wireshark 直接抓包只能得到十六进制的字节，还要人工将每个字段进行转换，所以就想写一个解析这个协议的解析器。在网上找到一篇教程写的还不错，就拿过来翻译了一下。

<!--more-->

[原文链接点这里](https://mika-s.github.io/wireshark/lua/dissector/2017/11/04/creating-a-wireshark-dissector-in-lua-1.html)

本文将解释如何使用 Lua 编程语言在 Wireshark 中轻松创建协议解析器。当 Wireshark 没有自带你要处理的自定义的协议的解析器的时候，编写解析器就很有用。例如，如果 Wireshark 是这样的

![](https://mika-s.github.io/assets/creating-wireshark-dissectors-1/before.png)

很难判断数据部分中的各个字节代表什么。

Wireshark 是用 C 编写的，而 Wireshark 的解析器通常也是用 C 编写的。然而，Wireshark有一个 Lua 实现，它使不熟悉 C 的人更容易编写解析函数。对于不熟悉 Lua的人来说，Lua 是一种非常轻量级的编程语言，其被设计用于在应用程序中作为一种脚本语言，方便扩展应用程序的功能。

使用Lua的缺点是编写的解析器比用 C 编写的解析器慢（翻译者：其实处理小型的协议差距并不大）。

在开始编写解析器之前，我们需要先速成一下 Lua。我们并不需要详细了解这门语言，但我们必须知道基本的语法。

## Lua 速成课程

- Lua是多范式的，在一定程度上支持过程式、函数式编程，并且具有一些面向对象的编程特性。它没有现成的类、原型或继承，但是它们可以由程序员创建。
- Lua 是动态类型语言
- 变量作用域有`local`和`global`两种，默认为`gloabl`
- 不需要分号结尾，空格也不像 Python 那样要求的很严格
- 以`--`开头的行是注释
- 不要使用`++`或者`+=`，使用`i = i + 1 `
- 数据类型有：字符串、数字、布尔值、nil、函数、用户数据（userdata）、线程和 table。数字包含浮点数和整数，布尔值有 true 和 false。字符串可以用单引号也可以用双引号。userdata 和线程在这里并不重要
- nil是一个非值（non-value）。变量在被赋值之前其值为nil。
- 在条件判断中：nil 和 false 为假
- Lua 有一个名为 table 的类型，这也是它唯一的数据结构。Table 实现了关联数组。关联数组可以由数字和其他类型(如字符串)建立索引。它们没有固定的大小，可以动态添加元素。Table 通常被称为对象。可以这样创建：

```lua
new_table = {}
```

可以这样赋值：

```lua
new_table[20] = 10
new_table["x"] = "test"
a.x = 10	-- same as a["x"] = 10
```

Lua 可以使用函数，并且通常与 Javascript 中的对象非常相似。

- 条件判断

  ```lua
  if i == 0 then variable = 200
  elseif i == 1 then variable = 300
  else variable = 400 end
  ```

- 循环

  ```lua
  while i < 10 do
    i = i + 1
  end
  ```

  ```lua
  for i = 0, 10, 1 do
    print(i)
  end
  ```

  上面的意思是 *i = first, last, delta*, `break`可以使用，但是`continue`不能使用

- 函数可以这样声明：

  ```lua
  function add(arg1, arg2)
      return arg1 + arg2
  end
  ```

  调用

  ```lua
  local added_number = add(2,3)
  ```

  如果你看到一些函数是这样调用的

  ```lua
  a:func1()
  a.func2()
  ```

  这表明函数`func1`和`func2`属于表（对象）  `a`。使用冒号是将对象本身作为参数传递给函数的语法糖。这意味着`a:func1()` 代表 `a.func1(a)`。

上面这些就是 Lua 重要的语法了，你可以查看[Lua 5.2 参考手册](https://www.lua.org/manual/5.2/)来学习更详细的内容。

## 设置

Lua 脚本被放置在 plugins 文件夹的子文件夹中（翻译注：按照官方文件的说法，插件要放在子文件夹中，而 Lua 脚本可以直接放在 plugins 文件夹下），该子文件夹位于 Wireshark 根文件夹中。子文件夹以 Wireshark 版本命名。在 Widowns 上路径为*C:\Program Files\Wireshark\plugins\2.4.2*（翻译注：在 Linux 上通常为*~/.local/lib/wireshark/plugins*，如果不存在需要建立）。当Wireshark启动时，脚本会被加载。修改脚本后必须重新启动Wireshark，或者使用`Ctrl+Shift+L`重新加载所有Lua脚本。

我目前使用的是最新版本。我在这里所做的可能不适用于早期版本。

## 协议

本文要研究的最有趣的协议可能是一个 Wireshark 还不知道的定制协议，但是我使用的所有定制协议都是与工作相关的，我不能在这里发布关于它们的信息。因此，我们将转而研究 [MongoDB wire 协议](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/)。

（在 Wireshark 中已经有了一个 [Mongo 解析器](https://wiki.wireshark.org/Mongo)，但我不打算使用这个）

根据上面链接中的规范，MongoDB wire 协议是使用端口号 27017 的 TCP/IP 协议。字节顺序是小端，这意味着高位字节排在内存的高位。大多数协议都是大端协议。而唯一的区别只是字节的顺序。例如，如果我们有一个 int32，大端模式下是这些字节: `00 4f 23 11`，那么小端模式就是`11 23 4f 00`。这是我们在写解析器时必须考虑的。

在这篇文章中，我只看协议的头部。就像这样

```c
struct MsgHeader {
    int32 messageLength; // total message size, including this
    int32 requestID;  	 // identifier for this message
    int32 responseTo;	 // requestID from the original request
    					 //   (used in reponses from db)
    int32 opCode;		 // request type - see table below
}
```

我们可以看到它有 4 个 int32，每个 int32 包含4个字节，因为 4*8 = 32。

## 编写样本代码

让我们先建立一些样板代码，所有的解析器都会用到:

```lua
mongodb_protocol = Proto("MongoDB",  "MongoDB Protocol")

mongodb_protocol.fields = {}

function mongodb_protocol.dissector(buffer, pinfo, tree)
  length = buffer:len()
  if length == 0 then return end

  pinfo.cols.protocol = mongodb_protocol.name

  local subtree = tree:add(mongodb_protocol, buffer(), "MongoDB Protocol Data")
end

local tcp_port = DissectorTable.get("tcp.port")
tcp_port:add(59274, mongodb_protocol)
```

