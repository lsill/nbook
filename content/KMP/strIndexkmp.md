---
title: "kmp实现strings.Index"
date: 2021-12-01T17:37:48+08:00
draft: true
---



[28.实现strStr()](https://leetcode-cn.com/problems/implement-strstr/)

​	这道题我发现golang里面的内置函数strings.Index好像就是这种写法，不过写的更加的通用和难理解

​	先不追求那么高大上的解，这道题不用kmp照样可以解出来，而且时间复杂度和空间复杂度也没那么差，先考虑以下的一些特殊情况吧

- 当len(needle)等于0的时候，空字符串是所有字符串的子集，所以直接返回0就行
- 当需要查询的字符串haystact长度小于目标字符串needle的时候，在haystack一定找不到needle直接返回-1就可
- 当len(needle) == 1的时候，就是直接在haystact中找到第一个相等的字节返回对应索引就行了

  以上就是需要提前考虑的比较特殊的情况了，在go中字符串是不可写的字符切片，虽然不可写，但我们可以直接读啊。

假如我们在字符串haystact的 i 位置找到与needle[0]相等的字节，那么如果这里就是与needle开始相等的位置，那么haystact[i:len(needle)]就应该与needle相等，那么解就来了。

```go
func strStr(haystack string, needle string) int {
	n := len(needle)
	if n == 0 {
		return 0
	}
	if len(haystack) < n {
		return -1
	}
	if n == 1 {
		for i := 0; i < len(haystack); i++ {
			if needle[0] == haystack[i] {
				return i
			}
		}
		return -1
	}
	for i := 0; i < len(haystack); i++ {
		if haystack[i] == needle[0] {
			end := i + n
			if end > len(haystack) {
				return -1
			}
			if haystack[i:end] == needle {
				return i
			}
		}
	}
	return -1
}

```



​	但做题不是实际的生产环境，我们追求的是锻炼发散的思维，所以，即便可以解出来我们也需要尝试下kmp，看起来就很高大上的算法，总要看下。



