---
title: "goland激活"
date: 2021-12-10T14:23:48+08:00
draft: false
---

### goland激活

[原教程连接](https://zhile.io/2021/11/29/ja-netfilter-javaagent-lib.html)

#### 1. 版本试用

goland2021.3以后（也适用于jb下面的所有产品）

#### 2.激活过程

[下载](https://jetbra.in/files/ja-netfilter-all.zip) 文件（不知名网络热心大佬的链接，可能失效）

- 下载goland2021.3+
- 出现账号激活选项时候选择active code 去 [不知名热心大佬的库 ](https://jetbra.in/s )找到对应的key（如果网页失效了直接百度一个）这个时候就可以进入试用了
- 选择工具栏最上面的help下面的Edit Custom VM options这个时候就打开了文件goland64.exe.vmoptions，增加-javaagent:你的绝对路径\ja-netfilter-all\ja-netfilter.jar （例子：-javaagent:E:\goland\ja-netfilter-all\ja-netfilter.jar）（mac或者linux直接pwd绝对路径就可以了）
- 如果写错了打不开goland不要紧，C:\Users\用户名\AppData\Roaming\JetBrains\GoLand2021.3下面也可以找到goland64.exe.vmoptions，修改即可。

