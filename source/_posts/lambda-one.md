---
title: 读书笔记 | Lambda表达式了解一下(一)
date: 2018-02-07 20:57:40
tags:
  - Java8
categories: 
  - [Java,读书笔记]
toc: true
---

此读书笔记对应于Java8 In Action（Java8实战）的第三章

<!--more-->

## Function Interface相关
在介绍Lambda之前，先说一下Function Interface相关内容
### Function Interface

Function Interface 书上的定义为
>a functional interface is an interface that specifies exactly one abstract method

在[上一篇笔记](https://lza852.com/2018/02/06/behavior-parameterization/)中，定义了Predict&lt;T&gt;接口，这个接口只有一个抽象方法
<center>boolean test(T t)</center>

像这种**仅有一个抽象方法**的接口就被称为**函数接口**（Function Interface）

**注：函数接口可以有多个默认实现方法（default method）,但是只能有一个抽象方法**

### @FunctionInterface
在Java8中，新增了一个注解**@FunctionInterface**,将该注解标注到接口上，能够标明该接口为函数接口。如果在非函数接口上使用该标注，编译器就会报错，提示该接口不满足函数接口的定义
```java
@FunctionInterface
public interface predict<T>{
    boolean test(T t);
}
```
该注解类似于@Override，并非一定要用，但是使用这个注解能够更好地表明该接口是函数接口

### Function Descriptor
顾名思义，Function Descriptor即为Function Interface的描述符。还以Predict&lt;T&gt; 举例，由于该接口定义了一个接收泛型、返回布尔值的方法，它的函数描述符就可以表示为
<center>T -> boolean</center>

当接收参数为空的时候，可以用()表示

下面是Java8预先定义的一些函数接口和对应的函数描述符

|Function Interface|Function Descriptor|
|:-----:|:------:|
|Predict&lt;T&gt;| T -&gt; boolean|
|Consumer&lt;T&gt;| T -&gt; void |
|Function&lt;T, R&gt; | T -&gt; R |
|Supplier&lt;T&gt;|() -&gt; T |
|UnaryOperator&lt;T&gt;|T -&gt; T|
|BinaryOperator&lt;T&gt;|(T, T) -&gt; T |
|BiPredicate&lt;L, R&gt;|(L, R) -&gt; boolean |
|BiConsumer&lt;T, U&gt;|(T, U) -&gt; void|
|BiFunction&lt;T, U, R&gt;|(T, U) -&gt; R|

函数描述符主要用于对Lambda表达式的类型检查，只有符合函数描述符的Lambda表达式才会被使用，具体细节会在之后的笔记中进行说明。

## Lambda表达式
在[上一篇笔记](https://lza852.com/2018/02/06/behavior-parameterization/)中，使用到的Lambda为
<div align="center">
 (Apple apple) ->"RED".equals(apple.getColor()))
</div>

可以看到，Lambda表达式由三部分组成  
- 参数列表  
使用括号括起来，多个参数使用逗号隔开。如果没有参数，则使用()
- 箭头
- 表达式主体  

常见的Lambda形式如下
```java
(String s) -> s.length()  // String -> int
(String s) -> s.length() > 5  // String -> boolean
(int x, int y) -> {  
    System.out.println(x);  
    System.our.println(y);  
}  // 多个语句
() -> 42  // 空参数
```

可以看到，Lambda表达式也可以包含多个**语句**，但是需要用大括号括起来。

下面的Lambda表达式是错误的：
```java
(String s) -> {"HaHaHa";} // 大括号中只能包含语句，不能包含表达式
(String s) -> return "Alan" + s; // return是一个语句，必须要在大括号内
```

所以在写lambda表达式的时候一定要注意表达式和语句的区别

### 使用Lambda的场合
在调用方法的时候，如果该方法含有Function Interface形参，实际调用的时候就可以传入Lambda表达式，在上个笔记中，调用selectApple方法的时候就传入了一个Lambda表达式作为Predict&lt;T&gt;的实参

## 常用的Function Interface具体使用方法
介绍Function Interface的时候，给出了Java8中一些Function Interface，下面以Predict&lt;T&gt;, Consumer&lt;T&gt;, Function&lt;T, R&gt;为例，介绍具体的使用方法。
### Predict&lt;T&gt;
在上个笔记中已经涉及到了该函数接口的使用, 函数描述符为 T->boolean
```java
@FunctionInterface
public interface Predict{
    boolean test(T t);
}

public static <T> List<T> filter(List<T> list, Predicate<T> p) {
    List<T> results = new ArrayList<>();
    for(T s: list){
        if(p.test(s)){
            results.add(s);
        }
    }   
    return results;
}
```
调用示例
```java
Predict<String> nonEmptyString = (String s) -> !s.isEmpty();
List<String> nonEmpty = filter(listOfStrings, nonEmptyString);
```

### Consumer&lt;T&gt;
该接口定义了名为accept的抽象方法，函数描述符为 T->void, 可以使用这个接口实现一些对实体的操作，比如在列表的forEach方法中对列表的每一个元素执行操作
```java
@FunctionInterface
public interface Consumer<T>{
    void accept(T t);
}

public static <T> void forEach(List<T> list, Consumer<T> c){
    for (T t: list){
        c.accept(t);
    }
}

forEach(Array.asList(1,2,3,4,5)
        (Integer i) -> System.out.println(i)
        );
```

### Function&lt;T, R&gt;
该接口定义了名为apply的抽象方法，函数描述符为 T -> R, 可以使用这个接口实现从一种实体到另一种实体的转变。例如可以实现将一个Apple List转换为对应的重量List
```java
@FunctionInterface
public interface Function<T, R>{
    R apply(T t);
}

public static <T, R> List<R> map(List<T> list, Function<T, R> f){
    List<R> result = new ArrayList<>();
    for (T t: list){
        result.add(f.apply(T));
    }
    return result;
}

List<Integer> l = map(
    Array.asList("lambda", "in", "action");
    (String s) -> s.length()
);

```


### 自定义Function Interface
除了上述介绍的Java8中自定义的Function Interface，我们也可以随时在代码中自定义Interface，按照需求实现对应功能。比如需要接收两个泛型参数，返回一个泛型参数，就可以定义一个TripleFunction接口
```java
@FunctionInterface
public interface TripleFunction<T, R, F>{
    F apply(T, R);
}
```




