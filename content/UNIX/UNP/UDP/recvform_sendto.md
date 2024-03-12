---
title: "recvfrom和sendto函数"
date: 2023-12-27T16:53:35+08:00
draft: true
---

```c
#include<sys/socket..h>
ssize_t recvfrom(int sockfd, void *buff,size_t nbytes,int flags, struct sockaddr* from,socklen_t *addrlen);
ssize_t sendto(int sockfd,const void* buff,size_t nbytes,int flags, const struct sockaddr* to,socklen_t *addrlen);
// 均返回:若成功则为读或写的字节数，若出错则为-1
```
前三个参数sockfd、buff和nbytes等同于read和write函数的三个参数:描述符、指向读入或写出缓冲区的指针和读写字节数。

sendto的to参数指向一个含有数据报接收者的协议地址(例如IP地址及端口号)的套接字地址结构，其大小由addrlen参数指定。
recvfrom的from参数指向一个将由该函数在返回时填写数据报发送者的协议地址的套接字地址结构，而在该套接字地址结构中
填写的字节数则放在addrlen参数所指的整数中返回给调用者。注意，sendto的最后一个参数是一个整数值，而recvfrom的最
后一个参数是一个指向整数值的指针(即值-结果参数)。

recvfrom的最后两个参数类似于accept的最后两个参数:返回时其中套接字地址结构的内容告诉我们是谁发送了数据报(UDP情况下)
或是谁发起了连接(TCP情况下)。sendto的最后两个参数类似于connect的最后两个参数:调用时其中套接字地址结构被我们填入数据报将
发往(UDP情况下)或与之建立连接(TCP情况下)的协议地址。

这两个函数都把所读写数据的长度作为函数返回值。在recvfrom使用数据报协议的典型用途中，返回值就是所接收数据报中的用户数据量。

写一个长度为0的数据报是可行的。在UDP情况下，这会形成一个只包含一个IP首部(对于IPv4通常为20个字节，对于IPv6通常为40个字节)
和一个8字节UDP首部而没有数据的IP数据报。这也意味着对于数据报协议，recvfrom返回0值是可接受的:它并不像TCP套接字上read返回
0值那样表示对端已关闭连接。既然UDP是无连接的，因此也就没有诸如关闭一个UDP连接之类事情。如果recvfrom的from参数是一个空指针
，那么相应的长度参数(addrlen)也必须是一个空指针，表示我们并不关心数据发送者的协议地址。

recvfrom和sendto都可以用于TCP，尽管通常没有理由这样做。





