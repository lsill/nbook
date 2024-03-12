---
title: "getaddrinfo 函数IPV6"
date: 2023-12-29T18:38:15+08:00
draft: true
---

POSIX规范定义了getaddrinfo函数以及该函数为IPv4或IPv6 返回的信息。

注意一下几点：
- getaddrinfo在处理两个不同的输入:一个是套接字地址结构类型，调用者期待返回的地址结构符合这个类型;
另一个是资源记录类型，在DNS或其他数据库中执行的查找符合这个类型。
- 由调用者在hints结构中提供的地址族指定调用者期待返回的套接字地址结构的类型。如果调用者指定AF_INET，
getaddrinfo函数就不能返回任何sockaddr_in6结构:如果调用者指定AF_INET6，getaddrinfo函数就不能返
回任何sockaddr_in结构。
- POSIX声称如果调用者指定AF_UNSPEC，那么getaddrinfo函数返回的是适用于指定主机名和服务名且适合任意
协议族的地址。这就意味着如果某个主机既有AAAA记录又有A记录，那么AAAA记录将作为sockaadr_in6结构返回，
A记录将作为sockaddr_in结构返回。在sockaddr_in6结构中作为IPv4映射的IPv6地址返回A记录没有任何意义，
因为这么做没有提供任何额外信息:这些地址己在sockaddr_in结构中返回过了。
- POSIX的这个声明也意味着如果设置了AI_PASSIVE标志但是没有指定主机名，那么IPv6通配地址(IN6ADDR_ANY_INIT
或0:0)应该作为sockaddr_in6结构返回，同样IPv4通配地址(INADDR_ANY或0.0.0.0)应该作为sockaddr_in结构返回。
首先返回IPv6通配地址也是有意义的.
- 在hints结构的ai_family成员中指定的地址族以及在ai_flags成员中指定的AI_V4MAPPED和AI_ALL等标志决定了在
DNS中查找的资源记录类型(A和/或AAAA)，也决定了返回地址的类型(IPv4、IPV6和/或IPv4映射的IPV6)。图11-8对此作了汇总。
- 主机名参数还可以是IPv6的十六进制数串或IPv4的点分十进制数串。这个数串的有效性取决于由调用者指定的地址族。如果指定
AF_INET，那就不能接受IPv6的十六进制数串;如果指定AF_INET6，那就不能接受IPv4的点分十进制数串。然而如果指定的是
AF_UNSPEC，那么这两种数串都可以接受，返回的是相应类型的套接字地址结构。

(有人可能会争论说，如果指定了AF_INET6，那么点分十进制数串应该作为IPV4映射的IPV6地址在sockaddr_in6结构中返回。
然而得到同样结果另有简单的方法，就是在点分十进制数串前加上0::ffff:)

图11-8汇总了getaddrinfo如何处理IPv4和IPv6地址。“结果”一栏是在给定前三栏的变量后，该函数返回给调用者的结果。
“行为〞一栏则说明该函数如何获取这些结果。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/11_8.jpg)

图11-8仅仅说明getaddrinfo如何处理IPv4和IPv6，也就是返回给调用者的地址数目。返回给调用者的addrinfo结构的确切
数目还取决于指定的套接字类型和服务名。







