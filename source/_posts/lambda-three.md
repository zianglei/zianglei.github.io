---
title: 读书笔记 | Lambda表达式了解一下(三)
date: 2018-03-05 20:45:11
tags: Java8
categories:
  - [Java, 读书笔记]
toc: true
---

此读书笔记对应于Java8 in Action(Java8 实战)的第三章，主要介绍Lambda表达式中方法引用的相关知识

<!--more-->

## Method Reference(方法引用)
首先来举个例子
```java
inventory.sort((a1, a2) -> a1.getWeight().compareTo(a2.getWeight()))
```
这是我们之前见到的对inventory根据重量进行排序的代码，可以看到所用到的lambda表达式比较繁琐，如果使用Method Reference，具体代码为
```java
inventory.sort(comparing(Apple::getWeight)
```
sort的参数类型为Comparator<? super E>

Comparator有一个静态的帮助方法为comparing，接收一个函数，返回Comparator。则可以传入Method Reference

可以看到，Method Reference即为使用某个已经存在于某个类中的方法，并且可以将方法作为参数传入。这样做在大多数情况下都能提高程序的可读性，并且可以省去编写冗长的Lambda表达式的麻烦

### Method Reference 种类
Method Reference主要可以分为三类
1. 引用静态方法，比如Integer::parseInt
2. 引用类的属性，比如String::length
3. 引用已存在的实例，比如变量s，类型为String，那么就可以使用s::getLength，相当于s.getLength()

### Method Reference 语法
```java
(args) -> ClassName.staticMethod(args) //lambda
ClassName::staticMethod // method reference
```
```java
(arg0, rest) -> arg0.instanceMethod(rest) // lambda
ClassName::instanceMethod // method reference, arg0 is of type ClassName
```
```java
(args) -> expr.instanceMethod(args) // lambda
expr::instanceMethod(args) // method reference
```

### 构造函数引用
method reference除了可以引用已经存在的函数外，还可以引用构造函数，语法形式为ClassName::new, 例如
```java
Supplier<Apple> c1 = Apple:new; 
// A constructor reference to the default Apple() constructor
Apple a1 = c1.get();
// produce a new apple
```

```java
Function<Integer, Apple> c1 = Apple:new;
// A constructor reference to Apple(Integer weight) equals to Function interface
Apple a2 = c2.apply(110);
```

## 组合Lambda表达式
Lambda表达式可以进行组合。通过组合多个简单的Lambda表达式，我们可以得到一个复杂的表达式。下面是几个栗子：

### 逆序排序
```java
inventory.sort(comparing(Apple:getWeight).reversed());
```

### 链式排序
如果两个苹果重量相同，则比较两个苹果的产地，就可以用链式排序实现
```java
inventory.sort(comparing(Apple:getWeight)
.reversed()
.thenComparing(Apple::getCountry)
)
```

### 组合预测
可以使用.negate、.and、.or来进行组合，实现复杂的判断
```java
Predict<Apple> redAndHeavyAppleOrGreen = redApple
.and(a -> a.getWeight() > 150)
.or(a -> "green".equals(a.getColor()));
```

### 组合函数(Function)
可以使用andThen、compose实现Lambda函数表达式的组合
```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
Function<Integer, Integer> h = f.andThen(g); // 即为 h = g(f(x))
Function<Integer, Integer> d = f.compose(g); // 即为 d = f(g(x))
int hresult = h.apply(1);
int dresult = d.apply(1);
```

