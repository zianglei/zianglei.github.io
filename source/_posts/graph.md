---
title: 数据结构-图
date: 2020-04-20 09:00:00
categories: 数据结构
toc: true
---
关于图这种数据结构的一些知识
<!--more-->

顶点集V，边集E，相连的两个点互相**邻接**，顶点与边相互**关联**。

无向图/有向图：边无/有方向

路径：$\pi = <V_0,V_1,...V_k>$，长度$|\pi|=k$

简单路径：$V_i = V_j 除非 i = j$

简单图：没有两条边，它们所关联的两个点都相同

欧拉环路：经过所有的边一次且恰好一次的环路（例如一笔画问题）。

哈密尔顿环路：经过所有的点一次且恰好一次的环路。

边数与度数的关系：$E = \frac{1}{2}\sum{D(di)}$

## 数据结构表示

```c++
template <typename Tv, typename Te> class Graph {
private:
  void reset() {
    for(int i = 0; i < n; i++) {
      status(i) = UNDISCOVERED; dTime(i) = fTime(i) = -1;
      parent(i) = -1; priority(i) = INT_MAX;
      for(int j = 0; j < n; j++) {
        if(exists(i, j)) status(i, j) = UNDETERMINED;
      }
    }
  }
public:
  /*....*/
}
```

### 顶点类

```c++
typedef enum {UNDISCOVERED, DISCOVERED, VISITED} VStatus;
template <typename Tv> struct Vertex {
  Tv data; int inDegree, outDegree;
  VStatus status; // 状态
  int dTime, fTime; // 时间标签
  int parent; // 在遍历树中的父节点
  int priority; // 在遍历树中的优先级
  Vertex (Tv const & d) :
  	data(d), inDegree(0), outDegree(0), status(UNDISCOVERED),
  	dTime(-1), fTime(-1), parent(-1), priority(INT_MAX) {}
};
```

### 边类

```c++
typedef enum {UNDETERMINED, TREE, CROSS, FORWARD, BACKWARD} EStatus;
template <typename Te> struct Edge {
  Te data;
  int weight;
  EStatus status;
  Edge(Te const &d, int w):
  	data(d), weight(w), status(UNDETERMINED) {}
};
```

### 邻接矩阵

```c++
template <typename Tv, typename Te> class GraphMatrix: public Graph<Tv, Te> {
  private:
  	Vector<Vertex<Tv>> V; //顶点集
  	Vector<Vector<Edge<Te>*>> E; //边集
  public:
  	GraphMatrix() { n = e = 0;}
 		~GraphMatrix() {
      for(int j = 0; j < n; j++) {
        for(int k = 0; k < n; k++) {
          delete E[j][k];
        }
      }
    }
};
```

#### 邻居操作

```c++
int nextNbr(int i, int j) {
  while ((-1 < j) && !exist(i, --j)); // 逆向顺序查找，方便查找不到时返回-1，通知调用者查找已经结束
  return j;
} 
int firstNbr(int i) {
  return nextNbr(i, n);
} // 首个邻居
```

#### 边操作

```c++
bool exists(int i, int j) {
  return (0 <= i) && (i < n) && (0 <= j) && (j < n) && E[i][j] != NULL; // 短路求值
}

void insert(Te const * edge, int w, int i, int j) {
  if (exists(i, j)) return;
  E[i][j] = new Edge<Te>(edge, w);
  e++;
  V[i].outDegree++;
  V[j].inDegree++;
}

Te delete(int i, int j) {
  if (!exists(i, j)) return;
  Te eBak = edge(i, j);
  delete E[i][j]; E[i][j] = NULL;
  e--;
  V[i].outDegree--;
  V[j].inDegree--;
  return eBak;
}
```

#### 顶点操作

```c++
int insert(Tv const & vertex) {
  for (int i = 0; j < n; j++) E[j].insert(NULL); n++;
  E.insert(Vector<Edge<Te>*>(n, n, NULL));
  return V.insert(Vertex<Tv>(vertex));
}

int remove(int i) {
  for (int j = 0; j < n; j++) {
    if (exists(i, j)) // 删除所有出边
      delete E[i][j]; V[j].inDegree--;
  }
  E.remove(i); n--;
  for (int j = 0; j < n; j++) {
    if (exists(j, i)) // 删除所有入边
      delete E[j].remove(i); V[j].outDegree--;
  }
  Ev vBak = vertex(i);
  V.remove(i);
  return vBak;
}
```

