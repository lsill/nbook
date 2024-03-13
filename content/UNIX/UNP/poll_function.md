---
title: "poll函数"
date: 2023-12-25T16:48:34+08:00
draft: true
---

# poll函数
```c
#include <poll.h>
int poll(struct pollfd *fdarray, unsigned long nfds, int timeout);
// 返回:若有就绪描述符则为其数目，若超时期为0，若出错则为-1
```
第一个参数是指向一个结构数組第一个元素的指针。每个数组元素都是一个pollfd结构，用于指定测试某个给定描述符fa的条件。
```c
struct pollfd { 
    int fd;             // descriptor to check
    short events;       // events of interest on fd
    short revents;      // events that occurred on fd
);
```
要测试的条件由events成员指定，函数在相应的revents成员中返回该描述符的状态。(每个描述符都有两个变量，一个为调用值，另一个为返回结果，从而
避免使用值-结果参数。回想select函数的中间三个参数都是值-结果参数。)这两个成员中的每一个都由指定某个特定条件的一位或多位构成。
图6-23列出了用于指定events标志以及测试revents标志的一些常值。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/poll_eventsandrevents.jpg)
将该图分为三个部分:第一部分是处理输入的四个常值，第二部分是处理输出的三个常值，第三部分是处理错误的三个常值。
其中第三部分的三个常值不能在events中设置，但是当相应条件存在时就在revents中返回。
poll识别三类数据:普通(normal)、优先级带(priority band)和高优先級(high priority)。这些术语均出自基于流的实现

就TCP和UDP套接字而言，以下条件引起poll返回特定的revent。不幸的是，POSIX在其poll的定义中留了许多空洞(也就是说有多种方法可返回相
同的条件)。
- 所有正规TCP数据和所有UDP数据都被认为是普通数据。
- TCP的带外数据被认为是优先级带数据。
- 当TCP连接的读半部关闭时(醫如收到了一个来自对端的FIN)，也被认为是普通数据，随后的读操作將返回0。
- TCP连接存在错误既可认为是普通数据，也可认为是错误(POLLERR)。无论哪种情况，随后的读操作将返回-1，
并把errno设置成合适的值。这可用于处理诸如接收到RST或发生超时等条件。
- 在监听套接字上有新的连接可用既可认为是普通数据，也可认为是优先级数据。大多数实现视之为普通数据。
- 非阻塞式connect的完成被认为是使相应套接字可写。

结构数组中元素的个数是由nfds参数指定。

timeout参数指定poll函数返回前等待多长时间。它是一个指定应等待毫秒数的正值。图6-24给出了它的可能取值。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/poll_timeout.jpg)
INFTIN常值被定义为一个负值。如果系统不能提供毫秒级精度的定时器，该值就向上含入到最接近的支持值。

当发生错误时，poll函数的返回值为-1，若定时器到时之前没有任何描述符就绪，则返回0，否则返回就绪描述符的个数，即revents成员值
非0的描述符个数。

如果我们不再关心某个特定描述符，那么可以把与它对应的pollfd结构的events成员设置成一个负值。poll函数将忽略这样的pollfd结构的
events成员，返回时將它的revents成员的值置为0。

对于FD_SETSIZE以及就每个描述符中中最大描述符数目相比每个进程中最大描述符数日展开的讨论，有了poll就不再有那样的问题了，因为分配
一个pollfd结构的数组并把该数组中元素的数目通知内核成了调用者的责任。内核不再需要知道类似fa_set的固定大小的数据类型。












