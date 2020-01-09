---
title: 读书笔记 | Java8 行为参数化
date: 2018-02-06 16:55:42
tags: 
  - Java8
categories: 
  - [Java,读书笔记]
toc: true
---

此读书笔记对应于Java8 In Action（Java8实战）的第二章

<!--more-->
## 需求不断变化
某一天，一个农场主想让你从他存储苹果的仓库中筛选出红色的苹果。你心想，这个好办啊，随手就写出了代码

```java
public List<Apple> selectApple(List<Apple> inventory, String color){
    List<Apple> result = new ArrayList<>();
    for (Apple apple: inventory){
        if (apple.getColor().equals(color)){
            result.add(apple);
        }
    }
    return result;
}
```
然后调用
```java
List<Apple> result = selectApple(inventory, 'RED');
```

使用你的代码农场主很快就筛出了红苹果。过了两天，农场主又改变了需求，他想要筛选重量大于150g的苹果。虽然你很讨厌改需求，但是这个需求实现起来还算简单，直接将上面筛选颜色的更改一下就行了

```java
public List<Apple> selectApple(List<Apple> inventory, int weight){
    List<Apple> result = new ArrayList<>();
    for (Apple apple: inventory){
        if (apple.getWeight() > weight){
            result.add(apple);
        }
    }
    return result;
}
```
然后调用
```java
List<Apple> result = selectApple(inventory, 150);
```

隔了几天，农场主又改需求了，他还想把仓库中的**西瓜**按照颜色和重量分别存到其他不同的仓库。。。
<div style="text-align: center">
<img src="http://p3jggzq4i.bkt.clouddn.com/v2-e6ecb99d82832b3d63c3845948240f44_hd.jpg"/>
</div>

## 行为参数化
为了应对不断变化的需求，我们要对这些需求进行抽象

### 对不同属性的筛选
我们可以看到对属性的筛选，都是通过获取Apple的属性，然后返回一个boolean值。所以我们可以创建一个接口
```java
public interface ApplePredict{
    boolean test(Apple apple);
}
```
然后继承这个接口，分别实现对苹果的颜色和重量的筛选
```java
public class PredictRedApple implements ApplePredict{
    boolean test(Apple apple){
        return "RED".equals(apple.getColor());
    }
}   

public class PredictWeight implements ApplePredict{
    boolean test(Apple apple){
        return apple.getWeight > 150;
    }
}
```
然后改变之前写的SelectApple函数，将ApplePredict接口作为参数传入
```java
public List<Apple> selectApple(List<Apple> inventory, ApplePredict predict){
    List<Apple> result = new ArrayList<>();
    for (Apple apple: inventory){
        if (predict.test(apple)){
            result.add(apple);
        }
    }
    return result;
}
```
这样，在调用的时候传入不同的实现类就可以实现对苹果不同属性的筛选
```java
List<Apple> result = selectApple(inventory, new PredictWeight()); 
```

>这种实现方法叫做[策略模式](https://en.wikipedia.org/wiki/Strategy_pattern),通过事先实现不同的策略，在运行的时候就可以根据需要选择对应的策略，而不用改变程序框架

### 使用匿名类
上述的方法虽然能够解决需求变更问题，但是每有一个需求，我们就要继承接口新建一个类，很繁琐。使用匿名类能够一定程度上解决这个问题
```java
List<Apple> result = selectApple(inventory, new ApplePredict(){
    boolean test(Apple apple){
        return "RED".equals(apple.getColor());
    }
});
```
匿名类可以实现继承和实例化一个接口，这样就不用再新建一个类来继承这个接口了。实际应用中，比如安卓中的为按钮添加监听器，对列表进行自定义的排序等等，都能用到匿名类。

### 使用Lambda
匿名类能够减少继承接口的类的数目，但是也有缺点：
- 代码臃肿  
当实现的功能代码比较简单的时候，比如就只是一个布尔值表达式，却需要占用三四行空间，不太简练
- 变量作用域问题  
当匿名类中需要定义变量的时候，很容易就会搞错变量的作用域，导致输出不正确的结果，尤其是使用this的情况下，例如
```java
public class MeaningOfThis{
    public final int value = 4;
    public void doIt(){
        int value = 6;
        Runnable r = new Runnable(){
            public final int value = 5;
            public void run(){
                int value = 10;
                System.out.println(this.value);
            }
        };
    }
    public static void main(String.. args){
        MeaningOfThis m = new MeaningOfThis();
        m.doIt();
    }
}
```
>当执行的时候，程序输出的是5，因为this在匿名类中（Runnable）指的就是匿名类本身，而不是MeaningOfThis类

在Java8中引入了Lambda，可以替代匿名类，同时代码更简洁
```java
List<Apple> result = selectApple(inventory, (Apple apple)->"RED".equals(apple.getColor()));
```

### 使用泛型
对于筛选苹果或者西瓜，可以使用泛型来解决不同物体的问题
```java
public interface Predict<T>{
    boolean test(T t);
}

public static <T> List<T> select(List<T> inventory, Predict<T> predict){
    List<T> result = new ArrayList<>();
    for(T item: inventory){
        if(predict.test(item)){
            result.add(item);
        }
    }
    return result;
}
```

这样在调用的时候，就可以传入不同的实体类来筛选不同的物体
```java
List<Apple> result = select(inventory, (Apple apple)->"RED".equals(apple.getColor()));
List<Watermelon> result = select(inventory, (Watermelon wm)->"GREEN".equals(wm.getColor()));
```
