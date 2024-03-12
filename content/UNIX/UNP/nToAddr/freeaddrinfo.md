---
title: "freeaddrinfo 函数"
date: 2023-12-29T18:33:43+08:00
draft: true
---

由getaddrinfo返回的所有存储空间都是动态获取的(臂如来自malloc调用)，包括addrinfo结构、
ai_addr结构和ai_canonname字符串。这些存储空间通过调用freeaddrinfo返还给系统.

```c
#include <netdb.h>
void freeaddrinfo(struct addrinfo *ai);
```
ai参数应指向由getaddrinfo返回的第一个addrinfo结构。这个链表中的所有结构以及由它们指向的任何动态存储空间
(警如套接字地址结构和规范主机名)都被释放掉。

假设我们调用getaddrinfo，遍历返回的addrinfo结构链表后找到所需的结构。如果我们为保存其信息而仅仅复制这个
addrinfo结构，然后调用freeaddrinfo，那就引入了一个潜藏的错误。原因在于这个addrinfo结构本身指向动态分配
的内存空间(用于存放套接字地址结构和可能有的规范主机名)，因此由我们保存的结构指向的内存空间己在调用freeaddrinfo
时返还给系统，稍后可能用于其他目的。