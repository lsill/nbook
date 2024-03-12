---
title: "getservbyname和getservbyport 函数"
date: 2023-12-29T16:36:33+08:00
draft: true
---

像主机一样，服务也通常靠名字来认知。如果我们在程序代码中通过其名字而不是其端口号来指代一个服务，
而且从名字到端口号的映射关系保存在一个文件中(通常是/etc/services)，那么即使端口号发生变动，
我们需修改的仅仅是/etc/services文件中的某一行，而不必重新编译应用程序。getservbyname函数
用于根据给定名字查找相应服务。

```c
#include <netdb.h>
struct servent* getservbyname(const char* servname,const char* protoname);
// 返回:若成功則为非空指针，若出错则为NULL
```

servent结构
```c
struct servent
{
    char *s_name; /* official service name*/
    char **s_aliases;   /* alias list.*/
    int s_port;     /* port number,network byte order*/
    char *s_proto;  /*protocol to use*/
};
```
服务名参数servname必须指定。如果同时指定了协议(即protoname参数为非空指针)，那么指定服务必须有匹配的协议。
有些因特网服务既用TCP也用UDP提供，其他因特网服务则仅仅支持单个协议(例如FTP要求使用TCP)。如果protoname
未指定而servname指定服务支持多个协议，那么返回哪个端口号取决于实现。通常情况下这种选择无关紧要，因为支持
多个协议的服务往往使用相同的TCP端口号和UDP端口号，不过这点并没有保证。

servent结构中我们关心的主要字段是端口号。既然端口号是以网络字节序返回的，把它存放到套接字地址结构时绝对
不能调用htons。

本函数的典型调用如下:
```c
strcut servent *sptr;

sptr = getservbyname("domain", "udp");  // dns using udp
sptr = getservbyname("ftp", "tcp");     // ftp using tcp
sptr = getservbyname("ftp", NULL);    // ftp use tcp
sptr = getservbyname("ftp", "udp");     // this call will fail 
```
既然FTP仅仅支持TCP，第二个调用和第三个调用等效，第四个调用则会失败。以下是田/etc/services文件中典型的文本行:
grep -e ^ftp -e ^domain /etc/services
```input
ftp-data         20/udp     # File Transfer [Default Data]
ftp-data         20/tcp     # File Transfer [Default Data]
ftp              21/udp     # File Transfer [Control]
ftp              21/tcp     # File Transfer [Control]
domain           53/udp     # Domain Name Server
domain           53/tcp     # Domain Name Server
ftp-agent       574/udp     # FTP Software Agent System
ftp-agent       574/tcp     # FTP Software Agent System
ftps-data	989/udp     # ftp protocol, data, over TLS/SSL
ftps-data	989/tcp     # ftp protocol, data, over TLS/SSL
ftps		990/udp     # ftp protocol, control, over TLS/SSL
ftps		990/tcp     # ftp protocol, control, over TLS/SSL
domaintime      9909/udp    # domaintime
domaintime      9909/tcp    # domaintime
```

(这里查看services ftp其实使用了udp，domain也用了tcp（查询的数据太大，需要跨区域传输时候会使用tcp）,可能是unp这本书出的太久了，linux有过更新)

getservbyport函数
```c
#include <netdb.h>
struct servent *getservbyport(int port,const char* protoname);
// 返回:若成功则为非空指针，若出错则为NULL
```
port参数的值必须为网络字节序。本函数的典型调用如下:
```
struct servent *sptr;
sptr = getservbyport(htons(53),"udp"); /*DNS using UDP*/,
sptr = getservbyport(htons(21),"tcp"); /* FTP using TCP*/
sptr = getservbyport(htons(21),NULL);  /*FTP using TCP */
sptr = getservbyport(htons(21),"udp");  /*this call will fail */
```

因为UDP上没有服务使用端口21，所以最后一个调用将失败。

必须清楚的是，有些端口号在TCP上用于一种服务，在UDP上却用于完全不同的另一种服务。例如:
grep 514 /etc/services
```
shell           514/tcp     # cmd
syslog          514/udp #
```
表明端口514在TCP上由rsh命令使用，在UDP上却由syslog守护进程使用。512~514范围内的端口都有这个特性。






