---
title: "I/O模型"
date: 2023-12-22T15:47:33+08:00
draft: true
---

## 1. I/O模型
unix下可用的5种I/O模型基本区别：
- 阻塞式I/O;
- 非阻塞式I/O;
- I/O复用(select和po11);
- 信号驱动式I/O(SIGIO);
- 异步I/O(POSIX的aio_系列函数)。

### 1. 阻塞式I/O模型
最流行的IO模型是阻塞式I/O(blockingI/O)模型。默认情形下，所有套接字都是阻塞的。

### 2. 非阻塞式I/O模型
进程把一个套接字设置成非阻塞是在通知内核:当所请求的I/O操作非得把本进程投入睡眠才能完成时，不要
把本进程投入睡眠，而是返回一个错误。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/noblockIO.jpg)

前三次调用recvfrom时没有数据可返回，因此内核转而立即返回一个EVOULDBLOCK错误。第四次调用recvfrom时
己有一个数据报准备好，它被复制到应用进程级冲区，于是recvfrom成功返回。我们接着处理数据。
当一个应用进程像这样对一个非阻塞描述符循环调用recvfrom时，我们称之为轮询〈polling)。应用进程持线轮
询内核，以查看某个操作是否就绪。这么做往往耗费大量CPU时间，不过这种模型偶尔也会遇到，通常是在专门提供
某一种功能的系统中才有。

### 3. I/O复用模型
有了I/O复用(I/O multiplexing)，我们就可以调用select或poll，阻塞在这两个系统调用中的某一个之上，而
不是阻塞在真正的I/O系统调用上。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/IO_multiplexing.jpg)

我们阻塞于select调用，等待数据报套接字交为可读。当select返回套接字可读这一条件时，我们调用recvfrom
把所读数据报复制到应用进程缓冲区。

比较图6-3和图6-1，I/O复用并不显得有什么优势，事实上由于使用select需要两个而不是单个系统调用，I/O复用还稍有劣势
。不过使用select的优势在于我们可以等待多个描述符就绪。

### 4. 信号驱动式I/O模型
我们也可以用信号，让内核在描达符就绪时发送SIGIO信号通知我们。我们称这种模型为信号驱动式I/O
(signal-drivenI/O)，图6-4是它的概要展示。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/signal_driveIO.jpg)
我们首先开启套接字的信号驱动式I/O功能，并通过sigaction系统调用安装一个信号处理函数。该系统调用将立即返回，我们的进程继续
工作，也就是说它没有被阻寒。当数据报准备好读取时，内核就为该进程产生一个SIGIO信号。我们随后既可以在信号处理两数中调用
recvfrom读取数据报，并通知主循环数据己准备好待处理，也可以立即通知主循环，让它读取数据报。无论如何处理SIGIO信号，这种
模型的优势在于等待数据报到达期间进程不被阻塞。主循环可以继续执行，只要等待来自信号处理函数的通知:既可以是数据已准备好被
处理，也可以是数据报已准各好被读取。

### 5. 异步I/O模型
异步I/O(asynchronousI/0)由POSIX规范定义。演变成当前POSIX規范的各种早期标准所
定义的实时函数中存在的差异已经取得一致。一般地说，这些函数的工作机制是:告知内核启动某个操作，并让内核在整个操作(包括将数据
从内核复制到我们自己的缓冲区)完成后通知我们。这种模型与信号驱动模型的主要区别在于:信号驱动式I/O是由内核通知我们何时可以启
动一个I/O操作，而异出I/O模型是由內核通知我们I/O操作何时完成。图6-5给出了一个例子。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/asyncIO.jpg)
我们调用aio_reaa函数(POSIX异步I/O函数以aio_或lio_开头)，给内核传递描述符、缓冲区指针、缓冲区大小(与read相同的三个参数)
和文件信移(与lseek类似)，并告诉内核当整个操作完成时如何通知我们。该系统调用立即返回，而且在等待I\O完成期间，我们的进程不被
阻塞。本例子中我们假设要求内核在操作完成时产生某个信号。该信号直到数据已复制到应用进程缓冲区才产生，这一点不同于信号驱动式I/O模型。

### 6. 各种I/O比较
前4种模型的主要区别在于第一阶段，因为它们的第二阶段是一样的:在数据从内核复制到调用者的缓冲区期间，进程阻塞于recvfrom调用。
相反，异步I/O模型在这两个阶段都要处理，从而不同于其他4种模型。

### 7 同步I/O0和异步I/O 对比
POSIX把这两个术语定义如下:
- 同步I/O操作(synchronous I/O opetation)导致请求进程阻塞，直到I/O操作完成;
- 异步I/O操作(asynchronous I/0 opetation)不导致请求进程阻塞。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/fiveIO_compare.jpg)






