---
title: "gethostbyname函数"
date: 2023-12-28T17:51:29+08:00
draft: true
---

杳找主机名最基本的两数是gethostbyname。如果调用成功，它就返回一个指向hostent结构的指针，该结构中
含有所查找主机的所有IPv4地址。这个函数的局限是只能返回IPv4地址。
```c
#include <netdb.h>
struct hostent *gethostbyname(const char *hostname);
// 返回:若成功则为非空指针，若出错则为NULL 且设置h_errno
```
hostent结构
```c
struct hostent {
    char *n_name; /* official (canonical) name of host. /*
    char **h_aliases; /* pointer to array of pointers to alias names */
    int h_addrtype; /* host address type: AF_INET */ 
    int h_length; /* length of address: 4 */
    char **th_addr_list; /*ptr to array of ptrs with IPv4addrs*/
 };
```

按照DNS的说法，gethostbyname执行的是对A记录的查询。它只能返回IPv4地址。

图11-2所示为hostent结构和它所指向的各种信息之间的关系，其中假设所查询的主机名有2个别名和3个IPv4地址。
在这些字段中，所查询主机的正式主机名(official host)和所有别名(alias)都是以空字符结尾的C字符串。

返回的h_name称为所查询主机的规范(canonical)名字。以CNAME录例子为例，主机ftp.unpbook.com的规范名字是
linux.unpbook.com。另外，如果我们在主机aix上以一个非限定主机名(例如solaris)调用gethostbyname，那么
作为规范名字返回的是它的FQDN(即solaris.unpbook.com)。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/11_2.jpg)

gethostbyname与其他套接字函数的不同之处在于:当发生错误时，它不设置errno变量，而是將全局整数变量h_errno设置为在
头文件<netdb.h>中定义的下列常值之一:
- HOST_NOT_FOUND;
- TRY_AGAIN;
- NO_RECOVERY;
- NO_DATA(等同于NO_ADDRESS)。

NO_DATA错误表示指定的名字有效，但是它没有A记录。只有MX记录的主机名就是这样的一个例子。

如今多数解析器提供名为hstrerror的函数，它以某个h_errno值作为唯一的参数，返回的是一个const char*指针，指向相应错误
的说明。