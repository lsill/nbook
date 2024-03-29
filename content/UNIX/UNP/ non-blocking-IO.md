---
title: "非阻塞IO"
date: 2024-03-13T16:21:13+08:00
draft: true
---

## 1. 概述
套接字的默认状态是阻塞的。这就意味者当发出一个不能立即完成的套接字调用时，其进程将被投入睡眠，
等待相应操作完成。可能阻塞的套接字调用可分为以下四类。

(1)输入操作，包括read、readv、recv、recvfrom和recvmsg共5个函数。如果某个进程对一个阻塞
的TCP套接字(默认设置)调用这些输入函数之一，而且该套接字的接收缓冲区中没有数据可读，该进程将
被投入睡眠，直到有一些数据到达。既然TCP是字节流协议，该进程的唤醒就是只要有一些数据到达，这
些数据既可能是单个字节，也可以是一个完整的TCP分节中的数据。如果想等到某个固定数目的数据可读
为止，那么可以调用我们的readn函数，或者指定MSG_WAITALL标志。

既然UDP是数据报协议，如果一个阻塞的UDP套接字的接收缓冲区为空，对它调用输入函数的进程将被投
入睡眠，直到有UDP数据报到达。

对于非阻塞的套接字，如果输入操作不能被满足(对于TCP套接字即至少有一个字节的数据可读，对于UDP
套接字即有一个完整的数据报可读)，相应调用将立即返回一个EMOULDBLOCK错误。

(2)输出操作，包括write、writev、send、sendto和sendmsg共5个函数。对于一个TCP套接字，内核
将从应用进程的缓冲区到该套接字的发送缓冲区复制数据。对于阻塞的套接字，如果其发送缓冲区中没有空
间，进程将被投入睡眠，直到有空间为止。

对于一个非阻塞的TCP套接字，如果其发送缓冲区中根本没有空间，输出函数调用将立即返回一个EWOULDBLOCK
错误。如果其发送缓冲区中有一些空间，返回值将是内核能够复制到该缓冲区中的字节数。这个字节数也称为不
足计數(short count)。

UDP套接字不存在真正的发送缓冲区。内核只是复制应用进程数据并把它沿协议栈向下传送，渐次冠以UDP首部和
IP首部。因此对一个阻塞的UDP套接字(默认设置)，输出函数调用将不会因与TCP套接字一样的原因而阻塞，不
过有可能会因其他的原因而阻塞。

(3)接受外来连接，即accept函数。如果对一个阻塞的套接字调用accept函数，并且尚无新的连接到达，调用进程
将被投入睡眠。

如果对一个非阻塞的套接字调用accept函数，并且尚无新的连接到达，accept调用将立即返回一个ENOULDBIOCK错误。

(4)发起外出连接，即用于TCP的connect函数。connect同样可用于UDP，不过它不能使一个“真正”的连接建立起来，
它只是使内核保存对端的IP地址和端口号。TCP连接的建立涉及一个三路握手过程，而且connect函数一直要等到客户
收到对于自己的SYN的ACK为止才返回。这意味着TCP的每个connect总会阻塞其调用进程至少一个到服务器的RTT时间。

如果对一个非阻塞的TCP套接字调用connect，并且连接不能立即建立，那么连接的建立能照样发起(臂如送出TCP三路
握手的第一个分组)，不过会返回一个EINPROGRESS错误。注意这个错误不同与上述三个情形中返回的错误。另请注意
有些连接可以立即建立，通常发生在服务器和客户处于同一个主机的情况下。因此即使对于一个非阻塞的connect，我
们也得预备connect成功返回的情况发生。

## 2. 非阻塞读和写:str_cli函数
举例来说，如果在标准输入有一行文本可读，我们就调用read读入它，再调用writen把它发送给服务器。然而如果套接字
发送缓冲区已满，writen调用将会阻塞。在进程阻塞于writen调用期间，可能有来自套接字接收缓冲区的数据可供读取。
类似的，如果从套接宇中有一行输入文本可读，那么一旦标准输出比网络还要慢，进程照样可能阻塞于后续的write调用。
本节的目标是开发这个函数的一个使用非阻塞式I/O的版本。这样可以防止进程在可做任何有效工作期间发生阻塞。

不幸的是，非阻塞式IO的加入让本函数的缓冲区管理显著地复杂化了，因此将分片介绍这个函数。套接字上使用标准I/O的
潜在问题和困难在非阻塞式I/O操作中显得尤为突出。本例子中继续避免使用标准I/O。

