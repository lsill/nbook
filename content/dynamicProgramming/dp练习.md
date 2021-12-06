---
title: "dp练习"
date: 2021-12-06T18:19:15+08:00
draft: false
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

这道题感觉一下子就难了好多，想了一段时间

还是按照公式列出

- 虽然给出了obstacleGrid二维数组，但有条件限制，还是在申请一个dp数组（给出的应该可以复用吧）
- 已知obstacleGrid[i] [j] == 1的话，就是不可到达，那么obstacleGrid[0] [0] == 1就可以直接返回了，出发点无法到达，那么后面的都无法到达。
- obstacleGrid[i] [j] ！= 1， 那么设置dp[0] [0] == 1,如果obstacleGrid[i] [j] == 0, 那么dp[i] [j] = 0。由出发点向右或者向下， 都只有一种可能性, dp[0] [j] = dp[0] [j-1],dp[i] [0] = dp[i-1] [0]。
- 遍历
- 写的比较糟糕，不好看 ，可以优化

解：

```
func UniquePathsWithObstacles3(obstacleGrid [][]int) int {
	m, n := len(obstacleGrid), len(obstacleGrid[0])
	// 起点不可到达，直接返回
	if obstacleGrid[0][0] == 1 {
		return 0
	}
	// 构造dp数组
	arr := make([][]int, m)
	for i := 0; i < m; i++ {
		arr[i] = make([]int, n)
	}
	// dp数组初始化，出发点只有一种可能性
	arr[0][0] = 1
	for i := 1; i < m; i++ {
		if obstacleGrid[i][0] == 1 {
			// 遇到障碍物直接返回
			arr[i][0] = 1
			break
		}
		// 只有一种走法
		arr[i][0] = arr[i-1][0]
	}
	for i := 1; i < n; i++ {
		if obstacleGrid[0][i] == 1 {
			// 遇到障碍物直接返回
			arr[0][i] = 0
			break
		}
		// 只有一种走法
		arr[0][i] = 1
	}
	// 遍历
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			if obstacleGrid[i][j] == 1 {
				arr[i][j] = 0
				continue
			}
			arr[i][j] = arr[i][j-1] + arr[i-1][j]
		}
	}
	return arr[m-1][n-1]
}
```

#### [343. 整数拆分](https://leetcode-cn.com/problems/integer-break/)

这道题一言难尽

首先还是套公式

- 从题意可以得到数组dp
- 由于n需要被当成数组下标，所以dp长度需要是n+1，dp[0] = 0, dp[1]=,dp[2]=2(因为1*1 < 2在被当乘数因子的时候会直接选择2),dp[3]=3（同理dp[2]）
- 这道题最难的部分就是状态转移方程了，那么开始分解4,4可以分解成2+2和1+3,5可以分解成3+2和2+3那么可以得到状态转移方程dp[n] = dp[i-2]*dp[2] > dp[i-3] * dp[3] (其实可以把4拿出来省去-2的步骤，但我懒的在写了)
- 遍历
- 优化：其实可以不用动态规划做的，但因为做完是专项训练，思路没有打开，还是自己太菜

解：(完全可以优化的)

```
func IntegerBreak(n int) int {
	if n == 2 {
		return 1
	}
	if n == 3 {
		return 2
	}
	dp := make([]int, n+2)
	dp[1] = 1
	dp[2] = 2
	dp[3] = 3
	for i := 4; i <= n; i++ {
		a2 := dp[i-2] * dp[2]
		a3 := dp[i-3] * dp[3]
		if a2 > a3 {
			dp[i] = a2
		} else {
			dp[i] = a3
		}
	}
	return dp[n]
}
```



