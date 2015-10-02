title: 算法学习之分治法(divide and conquer)
date: 2015-06-18 18:50:50
tags: [分治法,算法]
categories: 算法学习
---

## 什么是分治法
字面上的解释是“分而治之”，就是把一个复杂的问题分成两个或更多的相同或相似的子问题，直到最后子问题可以简单的直接求解，原问题的解即子问题的解的合并。

## 解决问题的流程

![divide and conquer](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer00.png)

## 分治法适用情况

    分治法所能解决的问题一般具有以下几个特征：

    1. 该问题的规模缩小到一定的程度就可以容易地解决

    2. 该问题可以分解为若干个规模较小的相同问题，即该问题具有最优子结构性质。

    3. 利用该问题分解出的子问题的解可以合并为该问题的解；

    4. 该问题所分解出的各个子问题是相互独立的，即子问题之间不包含公共的子问题。

第一条特征是绝大多数问题都可以满足的，因为问题的计算复杂性一般是随着问题规模的增加而增加；

第二条特征是应用分治法的前提它也是大多数问题可以满足的，此特征反映了递归思想的应用；、

第三条特征是关键，能否利用分治法完全取决于问题是否具有第三条特征，如果具备了第一条和第二条特征，而不具备第三条特征，则可以考虑用贪心法或动态规划法。

第四条特征涉及到分治法的效率，如果各子问题是不独立的则分治法要做许多不必要的工作，重复地解公共的子问题，此时虽然可用分治法，但一般用动态规划法较好。

## 应用问题

### 归并排序

可以看到排序问题正好满足以上提到的四个特征。
归并排序通过将待排序数组分为两个部分，递归地处理它们，最终将两个排序好的部分合并起来。
值得一提的是归并排序通过数组的下标分割子问题，即每次都把数组分为两半。

**伪代码如下**
![merge_sort](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer01.png)

![merge](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer02.png)

**例子**
![例子](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer03.png)

### 快速排序

跟归并排序的思想是类似的，但是快速排序是根据值来分割子问题，即把数组根据一个值分为大于和小于它两个部分。

**伪代码如下**

![quick_sort](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer04.png)

![partition](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer05.jpg)

**例子**

![例子](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer06.jpg)


### 二分搜索

对一个有序的数组进行二分搜索，每次比较中间的值，key大于它说明在右边，小于说明左边。

**伪代码如下**

![binary_search](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer07.png)

**例子**

![例子](http://7xjtfr.com1.z0.glb.clouddn.com/divide_and_conquer_divide_and_conquer08.png)


### 大整数乘法


对于任意位数的2个数相乘 $a * b$ ，写成：
{% math_block %}
a = a_{1} * 10^{(n_{1}/2)} + a_{0}
{% endmath_block %}
$n_{1}$为$a$的位数

{% math_block %}
b = b_{1} * 10^{(n_{2}/2)} + b_{0}　　　　　  
{% endmath_block %}
$n_{2}$为$b$的位数

分治策略就是基于以上变换，将a，b写成前一半数字和后一半数字相加的形式，例如若$a = 5423678$，那么$a\_{1} = 542$  $a\_{0} = 3678$（注意若不是偶数截取较小一半）

这样a和b相乘就可以写为：
{% math_block %}
a * b = { a_{1} * 10^{(n_{1}/2)} + a_{0} } * { b_{1} * 10^{(n_{2}/2)} + b_{0} }
{% endmath_block %}
展开后整理得：
{% math_block %}
a * b = a_{1}*b_{1} * 10^{[(n_{1}+n_{2})/2]} + a_{1}*b_{0} * 10^{(n_{1}/2)} + a_{0}*b_{1} * 10^{(n_{2}/2)} + a_{0}*b_{0}　
{% endmath_block %}
这样就很容易递归的来求$a * b$，如果嫌分解后的数还太大，就可以继续分解。（你可以自己规定在何时结束递归）

## 总结

分治法实际上就是类似于数学归纳法，找到解决本问题的求解方程公式，然后根据方程公式设计递归程序。
1. 一定是先找到最小问题规模时的求解方法
3. 然后考虑随着问题规模增大时的求解方法
3. 找到求解的递归函数式后（各种规模或因子），设计递归程序即可。


参考资料：

[五大常用算法之一：分治算法](http://www.cnblogs.com/steven_oyj/archive/2010/05/22/1741370.html)

[大整数乘法和Strassen矩阵乘法](http://www.cnblogs.com/kkgreen/archive/2011/06/12/2078668.html)
