---
title: "套接字编程简介"
date: 2023-12-08T17:32:28+08:00
draft: true
---

## 通用套接字地址结构sockaddr_storage
```
struct sockaddr_storage {
    sa_family_t  ss_family;  // 地址族

    // 实现特定的填充，确保结构体的大小足够容纳所有类型的地址
    // 并且对齐到适当的边界。
    char         __ss_pad1[_SS_PAD1SIZE];
    int64_t      __ss_align;
    char         __ss_pad2[_SS_PAD2SIZE];
};
```
在实际的网络编程中，sockaddr_storage 结构体通常用于以下情况：
- 接收任意类型的地址：当你的程序需要接受不同类型的套接字地址时（例如，同时处理 IPv4 
和 IPv6 连接），可以使用 sockaddr_storage 结构体来通用地存储地址信息。
- 与 sockaddr 类型兼容：sockaddr_storage 可以被安全地强制转换为 sockaddr 类型
，用于那些需要 sockaddr * 类型参数的系统调用（如 bind(), connect(), accept() 等）。

这种结构体的使用提高了网络程序的灵活性和兼容性，使其能够适应不同的网络环境和地址类型。

在不同的操作系统和网络编程环境中，sockaddr_storage 结构的具体实现可能会有所不同。在某些系统
中，会存在一个名为 ss_len 的字段，用于表示地址的长度。然而，在 POSIX 标准中，sockaddr_storage 
结构并没有定义这样一个字段。

## 套接字结构比较
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/socket_struct_com.jpg)


## 套接字函数
```c
#include<sys/socket.h>
int socket(int family,int type,int protocol);
//返回:若成功則为非负描达符，若出钱则为-1
```
AF_ 前缀表示地址族，PF_ 前缀表示协议族

socket函数的family值

| family   | 说明      |
|----------|---------|
| AF_INET  | IPv4协议  |
| AF_INET6 | IPv6协议  |
| AF_LOCAL | Unix域协议 |
| AF_ROUTE | 路由套接字   |
| AF_KEY   | 秘钥套接字   |

socket函数的type常值

| type           | 说明      |
|----------------|---------|
| SOCK_STREAM    | 字节流套接字  |
| SOCK_DGRAM     | 数据报套接字  |
| SOCK_SEQPACKET | 有序分组套接字 |
| SOCK_RAW       | 原始套接字   |

socket函数中family和type参数的组合

| ---            | AF_INET  | AF_INET6 | AF_LOCAL | AR_ROUTE | AF_KEY |
|----------------|----------|----------|----------|----------|--------|
| SOCK_STREAM    | TCP&SCTP | TCP&SCTP | 是        | ---      | ---    |
| SOCK_DGRAM     | UDP      | UDP      | 是        | ---      | ---    |
| SOCK_SEQPACKET | SCTP     | SCTP     | 是        | ---      | ---    |
| SOCK_RAW       | IPv4     | IPv6     | ---      | 是        | 是      |


## connect函数
TCP客户用connect函数来建立与TCP服务器的连接。
```c
#include<sys/socket.h>
int connect(int sockfd,const struct sockaddr *servaddr,socklen_t addrien);
// 返回:若成功則为0。若出错则为-1
```
如果是TCP套接字，调用connect函数将激发TCP的三路握手过程，而且仅在连接建立成功或出错时才返回，
其中出错返回可能有以下几种情况。
- 若TCP客户没有收到SYN分节的响应，则返回ETIMEDOUT错误。举例来说，调用connect函数时，
  4.4BSD内核发送一个SYN，若无响应则等待6s后再发送一个，若仍无响应则等待24s后再发送一个。
  若总共等了75s后仍末收到响应则返回本错误。
- 若对客户的SYN的响应是RST(表示复位)，则表明该服务器主机在我们指定的端口上没有进程在等待
与之连接(例如服务器进程也许没在运行)。这是一种硬错误(hard error)，客户一接收到RST就马上
返回ECONNREFUSED错误。
RST是TCP在发生错误时发送的一种TCP分节。产生RST的三个条件是:目的地为某端口的SYN到达，然而
该端又上没有正在监听的服务器(如前所述);TCP想取消一个己有连接:TCP接收到一个根本不存在的连
接上的分节。
- 若客户发出的SYN在中间的某个路由器上引发了一个“destination unreachable”(目的地不可达)
ICMP错误，则认为是一种软错误(soft error)。客户主机内核保存该消息，并按第一种情况中所述的
时间间隔继续发送SYN。若在某个规定的时间(4.4BSD规定75s)后仍未收到响应，则把保存的消息〈即
ICMP错误)作为EHOSTUNREACH或ENETUNREACH错误返回给进程。以下两种情形也是有可能的:一是按
照本地系统的转发表，根本没有到达远程系统的路径:二是connect调用根本不等待就返回。

