---
title: 向量
date: 2020-02-28 14:46
categories:
	- [数据结构]
toc: true
---

《数据结构》学习笔记--向量

在给向量规定的ADT中，有几种操作

- 去重：区间去重，单个去重($$O(n^2)$$)，排序后进行去重（$$O(n)$$)
- 查找：顺序查找$$O(n)$$，二分查找$$O(log(n))$$ (两种改进版，斐波那契查找)，插值查找
- 排序：冒泡排序（两种优化措施），归并排序

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

### 加倍式扩容

在初始容量为1的满向量，插入n=2^m的过程中，则在第1，2，4，8，16…次插入时都扩容，时间成本为1，2，4，8。。。2^m = n，几何级数，时间复杂度与末项相同，即O(n)

则分摊复杂度为O(1)

Tips：加倍扩容多少倍合适？使用2倍的扩容因子扩容后占用的空间总是大于之前分配的所有空间总和，对缓存不友好

## 分摊复杂度

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

template <typename T>
Rank Vector<T>::expand() {
  if (_size < _capacity) return;
  if (_capacity < DEFAULT_CAPACITY) _capacity = DEFAULT_CAPACITY;
  T* oldElem = _elem;
  _elem = new T[_capacity <<= 1];
  for (int i = 0; i < _size; i++) {
    _elem[i] = oldElem[i];
  }
  delete [] oldElem;
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

## 去重

```c++
// 删除重复元素，返回被删除元素数目
// 时间复杂度为O(n^2)，因为无论是查找还是移动都是O(n)，
// 所以加上循环，时间复杂度就为O(n^2)
template <typename T>
int Vector<T>::deduplicate() {
    int oldsize = _size;
    Rank r = 1;
    while (r < _size) {
        find(_elem[r], 0, r) < 0 ? r++ : remove(r); // 注意是移除当前下标为r的元素
    }
    return oldsize - _size;
}
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

### 有序性

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

#### 有序向量去除重复元素低效版（O(n^2))

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

#### 有序向量去除重复元素高效版（O(n))

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

![唯一化高效算法流程图](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200228152750_image-20200228152737081.png)

#### 二分查找（A）

##### 语义约定

至少应该便于自身向量的维护：`V.insert(1 + V.search(e), e)`，即便查找失败，也应该返回新元素适当插入的位置；若允许重复元素，则每一组也需要按照插入的顺序排列

**约定：在有序向量`V[lo, hi)`中，确定`不大于e的最后一个`元素**

```c++
// 简单版本二分查找
static Rank Vector<T>::binarySearch(T* A, T e, Rank lo, Rank hi) {
  while (lo < hi) {
    Rank mid = (lo + hi) / 2;
    if (e < A[mid]) hi = mid;
    else if (A[mid] < e) lo = mid + 1;
    else return mid;
  }
  return -1;
}
```

时间复杂度分析 T(n) = T(n/2) + O(1) = O(logn) （O(1)是因为每次比较只有1～2次比较次数，为常数项）

空间复杂度为$$O(1)$$，如果是递归版本，空间复杂度为$$O(log(n))$$

##### 查找长度

二分查找的计算包括元素之间的比较和秩的算术运算，由于元素的比较有时不可能在常数时间内完成， 因此我们主要关注在二分查找中的比较次数，也即**查找长度**。

对于上述二分查找实现而言，成功查找长度和失败查找长度均为O(1.5logN)（通过举例得到，实际递推公式参考《数据结构》一书）。

##### 改进余地

在二分查找的过程中，即使是成功查找，相同迭代次数下对应的查找长度也不近相等，这是因为向左和向右的比较次数不同，从而导致查找长度的不均衡。

#### fibonacci查找

由于向左的比较次数较少，因此fibonacci查找的思路是调整前后区域的宽度，适当的加长前向量，缩短后向量

![fibonacci查找示意图](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200229133124_KsD97R.png)

```c++
#include "..\fibonacci\Fib.h"
template <typename T> static Rank fibSearch(T* A, T e, Rank lo, Rank hi) {
  Fib fib(hi - lo);
  while(lo < hi) {
    while (hi - lo < fib.get()) fib.prev();	// 调整斐波那契数列，直到得到不小于n的斐波那契数，这一步花费O(logN)
   	Rank mid = lo + fib.get() - 1;
    if (e < A[mid]) hi = mid;
    else if (A[mid] < hi) lo = mid + 1;
    else return mid;
  }
  return -1;
}
```

对于fibonacci查找而言，查找长度为O(1.44logN)（实际递推公式参考《数据结构》一书）。

![最优的中轴点推导](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200229135020_a24axD.png)

上图证明斐波那契查找法平均查找长度最小，但是缺点是需要额外生成斐波那契数列。

#### 二分查找改进版（B）

```c++
template <typename T> static Rank binSearch(T* A, T const & e, Rank lo, Rank hi) {
  while (hi - lo > 1) { // 有效查找区间的宽度缩短为1时，算法就会终止
    Rank mid = (lo + hi) >> 1;
    (e < A[mid]) ? hi = mid : lo = mid;
  }
  return (e == A[lo]) ? lo : -1;
}
```

这样做将向左和向右的成本都降为1，最好（坏）的情况变坏（好），整体性能更趋稳定

但是上述的都没有满足之前提到的约定的语义：**在有序向量`V[lo, hi)`中，确定`不大于e的最后一个`元素**

#### 二分查找改进版（C）

```c++
// 满足约定的语义
template <typename T> static Rank binSearch(T* A, T const& e, Rank lo, Rank hi) {
  while (lo < hi) {
    Rank mid = (lo + hi) >> 1;
    (e < A[mid]) ? hi = mid : lo = mid + 1;
  }
  return --lo;
}
```

##### 算法证明

可行性：需证明`A[0, lo) <= e < A[hi, n)`

![image-20200301105915755](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200301105920_image-20200301105915755.png)

#### 插值查找

假设数组内的元素是有序且**按照某种规律随机分布**

假设按照线性分布的规律，则中轴点的计算方法为
$$
mi = lo + (hi - lo) * \frac{(e - A[lo])}{(A[hi]-A[lo])}
$$

```c++
template <typename T> static Rank interpolate(T* A, T const & e, Rank lo, Rank hi) {
  while (lo < hi) {
    Rank mi = lo + (hi - lo - 1) * (e * 1.0 - A[lo])/(A[hi - 1] - A[lo]);
    if (mi < lo) return mid; // 在数组内部不存在的时候返回mid
    if (e < A[mi]) hi = mi;
    else if (A[mi] < e) lo = mi + 1;  // 此处采用了 mi + 1，因为这样才不会导致mi == lo 时无限循环
    else return mi;
  }
  return --lo; // 当查找不到的时候返回上一个值
}
```

经过证明可知每次插值都会将规模缩小为$$\sqrt{n}$$，则对比与二进制数位来说相当于砍了一半，即为$$\frac{1}{2}\log_2{n}$$

同样经过$$\log{n}$$步，所以时间复杂度为O($$\log{n}\log{n}$$)

**缺点：插值查找很容易受到干扰，尤其是在向量不满足线性分布规律的前提下，因此其仅适用于非常大的搜索区间范围内。当搜索空间变小后，可以使用二分查找（搜索空间中等）或者顺序查找（搜索空间较小**

## 排序

### 起泡排序

```c++
// 最简单版本 O(n^2)
void bubble(T* A, int lo, int hi) {
  for (int i = hi - 1; i >= lo; i--) {
    for (int j = lo; j < i; j++) {
      if (A[j] > A[j + 1]) swap(j, j+1);
    }
  }
}
```

优化：在逐趟扫描交换过程中，如果某一趟没有发生逆序对的扫描交换，则表明已经有序，直接跳出

```c++
void bubble(T* A, int lo, int hi) {
  for (int i = hi; i >= lo; i--) {
    int sorted = 1;
    for (int j = lo + 1; j < hi; j++) {
      if (A[j-1] < A[j]) {
        swap(j-1, j);
        sorted = 0;
      }
    }
    if (sorted) return;
  }
}
```

再优化：在以下情况中，如果后面红色的部分已经有序，则实际上是将前面绿色的部分（长度为r）进行排序，总时间为$$O(nr)$$ ，如果能有一种技巧检测出有序的情况，使得实际排序的序列为$$\sqrt{n}$$，则实际的时间复杂度为$$O(n)$$

![image-20200301141702155](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200301144202_image-20200301141702155.png)

```c++
void bubble(T* A, int lo, int hi) {
  int last;    
  for (int i = hi; i > lo; i = last) {
    last = lo; // 注意每次循环都要初始化逆序对为[lo - 1, lo]
    for (int j = lo + 1; j < i; j++) {
      if (A[j-1] > A[j]) {
        last = j;
        swap(j-1, j);
      }
    }
  }
}
```

#### 分析

冒泡排序的最后为$$O(n)$$，最坏为$$O(n^2)$$ ，上述优化只是针对一般情况而言。

稳定性：冒泡排序是稳定的

### 归并排序

前提：在基于比较的排序算法中，最好的时间复杂度为$$\Omega({n\log{}})$$

思路：基于分治的思想，将要排序的向量一分为二，直到有序，然后再分别合并。主要问题就是如何将两个有序的序列进行合并

时间复杂度：$$T(n) = T(n/2) + O(n)$$，可知时间复杂度为$$O(n\log(n))$$。归并排序的最好、最坏时间复杂度均为$$O(nlog(n))$$

```c++
void mergeSort(T* A, int lo, int hi) { // [lo, hi)
  if ((hi - lo) < 2) return;
  int mid = (lo + hi) >> 1;
  mergeSort(A, lo, mid);
  mergeSort(A, mid, hi);
  merge(A, lo, mid, hi);
}

void merge(T* A, int lo, int mid, int hi) {
  int lb = mid - lo;
  T* B = new T[lb];
  for (int i = 0; i < lb; i++) B[i] = A[lo + i];
  int lc = hi - mid; T* C = A + mid;
//  for(int i = lo, j = 0, k = 0; (j < lb) || ( k < lc);) {
//    if ((j < lb) && ((lc <= k) || (B[j] <= C[k])) A[i++] = B[j++];
//    if ((k < lc) && ((lb <= j) || (C[k] < B[j]))	) A[i++] = C[k++];
//  }
  for(int i = lo, j = 0, k = 0; j < lb;) {
    A[i++] = (lc <= k || B[j] <= C[k]) ? B[j++] : C[k++];
  }
}

// TODO: 归并排序自下而上实现
```

## 总结

在给向量规定的ADT中，有几种操作

- 去重：区间去重，单个去重($$O(n^2)$$)，排序后进行去重（$$O(n)$$)
- 查找：顺序查找O(n)，二分查找$$O(log(n))$$ (两种改进版，斐波那契查找)，插值查找
- 排序：冒泡排序（两种优化措施），归并排序