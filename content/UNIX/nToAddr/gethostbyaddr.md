---
title: "gethostbyaddr函数"
date: 2023-12-29T16:30:13+08:00
draft: true
---

gethostbyaddr函数试图由一个二进制的IP地址找到相应的主机名，与
[gethostbyname](https://github.com/lsill/nbook/blob/main/content/UNIX/nTOAddr/gethostbyname.md)
的行为刚好相反。

```c
#include snetdb.h>
struct hostent *gethostbyaddr(const char *addr, sockie n_t len, int family);
// 返回:若成功则为非空指针，若出错則为NULL 且设置h_errno
```

本函数返回一个指向hostent结构的指针。我们感兴趣的字段通常是存放规范主机名的h_name。

addr参数实际上不是char*类型，而是一个指向存放IPV4地址的某个in_addr结构的指针,len参数是这个结构的大小:
对于IPv4地址为4。family参数为AF_INET。

按照DNS的说法，gethostbyaddr在 in_adar.arpa 域中向一个名字服务器查询PTR记录。
