---
title: "匹配队列的实现"
date: 2022-01-13T12:13:48+08:00
draft: true
---

## 1. 程序设计定理

​	在写功能的时候我们需要考虑的不仅仅只是如何去实现策划的需求，怎么把功能给写出来，还要考虑后面的复用性可读性。复用性是为了少些代码，类似的功能其实写好了就可以让策划通过配表自己实现，这是很重要的，这不仅仅是减少工作量，而且是为了减少bug，**只要不写代码就不会有bug**。可读性是为了后面人接手，毕竟我们都不想自己的代码被称作“祖传屎山”。



## 2. 匹配的机制

​	目前流行的匹配机制一个是组成一个队伍（可以不满员匹配过程中填充其他队员），这个队伍去对抗怪物（`pve`）或者玩家（`pvp`），这和传统`mmogpg`组队最大的区别就是减少了队伍组队等待队员的过程，快节奏的时代嘛。

 	匹配我大致将它分为两种：

- 为了找队友：这是一般手游的做法，手游一般轻社交，等队伍满员才能开始副本节奏就很慢。表现就是1到n个人准备后就直接开始了，然后满员就开始游戏。

- 为了找对手：这种更加常见，`lol`的匹配就是这样的，当然这种也包含了找队友的实现。



## 3. 寻找队友实现方案

​	这里假设我们已经实现了队伍机制，并且队伍最大规模为4

流程图如下：

![match](/Users/lsill/Documents/match_pic.png)

	在寻找队友的过程中，我们一般考虑的是还需要多少个人才能开始这句游戏。

| 队伍人数 | 需要队员数量 | 匹配方案 |
|:------|:------|:-------|
| 4 | 0 | 0(满员直接开始游戏) |
| 3 | 1 | 1:可以从1的等待队列寻找 |
| 2 | 2 | 2:可以从2的等待队列寻找，也可以从1的等待队列找2个1 |
| 1 | 3 | 3:可以从3的等待队列找，也可以从2和1的等待队列各找一个，从1的等待队列找三个|


实现伪码
```
type numMatch func(sm *SingleMatch, teams []*teaming.Team) bool

// matchFuncTable 还需要多少人就去匹配，目前最多是4人的，以后出现5人匹配在加就行
var matchFuncTable = map[int]numMatch{
	0: nil,
	1: needOneMatch,   // 就差一个人开车
	2: needTwoMatch,   // 差2人开车
	3: needThreeMatch, // 还差3个人才能开车
}

// needThreeMatch 还需要三个人
func needThreeMatch(sm *SingleMatch, teams []*teaming.Team) bool {
	if 等待队列找到 {
		return true
	}
	return false
}

// needTwoMatch 还需要两个人
func needTwoMatch(sm *SingleMatch, teams []*teaming.Team) bool {
	if 等待队列找到 {
		return true
	}
	return false
}

// needOneMatch 还需要一个人
func needOneMatch(sm *SingleMatch, teams []*teaming.Team) bool {
	if 等待队列找到 {
		return true
	}
	return false
}

type SingleMatch struct {
	timeout  time.Duration // 超时时间
	duration time.Duration // 等待持续时间

	finder map[int64]*list.Element
	teams  []*list.List
}

func (sm *SingleMatch) Match(team *teaming.Team) bool {
	count := team.Count()
	needNum := sm.MemberSize() - count
	if needNum < 0 {
		return false
	}
	if needNum == 0 {
		匹配成功()
		return true
	}
	funcMatch, ok := matchFuncTable[needNum]
	if !ok {
		return false
	}
	if funcMatch(sm, []*teaming.Team{team}) {
		return true
	}
	// 加入匹配队列
	sm.finder[team.TeamId()] = sm.teams[count-1].PushBack(team)
	return true
}


```


