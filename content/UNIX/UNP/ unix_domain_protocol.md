---
title: "Unix 域协议"
date: 2024-03-12T16:20:32+08:00
draft: true
---

## 1. 概述
Unix域协议并不是一个实际的协议族，而是在单个主机上执行客户/服务器通信的一种方法，所用API就是在不同主机上执行
客户/服务器通信所用的API(套接字API)。Unix域协议因此可视为IPC方法之一。

Unix城提供两类套接字:字节流套接字(类似TCP)和数据报套接字(类似DUP)。尽管也提供原始套接字，不过它的语义不曾见
于任何文档，未见过任何使用它的程序，POSIx也没有它的定义。

使用Unix城套接字有以下3个理由。
1. 在源自Berkeley的实现中，Unix域套接字往往比通信两端位于同一个主机的TCP套接字快出一倍。X Window System
发挥了Unix域套接字的这个优势。当一个X11客户启动并打开到X11服务器的连接时，该客户检查DISPLAY环境变量的值，其
中指定服务器的主机名、窗口和屏幕。如果服务器与容户处于同一个主机，客户就打开一个到服务器的Unix域字节流连接，否
则打开一个到服务器的TCP连接。
2. Unix域套接字可用于在同一个主机上的不同进程之间传递描达符。
3. Unix域套接字较新的实现把客户的凭证(用户ID和组ID)提供给服务器，从而能够提供额外的安全检查措施。Unix域中用于
标识客户和服务器的协议地址是普通文件系统中的路径名。IPv4协议地址由一个32位地址和一个16位端口号构成，IPv6协议地
址则由一个128位地址和一个16位端口号构成。这些路径名不是普通的Unix文件:除非把它们和Unix域套接字关联起来，否则无
法读写这些文件。

## 2. Unix域套接字地址结构
<sys/un.h>中定义的Unix域套接字地址结构
```c
struct sockaddr_un {
    sa_family_t sun_family; /* AF_LOCAL */
    char    sun_path [104]; /* null-terminated pathname */
};
```
存放在sun_path数组中的路径名必须以空字符结尾。实现提供的SUN_LEAN宏以一个指向sockaddr_un结构的指针为参数并返回该结构的
长度，其中包括路径名中非空字节数。未指定地址通过以空字符串作为路径名指示，也就是一个sun_path[0]值为0的地址结构。它等价于
IPv4的INADR_ANY常值以及IPv6的IN6ADDR_ANY_INIT常值。

## 3. socketpair 函数
socketpair函数创建两个随后连接起来的套接字。本函数仅适用于Unix域套接宇。
```c
#include<sys/socket.h>
int socketpair(int family,int type,int protocol,int sockfd[2]):
// 返回:若成功到为非0，若出错则为-1
```
family参数必须为AF_LOCAL，protocol参数必须为0。type参数既可以是SOCK_STREAM，也可以是SOCK_DGRAM。新创建的两个套接字
描述符作为sockfd[0]和sockfd[1]返回。

指定type参数为SOCK_STRAEM调用socketpair得到的结果称为流管道(stream pipe)。它与调用pipe创建的普通Unix管道类似，差别
在于流管道是全双工的，即两个描述符都是既可读又可写。图15-7展示了调用socketpair创建的流管道。

## 4. 套接字函数
当用于Unix域套接字时，套接字函数中存在一些差异和限制。我们尽量列出POSIX的要求，并指出并非所有实现目前都己达到这个级别。
1. 由bind创建的路径名默认访问权限应为0777(属主用户、组用户和其他用户都可读、可写并可执行)，并按照当前umask值进行修正。
2. 与Unix域套接字关联的路径名应该是一个绝对路径名，而不是一个相对路径名。避免使用后者的原因是它的解析依赖于调用者的当前
工作目录。也就是说，要是服务器捆绑一个相对路径名，客户就得在与服务器相同的目录中(或者必领知道这个目录)才能成功调用connect或sendto。
POSIX声称给Unix域套接字捆绑相对路径名将导致不可预计的结果。
3. 在connect调用中指定的路径名必须是一个当前绑定在某个打开的Unix城套接字上的路径名，而且它们的套接字类型(字节流或数据报)
也必须一致。出错条件包括:(a)该路径名己存在却不是一个套接字:(b〕该路径名己存在且是一个套接字，不过没有与之关联的打开的描述符:
(c)该路径名己存在且是一个打开的套接字，不过类型不符(也就是说Unix域字节流套接字不能连接到与Unix域数据报套接字关联的路径名，反之亦然)。
4. 调用connect连接一个Unix域套接字涉及的权限测试等同于调用open以只写方式访问相应的路径名。
5. Unix域字节流套接字类似TCP套接字:它们都为进程提供一个无记录边界的字节流接口。
6. 如果对于某个Unix域字节流套接字的connect调用发现这个监听套接字的队列己满，调用就立即返回一个BCONNREFUSED错误。这一点不同
于TCP:如果TCP监听套接字的队列己满，TCP监听端就忽略新到达的SYN，而TCP连接发起端将数次发送SYN进行重试。
7. Unix域数据报套接字类似于UDP套接字:它们都提供一个保留记录边界的不可靠的数据报服务。
8. 在一个末绑定的Unix域套接字上发送数据报不会自动给这个套接字捆绑一个路径名，这一点不同于UDP套接字:在一个未綁定的UDP套接字上发送UDP
数据报导致给这个套接字捆绑一个临时端口。这一点意味者除非数据报发送端已经捆绑一个路径名到它的套接字，否则数据报接收端无法发回应答数据报。
类似地，对于某个Unix域数据报套接字的connect调用不会给本套接字捆鄉一个路径名，这一点不同于TCP和UDP。

## 5. Unix域字节流客户/服务器程序
- [unixstrserv01](https://github.com/lsill/unpvnote/blob/main/unixdomain/unixstrserv01.c?plain=1#L4)
- [unixstrcli01](https://github.com/lsill/unpvnote/blob/main/unixdomain/unixstrcli01.c?plain=1#L4)

## 6. 






