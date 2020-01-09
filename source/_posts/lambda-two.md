---
title: 读书笔记 | Lambda表达式了解一下(二)
date: 2018-02-14 15:09:38
tags: Java8
categories:
  - [Java, 读书笔记]
toc: true
---

此读书笔记对应于Java8 in Action(Java8 实战)的第三章，主要介绍Lambda表达式类型推断相关知识

<!--more-->

## 类型检查
当我们将Lambda表达式传入函数的时候，编译器如何检查Lambda表达式的类型和对应的形参类型一致？
### 上下文
在传入Lambda表达式的时候，编译器首先会从Lambda的上下文(context)来获得Lambda表达式的类型。这里的上下文指的是函数声明中对应的形参或者函数体内部被赋值为Lambda表达式的变量，它们的类型被成为Lambda表达式的**目标类型**(target type),比如
```java
filter(List<Apple> inventory, Predict<Apple> p) // p即为传入的Lambda表达式的context，p对应的类型即为传入的Lambda表达式的目标类型
```
### 类型检查的具体过程
编译器检查传入的Lambda表达式是否有效的过程如下:
1. 通过Lambda表达式的上下文获得表达式对应的目标类型（目标类型通常为Function Interface)
2. 通过对应的Function Interface里定义的抽象方法，获得对应的Function Descriptor
3. 检查Function Descriptor和传入的Lambda表达式的Function Descriptor是否一致，如果一致，则语句合法

**注:如果Lambda表达式会抛出异常，则Function Interface定义的抽象方法也要抛出对应类型的异常**

由上述过程可以看到，类型检查主要是通过Function Descriptor来判断语句是否合法，则只要满足Function Descriptor,一个Lambda表达式可以赋值给不同的Funtion Interface
```java
Comparator<Apple> c1 = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
BiFunction<Apple, Apple, Integer> = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
ToIntBiFunction<Apple, Apple> = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```   

## 类型推断
在编写Lambda表达式的时候，可以省略形参对应的类型,如
```java
(Apple a) -> a.getWeight()
a -> a.getWeight() // 省略形式
```
由于在类型检查的时候是通过检查Function Descriptor，则可以通过Function Descriptor来推测传入的Lambda的参数类型。  
当只有一个参数且省略参数类型的时候，可以省略小括号

### 一些特殊规则
#### void的兼容性
当Function Descriptor返回类型为void的时候，即使传入的Lambda表达式返回类型并不是void类型，依然合法
```java
Predictor<String> s = s -> list.add(s);
Consumer<String> s = s -> list.add(s);　// list.add(s)返回一个boolean值，但是Consumer<String>定义的返回值为void
```
#### &lt;&gt;
在Java中,可以使用&lt;&gt;(Diamond opetator)来让编译器推测类型，一个特定的实例可以实例可以在两个或多个不同的上下文中出现
```java
List<String> listOfStrings = new ArrayList<>();
List<Integer> listOfIntegers = new ArrayList<>();
```

## 闭包
闭包的定义为
>a closure is an instance of a function that can reference nonlocal variables of that
function with no restrictions.

通俗来讲，闭包就是一个依赖外部变量的函数，可以访问函数外部的变量. 在Java中，内部类、匿名类都经常作为闭包(由此看出闭包也可以作为参数传入另外的函数当中)

详细的解释可以参考[知乎问题](https://www.zhihu.com/question/21395848)

### Lambda表达式使用外部变量
Lambda表达式可以替代匿名函数，自然也可以成为闭包。既然闭包能够使用外部的变量，但是Java中的闭包却必须要求被闭包使用的外部变量用final修饰。
```java
final int portNumber = 1337;
Runnable r = () -> System.out.println(portNumber);
```  
这是因为Java实现的闭包并不是将使用的外部变量的**引用**传入闭包内，而是将**外部变量的值**拷贝一份传入闭包中（这种方法称之为capture-by-value，即通过捕获外部变量的值来实现外部变量的使用)。因此，为了保证变量值的一致性，则必须使用final来修饰被使用的外部变量，从而保证外部变量只能被赋值一次。 

另一个用处是在线程的情况下，比如常用的Runnable。例如，如果Lambda位于线程a，使用的外部变量在另外一个线程b，如果Lambda使用的是外部变量的引用，线程b在Lambda表达式调用该变量之前就已经将这个变量对应的内存空间释放，则调用该变量的时候就会找不到该变量对应的值，进而报错。

### Lambda表达式使用外部类 
除了外部变量，闭包也可以使用外部的实例(instance)，但是实例变量并不用使用final修饰，因为实例是存放在堆中，而变量是存放在栈中。堆是所有线程共用的，所以不同线程之间可以共用一个类实例  

总而言之，在Java中闭包能够使用的外部变量只能是只被赋值一次的变量。 

