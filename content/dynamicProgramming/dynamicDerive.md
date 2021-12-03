---
title: "动态规划思路"
date: 2021-12-02T18:36:15+08:00
draft: false
---

#### 动态规划问题的的通用推导

- 我该如何确定我要的Dp数组
- 怎么推导递推公式
- dp数组怎么初始化
- 遍历顺序从下开始还是从上
- 代码如何写更好

#### 代码无法ac的自证

- 我有推导状态转移公式了吗
- 我打印dp数组的日志了吗
- 打印出来的dp数组和我想象的一样了吗

##### 例题1：

菲波那切数列

维基百科解释：用文字来说，就是斐波那契数列由0和1开始，之后的斐波那契数就是由之前的两数相加而得出。首几个斐波那契数是：0,1,1,2,3,5,8,13...

**解：**

```golang
func Fib(n int) int {
	if n == 0 {
		return 0
	}
	if n == 1 {
		return 1
	}
	// 确定dp数组
	dp := make([]int, n+1)
	// 初始化dp数组
	dp[0] = 0
	dp[1] = 1
	// 一维遍历
	for i := 2; i <= n; i++ {
		// 状态转移方式确定
		dp[i] = dp[i-1] + dp[i-2]
	}
	return dp[n]
}
```

 做leetcode的时候顺手就这么写了，空间复杂度占用较高，从上述代码可以看到 我们完全可以维护两个常数，用来代表dp数组

```golang
func fib(n int) int {
	if n == 0 {
		return 0
	}
	if n == 1 {
		return 1
	}
	// dp[2], dp[0] = 0, dp[1]=1
	pre, next := 0, 1
	for i := 2; i <= n; i++ {
		tmp := pre + next
		pre = next
		next = tmp
	}
	return next
}
```


leetcode [70.爬楼梯](https://leetcode-cn.com/problems/climbing-stairs/) 是上述菲波那切数列的变体，代码都不需要有任何改变

[746. 使用最小花费爬楼梯 ](https://leetcode-cn.com/problems/min-cost-climbing-stairs/) 是上题的进阶，我做的时候有点钻牛角尖为什么到达顶点没有任何消耗，也算是没有理解题意。

 1.从题意可得，长度为n的楼梯，从n-1或者n-2到达n都不消耗任何的体力。

 2.构造dp数组，长度为n的台阶，那么初始化dp = make([]int, n)

 3.初始化，我如果要到达i=0这个台阶我只有一种可能，那么很明显就是dp[0]=cost[0]，同理，到达i=1台阶有两种方式，一种是走两步 0、1，一种是直接走两步到2，很明显dp[1] = cost[1]< dp[1] + d[0]

 4.我如果想要到达i=2的楼梯，我只能从i=0或者i=1上出发，那么我选择消耗体力小的，就是我到达i=2消耗的最小体力，也就是dp[2]=dp[1] > dp[0]?dp[0]:dp[1]，从这里就可以推导出状态转移方程dp[i]= dp[i-1] > dp[i-2]?dp[i-2]:dp[i-1]
 
 5.因为i阶梯只能前两个阶梯相关，所以可以用两个常数代替dp数组

解：

```
func MinCostClimbingStairs(cost []int) int {
	n := len(cost)
	// 确认dp数组 dp[n-1]
	pre, next := cost[0], cost[1]
	for i := 2; i < n; i++ {
		// 状态转移公式 dp[i] = dp[i-1] > dp[i-2] ? dp[i-2]:dp[i-1]
		tmp := cost[i]
		if pre > next {
			tmp += next
		} else {
			tmp += pre
		}
		pre = next
		next = tmp
	}
	if next > pre {
		return pre
	}
	return next
}
```




