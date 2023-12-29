---
title: "getaddrinfo 函数"
date: 2023-12-29T16:36:33+08:00
draft: true
---
gethostbyname和gethostbyaddr这两个函数仅仅支持IPv4。最终结果是getaddrinfo函数。
getaddrinfo函数能够处理名字到地址以及服务到端口这两种转换，返回的是一个sockaddr结构
而不是一个地址列表。这些sockaddr结构随后可由套接字函数直接使用。如此一来，getaddrinfo
函数把协议相关性完全隐藏在这个库函数内部。应用程序只需处理由getaddrinfo填写的套接字地址
结构。

```c
include <netdb.h>
int getaddrinfo(const char *hostname, const char *service,
const struct addrinfo *hints, struct addrinto **result) ;
// 返回:若成功则为0，若出错则为非0 
```
addrinfo结构定义
```c
struct addrinfo
{
    int ai_flags;   /* AI_PASSIVE, AI_CANONNAME */
    int ai_family    / * AF_xxx */
    int ai_socktype;    /* SOCK_xxx */
    int ai_protocol;    /* 0 or IPPROTO_xxx for IPv4 and IPv6 */
    socklen_t ai_addrlen; /* length of ai_addr */
    char *ai_canonname; /* ptr to canonical name for host /*
    struct sockaddr *ai_addr;   /* ptr to socket address structure */
    struct addrinfo *ai_next;  /* ptr to next structure in linked list */
};

```

其中hostname参数是一个主机名或地址串(IPv4的点分十进制数串或IPv6的十六进制数串)。service参数是一个服务名
或十进制端口号数串。

hints参数可以是一个空指针，也可以是一个指向某个addrinfo结构的指针，调用者在这个结构中填入关于期望返回的信息
类型的暗示。举例来说，如果指定的服务既支持TCP也支持UDP(例如指代某个DNS服务器的domain服务)，那么调用者可以把
hints结构中的ai_socktype成员设置为SOCK_DGRAN，使得返回的仅仅是适用于数据报套接字的信息。

hints结构中调用者可以设置的成员有:
- ai_flags(零个或多个或在一起的AI_xxx值):
- ai_family(某个AF_xxx值);
- ai_socktype(某个SOCK_xxx值);
- ai_protocol.

其中ai_flags成员可用的标志值及其含义如下。
- AI_PASSIVE：套接字将用于被动打开。
- AI_CANONNAME：告知getaddrinfo函数返回主机的规范名字。
- AI_NUMERICHOST：防止任何类型的名字到地址映射，hostname参数必须是一个地址串。
- AI_NUMERICSERV：防止任何类型的名字到服务映射,service参数必须是一个十进制端口号数串。
- AI_V4MAPPED:如果同时指定ai_family成员的值为AF_INET6，那么如果没有可用的AAAA记录，就返回与A记录对应的
IPv4映射的IPv6地址。
- AI_ALL:如果同时指定AI_V4MAPPED标志，那么除了返回与AAAA记录对应的IPv6地址外，还返回与A记录对应的IPv4
映射的IPv6地址。
- AI_ADDRCONFIG:按照所在主机的配置选择返回地址类型，也就是只查找与所在主机同馈接又以外的网络接口配置的IP
地址版本一致的地址。

如果hints参数是一个空指针，本函数就假设ai_flag、ai_socktype和ai_protocol的值均为0，ai_family的值为AF_UNSPEC。

如果本函数返回成功(0)，那么由result参数指向的变量己被填入一个指针，它指向的是由其中的ai_next成员串接起来的addrinfo
结构链表。可导致返回多个addrinfo结构的情形有以下两个。

(1)如果与hostname参数关联的地址有多个，那么适用于所请求地址族(可通过hints结构的ai_family成员设置)的每个地址都返回一
个对应的结构。

(2)如果service参数指定的服务支持多个套接字类型，那么每个套接字类型都可能返回一个对应的结构，具体取决于hints结构的
ai_socktype成员。(注意，getaddrinfo的多数实现认为只能按照由ai_socktype成员请求的套接字类型端口号数串到端口的转
换，如果没有指定这个成员，那就返回一个错误。) 

举例来说，如果在没有提供任何暗示信息的前提下，请求查找有2个IP地址的某个主机上的domain服务，那将返回4个addrinfo结构，分别是:
- 第一个IP地址组合SOCK_STREAN 套接字类型;
- 第一个IP地址组合SOCK_DGRAN 套接字类型;
- 第二个IP地址组合SOCK_STREAN 套接字类型;
- 第二个IP地址组合SOCK_DGRAN 套接字类型。

图11-5展示了本例子。当有多个addrinfo结构返回时，这些结构的先后顺序没有保证，也就是说，我们并不能假定TCP服务总是
先于UDP服务返回。

（尽管没有保证，本函数的实现却应该按照DNS返回的顺序返回各个IP地址。有些解析器允许系统管理员在/etc/resolv.conf文
件中指定地址的排序顺序。IPv6可指定地址选择規則，可能影响由getaddrinfo返回地址的顺序。)

在addrinfo结构中返回的信息可现成用于socket调用，随后现成用于适合客户的connect或sendto调用，或者是适合服务器的
bind调用。socket函数的参数就是addrinfo结构中的ai_family、ai_socktype和ai_addr成员。connect或bina函数的
第二个和第三个参数就是该结构中的ai_addr(一个指向适当类型套接字地址结构的指针，地址结构的内容由getaddrinfo函数填
写)和ai_addrlen(这个套接字地址结构的大小)成员。