维护着两个缓冲区:to容纳从标准输入到服务器去的数据，fr容纳自服务器到标准输出来的数据。图16-1展示了to缓冲区
的组织和指向该缓冲区中的指针。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_1.jpg)

其中toiptr指针指向从标准输入读入的数据可以存放的下一个字节。tooptr指向下一个必须写到套接字的字节。有
(toiptr-tooptr)个字节需写到套接字。可从标准输入读入的字节数是(&to[MAXLINE]-toiptr)。一旦tooptr
移动到toiptr，这两个指针就一起恢复到缓冲区开始处。

图16-2展示了fr缓冲区相应的组织
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_2.jpg)

[str_cli](https://github.com/lsill/unpvnote/blob/main/nonblock/strclinonb.c?plain=1#L4)

[gf_time函数](https://github.com/lsill/unpvnote/blob/main/lib/gf_time.c?plain=1#L4)
gf_time函数返回一个含有当前时间的字符串，包括微秒，格式如下。 12:34:56.123456

![非阻塞式I/O例子的时间线](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_8.jpg)

### 1. str_cli 的较简单版本
上面给出的str_cli函数非阻塞版本比较复杂-—约有135行代码。我们知道代码长度从20行倍增到40行的努力是值得的，因为在批量模式下执行速度
几乎提高了30倍，而且在阻塞的描述符上使用select并不太复杂。然布考虑到结果代码的复杂性，把应用程序编写成使用非阻塞式I/O的努力是否照
样值得呢?回答是否定的。每当我们发现需要使用非阻空式I/O时，更简单的办法通常是把应用程序任务划分到多个进程(使用fork)或多个线程。

下面是str_cli函数的另一个版本，该函数使用fork把当前进程划分成两个进程。这个函数一开始就调用fork把当前进程划分成一个父进程和一个
子进程。子进程把来自服务器的文本行复制到标准输出，父进程把来自标准输入的文本行复制到服务器，如图16-9所示。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_9.jpg)

[str_cli fork简易版](https://github.com/lsill/unpvnote/blob/main/nonblock/strclifork.c?plain=1#L4)

上面代码明确地指出所用TCP连接是全双工的，而且父子进程共字同一个套接字:父进程往该套接字中写，子进程从该套接字中读。尽管套接字只
有一个，其接收缓冲区和发送缓冲区也分别只有一个，然而这个套接字却有两个描述符在引用它:一个在父进程中，另一个在子进程中。

同样需要考虑进程终止序列。正常的终止序列从在标准输入上遇到EOF之时开始发生。父进程读入来自标准输入的EOF后调用shutdown发送FIN。
(父进程不能调用close，会导致子进程也会被关闭)但当这发生之后，子进程需继续从服务器到标准输出执行数据复制，直到在套接字上读到EOF。

服务器进程过早终止也有可能发生。要是发生这种情况，子进程将在套接字上读到EOF。这样的子进程必须告知父进程停止从标准输入到套接字
复制数据。上面代码中子进程向父进程发送一个SIGTERM信号，以防父进程仍在运行。如此处理的另一个手段是子进程无为地终止，使得父进程
(如果仍在运行的话)捕获一个SIGCHLD信号。

父进程完成数据复制后调用pause让自己进入睡眠状态，直到捕获一个信号(子进程来的SIGTERM信号)，尽管它不主动捕获任何信号。SIGTERM
信号的默认行为是终止进程，这对于本例子是合适的。让父进程等待子进程的目的在于精确测量调用此版str_cli函数的TCP客户程序的执行时钟
时间。正常情况下子进程在父进程之后结束，然而用于测量时钟时间的是shell内部命令time，它要求父进程持续到测量结束时刻。

注意该版本相比本节前面给出的非阻塞版本体现的简单性。非阻塞版本同时管理4个不同的I/O流，而且由于这4个流都是非阻寨的，不得不考虑对
于所有4个流的部分读和部分写问题。然而在fork版本中，每个进程只处理2个I/O流，从一个复制到另一个。这里不需要非阻塞式1O，因为如果
从输入流没有数据可读，往相应的输出流就没有数据可写。

### 3. 非阻塞connect
当在一个非阻塞的TCP套接字上调用connect时，connect将立即返回一个EINPROGRESS错误，不过已经发起的TCP三路握手继续进行。接者
使用select检测这个连接或成功或失败的已建立条件。非阻塞的connect有三个用途。

(1)我们可以把三路握手叠加在其他处理上。完成一个connect要花一个RTT时间，而RTT波动范围很大，从局域网上的几个毫秒到几百个毫秒
甚至是广域网上的几秒。这段时间内也许有我们想要执行的其他处理工作可执行。

(2)可以使用这个技术同时建立多个连接。这个用途已随着Web浏览器变得流行起来。

(3)既然使用select等待连接的建立，可以给select指定一个时间限制，使得能够缩短connect的超时。许多实现有着从75秒钟到数分钟
的connect超时时间。应用程序有时想要一个更短的超时时间，实现方法之一就是使用非阻塞connect。

非阻塞connect虽然听似简单，却有一些必须处理的细节。
- 尽管套接字是非阻塞的，如果连接到的服务器在同一个主机上，那么当调用connect时，连接通常立刻建立。
- 源自Berkeley的实现(和POSIX)有关于select和非阻寨connect的以下两个规则:(1)当连接成功建立时，描述符变为可写
(2)当连接建立遇到错误时，描述符变为既可读又可写。

### 4. 非阻塞connect:时间获取客户程序
[connect_nonb](https://github.com/lsill/unpvnote/blob/main/lib/connect_nonb.c?plain=1#L4)
套接字的各种实现以及非阻塞connect会带来移植性问题。首先，调用select之前有可能连接已经建立并有来自对端的数据到达。这种情况
下即使套接字上不发生错误，套接字也是既可读又可写，这和连接建立失败情况下套接字的读写条件一样。上中的代码通过调用getsockopt
并检查套接字上是否存在待处理错误来处理这种情形。

其次，既然不能假设套接字的可写(而不可读)条件是select返回套接字操作成功条件的唯一方法，下一个移植性问题就是怎样判断连接建立
是否成功。张贴到Usenet上的解决办法各式各样。这些方法可以上面代码中的getsockopt调用。

(1)调用getpeername代替getsockopt。如果getpeername以ENOTCONN错误失败返回，那么连接建立已经失败，必须接着以SO_ERROR
调用getsockopt取得套接字上待处理的错误。

(2)以值为0的长度參数调用read。如果read失败，那么connect己经失败，read返回的errno给出了连接失败的原因。如果连接建立成功，
那么read应该返回0。

(3)再调用connect一次。它应该失败，如果错识是EISCONN，那么套接字已经连接，也就是说第一次连接已经成功。

不幸的是，非阻塞connect是网络编程中最不易移植的部分。使用该技术必须准备应付移植性问题，特别是对于较老的实现。避免移植性问题
的一个较简单技术是为每个连接创建一个处理线程。

**被中断的connect**
对于一个正常的阻塞式套接字，如果其上的connect调用在TCP三路握手完成前被中断(臂如说捕获了某个信号)，将会发生什么呢?假设被中断
的connect调用不由内核自动重启，那么它将返回EINTR。不能再次调用connect等待未完成的连接继续完成。这样做將导致返回EADDRINUSE
错误。

这种情形下我们只能调用select，就像本节对于非阻塞connect所做的那样。连接建立成功时select返回套接字可写条件，连接建立失败时
select返回套接字既可读又可写条件。

## 5.非阻塞connect:Web客户程序
非阻塞connect的现实例子出自Netscape的Web客户程序。客户先建立一个与某个Web服务器的HTTP连接，再获取一个主页(homepage)。
该主页往往含有多个对于其他网页(Webpage)的引用。客户可以使用非阻塞connect同时获取多个网页，以此取代每次只获取一个网页的
串行获取手段。图16-12展示了一个并行建立多个连接的例子。最左边情老表示串行执行所有3个连接。假设第一个连接耗用10个时间单位，
第二个耗用15个，第三个耗用4个，总计29个时间单位。

中间情形并行执行2个连接。在时刻0启动前2个连接，当其中之一结束时，启动第三个链接。总计耗时差不多减半，从29变为15，不过必
须意识到这是就理想情况而言。如果并行执行的连接共享同一个低速链路(售如说客户主机通过一个拨号调制解调器链路接入因特网)，那
么每个连接可能彼此竞用有限的资源，使得每个连接都可能耗用更长的时间。举例来说，10个时间单位的连接可能变为15，15个时间单位
的可能变为20，4个时间单位的可能变为6。即便如此，总计耗时将是21，仍然短于串行执行的情形。

最右边情形并行执行所有3个连接，其中再次假设这3个连接之间没有干扰(理想情况)。然而就选择的例子时间而言，本情形的总计耗时和
中间情形的一样，都是15个时间单位。在处理Web客户时，第一个连接独立执行，来自该连接的数据含有多个引用，随后用于访问这些引
用的多个连接则并行执行，如图16-13所示。

为了进一步优化连接执行序列，客户可以在第一个连接尚未完成前就开始分析从中陆续返回的数据，以便尽早得悉其中含有的引用，并尽
快启动相应的额外连接。既然准备同时处理多个非阻塞connect，我们就不能使用上面代码connect_nonb函数，因为它直到连接已
经建立才返回。必须自行管理这些(可能尚末成功建立的)连接。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_12.jpg)
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_13.png)
- [web.h](https://github.com/lsill/unpvnote/blob/main/nonblock/web.h?plain=1#L3)
- [web.c](https://github.com/lsill/unpvnote/blob/main/nonblock/web.c?plain=1#L4)
- [homepage](https://github.com/lsill/unpvnote/blob/main/nonblock/homepage.c?plain=1#L4)
- [start_connect](https://github.com/lsill/unpvnote/blob/main/nonblock/start_connect.c?plain=1#L4)
- [write_get_cmd](https://github.com/lsill/unpvnote/blob/main/nonblock/write_get_cmd.c?plain=1#L4)

**同时连接的性能**

同时建立多个连接的性能收益如何呢?图16-20给出了获取某个Web服务器的主页并后跟来自该服务器的9个图像文件所需的时钟时间。
到该服务器的RTT约为150ms。主页的大小为4017字节，9个图像文件的平均大小为1621字节。TCP分节大小为512字节。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/16_20.png)

主要的性能改善是在同时连接数为3的时候取得的(时钟时间减半)，同时连接数为4或更多的时候性能增长要少得多。

## 6. 非阻塞accept
当有一个己完成的连接准备好被accept时，select将作为可读描述符返回该连接的监听套接字。因此，如果使用select在某个监听
套接字上等待一个外来连接，那就没有必要把该监听套接字设置为非阻塞，这是因为如果select告诉我们该套接字上已有连接就绪，
那么随后的accept调用不应该阻塞。

不幸的是，这里存在一个可能让我们掉入陷阱的定时问题[Gierth 1996]。为了查看这个问题，TCP回射客户程序改写成建立连接后发送一个RST到服务器。
[tcpcli03](https://github.com/lsill/unpvnote/blob/main/nonblock/tcpcli03.c?plain=1#L3)

在select返回监听套接字的可读条件之后但在调用accept之前暂停。如下面这段代码中.
```c
if (FD_ISSET(listenfd, &rset)) { /* new client connection */ 
    printf ("listening socket readable\n");
    sleep(5);
    clilen = sizeof (cliaddr);
    connfd = Accept (listenfd, (SA *) &cliaddr, &clilen);
```
模拟一个繁忙的服务器，它无法在select返回监听套接字的可读条件后就马上调用accept。通常情况下服务器的这种迟钝不成问题(实际上这就是要维护一个己
完成连接队列的原因)，但是结合上连接建立之后到达的来自容户的RST，问题就出现了。

当客户在服务器调用accept之前中止某个连接时，源自Berkeley的实现不把这个中止的连接返回给服务器，而其他实现应该返回ECONNABORTED错误，却往往代之以
返回EPROTO错误。考虑一个源自Berkeley的实现上的如下例子。
- 客户如`tcpcli03`所示建立一个连接并随后中止它。
- select向服务器进程返回可读条件，不过服务器要过一小段时间才调用accept。
- 在服务器从select返回到调用accept期间，服务器TCP收到来自客户的RST。
- 这个已完成的连接被服务器TCP驱除出队列，我们假设队列中没有其他已完成的连接。
- 服务器调用accept，但是由于没有任何已完成的连接，服务器于是阻塞。

服务器会一直阻塞在accept调用上，直到其他某个客户建立一个连接为止。但是在此期间，服务器单纯阻塞在accept调用上，
无法处理任何其他己就绪的描述符。

本问题的解决办法如下。

(1)当使用select获悉某个监听套接字上何时有己完成连接准备好被accept时，总是把这个监听套接字设置为非阻塞。

(2)在后续的accept调用中忽略以下错误:ENOULDBLOCK(源自Berkeley的实现，客户中止连接时)、
ECONNABORTED(POSIX实现，客户中止连接时)、EPROTO(SVR4实现，客户中止连接时)和EINTR(如果有信号被捕获)。

## 7. 小结
select通常结合非阻塞式I/O一起使用，以便判断描述符何时可读或可写。这个版本的客户程序是给出的所有版本中执行速度最快的，
尽管其代码修改确非易事。在这之后展示说明使用fork把客户程序划分成两部分由不同进程分别执行要简单得多。

非阻塞connect使我们能够在TCP三路握手发生期间做其他处理，而不是光阻塞在connect上。不幸的足，非阻塞connect不可移植，
不同的实现有不同的手段指示连接已成功建立或己碰到错误。我们使用非阻塞connect开发了一个新型客户程序，它类似同时打开多
个TCP连接以较少从单个服务器取得多个文件所需时钟时间的Web客户程序。如此发起多个连接可以减少时钟时间，不过考感到TCP的
拥塞避免机制，它是对网络不利的。


## 习题
Q:父进程必须调用shutdown而不是close。这是为什么?

A:套接字描述符是在父子进程之间共享的，因此它的引用计数为2。要是父进程调用close，那么这只是把该引用计数由2减为1，
而且既然它仍然大于0，FIN就不发送。这就是使用shutdown函数的另一个理由:即使描述符的引用计数仍然大于0，FIN也被强迫发送出去。

Q:如果服务器进程过早终止，而客户子进程收到来自服务器的EOF后不通知父进程就终止，将会发生什么?

A:父进程将继续写出到己经接收FIN的套接字。它发送给服务器的第一个分节将引发RST响应。此后的那个write调用将导致内核像
向父进程发送SIGPIPE信号。
（当服务器进程过早终止，TCP连接会被关闭，客户端套接字会接收到EOF（即read函数返回0）。如果客户端子进程在收到EOF后
立即终止而不通知其父进程，父进程可能仍然尝试向这个已经关闭的套接字写入数据。当父进程尝试向一个已经接收到对端FIN标志
（即对端关闭连接）的套接字写入数据时，TCP协议规定，接收到FIN的一方（此处为客户端）不应再发送数据。因此，客户端TCP
会向服务器发送一个RST（重置）标志，以告知对方出现了错误（试图向一个已经关闭的连接发送数据）。在父进程的下一个write
调用时，因为套接字已经收到了RST标志，操作系统会给父进程发送一个SIGPIPE信号。默认情况下，接收到SIGPIPE信号会导致
进程终止。如果父进程没有处理这个信号（比如忽略它或者提供一个处理函数），那么它将因为这个信号而终止。简而言之，如果服
务器过早终止，客户端子进程在不通知父进程的情况下终止，那么父进程可能会在尝试写入已关闭的套接字时收到一个SIGPIPE信号
，并因此终止。这就是为什么通常需要在网络编程中妥善处理EOF和信号的原因。）

Q:如果父进程在子进程之前意外死亡，而子进程随后从套接字读到EOF，将会发生什么?

A:当子进程调用getppid向父进程发送SIGTERU信号时，所返回的进程ID将是1即init进程，它是所有孤儿进程的继父(也就是说
它继承所有其父进程在子进程仍在运行时就终止的那些子进程)。子进程试图向init进程发送这个信号，但是没有足够的权限。然
而如果这个客户程序有机会以超级用户特权运行，从而允许它向init发送信号，那么在发送该信号之前应该检测getppid的返回值。
（子进程检测到其父进程已经终止（getppid()返回1），它应该避免向init进程发送信号。在实际编程中，子进程应该在尝试向父
进程发送信号之前检查父进程的状态，以避免不必要或潜在的危险操作。）

Q:来自对端的数据有可能在本端的connect调用返回前到达套接字。这是如何发生的?

A:如果服务器在accept调用返回之后立即发送数据，而当三路握手的第二个分节到达以在容户端完成连接的时候客户主机却比较忙，
那么来自服务器的数据可能在客户的connect调用返回之前到达。举例来说，SMTP服务器在未从中读之前就立即往一个新建立的连
接中写，以便给客户发送一个问候消息。

这种情况通常发生在以下场景中：
1. 客户端发起连接请求（TCP三次握手的第一次分节）。
2. 服务器接收到连接请求，并发送回应（TCP三次握手的第二次分节），同时accept调用返回，服务器立即发送一些数据（比如SMTP服务器发送问候消息）。
3. 由于客户端主机比较忙，处理第二次分节的完成连接的动作（TCP三次握手的第三次分节）和connect调用的返回被延迟。
4. 在connect调用返回之前，服务器发送的数据已经到达客户端。

这种情况下，客户端的套接字缓冲区中可能已经有了来自服务器的数据，即使connect调用还没有返回。因此，一旦connect返回，
客户端就可以立即开始读取这些数据，而不需要等待服务器再次发送。

这个现象说明了TCP连接建立的过程是异步的，数据传输可以在连接的各个阶段发生，而不仅仅是在连接完全建立之后。这也意味着在
编写网络程序时，需要考虑到这种异步性，确保程序能够正确处理各种网络事件的顺序和时机。















