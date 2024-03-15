---
title: "ioctl操作"
date: 2024-03-14T17:13:43+08:00
draft: true
---

## 1. 概述
ioctl函数传统上一直作为那些不适合归入其他精细定义类别的特性的系统接口。POSIX致力于摆脱处于标准化过程中的
特定功能的ioctl接口，办法是为它们创造一些特殊的函数以取代ioctl请求。举例来说，Unix终端接口传统上使用
ioctl访问，然而POSIX为终端创造了12个新函数:tcgetattr用于获取终端属性，tcflush用于冲刷待处理输入或输出，
等等。类似地，POSIX替换了一个用于网络的ioctl请求:新的sockatmark函数取代SIOCATMARK ioctl。尽管如此，
为与网络编程相关且依赖于实现的特性保留的ioctl请求为数依然不少，它们用于获取接口信息、访问路由表、访问ARP
高速缓存，等等。

本章给出与网络编程相关的ioctl请求的概貌，其中有许多依赖于具体的实现。此外，包括源自4.4BSD的系统和Solairs2.6
及以后版本在内的一些实现改用AF_ROUTE域套接字(路由套接字)来完成其中许多操作。

网络程序(特别是服务器程序)经常在程序启动执行后使用ioctl获取所在主机全部网络接口的信息，包括:接口地址、是否
支持广播、是否支持多播，等等。我们将自行开发用于返回这些信息的函数，在本章提供一个使用ioctl的实现。


## 2. ioctl 函数
本函数影响由fd参数引用的一个打开的文件。
```c
#include<unistd.h>
int ioctl(int fd,int request,.../* void* arg */);
// 返回:若成功則为0，若出错则为-1
```
其中第三个参数总是一个指针，但指针的类型依賴于request参数。

可以把和网络相关的请求(request)划分为6类:
- 套接字操作;
- 文件操作;
- 接口操作;
- ARP高速缓存操作; 
- 路由表操作;
- 流系统

某些ioctl採作和某些fcntl操作功能重叠(譬如把套接字设置为非阻塞)，而且某些操作可以使用ioctl以不止一种方式指定(
譬如设置套接字的进程组属主)。图17-1列出了网络相关ioctl请求的request参数以及arg地址必须指向的数据类型。
以下各节详细讲解这些请求。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/17_1.png)


## 3. 套接字操作
明确用于套接字的ioctl请求有3个。它们都要求ioctl的第三个参数是指向某个整数的一个指针。

- SIOCATMARK:如果本套接字的读指针当前位于带外标记，那就通过由第三个参数指向的整数返回一个非0值;
否则返回一个0值。POSIX以函数sockatmark替换本请求。
- SIOCGPGRP:通过由第三个参数指向的整数返回本套接字的进程ID或进程组ID，该ID指定针对本套接字的
SIGIO或SIGURG信号的接收进程。
- SIOCSPGRP：把本套接字的进程ID或进程组ID设置成由第三个参数指向的整数，该ID指定针对本套接字的SIGIO或
SIGURG信号的接收进程。本请求和fcntl的F_SETOMN命令等效。


## 4. 文件操作
下一组请求以FIO打头，它们可能还适用于除套接字外某些特定类型的文件。本节仅仅讨论适用于套接字的请求。
以下5个请求都要求ioctl的第三个参数指向一个整数。
- FIONBIO :根据ioctl的第三个参数指向一个0值或非0值,可清除或设置本套接字的非阻塞式I/O标志。本请求
和O_NONBLOCK文件状态标志等效，而可以通过fcntl的F_SETFL命令清除或设置该标志。
- FIOASYNC :根据ioctl的第三个参数指向一个0值或非0值，可清除或设置针对本套接字的信号驱动异步I/O标志，
它决定是否收取针对本套接字的异步I/O信号(SIGIO)。本请求和O_ASYNC文件状态标志等效，而可以通过fcntl的
E_SETEL命令清除或设置该标志。
- FIONREAD :通过由ioctl的第三个参数指向的整数返回当前在本套接字接收缓冲区中的字节数。本特性同样适用
于文件、管道和终端。
- FIOSETONN:对于套接字和SIOCSPGRP等效。
- FIOGETOWN:对于套接字和SIOCGPGRP等效。