如果在hints结构中设置了AI_CANONNAME标志，那么本函数返回的第一个addrinfo结构的ai_canonname成员指向所查找主机
的规范名字。按照DNS的说法，规范名字通常是FQDN。诸如telnet之类程序往往使用这个标志以显示所连找到主机的规范名宇，
这样即使用户给定的是一个简单名字或别名，他们也能搞清真正查找的名字.

图11-5给出了执行下列程序片段返回的信息。
```
struct addrinfo hints, *res;
bzero(&hints, sizeof(hints));
hints.ai_flags = AI_CANONNAME;
hints.ai_family = AF_INET;
getaddrinto("freebsd4", "domain", shints, &res);
```
图中除res变量外的所有内容都是由getaddrinfo函数动态分配的内存空间(譬如来自malloc调用)。我们假设主机freebsd4
的规范名字是freebsd4.unpbook.com，并且它在DNS中有2个IPv4地址。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/11_5.jpg)

端口53用于domain服务。这个端口号在套接字地址结构中按照网络字节序存放。返回的ai_protocol值或为IPPROTO_TCP，或为
IPPROTO_UDP。要是ai_family和ai_socktype组合能够完全指定TCP或UDP协议，那么返回的ai_protocol值为0也可以接受。
也就是说，如果系统没有实现除TCP外的其他SOCK_STREAM协议(例如SCTP)，套接字类型值为SOCK_STREAM的那两个addrinfo
结构协议值可为0;同样地，如果系统没有实现除UDP外的其他SOCK_DGRAM协议，套接字类型值为SOCK_DGRAM的那两个addrinfo
结构协议值可为0。最安全的做法是让getaddrinfo总是返回明确的协议值。

图11-6汇总了根据指定的服务名(可以是一个十进制端口号数串)和ai_socktype暗示信息为每个通过主机名查找获得的IP地址返回
addrinfo结构的数目。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/11_6.jpg)

在不考虑SCTP的前提下，只有在末提供ai_socktype暗示信息时才可能为每个IP地址返回多个addrinfo结构，此时或者服务以名字
标识并且同时支持TCP和UDP(在/etc/services文件中指明)，或者服务以端口号标识。

如果枚举getaddrinfo所有64种可能的输入(因为它共有6个二值输入变量)，那么许多是无效的，有些则没有多大意义。为此我们只查
看一些常见的输入。
- 指定hostname和service。这是TCP或UDP客户进程调用getaddrinfo的常规输入。该调用返回后，TCP客户在一个循环中针对每个
返回的IP地址，逐一调用socket和connect，直到有一个连接成功，或者所有地址尝试完毕为止。

对于UDP客户，由getaddrinfo填入的套接字地址结构用于调用sendto或connect。如果客户能够判定第一个地址看来不工作(其手段不
外乎或者在己连接的UDP套接字上收到出错消息，或者在未连接的套接字上经历消息接收超时)，那么可以尝试其余的地址。如果客户清楚
自己只处理一种类型的套接字(例如Telnet和FTP客户只处理TCP，TFTP客户只处理UDP)，那么应该把hints结构的ai_socktype成员
设置成SOCK_STREAM或SOCK_DGRAM。

- 典型的服务器进程只指定service而不指定hostname，同时在hints结构中指定AI_PASSIVE标志。返回的套接字地址结构中应含有
一个值为INADDR_ANY(对于IPv4)或IN6ADDR_ANY_INIT(对于IPv6)的IP地址。TCP服务器随后调用socket、bind和listen。如果
服务器想要malloc另一个套接字地址结构以从accept获取客户的地址，那么返回的ai_addrlen值给出了这个套接字地址结构的大小。

UDP服务器将调用socket、bind和recvfrom。如果服务器想要malloc另一个套接字地址结构以从recvfrom获取客户的地址，那么返回
的ai_addrlen值给出了这个套接字地址结构的大小。与典型的客户一样，如果服务器清楚自己只处理一种类型的套接宇，那么应该把hints
结构的ai_socktype成员设置成SOCK_STREAM或SOCK_DGRAM。这样可以避免返回多个结构，其中可能出现错误的ai_socktype值。

- 到目前为止，我们展示的TCP服务器仅仅创建一个监听套接字，UDP服务器也仅仅创建一个数据报套接字。这也是我们讨论上一点隐含的一
个假设。服务器程序的另一种设计方法是使用select或poll函数让服务器进程处理多个套接字。这种情形下，服务器将遍历由getaddrinfo
返回的整个addrinfo结构链表，并为每个结构创建一个套接字，再使用select或poll。

(这个技术的问题在于，getaddrinfo返回多个结构的原因之一是该服务可同时由IPV4和IPV6处理。这两个协议并非完全独立。也就
是说，如果我们为某个给定端口创建了一个IPV6监听套接宇，那么没有必要为同一个端口再创建一个IPV4套接字，因为来自IPV4客户的连接将
由协议栈和IPV6监听套接守自动处理，而不论是否设置了IPV6_V6ONLY套接字选项。）

尽管getaddrinfo函数确实比gethostbyname和getservbyname这两个函数“好”(它方便我们编写协议无关的程序代码，单个函数能够同时
处理主机名和服务，所有返回信息都是动态而不是静态分配的)，不过它仍然没有像期待的那样好用。问题在于我们必须先分配一个hints结构，
把它清零后填写需要的字段，再调用getaddrinfo，然后遍历一个链表逐一尝试每个返回地址。

getaddrinfo解决了把主机名和服务名转换成套接字地址结构的问题。



