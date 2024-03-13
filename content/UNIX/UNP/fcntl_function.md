---
title: "fcntl函数"
date: 2023-12-27T14:49:25+08:00
draft: true
---

与代表“file control”(文件控制)的名字相符，fcntl函数可执行各种描述符控制操作。在讲解函数及其
如何影响套接字之前，我们需要看得远一点。图7-20汇总了由fcntl、ioctl和路由套接字执行的不同操作。

其中前六个操作可由任何进程应用于套接字，接着两个操作(接口操作)比较少见，不过也是通用的，后两个
操作(ARP和路由表操作)由诸如ifconfig和route之类管理程序执行。

执行前四个操作的方法不止一种，不过我们在最后一列指出，POSIX规定fcntl方法是首选的。还指出，
POSIX提供sockatmark函数作为测试是否处于带外标志的首选方法。最后一列空白的其余操作没有被
POSIX标准化。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/7_20.jpg)

fcntl函数提供了与网络编程相关的如下特性:
- 非阻塞式I/O。通过使用F_SETFL命令设置Q_NONBLOCK文件状态标志，我们可以把一个套接字设置为非阻塞型。
- 信号驱动式I/O。通过使用F_SETFL.命令设置O_ASYNC文件状态标志，我们可以把一个套接字设置成一旦其状态
发生变化，内核就产生一个SIGIO信号。
- F_SETOWN 命令允许我们指定用于接收SIGIO和SIGURG信号的套接字属主(进程ID或进程组ID)。其中SIGIO信
号是套接字被设置为信号驱动式I/O型后产生的，SIGURG信号是在新的带外数据到达套接字时产生的。
F_GETOWN 命令返回套接字的当前属主。

```c
#include <fcntl.h>
int fcntl(int fd, int cmd,... /*int arg*/);
// 返同:若成功則取决于cmd，若出错则为-1
```
每种描述符(包括套接字描述符)都有一组由F_GETFL命令获取或由F_SETFL命令设置的文件标志。其中影响套接字描述符
的两个标志是:
- NONBLOCK: 非阻塞式I/O;
- O_ASYNC: 信号驱动式I/O;

非阻塞设置例子：
```c
int flags;
// set a socket as nonblocking
if ((flags = fcntl(fd, F_GETFL, 0)) < 0)
    err_sys("F_GETFL error");
flags |= O_NONBLOCK; 
if ((flags = fcntl(fd, F_SETFL, 0)) < 0)
    err_sys("F_SETFL error");
```


信号SIGIO和SIGURG与其他信号的不同之处在于，这两个信号仅在己使用F_SETOWN命令给相关套接字指派了属主后才会产生。
F_SETOWN命令的整数类型arg参数既可以是一个正整数，指出接收信号的进程ID，也可以是一个负整数，其绝对值指出接收信
号的进程组ID。F_GETOWN 命令把套接字属主作为fcntl函数的返回值返回，它既可以是进程ID(一个正的返回值)，也可以是
进程组ID(一个除-1以外的负值)。指定接收信号的套接字属主为一个进程或一个进程组的差别在于:前者仅导致单个进程接收
信号，而后者则导致整个进程组中的所有进程(也许不止一个进程)接收信号。

使用socket函数新创建的套接字并没有属主。然而如果一个新的套接字是从一个监听套接字创建来的，那么套接字属主将由已
连接套接字从监听套接字继承而来(许多套接字选项也是这样继承)。










