---
title: "煎饼排序"
date: 2022-02-22T17:21:48+08:00
draft: false
---



​	在leetcode刷题的时候偶然看到了一道新的排序算法题，平常时候根本没用到过，比较新颖做个笔记

​	[原题](https://leetcode-cn.com/problems/pancake-sorting/)

leetcode题解的时候指出了如果求找到**最短解法是 NP 困难的**，大概搜了下，也没找到具体用处。

大概思路：

leetcode 举例子的解不太容易能引导出正确解

```
输入：[3,2,4,1]
输出：[4,2,4,3]
解释：
我们执行 4 次煎饼翻转，k 值分别为 4，2，4，和 3。
初始状态 arr = [3, 2, 4, 1]
第一次翻转后（k = 4）：arr = [1, 4, 2, 3]
第二次翻转后（k = 2）：arr = [4, 1, 2, 3]
第三次翻转后（k = 4）：arr = [3, 2, 1, 4]
第四次翻转后（k = 3）：arr = [1, 2, 3, 4]，此时已完成排序。 
```

其中省略了一些步骤：

完本的顺序应该是：

```
[4 2 3 1] // 将最大值4先翻到最上面
[1 3 2 4]	// 将最大值4在翻到最下面，此时只需要关心子集arr[0:n-1]
[3 1 2 4]	// 将 子集中最大值3翻到上面
[2 1 3 4]	// 将子集中最大值3翻到下面，此时只需要关心自己arr[0:n-2]
[2 1 3 4]	
[1 2 3 4]
```

图解：

![pancake](https://github.com/lsill/nbook/blob/dev/static/images/sort/pancake.png?raw=true)



代码：

```go
func PancakeSort(arr []int) (ans []int) {
	for n := len(arr); n > 1; n-- {
		index := 0
		for i, v := range arr[:n] {
			if v > arr[index] {
				index = i
			}
		}
		if index == n-1 {
			continue
		}
		for i, m := 0, index; i < (m+1)/2; i++ {
			arr[i], arr[m-i] = arr[m-i], arr[i]
		}
		for i := 0; i < n/2; i++ {
			arr[i], arr[n-1-i] = arr[n-1-i], arr[i]
		}
		ans = append(ans, index+1, n)
	}
	return
}
```