## 5. 接口配置
需处理网络接口的许多程序沿用的初始步骤之一就是从内核获取配置在系统中的所有接口。本任务由SIOCGIFCONF
请求完成，它使用ifconf结构，ifconf又使用ifreq结构，图17-2给出了这两个结构的定义。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/17_2.png)
在调用ioctl前先分配一个缓冲区和一个ifconf结构，然后初始化后者。图17-3展示了这个iconf结构的初始化结果，
其中假设缓冲区的大小为1024字节。ioctl的第三个参数指向这样的ifconf结构。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/17_3.png)
假设内核返回2个ifreq结构，在ioctl返回时通过同一个ifconf结构所返回的值如图17-4所示。阴影区域为被ioctl
修改过的部分。缓冲区中填入了那2个ifreq结构。ifconf结构的ifc_len成员也被更新，以反映存放在缓冲区中的
信息量。本图假设每个ifreq结构占用32个字节。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/17_4.png)

指向某个ifreq结构的指针也用作图17-1所示接口类其余ioctl请求的一个参数。注意ifreq结构中含有一个联合，而众多
#define隐藏了这些字段实际上是该联合的成员这一事实。对于该联合某个成员的所有引用都使用如此定义的名宇。
注意有些系统往这个ifr_ifru联合中增添了许多依赖于实现的成员。

