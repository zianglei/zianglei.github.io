---
title: 向量
date: 2020-02-28 14:46
categories:
	- [数据结构]
toc: true
---

《数据结构》学习笔记--向量

<!--more-->

## ADT和DS

抽象数据类型是在数据模型上定义的一套操作（这些操作具体是如果实现的并不关心，所以叫抽象数据类型）数据结构是使用编程语言实现ADT的一套算法

## 向量的ADT定义

![向量ADT](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/1.png)

其中disordered() 判断是否有紧邻的逆序对，返回逆序对个数，如果为0，代表是有序向量

## 向量基于复制的构造

```c++
void Vector<T>::copyFrom(T const* A, Rank lo, Rank hi) {
    _elem = new T[_capacity=2*(hi - lo)];
    _size = 0;
    while (lo < hi) {
        _elem[_size++] = A[lo++];
    }
}
```

## 可扩充的向量

### 递增式扩容

在初始容量为0，插入n = m * I 的过程中，在第1，I+1，2I+1，，，（m-1)*I+1次插入时扩容，每扩容一次追加I个容量，则总共扩容了m次，扩容的时间成本为0、I、2I、3I、、(m-1)I，

算术级数，复杂度是末项的平方，即为O(n^2)，分摊成本为O(n)

###加倍式扩容

在初始容量为1的满向量，插入n=2^m的过程中，则在第1，2，4，8，16…次插入时都扩容，时间成本为1，2，4，8。。。2^m = n，几何级数，时间复杂度与末项相同，即O(n)

则分摊复杂度为O(1)

Tips：加倍扩容多少倍合适？使用2倍的扩容因子扩容后占用的空间总是大于之前分配的所有空间总和，对缓存不友好

## **分摊复杂度**

分摊复杂度所考虑的一串操作序列一定是真实可行的，并且该序列必须要足够长

![分摊复杂度](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/2.png)

## 插入

```c++
template <typename T>
Rank Vector<T>::insert (Rank r, T const &e) {
    expand(); // 如果有必要，需要扩容
    for (int i = _size; i > r; i--) _elem[i] = _elem[i - 1];
    _elem[r] = e; _size++;
    return r;
}
```

## 区间删除

```c++
template <typename T>
int Vector<T>::remove(Rank lo, Rank hi) {
    if (lo == hi) return 0;
    while (hi < _size) _elem[lo++] = _elem[hi++]; // 从前往后移动
    _size = lo; // 抛弃[lo, +size = hi) 区间
    shrink(); // 若有必要，则缩容
    return hi - lo; // 返回被删除元素的个数
}
```

## 单元素删除

```c++
template <typename T>
T Vector<T>::remove(Rank r) {
    T t = _elem[r];
    remove(r, r + 1);
    return t;
}
```

## 查找

```c++
/**
* 无序查找，返回最后一个元素的位置，失败时，返回lo-1
*/
template <typename T>
Rank Vector<T>::find (T const& e, Rank lo, Rank hi) const {
    while ((lo < hi--) && (e != _elem[hi])); // 逆序查找
    return hi;
}
```

此查找算法是输入敏感算法：最优O(1)，最坏O(n)

## 唯一化

```c++
// 删除重复元素，返回被删除元素数目
// 时间复杂度为O(n^2)，因为无论是查找还是移动都是O(n)，
// 所以加上循环，时间复杂度就为O(n^2)
template <typename T>
int Vector<T>::deduplicate() {
    int oldsize = _size;
    Rank r = 1;
    while (r < _size) {
        find(_elem[r], 0, r) < 0 ? r++ : remove(r);
    }
    return oldsize - _size;
}·
```

### 算法证明

![正确性](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200228144323_F6484D8F-6635-4321-9F47-3C6501D4EE7D.png)

![单调性](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200228144339_C1A71BA3-7A8D-4298-8AF2-6A7EE663A928.png)

## 遍历

```c++
template <typename T>
void Vector<T>::traverse(void (*visit) (T&))
{ for (int i = 0; i < _size; ++i) visit(_elem[i]);}

template <typename T> template <typename VST>
void Vector<T>::traverse(VST& visit) 
{ for (int i = 0; i < _size; ++i) visit(_elem[i]);}
```

## 有序向量

### 唯一性

#### 有序性

有序序列中，任意一对相邻元素顺序；无序序列中，总有一对相邻元素逆序

```c++
template <typename T>
void Vector<T>::disordered() {
  int n = 0;
  for (int i = 1; i < _size; i++) {
    n += (_elem[i - 1] > _elem[i]);
  }
  return n; // n为0代表有序
}
```

#### 有序向量唯一化低效版（O(n^2))

```c++
template <typename T>
int Vector<T>::uniquify() {
  int i = 0;
  int old_size = _size;
  while (i < _size - 1) {
    _elem[i] == _elem[i + 1] ? remove(i + 1): i++;
  }
  return old_size - _size;
}
```

分析：最坏情况下，每次调用remove都有O(n)的复杂度，则总时间复杂度为O(n^2)

#### 有序向量唯一化高效版（O(n))

```c++
template <typename T>
int Vector<T>::uniquify() {
  int i = 0, j = 0;
  while (++j < _size) {	// 逐一扫描
     if (_elem[i] != _elem[j]) _elem[++i] = _elem[j]; // 当发现不同元素时，向前移动
  }
  _size = ++i; shrink(); // 截断末尾
  return j - i;
}
```

分析：时间复杂度为O(n)

**![唯一化高效算法流程图](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200228152750_image-20200228152737081.png)**

