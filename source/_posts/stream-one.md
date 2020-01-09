---
title: 读书笔记 | Stream（一）
date: 2018-03-06 21:41:58
tags: 
  - Java8
  - Stream
categories:
  - [Java, 读书笔记]
  - Stream
---


Java8 一个最大的特点就是引入了Stream，Stream顾名思义为流，即可以使用流式处理方式对列表等进行处理
<!--more-->

## Stream简单介绍
```java
List<String> lowCaloricDishesName = 
    menu.stream()
        .filter(d -> d.getCalories() < 400)
        .sorted(comparing(Dish::getCalories))
        .map(Dish::getName)
        .collect(toList())
```
上述代码实现了对一个列表的元素先过滤出卡路里小于400的元素，然后根据卡路里进行排序，最后获得列表中元素的名字，返回为List
如果在Java7中，需要写好几个for循环才能实现这一串功能，而使用Stream之后就可以更方便地实现，并且具有很高的可读性，一目了然。

同时，Stream也可以很方便地实现并行处理，在代码中只需要将stream换成parallelStream即可。Stream其实和SQL语句有些类似，都是采用了声明式方法。程序猿不需要关心如何实现具体的功能，只需要告诉计算机需要做什么就可以。

### Stream的严谨定义
Stream的严谨定义为
> A sequence of elements from a source that supports data processing operations

也就是一个支持数据操作的元素序列。从这个定义中可以看到Stream与三个因素有关
- 元素序列（Sequence of elements)
和Java中的Collection类似，Stream也是一个元素序列。但是与Colleciton不同的是Stream定义了关于元素的一系列操作，例如**filter**，**sorted**等等，Collection是存储数据，而Stream是处理数据
- 源（Source）
Stream不仅限于可以处理Collection，只要是提供数据的源都可以转换为Stream进行处理，例如arrays，I/O接口等等
- 数据处理操作(Data processing operations)
Stream定义了一系列数据处理操作，例如filter、map、reduce、find、match等等，可以很方便地对数据进行处理，实现类似SQL那样地声明式操作

### Stream的特点
#### 流水线
由于Stream中定义的方法的返回类型都是Stream，所以这些操作可以像流水线一样串起来，能够极大地提高可读性，并且还可以进行一些优化，比如laziness和short-circuiting(短路)
#### 内部循环
Stream内部隐含了循环操作，不想Collections那样要在外部实现循环来对数据进行处理

## 开始使用Stream
我们以一段代码来学习使用Stream
```java
List<String> threeHighCaloricDishNames = 
    menu.stream()
        .filter(d -> d.getCalories() > 3000)
        .map(Dish::getName)
        .limit(3)
        .collect(toList());
System.our.println(threeHighCaloricDishNames);
```
在上面的代码中，我们使用了四个数据操作方法
- filter
filter是对数据源中的数据按照条件进行过滤，我们传入了一个lambda表达式，表明过滤出卡路里大于3000元素
- map
map是将数据源中的数据转变成另外一类数据，在此处，我们将过滤出的dish的name提取出来，形成新的Stream
- limit
limit表示对Stream进行截断，保留前三个
- collect
collect是将Stream转换为其他非Stream的形式，比如数字、列表等等。在此处我们将Stream转变为List，但两者中的元素是相同的，都为Dish的Name

可以看到使用一系列Stream操作方法就可以将一个列表进行复杂的数据操作，而形式却是非常简单的。

在Stream执行过程中，编译器会对一些操作进行优化，提高执行效率。比如，在上述代码中，编译器并不会按顺序执行操作，而是首先进行截断（即先执行limit），然后再对截断后的Stream进行处理，这种优化称之为short-circuiting；在执行filter和map的时候，并不是先执行完filter和map，而是在实现循环的hi后将filter和map合并，这种优化称之为loop fusion

## Stream操作类型
Stream常用的操作方法包括filter、map、limit等等，这些操作方法都可以分为两种类型Intermediate operations和Terminal operations
### Intermediate operations(中间操作方法)
像filter、sorted等返回Stream的方法称之为中间操作方法。这些方法并不会按照编写顺序执行，而是在遇到Terminal operations的时候才会按照一定的顺序执行，因为编译器要对这些方法进行优化（例如上述提到的loop fusion等）来提高效率
### Terminal operations(终点操作方法)
Terminal operations常用的是collect、forEach，这一类方法返回的是非Stream类型
### 常用的Stream操作方法
|Operation|Type|Return type|Argument operation|Function Descriptor|
|:--:|:--:|:--:|:--:|:--:|
|filter|Intermediate|Stream<T>|Predict<T>|T->boolean|
|map|Intermediate|Steam<T>|Function<T, R>|T->R|
|limit|Intermediate|Stream<T>|
|sorted|Intermediate|Stream<T>|Comparator<T>|(T,T)->int|
|distinct|Intermediate|Stream<T>|

|Operation|Type|Purpose|
|:--:|:--:|:--:|
|forEach|Terminal|对每一个元素使用一个lambda表达式处理，返回void|
|count|Terminal|返回stream中元素的个数|
|collect|Terminal|将Stream转换为集合，例如List、Map|



