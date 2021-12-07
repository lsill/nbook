---
title: "dp树相关练习"
date: 2021-12-07T14:19:15+08:00
draft: false
---

 [96. 不同的二叉搜索树](https://leetcode-cn.com/problems/unique-binary-search-trees/)

这道题获得思路不难，难的是去推导状态转移方程的时候推导的是错的，然后做的时候一直不对，然后不磨叽了，直接看答案了。

上图

![lc96.png](https://github.com/lsill/nbook/blob/main/static/images/lc/lc96.png?raw=true)

套公式

- 由题意可知一维数组就可以满足
- 当左子树或者右子树节点个数<=1的时候，可能性必为1
- dp[0] = 1,dp[1]=1,那么dp[2] = dp[1] * dp[0] + dp[0] *dp[1] (此处一定为乘法，我用了加法推导错误), 因为是 [二叉搜索树](https://baike.baidu.com/item/%E4%BA%8C%E5%8F%89%E6%90%9C%E7%B4%A2%E6%A0%91/7077855?fr=aladdin) 所以当n==3时候根节点为3,左子树必为1,2组成的，根节点为2的时候左子树是1，右子树是2，进一步推导

```golang
		for j := 1; j <= i; j++ {
			dp[i] += dp[j-1] * dp[i-j]
		}
```

- 遍历
- 优化：可以用数学做法和直接算出所有答案

解：

```go
func NumTrees(n int) int {
	dp := make([]int, n+1)
	dp[0] = 1
	dp[1] = 1
	for i := 2; i <= n; i++ {
		for j := 1; j <= i; j++ {
			dp[i] += dp[j-1] * dp[i-j]
		}
	}
	return dp[n]
}

```