## 6. get_ifi_info 函数
既然很多程序需知道系统中的所有接口，于是开发一个名为get_ifi_info的函数，它返回一个结构链表，
其中每个结构对应一个当前处于“up”(在工，状态的接口。在本节使用SIOCGIFCONF ioctl实现这个函数。

首先在一个名为unpifi.h的新头文件中定义iti_info结构，如下所示：

[unpifi](https://github.com/lsill/unpvnote/blob/main/lib/unpifi.h?plain=1#L4)

[prifinfo](https://github.com/lsill/unpvnote/blob/main/ioctl/prifinfo.c?plain=1#L4)

[get_ifi_info](https://github.com/lsill/unpvnote/blob/main/lib/get_ifi_info.c?plain=1#L4)

## 7. 接口操作
SIOCGIFCONF请求为每个已配置的接口返回其名字以及一个套接字地址结构。接着可以发出多个接口类其他请求以设置或
获取每个接口的其他特征。这些请求的获取(get)版本(STOCGxxx)通常由netstat程序发出，设置(set)版本(STOCGxxx)
通常由ifconfig程序发出。任何用户都可以获取接口信息，设置接口信息却要求具备超级用户杈限。

这些请求接受或返回一个ifreq结构中的信息，而这个结构的地址则作为ioctl调用的第三个参数指定。接又总是以其名字
标识，在ifreq结构的ifr_name成员中指定，如1e0、100、ppp0等。

这些请求中有许多使用套接字地址结构在应用进程和内核之间指定或返回具体接口的IP地址或地址掩码。对于Pv4，这个地址
或掩码存放在一个网际网套接字地址结构的sin_addr成员中;对于IPv6，它是一个IPv6套接字地址结构的sin6_addr成员。

- SIOCGIFADDR :在ifr_addr成员中返回单播地址。
- SIOCSIFADDR :用ifr_addr成员设置接口地址。这个接口的初始化函数也被调用。
- SIOCGIFFLAGS :在ifr_flags成员中返回接口标志。这些标志的名字格式为IFF_xxx,在<net/if.h>头文件中定义。
举例来说，这些标志指示接口是否处于在工状态(IFF_UP)，是否为一个点到点接口(IFF_POINTOPOINT), 是否支持广播
(IPF_BROADCAST)，等等。
- SIOCSIFFLAGS :用ifr_flags成员设置接口标志。
- SIOCGIFDSTADDR : 在ifr_dstaddr成员中返回点到点地址。
- STOCSIFDSTADDR :用ifr_astaddr成员设置点到点地址。
- SIOCGIFBRDADDR :在ifr_broadaddr成员中返回广播地址。应用进程必须首先获取接口标志，然后发出正确的请求:
对于广播接口为SIOCGIFBRDADDR，对于点到点接口为SIOCGIFDSTADDR。
- SIOCSIFBRDADDR :用ifr_broadaddr成员设置广播地址。
- SIOCGIFNETMASK在 :ifr_addr成员中返回子网掩码。
- SIOCSIFNETMASK :用ifr_addr成员设置子网掩码。
- SIOCGIFMETRIC :用ifr_metric成员返回接口测度。接口测度由内核为每个接口维护，不过使用它的是路由守护进程routed。
接口测度被routea加到跳数上(使得某个接口更不被看好)。
- SIOCSIFMETRIC ：用ifr_metric成员设置接口的路由测度。本节讲述的是通用的接口请求。许多实现中都加入了其他的请求。


## 8. ARP 高速缓存操作
ARP高速缓存也通过ioctl函数操纵。使用路由域套接字的系统往往改用路由套接字访问ARP高速缓存。这些请求使用一个如
图17-12所示的arpreq结构，它定义在头文件<net/if_arp.h>中。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/17_12.png)

ioctl的第三个参数必须指向某个arpreq结构。操纵ARP高速缓存的ioctl请求有以下3个。

- SIOCSARP :把一个新的表项加到ARP高速缓存，或者修改其中已经存在的一个表项。其中arp_pa是一个含有IP地址的网际
网套接字地址结构，arp_ha则是一个通用套接字地址结构，它的sa_family值为AF_UNSPEC,sa_data中含有硬件地址(例
如6字节的以太网地址)。ATF_PERM和ATF_PUBL这两个标志也可以由应用程序指定。另外两个标志(ATE_INUSE和ATF_COM)则由内核设置。
- SIOCDARP :从ARP高速缓存中删除一个表项。调用者指定要删除表项的网际网地址。
- SIOCGARP :从ARP高速缓存中获取一个表项。调用者指定网际网地址，相应的硬件地址(例如以太网地址)随标志一起返回。

只有超级用户才能增加或删除表项。这3个请求通常由arp程序发出。

一些较新的系統不支特这些与ARP相关的ioctl请求，而改用路由套接宇执行这些ARP操作。

注意ioctl没有办法列出ARP高速缓存中的所有表项。当指定-a标志(列出ARP高速缓存中的所有表项)执行arp命令时，大多数
版本的arp程序通过读取内核的内存(/dev/kmem)获得ARP高速缓存的当前内容。

## 9. 路由表操作
有些系统提供2个用于操纵路由表的ioctl请求。这2个请求要求ioctl的第三个参数是指向某个rtentry结构的一个指针，该结构定义
在<net/route.h>头文件中。这些请求通常由route程序发出。只有超级用户才能发出这些请求。在支持路由域套接字的系统中，这些
请求改由路由套接字而不是ioctl执行。

- SIOCADDRT :往路由表中增加一个表项。
- SIOCDELRT :从路由表中删除一个表项。

ioctl没有办法列出路由表中的所有表项。这个操作通常由netstat程序在指定 -r 标志执行时完成。netstat程序通过读取内核的内存
(/dev/kmem)获得整个路由表。与ARP高速缓存的列示一样.

## 10. 小结
用于网络编程的ioctl命令可划分为6类:
- 套接字操作(是否位于带外标记等);
- 文件操作(设置或清除非阻塞标志等);
- 接口操作(返回接口列表，获取广播地址等);
- ARP表操作(创建、修改、获取或删除);
- 路由表操作(增加或删除);
- 流系统。

使用其中的套接字操作和文件操作，而接口列表的获取是一个相当常用的操作，为此开发了一个完成本操作的函数。

## 习题
Q:由SIOCGIFBRDADDR请求返回的广播地址是通过ifreq结构的ifr_broadaddr成员返回的。然而查看TCPV2第173，注意到它是在
ifr_dstaddr成员中返回的。这里有问题吗?

A:无关紧要，因为图17-2中union的前3个成员都是套接字地址结构。




























