title: 算法学习之减治法(decrease and conquer)
date: 2015-06-22 10:55:13
tags: [算法, 减治法]
categories: 算法学习
---

# 什么是分治法
减治技术利用了一个问题给定实例的解和同样问题较小实例的解之间的某种关系。一旦建立了这种关系，就可以从顶至下递归的来用该关系，也可以从底至上非递归的来运用该关系：
1. 减去一个常量
2. 减去一个常量因子
3. 减去的规模是可变的

# 分治法例子

## 减去一个常量

### 拓扑排序

#### 定义
定义：将有向图中的顶点以线性方式进行排序。即对于任何连接自顶点u到顶点v的有向边uv，在最后的排序结果中，顶点u总是在顶点v的前面。

#### 两张实现算法

##### Kahn算法

    L← Empty list that will contain the sorted elements
    S ← Set of all nodes with no incoming edges
    while S is non-empty do
        remove a node n from S
        insert n into L
        foreach node m with an edge e from n to m do
            remove edge e from the graph
            if m has no other incoming edges then
                insert m into S
    if graph has edges then
        return error (graph has at least onecycle)
    else 
        return L (a topologically sortedorder)

不难看出该算法的实现十分直观，关键在于需要维护一个入度为0的顶点的集合：
每次从该集合中取出(没有特殊的取出规则，随机取出也行，使用队列/栈也行，下同)一个顶点，将该顶点放入保存结果的List中。
紧接着循环遍历由该顶点引出的所有边，从图中移除这条边，同时获取该边的另外一个顶点，如果该顶点的入度在减去本条边之后为0，那么也将这个顶点放到入度为0的集合中。然后继续从集合中取出一个顶点…………
 
当集合为空之后，检查图中是否还存在任何边，如果存在的话，说明图中至少存在一条环路。不存在的话则返回结果List，此List中的顺序就是对图进行拓扑排序的结果。

[例子](http://7xjtfr.com1.z0.glb.clouddn.com/decreace_00.png)

对上图进行拓扑排序的结果：
2->8->0->3->7->1->5->6->9->4->11->10->12

**复杂度分析：**
初始化入度为0的集合需要遍历整张图，检查每个节点和每条边，因此复杂度为O(E+V);
然后对该集合进行操作，又需要遍历整张图中的，每条边，复杂度也为O(E+V);
因此Kahn算法的复杂度即为O(E+V)。

##### 基于DFS的拓扑排序

    L ← Empty list that will contain the sorted nodes
    S ← Set of all nodes with no outgoing edges
    for each node n in S do
        visit(n) 
    function visit(node n)
        if n has not been visited yet then
            mark n as visited
            for each node m with an edge from m to ndo
                visit(m)
            add n to L

DFS的实现更加简单直观，使用递归实现。利用DFS实现拓扑排序，实际上只需要添加一行代码，即上面伪码中的最后一行：add n to L。
需要注意的是，将顶点添加到结果List中的时机是在visit方法即将退出之时。
这个算法的实现非常简单，但是要理解的话就相对复杂一点。
关键在于为什么在visit方法的最后将该顶点添加到一个集合中，就能保证这个集合就是拓扑排序的结果呢？
因为添加顶点到集合中的时机是在dfs方法即将退出之时，而dfs方法本身是个递归方法，只要当前顶点还存在边指向其它任何顶点，它就会递归调用dfs方法，而不会退出。因此，退出dfs方法，意味着当前顶点没有指向其它顶点的边了，即当前顶点是一条路径上的最后一个顶点。

**复杂度分析：**
复杂度同DFS一致，即O(E+V)。具体而言，首先需要保证图是有向无环图，判断图是DAG可以使用基于DFS的算法，复杂度为O(E+V)，而后面的拓扑排序也是依赖于DFS，复杂度为O(E+V)
     
还是对上文中的那张有向图进行拓扑排序，只不过这次使用的是基于DFS的算法，结果是：
8->7->2->3->0->6->9->10->11->12->1->5->4

##### 两种实现算法的总结

这两种算法分别使用链表和栈来表示结果集。
对于基于DFS的算法，加入结果集的条件是：顶点的出度为0。这个条件和Kahn算法中入度为0的顶点集合似乎有着异曲同工之妙，这两种算法的思想犹如一枚硬币的两面，看似矛盾，实则不然。一个是从入度的角度来构造结果集，另一个则是从出度的角度来构造。
 
实现上的一些不同之处：
Kahn算法不需要检测图为DAG，如果图为DAG，那么在出度为0的集合为空之后，图中还存在没有被移除的边，这就说明了图中存在环路。
而基于DFS的算法需要首先确定图为DAG，当然也能够做出适当调整，让环路的检测和拓扑排序同时进行，毕竟环路检测也能够在DFS的基础上进行。
二者的复杂度均为O(V+E)。

## 减去一个常量因子

### Russian Peasant Multiplication
计算两个整数的积

#### 原理
n \* m = n/2 \* m  n为偶数
n \* m  = (n-1)/2 \* 2m + m  (if n > 1  and m if n = 1) n为奇数

#### 例子
![例子](http://7xjtfr.com1.z0.glb.clouddn.com/decreace_01.png)


## 减去的规模是可变的

### Selection Problem
找出数组中第n小的数

#### 原理
类似与快速排序,只是把key换做数组的下标,通过数的下标作为分割的依据

#### 例子
![例子](http://7xjtfr.com1.z0.glb.clouddn.com/decreace_02.png)

**参考资料:**
[减治法（一）](http://www.cnblogs.com/kkgreen/archive/2011/06/17/2083915.html)
[拓扑排序的原理及其实现](http://blog.csdn.net/dm_vincent/article/details/7714519)
