---
title: 数据结构-列表
date: 2020-04-15 10:00:03
categories: 数据结构
toc: true
---
《数据结构》列表笔记
<!--more-->

### ADT接口

#### 列表节点

![列表节点ADT](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200304174033_image-20200304174030188.png)

```c++
#define Posi(T) ListNode<T>* // 列表节点位置(template alias)
template <typename T>
struct ListNode {
  T data; // 数值
  Posi(T) pred;
  Posi(T) succ;
  ListNode() {}
  ListNode(T e, Posi(T) p = NULL, Posi(T) s = NULL)
    : data(e), pred(p), succ(s) {}
  Posi(T) insertAsPred(T const & e);
  Posi(T) insertAsSucc(T const & e);
}
```

#### 列表

![列表ADT—1](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200304174516_image-20200304174514421.png)

![列表ADT-2](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200304174547_image-20200304174544825.png)

```c++
template <typename T> class List {
private:
  int _size; Posi(T) header; Posi(T) trailer;
protected:
  void init() {
    header = new ListNode<T>();
    trailer = new ListNode<T>();
    header->succ = trailer; header->pred = NULL;
    trailer->pred = header; trailer->succ = NULL;
    _size = 0;
  }
}
```

### 循秩访问

```c++
template <typename T>
T List<T>::operator[](Rank r) const { // 效率为O(r), 不建议常用
  Posi(T) p = first(); // 从首节点出发
  while (0 < r--) p = p->succ;
  return p->data;
}
```

### 查找

```c++
template <typename T> // 在p的n个前驱中找到e，从外部调用时，0<= n <= rank(p) < _size;
Posi(T) List<T>::find(T const & e, int n, Posi(T) p) const {
  while (0 < n--) {
    if (e == (p = p->pred)->data) return p;
  }
  return NULL;
}
```

### 插入

```c++
template <typename T>
Posi(T) List<T>::insertBefore(Posi(T) p, T const &e) {
  _size++; return p->insertAsPred(e); 
}

template <typename T> // 作为该节点的前驱插入
Posi(T) ListNode<T>::insertAsPred(T const &e) {
  Posi(T) x = new ListNode(e, pred, this);
  pred->succ = x; pred = x; return x;
}

template <typename T> // 基于复制的构造
void List<T>::copyNodes(Posi(T) p, int n) {
  init();
  while (n--) {
    insertAsLast(p->data); p = p->succ;
  }
}

template <typename T> // 作为最后元素插入
Posi(T) List<T>::insertAsLast(T const & e) {
  return insertBefore(trailer);
}
```

### 删除

```c++
template <typename T>
T List<T>::remove(Posi(T) p) {
  T e = p->data;
  p->pred->succ = p->succ;
  p->succ->pred = p->pred;
  delete p; _size--; return e;
}
```

### 析构

```c++
template <typename T>
List<T>::~List() {
  clear(); delete header; delete trailer;
}

template <typename T> int List<T>::clear() {
  int oldSize = _size;
  while (0 < _size--) {
    remove(header->succ); // 反复删除首节点
  }
  return oldSize;
}
```

### 去重

```c++
template <typename T> int List<T>::deduplicate() {
  if (_size < 2) return 0;
  int oldSize = _size;
  Posi(T) p = first(); Rank r = 1; // p从首节点开始
  while (trailer != (p = p->succ)){ // 访问到末尾
    Posi(T) q = find(p->data, r, p);
    q ? remove(q) : r++;
  }
  return oldSize - _size;
}
```

### 有序列表

#### 去重

```c++
// 算法复杂度为O(n)
template <typename T> int List<T>::uniquify() {
  if (_size < 2) return 0;
  int oldSize = _size;
  Posi(T) p = first(), *q; // q 一直作为p的后继
  while (trailer != (q = p->succ)) {
    if (p->data != q->data) p = q;
    else remove(q);
  }
  return oldSize - _size;
}
```

#### 查找

```c++
// 找到p的n个前驱中不大于e的节点
template <typename T> Posi(T) List<T>::search(T const & e, int n, Posi(T) p) {
  while (0 < n--) {
    if ((p=p->pred)->data <= e) break;
  }
  return p;
}
// 时间复杂度为O(n)，因为链表是基于位置访问；所以不能使用二分查找将时间复杂度降为O(log2n)
```

### 选择排序

类似挑苹果，每次从盆里挑出最大的，然后放到桌子上。每一次做的就是选择，基于选择的排序就是选择排序

起泡排序就是选择排序的一种，但是效率太低（需要一步一步将最大的数移动到有序区域到分界点）。所以在列表中，可以直接移动，可以提高效率

时间复杂度为$$O(n^2)$$ ，最好情况下

```c++
template <typename T> void List<T>::selectionSort(Posi(T) p, int n) {
  Posi(T) head = p->pred; Posi(T) tail = p;
  for(int i = 0; i < n; i++) tail = tail->succ; // tail指向有序区的起始节点
  while (n > 1) {
    insertBefore(tail, remove(selectMax(head->succ, n)));
    tail = tail->pred; n--;
  }
}

template <typename T> Posi(T) List<T>::selectMax(Posi(T) p, int n) {
  Posi(T) max = p;
  while (1 < n--) { // n > 1
    if(max->data <= (p = p->succ)->data) max = p;
  }
  return max;
}
```

![选择排序实现思路](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200304190030_image-20200304190019452.png)

### 插入排序

```c++
template <typename T> void List<T>::insertionSort(Posi(T) p, int n) {
  for(int r = 0; r < n; r++) {
    insertAfter(search(p->data, r, p), p->data);
    p = p->succ; remove(p->pred);
  } // n次迭代，每次O(r+1)
}
```

时间复杂度为$$O(n^2)$$，最好情况下只需一次比较，时间复杂度为$$O(n)$$，最坏情况为$$O(n^2)$$

#### 平均性能

![平均性能](http://ziangleiblog.oss-cn-beijing.aliyuncs.com/uPic/20200306233639_image-20200306233636909.png)

#### 逆序对

在插入过程中，我们需要对之前排序好的序列进行查找，然后插入；则插入的位置之后的所有有序元素均与被插入的元素组成逆序对；因此插入的元素向前移动了多少，它的逆序对就有多少；