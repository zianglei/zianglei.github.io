---
title: "CS110L Week 3"
date: 2021-09-07T21:15:25+08:00
toc: true
tags:
- Rust
- CS110L
categories:
- Course
description: Processes! 🕶️Big brother is watching you!
---

## Part 1

使用 Rust 实现一个小工具 inspect-fds，查看进程打开的文件描述符信息。

### 找到一个进程打开的所有文件描述符

一个进程打开的文件描述符存放在 `/proc/$pid/fd` 文件夹下。

因此使用 std::fs::read_dir 遍历当前路径，由于要求 list_fds 函数返回 Option<Vec\<usize>>，但是 std::fs::read_dir 返回的是 Result，因此我们可以使用 Result.ok()? 将 Result 转换为 Option，当 Result 返回 Err 的时候，会转换为 None

```rust
for entry in fs::read_dir(fd_path).ok()? {
     let entry = entry.ok()?;
     let fd = entry.file_name()
                .into_string().ok()?
                .parse::<usize>().ok()?;
     open_fds.push(fd);
}
```

注意在使用 vscode remote 远程连接时，每个进程会多出 19 和 99 两个文件描述符，查看它们指向的描述符分别是 /dev/ptmx 和 vscode remote 创建的文件。

/dev/ptmx 是 Linux 的伪终端，当打开时，会自动创建 master 和 slave（slave 位于 /dev/pts/ 下），为了实现虚拟化的目标，伪终端模拟器通过master发送键盘输入，bash 等终端进程通过 slave 接收，然后将执行结果通过 slave 发送，伪终端模拟器通过 master 接收处理结果，显示到显示器上。

### 读取文件描述符信息

由于 /proc/\$pid/fd 下所有的文件都是软链接，因此需要使用 `fs::read_link` 获取文件描述符指向的真实的文件，然后使用 `fs::read_to_string` 读取 /proc/\$pid/fdinfo/\$fd 下的信息，然后使用正则表达式提取信息即可得到文件描述符的 AccessMode、Cursor。

使用正则表达式

```rust
let re = Regex::new(r"pos:\s*(\d+)").unwrap();
Some(
    re.captures(fdinfo)?
        .get(1)?
        .as_str()
        .parse::<usize>()
        .ok()?,
    )
```

## Part 2

实现一个通用的链表，练习泛型和 trait 的用法

Box\<T> 表示在堆上为对象分配内存，返回指向内存的指针

### 实现泛形

```rust
impl<T> LinkedList<T> {
    head: Option<Box<Node<T>>>,
    size: usize,
}
```

### 实现 Clone trait

```rust
impl<T: Clone> Clone for Node<T> {
    fn clone(&self) -> Self {
        // next 会自动克隆下一个节点
        Node { value: self.value.clone(), next: self.next.clone() }
    }
}

```

### 实现 PartialEq trait

```rust
impl<T: PartialEq> PartialEq for LinkedList<T> {
    fn eq(&self, rhs: &LinkedList<T>) -> bool {
        if self.size != rhs.size {
            return false;
        } 
        let mut l: &Option<Box<Node<T>>> = &self.head;
        let mut r: &Option<Box<Node<T>>> = &rhs.head;
        loop {
            match l {
                Some(node) => {
                    // node 类型是 &Box<Node<T>>, r 类型是&Option<Box<Node<T>>，如果直接 unwrap，得到的就是 Box<Node<T>>，不能做比较，因此我们需要使用 as_ref，转换成 Option<&Box<Node<T>>，然后再 unwrap
                    let rnode = r.as_ref().unwrap();
                    if *node != *rnode {
                        return false
                    }
                    l = &node.next;
                    r = &rnode.next;
                },
                None => break,
            }
        }
        true
    }
}
```

注意比较时候的类型转换。

Option::AsRef 和通常的 as_ref 不同，会将 &Option\<T> 转换成 Option<&T>，也就是说刚开始对 Option 是没有所有权的，通过 as_ref 转换成了 Option<&T>，就有了所有权，可以调用 unwrap 获取里面的 T 的引用。

### 实现 Iterator/IntoIterator trait

引用自 [这篇](http://xion.io/post/code/rust-for-loop.html) 文章，在 Rust 中，为了实现遍历所有元素，通常使用迭代器访问元素，迭代器起到的作用是获取当前值、向前指向下一个值、如果全部已经遍历完就返回 None

如果一个类型实现了 Iterator trait，就可以转换成一个迭代器，调用 next 方法获得集合中的值

而通常使用的 for loop 其实是 IntoIterator trait 的语法糖，其中的 into_iter 方法会返回一个迭代器

```rust
trait Iterator {
  type Item
}
trait IntoIterator {
  type Item
  type IntoIter
  fn into_iter(self) -> Iterator; // 可以看到默认的迭代器会将原本的集合的所有权消耗掉，转变成一个迭代器，因此 for loop 使用过后集合就不能再此被使用
}

impl IntoIterator for &Vec<T> {
  type Item
  type IntoIter
  fn into_iter(self) -> Itearator<Item=&T> {
    // 转换成引用迭代器
  }
}
```

对于我们的 LinkedList 来说，为了实现

```rust
for item in &list {
  ...
}
```

我们需要为 &LinkedList\<T> 实现 IntoIterator，将其转换成 LinkedListIter，LinkedListIter 需要实现 Iterator trait，对链表里的 value 进行克隆

```rust
pub struct LinkedListIter<'a, T> {
    current: &'a Option<Box<Node<T>>>
}

impl<T: Clone> Iterator for LinkedListIter<'_, T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        match self.current {
            Some(node) => {
              	// 此处直接使用拷贝
                Some(node.value.clone())
            },
            None => None
        }
    }
}

// 引用情况下
impl<'a, T: Clone> IntoIterator for &'a LinkedList<T> {

    type Item = T;
    type IntoIter = LinkedListIter<'a, T>;

    fn into_iter(self) -> LinkedListIter<'a, T> {
        LinkedListIter { current: &self.head }
    }
}
```

如果不使用拷贝形式，而是用引用形式，则可以按照下面方法实现

```rust
impl<'a, T: Clone> IntoIterator for &'a LinkedList<T> {

    type Item = &'a T; // 此处变为引用
    type IntoIter = LinkedListIter<'a, T>;

    fn into_iter(self) -> LinkedListIter<'a, T> {
        LinkedListIter { current: &self.head }
    }
}

impl<'a, T: Clone> Iterator for LinkedListIter<'a, T> {
    type Item = &'a T; // 此处变为引用
    fn next(&mut self) -> Option<&'a T> { // Option 返回的类型为引用值
        match self.current {
            Some(node) => {
                self.current = &node.next;
                Some(&node.value) // 返回引用
            },
            None => None
        }
    }
}
```



如果需要实现

```rust
for item in list {
  ...
}
```

我们可以直接为 LinkedList\<T> 实现 Iterator

```rust
impl <T: Clone> Iterator for LinkedList<T> {
  type Item
  fn next(&mut self) -> Option<T>
}
```



