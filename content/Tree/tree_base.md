---
title: "Test"
date: 2021-12-15T17:19:48+08:00
draft: false
---



[原文连接](https://programmercarl.com/%E4%BA%8C%E5%8F%89%E6%A0%91%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80.html#%E5%85%B6%E4%BB%96%E8%AF%AD%E8%A8%80%E7%89%88%E6%9C%AC) 



[toc]

## 一. 二叉树的种类



### 1.满二叉树

![tree_man](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_man.png?raw=true)

如图所示，叶节点数全为0或者全为2的节点，并且所有叶节点数为0的节点都在同一层，这颗二叉树就是满二叉树，假设深度为k，则有**2^k-1**节点的二叉树









### 2.完全二叉树

![tree_woner](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_wonder.png?raw=true)

​	完全二叉树：除了最底层节点可能没填满外，其余层节点数都达到最大值，并且左边节点为空的情况下，右边节点不能有值，必须先填满子树，才可以填右子树。最底层是第h层，则h层包含**1~2^h-1个节点**。

​	**优先级队列是一个堆，堆就是一颗完全二叉树，同时保证父子节点的顺序关系**。









### 3.二叉搜索树

![tree_search](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_search.png?raw=true)



如图所示：

- 二叉搜索树的每个节点都有值
- 如果它的左子树不为空，则左子树上面的所有节点的值都小于它的根节点的值
- 如果它的右子树不为空，则左子树上面的所有节点的值都大于它的根节点的值
- 它的左右子树也满足上面条件





### 4.  平衡二叉搜索树

​	平衡二叉搜索树：又被称为AVL（Adelson-Velsky and Landis）树，且具有以下性质：它是一棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树。

![tree_balance_search](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_balance_search.png?raw=true)

如图所示如果是平衡二叉搜索树必须满足以下几点：

- 必须是二叉搜索树
- 左右两个子树的高度差的绝对值不超过1

c++的map、set、multimap、multiset的底层实现都是平衡二叉搜索树，所以map、set的增删操作时间复杂度都是logn，但go的[map](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)和c++的unordered_map底层实现是哈希表。





## 二. 二叉树的存储方式

**二叉树可以链式存储，也可以顺序存储。**



### 1. 链式存储

​	链式存储的方式是再结构内存放左右节点的指针，通过指针可以找到下一节点的位置，因此链式存储可以散落的分布在内存中。

如图所示：

![tree_point](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_point.png?raw=true)

通过存储左右节点的地址，来访问左右节点，如果没有左右节点，值为nil。



### 2.顺序存储

顺序存储一般都会想到数组，数组是在内存开辟的一片连续的空间。

如果要用数组来表示二叉树，会用计算的方式来算出数组的下标，然后通过下标访问目标。

如图所示：

![tree_array](https://github.com/lsill/nbook/blob/main/static/images/tree/tree_array.png?raw=true)

- 数组表示的二叉树索引id为0的一定是二叉树的根节点
- 如果父节点的下标是i,那么左节点的下标就是 i * 2 + 1，右节点的下标是  i * 2 + 2





## 3. 二叉树的遍历方式

有二叉树基本就是递归，而递归一般就两种遍历方式，一种是深度优先遍历（dfs）,一种是广度优先遍历(bfs)

这里就说这两种遍历方式了，可以自行google

