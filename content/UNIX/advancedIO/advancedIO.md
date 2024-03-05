---
title: "高级I/O函数"
date: 2024-03-04T17:15:56+08:00
draft: true
---

## 套接字超时
在涉及套接字的I/O操作上设置超时的方法有以下3种。
1. 调用alarm，它在指定超时期满时产生SIGALRM信号。这个方法涉及信号处理，而信号处理在不同的实现上存在差异，
而且可能干扰进程中现有的alarm调用。
2. 在select中阻塞等待I/O(select有内置的时间限制)，以此代替直接阻塞在read或write调用上。
3. 使用较新的SO_RCVTIMEO和SO_SNDTIMEO套接字选项。这个方法的问题在于并非所有实现都支持这两个套接字选项。

上述三个技术都适用于输入和输出操作(例如read、write及其诸如recvfrom、sendto之类的变体)，不过我们依然期待
可用于connect的技术，因为TCP内置的connect超时相当长(典型值为75秒钟)。select可用来在connect上设置超时的
先决条件是相应套接字处于非阻塞模式，而那两个套接字选项对connect并不适用。前两个技术适用手任何描述符，而第三
个技术仅仅使用于套接字描述符。 

### 1. 使用SIGALRM为connect设置超时
[connect_timeo](https://github.com/lsill/unpvnote/blob/main/lib/connect_timeo.c?plain=1#L4)

指出两点，第一点是使用本技术总能减少connect的超时期限，伹是无法延长内核现有的超时。源自Berkeley的内校中
connect的超时通常为75s。在调用我们的函数时，可以指定一个比75小的值(如10)，但是如果指定一个比75大的
值(如80)，那么connect仍将在75s后发生超时。

另一点是我们使用了系统调用(connect)的可中断能力，使得它们能够在内核超时发生之前返回。这一点不成问题的前提
是:我们执行的是系统调用，并且能够直接处理由它们返回的EINTR错误。我们将在29.7节碰到一个也执行系统调用的
库函数，不过系统调用返回EINTR时这个库函数重新执行同一个系统调用。在这种情形下我们仍能使用SIGALRM，不过
将在图29-10中看到，我们还不得不使用sigsetjmp和siglongjmp以绕过函数库对于EINTR的忽略。

尽管本例子相当简单，但在多线程化程序中正确使用信号却非常困难(见第26章)。因此我们建议只是在未线程化或单线
程化的程序中使用本技术。

### 2. 使用SIGALRM为recvform设置超时
[dg_cli](https://github.com/lsill/unpvnote/blob/main/advio/dgclitimeo3.c?plain=1#L4)

### 3. 使用select为recvform设置超时
[readable_timeo](https://github.com/lsill/unpvnote/blob/main/lib/readable_timeo.c?plain=1#L4)

创建等待描述符变为可写的名为writeable_timeo的类似函数。