ICMP（Internet Control Message Protocol，互联网控制消息协议）是一种用于互联网协议套件
（IP）的核心协议。它主要用于网络设备之间发送错误消息和运行状况查询，例如未能到达目的地、时间
超时等情况。ICMP 是一种无连接的协议，不像 TCP 或 UDP，它不用于常规的数据传输，而是用于传
输控制消息。

## bind函数
bind函数把一个本地协议地址赋子一个套接字。对于网际网协议，协议地址是32位的IPv4地址
或128位的IPv6地址与16位的TCP或UDP端口号的组合。
```c
(01
#include <sys/socket.h>
int bind(int sockd, const struct sockaddr *myaddr, socklen_t addrlen) ; 
// 返回:若成功则为0，若出错则为-
```

- 服务器在启动时捆绑它们的众所周知端口。如果一个TCP客户或服务器未曾调用bind捆绑一个端口，当
调用connect或listen时，内核就要为相应的套接字选择一个临时端口。让内核来选择临时端口对于
TCP客户来说是正常的，除非应用需要一个预留端口;然而对于TCP服务器来说却极为罕見，因为服务器
是通过它们的众所周知端口被大家认识的。
这个規則的例外是远程过程调用(Remote Procedure Call，RPC)服务器。它们通常就由内核为它们的
监听套接字选择一个临时端口、而该端口随后通过RPC端口映射器进行注册。客户在connect这些服务器之
前，必须与端口映射器联系以获取它们的临时端口。这种情況也造用于使用UDP的RPC服务器。
- 进程可以把一个特定的IP地址捆绑到它的套接字上，不过这个I地址必须属于其所在主机的网络接又之一。
对于TCP客户，这就为在该套接字上发送的IP数据报指派了源IP地址。对于TCP服务器，这就限定该套接字
只接收那些目的地为这个IP地址的客户连接。TCP客户通常不把IP地址捆绑到它的套接字上。当连接套接字
时，内核将根据所用外出网络接口选择源IP地址，而所用外出接口则取决于到达服务器所需的路径。如果
TCP服务器没有把IP地址捆绑到它的套接字上，内核就把客户发送的SYN的目的IP地址作为服务器的源IP
地址。

对于Iv4来说，通配地址由常值INADDR_ANY来指定，其值一般为0。它告知内核去选择IP地址。
```
struct sockaddr_in servaddr;
servaddr.sin_addr.s_addr=htonl(INADDR_ANY);/*wildcard*/
```
Iv6，我们就不能这么做了，因为128位的IPv6地址是存放在一个结构中的。(在C语言中，赋值语句的右边无法表
示常值结构。)为了解决这个问题，改写为:
```
struct sockaddr_in6 serv;
serv.sin6_addr = in6addr_any;   // wildcard
```

## 5. listen函数
listen两数仅由TCP服务器调用，它做两件事情。

1. 当socket函数创建一个套接字时，它被假设为一个主动套接字，也就是说，它是一个将调用connect发起连接
的客户套接字。listen函数把一个未连接的套接字转换成一个被动套接字，指示内核应接受指向该套接字的连接请
求。根据TCP状态转换图(图2-4)，调用listen导致套接字从CLOSED状态转换到LISTEN状态。
2. 本函数的第二个参数规定了内核应该为相应套接字排队的最大连接个数。

```
#include<sys/socket.h>
int listen(int sockfd,int backlog);
// 返回:若成功則为0，若出错则为-1
```
本函数通常应该在调用socket和bind这两个函数之后，并在调用accept函数之前调用。

内核为任何一个给定的监听套接字维护两个队列:
(1)未完成连接队列(incomplete connection queue)，每个这样的SYN分节对应其中一项:己由某
个客户发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程。这些套接宇处于SYN_RCVD状态(图2-4)。
(2)已完成连接队列(completed connection queue)，每个已完成TCP三路握手过程的客戶对应其中一项。
这些套接字处于ESTABLISHED状态(图2-4)。









