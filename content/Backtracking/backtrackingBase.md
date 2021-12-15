---
title: "回溯理论"
date: 2021-12-15T14:35:48+08:00
draft: true
---



[原文](https://programmercarl.com/%E5%9B%9E%E6%BA%AF%E7%AE%97%E6%B3%95%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80.html) （基本照着抄一遍加深自己的印象）（先看树）

### 一. 回溯法定义

​	回溯法也可以叫做回溯搜索法（在所有可能中搜索可能的子集）。

​	回溯感觉就是递归的一种，在递归结束后还原状态



### 二. 回溯的效率

​	回溯我觉得很难，就算知道怎么做我很多时候也写不出来代码，然后回溯法本质上就是**穷举所有可能**，在穷举所有可能的时候进行剪枝，断掉某一部分的可能性来优化这种穷举。



### 三. 回溯能解决什么问题

- 组合问题：N个数里面按一定规则找出k个数的集合
- 切割问题：一个字符串按一定规则有几种切割方式
- 子集问题：一个N个数的集合里有多少符合条件的子集
- 排列问题：N个数按一定规则全排列，有几种排列方式
- 棋盘问题：N皇后，解数独等等

**组合是不强调元素顺序的，排列是强调元素顺序的。**

例如：{1, 2} 和 {2, 1} 在组合上，就是一个集合，因为不强调顺序，而要是排列的话，{1, 2} 和 {2, 1} 就是两个集合了。

记住组合无序，排列有序，就可以了。



### 四. 如何理解回溯法

​	**回溯法解决的问题都可以抽象为树形结构**。

​	因为回溯法解决的都是在集合中递归查找子集，**集合的大小就构成了树的宽度，递归的深度，都构成树的深度。**

​	递归就要有终止条件，所以必然是一颗高度有限的树（N叉树）。



### 五. 回溯法模版

- 回溯函数模版返回值以及参数

  ​	在回溯算法中，函数名一般为backtracking，函数返回值一般是void(无返回值)

  ​	再来看一下参数，因为回溯算法需要的参数可不像二叉树递归的时候那么容易一次性确定下来，所以一般是先写逻辑，然后需要什么参数就填什么参数。

  ​	回溯函数伪代码如下：

  ```go
  func backtracking(参数) ｛｝
  ```

- 回溯函数终止条件

  ​	既然是树形结构，那么遍历树形结构就一定要有终止条件。

  ​	所以回溯也要有终止条件。


