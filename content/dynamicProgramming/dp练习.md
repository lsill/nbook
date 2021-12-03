---
title: "dp练习"
date: 2021-12-02T18:36:15+08:00
draft: true
---

#### [62. 不同路径](https://leetcode-cn.com/problems/unique-paths/)

这道题 以前做过，还是一次性ac的，难度不高

还是通用思路来解

- 这道题很明显是二维的dp数组（题目都画出来了），如何到达目标dp[m-1] [n-1]就是题目的解
- 从dp[0] [0]出发到dp[0] [1]和dp[1] [0] 都只有1种走法，那么dp[0] [1] = 1,dp[1] [0] = 1，从dp[0] [1] 出发 到dp[0] [2] 也只有一种走法，那么dp[0] [2] =1,同理可以得到dp[i] [0] = 1,dp[0] [i] =1
- 如果我想要到达dp[1] [1]位置，我有两种走法，从dp[1] [0]到达和从do[0] [1] 到达，那么我到达dp[1] [1]的走法就是dp[1] [1] = dp[0] [1] + dp[1] [0],由此可以得到递推公式dp[i] [j] = dp[i-1] [j] + dp[i] [j+1]
- 暂时没想到更好的写法...

解：

```
func uniquePaths(m int, n int) int {
	dp := make([][]int, m)
	for i := 0; i < m; i++ {
		dp[i] = make([]int, n)
	}
	for i := 0; i < m; i++ {
		dp[m][0] = 1
	}
	for i := 0; i < n; i++ {
		dp[0][i] = 1
	}
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = dp[i][j-1] + dp[i-1][j]
		}
	}
	return dp[m-1][n-1]
}

```



#### [63. 不同路径 II](https://leetcode-cn.com/problems/unique-paths-ii/)

