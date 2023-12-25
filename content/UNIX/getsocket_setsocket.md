---
title: "getsocket和setsocket函数"
date: 2023-12-25T18:06:30+08:00
draft: true
---

这两个函数仅用于套接字
```c
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *opilen);
int setsockopt (int sockfd, int level, int optname, const void *optval, socklen_t optlen) ;
// 均返回:若成功則为0，若出错則为- 1
```
其中sockfd必须指向一个打开的套接字描述符，level(级别)指定系统中解释选项的代码或为通用套接宇代码，或为某个特定
于协议的代码(例如IPV4、IPV6、TCP或SCTP)。

optval是一个指向某个变量(*optval）的指针，setsockopt从*optval中取得选项待设置的新值，getsockopt则把己获取
的选项当前值存放到*optval中。*oprval的大小由最后一个参数指定，它对于setsockopt是一个值参数，对于getsockopt
是一个值-结果参数。

图7-1和图7-2汇总了可由getsockopt获取或由setsockopt设置的选项。其中的“数据类型”列给出了指针optval必须指向的
每个选项的数据类型。我们用后跟一对花括号的记法来表示一个结构，如linger{}就表示struct linger.

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/socket_ip_opt.jpg)
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/socket_trans_opt.jpg)

套接字选项租分为两大基本类型:一是启用或禁止某个特性的二元选项(称为标志选项)，二是取得并返回我们可以设置或检查的特定值的
选项(称为值选项)。标有“标志”的列指出一个选项是否为标志选项。当给这些标志选项调用getsockopt函数时，*oprval是一个整数。
*optval中返回的值为0表示相应选项被禁止，不为0表示相应选项被启用。类似地，setsockopt函数需要一个不为0的*oprval值来
启用选项，一个为0的*optval值米禁止选项。如果“标志”列不含有“•"，那么相应选项用于在用户进程与系统之间传递所指定数据类型的值。