#### 邻接矩阵的优缺点

- 优点：判断关联，获得出入度数和删除度数时间复杂度都是常数，并且有良好的扩展性（适合稠密图）
- 缺点：O(n^2)空间，与实际边数无关

考察平面图（能够绘制在足够大的纸上），不相邻的边不会相交，这样的图满足欧拉公式（v - e + f - c = 1)

v为顶点数，e为边数，f为区域面数，c为连通域数（通常为1，因此欧拉公式可以变为v+f = e + 2)，则

$$e \leq 3 * n - 6 = O(n) << n^2$$，即整个空间的利用率约为1/n，

## 遍历

遍历是将非线形结构（半线形结构）转换为序列（线形结构）的常规方法

### 广度优先搜索（BFS）

先访问顶点S -> 访问顶点S的所有邻接顶点 -> 访问这些邻接顶点的未访问过的邻接顶点 -> ... -> 所有点都被访问

访问过后，所有访问过的边就形成了一个树，叫做**支撑树(Spanning Tree)** 

树的层次遍历是BFS的一个特例

```c++
template <typename Tv, typename Te>
// 顶点的状态：UNDISCOVERED -> DISCOVERED -> VISITE
void Graph<Tv, Te>::BFS(int v, int & clock) {
  Queue<int> Q; status(v) = DISCOVERED; Q.enqueue(v);
  while (!Q.empty()) {
    int v = Q.dequeue(); dTime(v) = ++clock;
    for (int u = firstNbr(v); -1 < u; u = nextNbr(v)) {
      /* 根据u的状态，分别处理 */
      if (UNDISCOVERED == status(u)) {
        status(u) = DISCOVERED; Q.enqueue(u);
        status(v, u) = TREE; parent(u) = v;
      } else {
        // 若u已被发现，则将（v,u)归类为跨边
        status(v, u) = CROSS;
      }
      status(v) = VISITED;
    }
  }
}
```

对于有多个连通域的图，可以对未遍历到的点再启动几次bfs

```c++
template <typename Tv, typename Te>
void Graph<Tv, Te>::bfs(int s) {
  reset(); int clock = 0; int v = s;
  do {
    if (UNDISCOVERED = status(v)) {
      BFS(v, clock);
    }
  } while (s != (v = (++v % n))); // 按序号访问
}
```

时间复杂度分析：

外层的while循环有O(n)，内层的for循环也有O(n)，同时内循环的次数为边数e（？？），所以总时间复杂度为O(n * n + e)，但是因为内层的for循环的常数非常小（邻接矩阵的某一行几乎总是处于高速缓存中），所以时间复杂度为O(n + e)，换成邻接表时间复杂度更显然为O(n + e)

#### 最短路径

BFS搜索发现的顶点顺序是按照深度的非降次顺序排列的，所以BFS发现的通路恰好是从顶点开始到该顶点的最短通路。

### 深度优先搜索（DFS）

自然语言描述：访问顶点S，若S尚有未被访问的邻居，则任取其一u，递归执行DFS(u)，否则返回

```c++
template <typename Tv, typename Te>
void Graph<Tv, Te>::DFS(int v, int & clock) {
  dTime(v) = ++clock; status(v) = DISCOVERED;
  for(int u = firstNbr(v); -1 < u; u = nextNbr(v)) {
    switch(status(u)) {
      case UNDISCOVERED: // u尚未发现
        status(v,u) = TREE; parent(u) = v; DFS(u, clock); break; // di gui 
      case DISCOVERED: // u已被发现但尚未访问完毕，
        status(v, u) = BACKWARD; break;	// 如果产生BACKWARD，证明有环路产生
      default: // u已经访问完毕（VISITED, 有向图），则视继承关系分为前向边或跨边
        	status(v, u) = dTime(v) < dTime(u) ? FORWARD : CROSS; break;
    }
  }
  fTime(v) = ++clock; status(v) = VISITED;
}
```

#### 括号引理

定义顶点的活动期为 active[u] = ( dTime[u], fTime[u] )

则给定有向图 G = (V, E) 及其任一DFS森林，则

- u是v的祖先 iff $active[u] \subseteq active[u]$
- u与v无关 iff $active[u] \cap active[v] = \oslash$

可以用此判断u和v是否有直系关系

#### 两者对比

BFS和DFS产生的边数是一样的
