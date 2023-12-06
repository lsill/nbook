---
title: "并发编程"
date: 2023-12-05T15:26:28+08:00
draft: true
---
# 并发编程
如果逻辑控制流在时间上重叠，那么它们就是`并发的(concurrent)`。这种常见的现象称为`并发(concurrency)`，
出现在计算机系统的许多不同层面上。硬件异常处理程序、进程和Linux信号处理程序都是大家很熟悉的例子。

并发情况：
序访问虚拟内存的一个未定义的区域。应用级并发在其他情况下也是很有用的:
- 访问慢速I/O设备。当一个应用正在等待来自慢速I/O设备(例如磁盘)的数据到达时，内核会运行其他进程
，使CPU保持繁忙。每个应用都可以按照类似的方式，通过交替执行I/O请求和其他有用的工作来利用并发。
- 与人交互。和计算机交互的人要求计算机有同时执行多个任务的能力。例如，他们在打印一个文档时，可能
想要调整一个窗口的大小。现代视窗系统利用并发来提供这种能力。每次用户请求某种操作(比如通过单击鼠
标)时，一个独立的并发逻辑流被创建来执行这个操作。
- 通过推迟工作以降低延迟。有时，应用程序能够通过推迟其他操作和并发地执行它们，利用并发来降低某些
操作的延迟。比如，一个动态内存分配器可以通过推迟合并，把它放到一个运行在较低优先级上的并发“合并”
流中，在有空闲的CPU周期时充分利用这些空闲周期，从而降低单个free操作的延迟。
- 服务多个网络客户端。迭代网络服务器是不现实的，因为它们一次只能为一个客户端提供服务。因此，一个
慢速的客户端可能会导致服务器拒绝为所有其他客户端服务。对于一个真正的服务器来说，可能期望它每秒为
成百上千的客户端提供服务，由于一个慢速客户端导致拒绝为其他客户端服务，这是不能接受的。一个更好的
方法是创建一个并发服务器，它为每个客户端创建一个单独的逻辑流。这就允许服务器同时为多个客户端服务
，并且也避免了慢速客户端独占服务器。
- 在多核机器上进行并行计算。许多现代系统都配备多核处理器，多核处理器中包含有多个CPU。被划分成并
发流的应用程序通常在多核机器上比在单处理器机器上运行得快，因为这些流会并行执行，而不是交错执行。
使用应用级并发的应用程序称为并发程序(concurrent program)。现代操作系统提供了三种基本的构造并
发程序的方法（如下）:
- 进程。用这种方法，每个逻辑控制流都是一个进程，由内核来调度和维护。因为进程有独立的虛拟地址空间
，想要和其他流通信，控制流必须使用某种显式的进程间通信(interprocess communication，IPC)机制。
- I/O多路复用。在这种形式的并发编程中，应用程序在一个进程的上下文中显式地调度它们自己的逻辑流。
逻辑流被模型化为状态机，数据到达文件描述符后，主程序显式地从一个状态转换到另一个状态。因为程序是一
个单独的进程，所以所有的流都共享同一个地址空间。
- 线程。线程是运行在一个单一进程上下文中的逻辑流，由内核进行调度。你可以把线程看成是其他两种方式的
混合体，像进程流一样由内核进行调度，而像I/O多路复用流一样共享同一个虚拟地址空间。

## 1. 基于进程的并发编程
构造并发程序最简单的方法就是用进程，使用那些大家都很熟悉的函数，像fork、exec和waitpid。
例如，一个构造并发服务器的自然方法就是，在父进程中接受客户端连接请求，然后创建一个新的子进
程来为每个新客户端提供服务。

为了了解这是如何工作的，假设我们有两个客户端和一个服务器，服务
器正在监听一个监听描述符(比如指述符3)上的连接请求。现在假设服务器接受了客户端1的连接请求，
并返回一个已连接描述符(比如指述符4)。在接受连接请求之后，服务器派生一个子进程，这个子进程
获得服务器描述符表的完整副本。子进程关闭它的副本中的监听描述符3，而父进程关闭它的已连接描
述符4的副本，因为不再需要这些描述符了。

因为父、子进程中的已连接描述符都指向同一个文件表表项，所以父进程关闭它的已连接描述符的副本
是至关重要的。否则，将永不会释放已连接描述符4的文件表条目，而且由此引起的内存泄漏将最终消
耗光可用的内存，使系统崩溃。

现在，假设在父进程为客户端1创建了子进程之后，它接受一个新的客户端2的连接请求，并返回一个新
的已连接描述符(比如描述符5)。然后，父进程又派生另一个子进程，这个子进程用己连接描述符5为它
的容户端提供服务。此时，父进程正在等待下一个连接请求，而两个子进程正在并发地为它们各自的客
户端提供服务。

### 1. 基于进程的并发服务器
```c
#include "csapp.h"

void echo(int connfd);

void sigchld_handler(int sig)
{
    while (waitpid(-1, 0, WNOHANG) > 0);
    return;
}

int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;

    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    Signal(SIGCHLD, sigchld_handler);
    listenfd = Open_listenfd(*argv[1]);
    while (1) {
        clientlen = sizeof(struct sockaddr_storage);
        connfd = Accept(listenfd, (SA*) &clientaddr, &clientlen);
        if (Fork() == 0) {
            Close(listenfd);
            echo(connfd);
            Close(connfd);
            exit(0);
        }
        Close(connfd);
    }
}
```
。关于这个服务器，有几点重要内容需要说明:
- 首先，通常服务器会运行很长的时间，所以我们必须要包括一个SIGCHLD处理程序，来回收僵死(zombie)
子进程的资源。因为当SIGCHLD处理程序执行时，SIGCHLD信号是阻塞的，而Linux信号是不排队的，所以
SIGCHLD处理程序必须准备好回收多个僵死子进程的资源。
- 其次，父子进程必须关闭它们各自的connfd副本。就像我们已经提到过的，这对父进程而言尤为重要，它
必须关闭它的已连接描述符，以避免内存泄漏。
- 最后，因为套接字的文件表表项中的引用计数，直到父子进程的connfd都关闭了，到客户端的连接才会终止。

### 2. 进程的优劣
对于在父、子进程间共享状态信息，进程有一个非常清晰的模型:共享文件表，但是不共享用户地址空间。
进程有独立的地址空间既是优点也是缺点。这样一来，一个进程不可能不小心覆盖另一个进程的虚拟内存，
这就消除了许多令人迷惑的错误一—这是一个明显的优点。

另一方面，独立的地址空间使得进程共享状态信息变得更加困难。为了共享信息，它们必须使用显式的
IPC(进程间通信)机制。(参见下面的旁注。)基于进程的设计的另一个缺点是，它们往往比较慢，因为
进程控制和IPC的开销很高。

## 2. 基于I/O多路复用的并发编程
假设要求你编写一个echo服务器，它也能对用户从标准输入键入的交互命令做出响应。在这种情况下，服务器
必须响应两个互相独立的I/O事件:1)网络客户端发起连接请求，2)用户在键盘上键入命令行。我们先等待哪个
事件呢?没有哪个选择是理想的。如果在accept中等待一个连接请求，我们就不能响应输人的命令。类似地，
如果在read中等待一个输人命令，我们就不能响应任何连接请求。

针对这种困境的一个解决办法就是I/O多路复用(I/0 multiplexing)技术。基本的思路就是使用select函数，
要求内核挂起进程，只有在一个或多个I/O事件发生后，才将控制返回给应用程序，就像在下面的示例中一样:
- 当集合{0，4}中任意描述符准备好读时返回。
- 当集合{1，2，7}中任意描述符准备好写时返回。
- 如果在等待一个I/O事件发生时过了152.13秒，就超时。

select是一个复杂的函数，有许多不同的使用场景。我们将只讨论第一种场景:等待一组描述符准备好读。
```c
#include <sys/select.h>
int select(int n, fd_set *fdset, NULL, NULL, NULL);
// 返回已准备好的描述符的非零的个数，若出错則为一1。
FD_ZERO(fd_set *fdset); // clear all bits in fdset
FD_CLR(int fd, fd_set *fdset);  // clear bit fd in fdset
FD_SET(int fd, fd_set *fdset);  // turn on bit fd in fdset
FD_ISSET(int fd, fd_set *fdset);    // Is bit fd in fdset on?
```
select函数处理类型为fd_set的集合，也叫做`描述符集合`。逻辑上，我们将描述符集合看成一个大小为n
的位向量(在2.1节中介绍过):bn-1,...,b1,b0 每个位bk对应于描述符k。当且仅当bk=1，描述符k才表
明是描述符集合的一个元素。只允许你对描述符集合做三件事:1)分配它们，2)将一个此种类型的变量赋值
给另一个变量，3)用FD_ZERO、FD_SET、FD_CLR和FD_ISSET宏来修改和检查它们。

针对我们的目的，select函数有两个输入:一个称为`读集合`的描述符集合(fdset)和该读集合的基数(n)(实
际上是任何描述符集合的最大基数)。select函数会一直阻塞，直到读集合中至少有一个描述符准备好可以读。
当且仅当一个从该描述符读取一个字节的请求不会阻塞时，描述符k就表示`准备好可以读了`。select有一个
副作用，它修改参数fdset指向的fd_set，指明读集合的一个子集，称为`准备好集合(ready set)`，这个
集合是由读集合中准备好可以读了的描述符组成的。该函数返回的值指明了准备好集合的基数。注意，由于这
个副作用，我们必须在每次调用select时都更新读集合。

### 1. 基于I/O多路复用的并发事件驱动服务器

```c
#include "csapp.h"

typedef struct {
    int maxfd;  // largest descriptor in read_set
    fd_set read_set;    // set of all active descriptors
    fd_set ready_set;   // subset of descriptors ready for reading
    int nready; // number of ready descriptors from select
    int maxi;   // high water index into client array
    int clientfd[FD_SETSIZE];   // set of active descriptors
    rio_t clientrio[FD_SETSIZE];    // set of active read buffers
}pool;

int byte_cnt = 0;   // counts total bytes received by server

void init_pool(int listenfd, pool* p);
void add_client(int connfd, pool* p);

int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    static pool pool;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    listenfd = Open_listenfd(*argv[1]);
    init_pool(listenfd, &pool);

    while (1)
    {
        // wait for listening/connected descriptor(s) to become ready
        pool.ready_set = pool.read_set;
        pool.nready = Select(pool.maxfd + 1, &pool.ready_set, NULL, NULL,NULL);

        // if listening descriptor ready, add new client to pool
        if (FD_ISSET(listenfd, &pool.ready_set))
        {
            clientlen = sizeof(struct sockaddr_storage);
            connfd = Accept(listenfd, (SA*)&clientaddr, &clientlen);
        }
        // echo a text line from each ready connected descriptor
        check_clients(&pool);
    }

}

void init_pool(int listenfd, pool* p)
{
    // Initially, there are no connected descriptor
    int i;
    p->maxi = -1;
    for (i = 0; i < FD_SETSIZE;i++)
        p->clientfd[i] = -1;
    // Initially, listenfd is only member of select read set
    p->maxfd = listenfd;
    FD_ZERO(&p->ready_set);
    FD_SET(listenfd, &p->ready_set);
}

void add_client(int connfd, pool* p)
{
    int i;
    p->nready--;
    for (i = 0; i < FD_SETSIZE;i++) // find an available slot
    {
        if (p->clientfd[i] < 0)
        {
            // add connected descriptor to the pool
            p->clientfd[i] = connfd;
            Rio_readinitb(&p->clientrio[i], connfd);

            // add the descriptor to descriptor set
            FD_SET(connfd, &p->ready_set);

            // update max descriptor and pool high water mark
            if (connfd > p->maxfd)
                p->maxfd = connfd;
            if (i > p->maxi)
                p->maxi = i;
            break;
        }
    }
    if (i == FD_SETSIZE) // couldn't find an empty slot
        app_error("add client error: too many clients");
}

```
init_pool函数初始化客戸端池。clientfd数组表示已连接描述符的集合，其中整数-1表示一个可用的槽位。
初始时，已连接描述符集合是空的，而且监听描述符是select读集合中唯一的描述符。

addclient函数添加一个新的客戸端到活动客戸端池中。在clientfd数组中找到一个空槽位后，服务器将这个
已连接描述符添加到数组中，并初始化相应的RIO读缓冲区，这样一来我们就能够对这个描述符调用rio_readlineb
。然后，我们将这个已连接描述符添加到select读集合，并更新该池的一些全局属性。maxfd变量记录了select
最大文件描述符。maxi变量记录的是到clientfd数组的最大素引，这样checkclients函数就无需搜素整个数组了。

### 2. I/O多路复用技术的优劣
事件驱动设计的一个优点是，它比基于进程的设计给了程序员更多的对程序行为的控制。例如，我们可以设想编写
一个事件驱动的并发服务器，为某些客户端提供它们需要的服务，而这对于基于进程的并发服务器来说，是很困难的。

另一个优点是，一个基于I/O多路复用的事件驱动服务器是运行在单一进程上下文中的，因此每个逻辑流都能访问
该进程的全部地址空间。这使得在流之间共享数据变得很容易。一个与作为单个进程运行相关的优点是，你可以利
用熟悉的调试工具，例如GDB,来调试你的并发服务器，就像对顺序程序那样。最后，事件驱动设计常常比基于进程
的设计要高效得多，因为它们不需要进程上下文切换来调度新的流。

事件驱动设计一个明显的缺点就是编码复杂。我们的事件驱动的并发echo服务器需要的代码比基于进程的服务器多
三倍，并且很不幸，随着并发粒度的减小，复杂性还会上升。这里的粒度是指每个逻辑流每个时间片执行的指令数
量。例如，在示例并发服务器中，并发粒度就是读一个完整的文本行所需要的指令数量。只要某个逻辑流正忙于读
一个文本行，其他逻辑流就不可能有进展。对我们的例子来说这没有问题，但是它使得在“故意只发送部分文本行然
后就停止”的恶意客户端的攻击面前，我们的事件驱动服务器显得很脆弱。修改事件驱动服务器来处理部分文本行不
是一个简单的任务，但是基于进程的设计却能处理得很好，而且是自动处理的。基于事件的设计另一个重要的缺点
是它们不能充分利用多核处理器。

## 3. 基于线程的并发编程
`线程(thread)`就是运行在进程上下文中的逻辑流。在本书里迄今为止，程序都是由每个进程中一个线程
组成的。但是现代系统也允许我们编写一个进程里同时运行多个线程的程序。线程由内核自动调度。每个线
程都有它自己的`线程上下文(thread context)`，包括一个唯一的整数`线程ID(ThreadID，TID)`、
栈、栈指针、程序计数器、通用目的奇存器和条件码。所有的运行在一个进程里的线程共享该进程的整个虚
拟地址空间。

基于线程的逻辑流结合了基于进程和基于I/O多路复用的流的特性。同进程一样，线程由内核自动调度，并
且内核通过一个整数ID来识别线程。同基于I/O多路复用的流一样，多个线程运行在单一进程的上下文中，
因此共享这个进程虛拟地址空间的所有内容，包括它的代码、数据、堆、共享库和打开的文件。

### 1. 线程执行模型
每个进程开始生命周期时都是单一线程，这个线程称为`主线程(main thread)`。在某一时刻，主线程创建
一个`对等线程(peer thread)`，从这个时间点开始，两个线程就并发地运行。最后，因为主线程执行一个
慢速系统调用，例如read或者sleep，或者因为被系统的间隔计时器中断，控制就会通过上下文切换传递到
对等线程。对等线程会执行一段时间，然后控制传递回主线程，依次类推。

在一些重要的方面，线程执行是不同于进程的。因为一个线程的上下文要比一个进程的上下文小得多，线程的
上下文切换要比进程的上下文切换快得多。另一个不同就是线程不像进程那样，不是按照严格的父子层次来组
织的。和一个进程相关的线程组成一个对等(线程)`池`，独立于其他线程创建的线程。主线程和其他线程的区
别仅在于它总是进程中第一个运行的线程。对等(线程)池概念的主要影响是，一个线程可以杀死它的任何对等
线程，或者等待它的任意对等线程终止。另外，每个对等线程都能读写相同的共享数据。

### 2. Posix 线程
Posix线程(Pthreads)是在C程序中处理线程的一个标准接口。它最早出现在1995年，而且在所有的Linux
系统上都可用。Pthreads定义了大约60个函数，允许程序创建、杀死和回收线程，与对等线程安全地共享数
据，还可以通知对等线程系统状态的变化。

下展示了一个简单的Pthreads程序。主线程创建一个对等线程，然后等待它的终止。对等线程输出“Hello,
world!\n”并且终止。当主线程检测到对等线程终止后，它就通过调用exit终止该进程。这是我们看到的第
一个线程化的程序，所以让我们仔细地解析它。线程的代码和本地数据被封装在一个线程例程(thread rou
tine)中。正如第二行里的原型所示，每个线程例程都以一个通用指针作为输人，并返回一个通用指针。如果
想传递多个参数给线程例程，那么你应该将参数放到一个结构中，并传递一个指向该结构的指针。相似地，如
果想要线程例程返回多个参数，你可以返回一个指向一个结构的指针。
```c
#include "csapp.h"

void *thread(void *vargp)
{
    printf("Hello world!\n");
    return NULL;
}

int main()
{
    pthread_t tid;
    Pthread_create(&tid, NULL, thread, NULL);
    Pthread_join(tid, NULL);
    exit(0);
}
```

### 3. 创建线程
线程通过调用pthread_create两数来创建其他线程。
```c
#include spthread.h>
typedef void * (func) (void *);
int pthread_create(pthread_t *t id, pthread_attr_t *attr, func *f, void *arg);
// 若成功雞返回0 ，若出錯则为非零。
```
pthread_create函数创建一个新的线程，并带着一个输人变量arg，在新线程的上下文中运行`线程例程f`。
能用attr参数来改变新创建线程的默认属性。改变这些属性已超出我们学习的范围，在我们的示例中，总是
用一个为NULL的attr参数来调用pthread_create函数。

当pthread_create返回时，参数tid包含新创建线程的ID。新线程可以通过调用pthread_self函数来获
得它自己的线程ID。

### 4. 终止线程
一个线程是以下列方式之一来终止的:
- 当顶层的线程例程返回时，线程会`隐式地终止`。
- 通过调用pthread_exit函数，线程会`显式地终止`。如果主线程调用pthread_exit，它会等待所有其他
对等线程终止，然后再终止主线程和整个进程，返回值为thread_return.

```c
#include <pthread.h>
void pthread_exit (void *thread_return);
// 从不返回
```
- 某个对等线程调用Linux的exit函数，该函数终止进程以及所有与该进程相关的线程。
- 另一个对等线程通过以当前线程ID作为参数调用pthread_cancel函数数来终止当前线程。

```c
#include <pthread.h>
int pthread_cancel (pthread_t tid);
// 若成功则返回0，若出错则为非0
```

### 5. 回收已终止线程的资源
线程通过调用pthread_join函数等待其他线程终止。
```c
#include spthread.h>
int pthread_join (pthread_t tid, void **thread_return);
若成功則返回0，若出错則为非零。
// 若成功則返回0 ，若出錯則为非零
```
pthread_join函数会阻塞，直到线程tid终止，將线程例程返回的通用(void*)指针赋值为thread_return
指向的位置，然后回收已终止线程占用的所有内存资源。

注意，和Linux的wait函数不同，pthread_join函数只能等待一个指定的线程终止。没有办法让pthread_wait
等待任意一个线程终止。这使得代码更加复杂，因为它迫使我们去使用其他一些不那么直观的机制来检测进程的终止
。实际上，Stevens在[110]中就很有说服力地论证了这是规范中的一个错误。


### 6. 分离线程
在任何一个时间点上，线程是`可结合的(joinable)`或者是`分离的(detached)`。一个可结合的线程能够
被其他线程收回和杀死。在被其他线程回收之前，它的内存资源(例如栈)是不释放的。相反，一个分离的线程
是不能被其他线程回收或杀死的。它的内存资源在它终止时由系统自动释放。

默认情况下，线程被创建成可结合的。为了避免内存泄漏，每个可结合线程都应该要么被其他线程显式地收回，
要么通过调用pthread_detach函数被分离。
```c
#include spthread.h>
int pthread_detach (pthread_t tid);
// 若成功則返四0，若出错則为非零。
```
pthread_detach函数分离可结合线程tid。线程能够通过以pthread_self()为参数的pthread_detach调
用来分离它们自己。

尽管我们的一些例子会使用可结合线程，但是在现实程序中，有很好的理由要使用分离的线程。例如，一个高性
能Web服务器可能在每次收到Web浏览器的连接请求时都创建一个新的对等线程。因为每个连接都是由一个单独
的线程独立处理的，所以对于服务器而言，就很没有必要(实际上也不愿意)显式地等待每个对等线程终止。在这
种情况下，每个对等线程都应该在它开始处理请求之前分离它自身，这样就能在它终止后回收它的内存资源了。

### 7. 初始化线程
pthread_once函数允许你初始化与线程例程相关的状态
```c
#include <pthread.h>
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once (pthread_once_t *once_control, void (*init_routine) (void));
// 总是返回0.
```

once_control变量是一个全局或者静态变量，总是被初始化为PTHREAD_ONCE_INIT。当你第一次用参数
once_control调用pthread_once时，它调用init_routine，这是一个没有输入参数、也不返回什么
的函数。接下来的以once_control为参数的pthread_once调用不做任何事情。无论何时，当你需要动态
初始化多个线程共享的全局变量时，pthread_once函数是很有用的

### 8. 基于线程的并发服务器
下展示了基于线程的并发echo服务器的代码。整体结构类似于基于进程的设计。主线程不断地等待连接请求，
然后创建一个对等线程处理该请求。虽然代码看似简单，但是有几个普遍而且有些微妙的问题需要我们更仔细
地看一看。第一个问题是当我们调用pthread_create时，如何将已连接描述符传递给对等线程。最明显的
方法就是传递一个指向这个描述符的指针，就像下面这样
```c
connfd= Accept(listenfd, (SA * aclientaddr, &clientlen); 
Pthread_create (&tid, NULL, thread, &connfd);
```
然后，我们让对等线程间接引用这个指针，并将它赋值给一个局部变量，如下所示
```c
void *thread(void *vargp) {
    int connfd = *((int *)vargp);
    //...
}
```

```c
#include "csapp.h"

void echo(int connfd);

void *thread(void *vargp);

int main(int argc, char **argv)
{
    int listenfd, *connfdp;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s<port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(*argv[1]);
    while (1) {
        clientlen = sizeof(struct sockaddr_storage);
        connfdp = Malloc(sizeof(int));
        *connfdp = Accept(listenfd, (SA*) &clientaddr, &clientlen);
        Pthread_create(&tid, NULL, thread, connfdp);
    }
}

void *thread(void *vargp);
{
    int connfd = *((int*)vargp);
    Pthread_detach(pthread_self());
    Free(vargp);
    echo(connfd);
    Close(connfd);
    return NULL
}
```
然而，这样可能会出错，因为它在对等线程的赋值语句和主线程的accept语句间引人了`竞争(race)`。如果赋值
语句在下一个accept之前完成，那么对等线程中的局部变量connfd就得到正确的描述符值。然而，如果赋值语句
是在accept之后才完成的，那么对等线程中的局部变量connfd就得到下一次连接的描述符值。那么不幸的结果就
是，现在两个线程在同一个描述符上执行输人和输出。为了避免这种潜在的致命竞争，我们必须将accept返回的
每个已连接描述符分配到它自己的动态分配的内存块，如第20~21所示。

另一个问题是在线程例程中避免内存泄漏。既然不显式地收回线程，就必须分离每个线程，使得在它终止时它的内存
资源能够被收回(第31行)。更进一步，我们必须小心释放主线程分配的内存块(第32行)。

## 4. 多线程程序中的共享变量
从程序员的角度来看，线程很有吸引力的一个方面是多个线程很容易共享相同的程序变量。然而，这种共享也是很
棘手的。为了编写正确的多线程程序，我们必须对所谓的共享以及它是如何工作的有很清楚的了解。为了理解C程序
中的一个变量是否是共享的，有一些基本的问题要解答:1)线程的基础内存模型是什么?2)根据这个模型，变量实例
是如何映射到内存的?3)最后，有多少线程引用这些实例?
一个变量是共享的，当且仅当多个线程引用这个变量的某个实例。为了让我们对共享的讨论具体化，我们将使用下图
中的程序作为运行示例。尽管有些人为的痕迹，但是它仍然值得研究，因为它说明了关于共享的许多细微之处。示例
程序由一个创建了两个对等线程的主线程组成。主线程传递一个唯一的ID给每个对等线程，每个对等线程利用这个ID
输出一条个性化的信息，以及调用该线程例程的,总次数。

### 1. 线程内存模型
一组并发线程运行在一个进程的上下文中。每个线程都有它自己独立的`线程上下文`，包括线程ID、栈、栈指针、
程序计数器、条件码和通用目的奇存器值。每个线程和其他线程一起共享进程上下文的剩余部分。这包括整个用户
虛拟地址空间，它是由只读文本(代码)、读/写数据、堆以及所有的共享库代码和数据区域组成的。线程也共享相
同的打开文件的集合。

从实际操作的角度来说，让一个线程去读或写另一个线程的寄存器值是不可能的。另一方面，任何线程都可以访问
共享虛拟内存的任意位置。如果某个线程修改了一个内存位置，那么其他每个线程最终都能在它读这个位置时发现
这个变化。因此，奇存器是从不共享的，而虚拟内存总是共享的。各自独立的线程栈的内存模型不是那么整齐清楚
的。这些栈被保存在虛拟地址空问的栈区域中，并且通常是被相应的线程独立地访问的。我们说通常而不是总是，
是因为不同的线程栈是不对其他线程设防的。所以，如果一个线程以某种方式得到一个指向其他线程栈的指针，那
么它就可以读写这个栈的任何部分。

### 2. 将变量映射到内存
多线程的C程序中变量根据它们的存储类型被映射到虛拟内存:
- 全局变量。全局变量是定义在函数之外的变量。在运行时，虛拟内存的读/写区域只包含每个全局变量的一个实例，
任何线程都可以引用。
- 本地自动变量。本地自动变量就是定义在函数内部但是没有static属性的变量。在运行时，每个线程的栈都包含
它自己的所有本地自动变量的实例。即使多个线程执行同一个线程例程时也是如此。例如，有一个本地变量tid的实例
，它保存在主线程的栈中。我们用tid.m来表示这个实例。再来看一个例子，本地变量myid有两个实例，一个在对等
线程0的栈内，另一个在对等线程1的栈内。我们将这两个实例分别表示为myid.p0和myid.p1。
- 本地静态变量。本地静态变量是定义在函数内部并有static属性的变量。和全局变量一样，虚拟内存的读/写区域
只包含在程序中声明的每个本地静态变量的一个实例。例如，即使示例程序中的每个对等线程都在第25行声明了cnt，
在运行时，虛拟内存的读/写区域中也只有一个cnt的实例。每个对等线程都读和写这个实例。

### 3. 共享变量
我们说一个变量v是`共享的`，当且仅当它的一个实例被一个以上的线程引用。例如，示例程序中的变量cnt就是
共享的，因为它只有一个运行时实例，并且这个实例被两个对等线程引用。在另一方面，myid不是共享的，因为
它的两个实例中每一个都只被一个线程引用。然而，认识到像msgs这样的本地自动变量也能被共享是很重要的.


## 5. 用信号量同步线程
共享变量是十分方便，但是它们也引人了`同步错误(synchronization error)`的可能性。。

### 1. 进度图

### 2. 信号量
Edsger Dijkstra，并发编程领域的先锋人物，提出了一种经典的解决同步不同执行线程问题的方法，这种方法
是基于一种叫做`信号量(semaphore)`的特殊类型变量的。信号量s是具有非负整数值的全局变量，只能由两种
特殊的操作来处理，这两种操作称为P和V:
- P(S):如果s是非零的，那么P将s减1，并且立即返回。如果，为零，那么就挂起这个线程，直到、变为非零，
而一个V操作会重启这个线程。在重启之后，P操作将、减1，并将控制返回给调用者。
- V(S):V操作将s加1。如果有任何线程阻塞在P操作等待s变成非零，那么V操作会重启这些线程中的一个，然后
该线程将s减1，完成它的P操作。

P中的测试和减1操作是不可分割的，也就是说，一旦预测信号量s变为非零，就会将s减1，不能有中断。V中的加1
操作也是不可分割的，也就是加载、加1和存储信号量的过程中没有中断。注意，V的定义中没有定义等待线程被重
启动的顺序。唯一的要求是V必须只能重启一个正在等待的线程。因此，`当有多个线程在等待同一个信号量时，你
不能预测V操作要重启哪一个线程`。

P和V的定义确保了一个正在运行的程序绝不可能进入这样一种状态，也就是一个正确初始化了的信号量有一个负值。
这个属性称为`信号量不变性(semaphore invariant)`，为控制并发程序的轨迹线提供了强有力的工具.

Posix标准定义了许多操作信号量的函数
```c
#include <semaphore.h>

int sem_init(sem_t *sem, 0, unsigned int value);
int sem_wait(sem_t *s); // P(s)
int sem_post(sem_t *s); // V(s)
```
sem_init函数将信号量sem初始化力value。每个信号量在使用前必須初始化。針对我们的目的，中间的
参数总是零。程序分别通过调用sem_wait和sem_post函数来执行P和V操作。为了简明，我们更喜欢使用
下面这些等价的P和V的包装两数:
```c
#include "csapp.h"
void P(sem_t *s); 
void V(sem_t *s); 
```

### 3. 使用信号量来实现互斥
信号量提供了一种很方便的方法来确保对共享变量的互斥访问。基本思想是将每个共享变量(或者一组相关的共享
变量)与一个信号量、(初始为1)联系起来，然后用P(s)和V(s)操作将相应的临界区包围起来。

以这种方式来保护共享变量的信号量叫做`二元信号量(binary semaphore)`，因为它的值总是0或者1。以提供
互斥为目的的二元信号量常常也称为`互斥锁(mutex)`。在一个互斥锁上执行P操作称为对互斥锁`加锁`。类似地，
执行V操作称为对互斥锁`解锁`。对一个互斥锁加了锁但是还没有解锁的线程称为占用这个互斥锁。一个被用作一组
可用资源的计数器的信号量被称为`计数信号量`。

### 4. 利用信号量来调度共享资源
除了提供互斥之外，信号量的另一个重要作用是调度对共享资源的访问。在这种场景中，一个线程用信号量操作
来通知另一个线程，程序状态中的某个条件己经为真了。两个经典而有用的例子是生产者-消费者和读者一写者问题。

#### 1. 生产者-消费者问题
生产者和消费者线程共享一个有n个槽的有限缓冲区。生产者线程反复地生成新的`项目(item)`，并把它们插人到
缓冲区中。消费者线程不断地从缓冲区中取出这些项目，然后消费(使用)它们。也可能有多个生产者和消费者的变
种。

因为插人和取出项目都涉及更新共享变量，所以我们必须保证对缓冲区的访问是互斥的。但是只保证互斥访问是不够
的，我们还需要调度对缓冲区的访问。如果缓冲区是满的(没有空的槽位)，那么生产者必须等待直到有一个槽位变为
可用。与之相似，如果缓冲区是空的(没有可取用的项目)，那么消费者必须等待直到有一个项目变为可用。

生产者-消费者的相互作用在现实系统中是很普遍的。例如，在一个多媒体系统中，生产者编码视频帧，而消费者
解码并在屏幕上呈现出来。缓冲区的目的是为了减少视频流的抖动，而这种抖动是由各个帧的编码和解码时与数据
相关的差异引起的。缓冲区为生产者提供了一个槽位池，而为消费者提供一个已编码的帧池。另一个常见的示例是
图形用户接口设计。生产者检测到鼠标和键盘事件，并将它们插人到缓冲区中。消费者以某种基于优先级的方式从
缓冲区取出这些事件，并显示在屏幕上。

我们将开发一个简单的包，叫做SBUF，用来构造生产者-消费者程序。在下一节里，我们会看到如何用它来构造一
个基于预线程化(prethbreading)的有趣的并发服务器。SBUF操作类型为sbu_ft的有限缓冲区。项目存放在一
个动态分配的n项整数数组(buf)中。front和rear素引值记录该数组中的第一项和最后一项。三个信号量同步对
缓冲区的访问。mutex信号量提供互斥的缓冲区访问。slots和items信号量分别记录空槽位和可用项目的数量。
```c
typedef struct {
    int *buf;   // buffer array
    int n;      // maximum number of slots
    int front;  // buf[(front + 1) %n] is fisrt item
    int rear;   // buf[rear%n] is last item
    sem_t mutex;    // protects accesses to buf
    sem_t slots;    // counts available slots;
    sem_t items;    // counts available items
} sbuf_t;
```
下给出了SBUF函数的实现。sbuf_init函数为缓冲区分配堆内存，设置front和rear表示一个空的缓冲区，并
为三个信号量赋初始值。这个函数在调用其他三个函数中的任何一个之前调用一次。sbuf_deinit函数是当应
用程序使用完缓冲区时，释放缓冲区存储的。sbuf_insert函数等待一个可用的槽位，对互斥锁加锁，添加项目，
对互斥锁解锁，然后宣布有一个新项目可用。sbuf_remove函数是与sbuf_insert函数对称的。在等待一个可用
的缓冲区项目之后，对互斥锁加锁，从缓冲区的前面取出该项目，对互斥锁解锁，然后发信号通知一个新的槽位可供
使用。
```c
#include "csapp.h"

typedef struct {
    int *buf;   // buffer array
    int n;      // maximum number of slots
    int front;  // buf[(front + 1) %n] is fisrt item
    int rear;   // buf[rear%n] is last item
    sem_t mutex;    // protects accesses to buf
    sem_t slots;    // counts available slots;
    sem_t items;    // counts available items
} sbuf_t;

// create an empty,bounded,shared FIFO buffer with n slots
void sbuf_init(sbuf_t *sp, int n)
{
    sp->buf = (int*)Calloc(n, sizeof(int));
    sp->n = n;
    sp->front = sp->rear = 0;
    Sem_init(&sp->mutex, 0, 1);// binary semaphore for locking
    Sem_init(&sp->slots, 0, n);// initially, buf has n empty slots
    Sem_init(&sp->items, 0, 0);// initially, buf has zero data items
}

void sbuf_deinit(sbuf_t *sp)
{
    Free(sp->buf);
}

void sbuf_insert(sbuf_t *sp, int item)
{
    P(&sp->slots);  // wait for available slot
    P(&sp->mutex);  // lock the buffer
    sp->buf[(++sp->rear) % (sp->n)] = item; // insert the item
    V(&sp->mutex);  // unlock the buff
    V(&sp->items);  // announce available item
}

int sbuf_remove(sbuf_t *sp)
{
    int item;
    P(&sp->items);  // wait for available item
    P(&sp->mutex);  // lock the buff
    item = sp->buf[(++sp->front) % (sp->n)];    // remove the item
    V(&sp->mutex);  // unlock the buffer
    V(&sp->slots);  // announce available slot
    return item;
}

```
#### 2. 读者-写者问题
读者-写者问题是互斥问题的一个概括。一组并发的线程要访问一个共享对象，例如一个主存中的数据结构，或者
一个磁盘上的数据库。有些线程只读对象，而其他的线程只修改对象。修改对象的线程叫做写者。只读对象的线程
叫做读者。写者必领拥有对对象的独占的访问，而读者可以和无限多个其他的读者共享对象。一般来说，有无限多
个并发的读者和写者。

读者-写者交互在现实系统中很常见。例如，一个在线航空预定系统中，允许有无限多个客户同时查看座位分配，
但是正在预订座位的客户必须拥有对数据库的独占的访问。再来看另一个例子，在一个多线程缓存Web代理中，无
限多个线程可以从共享页面缓存中取出已有的页面，但是任何向缓存中写人一个新页面的线程必领拥有独占的访问。

读者-写者问题有几个变种，分别基于读者和写者的优先级。第一类读者-写者问题，读者优先，要求不要让读者等待，
除非已经把使用对象的权限赋予了一个写者。换句话说，读者不会因为有一个写者在等待而等待。第二类读者-写者问
题，写者优先，要求一旦一个写者准备好可以写，它就会尽可能快地完成它的写操作。同第一类问题不同，在一个写者
后到达的读者必须等待，即使这个写者也是在等待。

如下给出了一个对第一类读者-写者问题的解答。同许多同步问题的解答一样，这个解答很微妙，极具欺骗性
地简单。信号量w控制对访问共享对象的临界区的访问。信号量mutex保护对共享变量readcnt的访问，
readcnt统计当前在临界区中的读者数量。每当一个写者进人临界区时，它对互斥锁w加锁，每当它离开临界
区时，对w解锁。这就保证了任意时刻临界区中最多只有一个写者。另一方面，只有第一个进人临界区的读者对
w加锁，而只有最后一个离开临界区的读者对w解锁。当一个读者进入和离开临界区时，如果还有其他读者在临
界区中，那么这个读者会忽略互斥锁w。这就意味着只要还有一个读者占用互斥锁w，无限多数量的读者可以
没有障碍地进人临界区。
```c
// global variables
int readcnt;//initially = 0
sem_t mutex, w;// both initially = 1

void read(void)
{
    while (1)
    {
        P(&mutex);
        readcnt++;
        if (readcnt == 1) // first in
            P(&w);
        V(&mutex);

        // critical section
        // reading happens

        P(&mutex);
        readcnt--;
        if (readcnt == 0) // last out
            V(&w);
        V(&mutex);
    }
}

void writer(void)
{
    while (1)
    {
        P(&w);
        // critical section
        // writing happens
        V(&w);
    }
}

```
对这两种读者-写者问颞的正确解答可能导致`饥饿(starvation)`，饥饿就是一个线程无限期地阻塞，无法进展。
例如，上示的解答中，如果有读者不断读者地到达，写者就可能无限期地等待。

#### 5. 综合：基于预线程化的并发服务器
预线程化的并发服务器:

一个预线程化的并发echo服务器。这个服务器使用的是有一个生产者和多个消费者的生产者-消费者模型
```c
#define NTHREADS 4
#define SBUFSIZE 16

void echo_cnt(int connfd);
void *thread(void *vargp);

sbuf_t sbuf;    // shared buffer of connected descriptors

int server(int argc, char **argv)
{
    int i, listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid;

    if (argc != 2) {
        fprintf(stderr, "usage: %s<port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(*argv[1]);

    sbuf_init(&sbuf, SBUFSIZE);
    for (i = 0; i < NTHREADS;i++) // create worker threads
        Pthread_create(&tid, NULL, thread, NULL);
    while (1)
    {
        clientlen = sizeof(struct sockaddr_storage);
        connfd = Accept(listenfd, (SA*) &clientaddr, &clientlen);
        sbuf_insert(&sbuf, connfd); // insert connfd in buff
    }
}

void *thread(void *vargp)
{
    Pthread_detach(pthread_self());
    while (1)
    {
        int connfd = sbuf_remove(&sbuf); // remove connfd from buffer
        echo_cnt(connfd);   // service client
        Close(connfd);
    }
}
```
echo_cnt:echo的一个版本，它对从客户端接收的所有字节计数
```c
static int byte_cnt; // byte counter
static sem_t mutex_c; // and the mutex_c that protect it

static void init_echo_cnt(void)
{
    Sem_init(&mutex_c, 0, 1);
    byte_cnt = 0;
}

void echo_cnt(int connfd)
{
    int n;
    char buf[MAXLINE];
    rio_t rio;
    static pthread_once_t once = PTHREAD_ONCE_INIT;

    Pthread_once(&once, init_echo_cnt);
    Rio_readinitb(&rio, connfd);
    while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
    {
        P(&mutex_c);
        byte_cnt += n;
        printf("server received %d (%d total) bytes on fd\n", n, byte_cnt, connfd);
        V(&mutex_c);
        Rio_writen(connfd, buf, n);
    }
}
```

## 6. 使用线程提高并行性
到目前为止，在对并发的研究中，我们都假设并发线程是在单处理器系统上执行的。然而，大多数现代机器具有
多核处理器。并发程序通常在这样的机器上运行得更快，因为操作系统内核在多个核上并行地调度这些并发线程
，而不是在单个核上顺序地调度。在像繁忙的Web服务器、数据库服务器和大型科学计算代码这样的应用中利用
这样的并行性是至关重要的，而且在像Web浏览器、电子表格处理程序和文档处理程序这样的主流应用中，并行
性也变得越来越有用。

所有程序的集合能够被划分成不相交的顺序程序集合和并发程序的集合。写顺序程序只有一条逻辑流。写并发
程序有多条并发流。并行程序是一个运行在多个处理器上的并发程序。因此，并行程序的集合是并发程序集合
的真子集。

将任务分配到不同线程的最直接方法是将序列划分成t个不相交的区域，然后给t个不同的线程每个分配一个区域
。为了简单，假设n是t的倍数，这样每个区域有n/t个元素。让我们来看看多个线程并行处理分配给它们的区域
的不同方法。

最简单也最直接的选择是将线程的和放入一个共享全局变量中，用互斥锁保护这个变量。图下给出了我们会如何
实现这种方法。在1部分，主线程创建对等线程，然后等待它们结束。注意，主线程传递给每个对等线程一个小
整数，作为唯一的线程ID。每个对等线程会用它的线程ID来决定它应该计算序列的哪一部分。这个向对等线程传
递一个小的唯一的线程ID的思想是一项通用技术，许多并行应用中都用到了它。在对等线程终止后，全局变量
gsum包含着最终的和。然后主线程用闭合形式解答来验证结果(第2部分)。


下面给出了每个对等线程执行的函数数。在第1部分，线程从线程参数中提取出线程ID，然后用这个ID来决定它
要计算的序列区域(第2部分)。在第3部分中，线程在它的那部分序列上选代操作，每次迭代都更新共享全局变量
gsum。注意，我们很小心地用P和V互斥操作来保护每次更新。
```c
void *sum_mutex(void *vargp)
{
    long myid = *((long*)vargp);    // extract the thread ID 第一部分
    // 第二部分
    long start = myid * nelems_per_thread;   // start element index
    long end = start + nelems_per_thread;// end element index
    long i;
    // 第三部分
    for (i = start;i < end;i++)
    {
        P(&mutex);
        gsum += i;
        V(&mutex);
    }
    return NULL;
}
```

同步开销巨大，要尽可能避免。如果无可避免，必须要用尽可能多的有用计算弥补这个开销.

一种避免同步的方法是让每个对等线程在一个私有变量中计算它自己的部分和，这个私有变量不与其他任何线程
共享，如下所示。主线程定义一个全局数组psum，每个对等线程i把它的部分和累积在psum[i]中。因为小心地
给了每个对等线程一个不同的内存位置来更新，所以不需要用互斥锁来保护这些更新。唯一需要同步的地方是主
线程必须等待所有的子线程完成。在对等线程结束后，主线程把psum向量的元素加起来，得到最终的结果。
```c
void *sum_mutex(void *vargp)
{
    long myid = *((long*)vargp);    // extract the thread ID 第一部分
    // 第二部分
    long start = myid * nelems_per_thread;   // start element index
    long end = start + nelems_per_thread;// end element index
    long i;
    // 第三部分
    for (i = start;i < end;i++)
    {
        psum[myid] += i;
    }
    return NULL;
}
```

使用局部变量来消除不必要的内存引用

```c
void *sum_mutex(void *vargp)
{
    long myid = *((long*)vargp);    // extract the thread ID 第一部分
    // 第二部分
    long start = myid * nelems_per_thread;   // start element index
    long end = start + nelems_per_thread;// end element index
    long i,sum;
    // 第三部分
    for (i = start;i < end;i++)
    {
        sum += i;
    }
    psum[myid] = sum;
    return NULL;
}
```

## 7. 其他并发问题
### 1. 线程安全
当用线程编写程序时，必须小心地编写那些具有称为`线程安全性(thread safety)`属性的函数。一个函数被称为
`线程安全的(thread-safe)`，当且仅当被多个并发线程反复地调用时，它会一直产生正确的结果。如果一个函数
不是线程安全的，我们就说它是`线程不安全的(thread-unsafe)`。
我们能够定义出四个(不相交的)线程不安全函数类:
- 第1类:不保护共享变量的函数。比如函数对一个未受保护的全局计数器变量加1。将这类线程不安全函数变成
线程安全的，相对而言比较容易:利用像P和V操作这样的同步操作来保护共享的变量。这个方法的优点是在调用
程序中不需要做任何修改。缺点是同步操作将减慢程序的执行时间。
- 第2类:保持跨越多个调用的状态的函数。一个伪随机数生成器是这类线程不安全函数的简单例子。请参考下面
中的伪随机数生成器程序包。rand函数是线程不安全的，因为当前调用的结果依赖于前次调用的中间结果。当调
用srand为rand设置了一个种子后，我们从一个单线程中反复地调用rand，能够预期得到一个可重复的随机数字
序列。然而，如果多线程调用rand函数，这种假设就不再成立了。
```c
unsigned next_seed = 1;
// rand - return pseudorandom integer in the range 0..32767
unsigned rand(void)
{
    next_seed = next_seed * 1103515245 + 12543;
    return (unsigned)(next_seed>>16) % 32768;
}
// srand - set the initial seed for rand
void srand(unsigned new_seed)
{
    next_seed = new_seed;
}
```
使得像rand这样的函数线程安全的唯一方式是重写它，使得它不再使用任何static数据，而是依靠调用者在参数
中传递状态信息。这样做的缺点是，程序员现在还要被迫修改调用程序中的代码。在一个大的程序中，可能有成百
上千个不同的调用位置，做这样的修改将是非常麻烦的，而且容易出错。
- 第3类:返回指向静态变量的指针的函数。某些函数，例如ctime和gethostbyname，将计算结果放在一个
static变量中，然后返回一个指向这个变量的指针。如果我们从并发线程中调用这些函数，那么将可能发生灾
难，因为正在被一个线程使用的结果会被另一个线程悄悄地覆盖了。有两种方法来处理这类线程不安全函数。
一种选择是重写函数，使得调用者传递存放结果的变量的地址。这就消除了所有共享数据，但是它要求程序员
能够修改函数的源代码。如果线程不安全函数是难以修改或不可能修改的(例如，代码非常复杂或是没有源代码
可用)，那么另外一种选择就是使用加锁-复制(lock-and-copy)技术。基本思想是将线程不安全函数与互斥锁
联系起来。在每一个调用位置，对互斥锁加锁，调用线程不安全函数，将函数返回的结果复制到一个私有的内存
位置，然后对互斥锁解锁。为了尽可能地减少对调用者的修改，你应该定义一个线程安全的包裝函数，它执行
加锁-复制，然后通过调用这个包装函数来取代所有对线程不安全两数的调用。例如，下面给出了ctime的一个
线程安全的版本，利用的就是加锁-复制技术。
```c
char *ctime_ts(const time_t *timep, char *privatep)
{
    char *sharedp;
    P(&mutex);
    sharddp = ctime(timep);
    strcpy(privatep, sharedp);// copy string from shared to private
    V(&mutex);
    return privatep;
}
```
- 第4类:调用线程不安全函数的函数。如果函数数f调用线程不安全函数g，那么f就是线程不安全的吗?
不一定。如果:是第2类函数，即依赖于跨越多次调用的状态，那么f也是线程不安全的，而且除了重写:
以外，没有什么办法。然而，如果:是第1类或者第3类函数，那么只要你用一个互斥锁保护调用位置和
任何得到的共享数据，f仍然可能是线程安全的。在上面中我们看到了一个这种情况很好的示例，其中
我们使用加锁-复制编写了一个线程安全函数，它调用了一个线程不安全的函数。

### 2. 可重入性
有一类重要的线程安全函数，叫做`可重入函数(reentrant function)`，其特点在于它们具有这
一种属性:当它们被多个线程调用时，不会引用任何共享数据。尽管`线程安全`和`可重入`有时会(不
正确地)被用做同义词，但是它们之间还是有清晰的技术差别，值得留意。所有函数的集合被划分成不
相交的线程安全和线程不安全函数集合。可重入函数集合是线程安全函数的一个真子集。

可重入函数通常要比不可重入的线程安全的函数高效一些，因为它们不需要同步操作。更进一步来说，
將第2类线程不安全函数转化为线程安全函数的唯一方法就是重写它，使之变为可重入的。

检查某个函数的代码并先验地断定它是可重入的，这可能吗?不幸的是，不一定能这样。如果所有的函数参数
都是传值传递的(即没有指针)，并且所有的数据引用都是本地的自动栈变量(即没有引用静态或全局变量)，
那么函数就是`显式可重入的(explicitly reentrant)`，也就是说，无论它是被如何调用的，都可以断
言它是可重入的。

然而，如果把假设放宽松一点，允许显式可重入西数中一些参数是引用传递的(即允许它们传递指针)，那么
我们就得到了一个`隐式可重入的(implicitly reentrant)函数`，也就是说，如果调用线程小心地传递
指向非共享数据的指针，那么它是可重人的。

### 3. 在线程化的程序中使用已存在的库函数
大多数Linux函数，包括定义在标准C库中的函数(例如malloc、free、realloc、printf和scanf)都是
线程安全的，只有一小部分是例外。图12-41列出了常见的例外。(参考[110]可以得到一个完整的列表。
strtok函数是一个已弃用的(不推荐使用)函数。asctime、ctime和localtime函数是在不同时间和数据格
式间相互来回转换时经常使用的函数。gethostbyname、gethostbyaddr和inet_ntoa函数是已弃用的网
络编程函数，已经分别被可重入的getaddrinfo、getnameinfo和inet_ntop函数取代。除了rand和
strtok以外，所有这些线程不安全函数都是第3类的，它们返回一个指向静态变量的指针。如果我们需要在一
个线程化的程序中调用这些函数中的某一个，对调用者来说最不惹麻烦的方法是加锁-复制。然而，加锁-复制
方法有许多缺点。首先，额外的同步降低了程序的速度。第二，像gethostbyname这样的函数返回指向复杂
结构的结构的指针，要复制整个结构层次，需要`深层复制(deep copy)`结构。第三，加锁-复制方法对像
rand这样依赖跨越调用的静态状态的第2类函数并不有效。

因此，Linux系统提供大多数线程不安全函数的可重入版本。可重入版本的名字总是以“_r〞后缀结尾。例如，
asctime的可重入版本就叫做asctime_r。我们建议尽可能地使用这些函数

### 4. 竞争
当一个程序的正确性依赖于一个线程要在另一个线程到达y点之前到达它的控制流中的x点时，就会发生`竞爭(race)`
。通常发生竞争是因为程序员假定线程将按照某种特殊的轨迹线穿过执行状态空间，而忘记了另一条准则规定:
多线程的程序必须对任何可行的轨迹线都正确工作。例子是理解竞争本质的最简单的方法。让我们来看看下面
的简单程序。主线程创建了四个对等线程，并传递一个指向一个唯一的整数ID的指针到每个线程。每个对等线
程复制它的参数中传递的ID到一个局部变量中(第22行)，然后输出一个包含这个ID的信息。它看上去足够简
单，但是当我们在系统上运行这个程序时，我们得到以下不正确的结果:
```c
// warnning: this code is buggy
#include "csapp.h"
#define N 4

void *thread(void *vargp);
int main()
{
    pthread_t tid[N];
    int i;
    
    for (i = 0; i < N;i++)
        Pthread_create(&tid[i], NULL, thread, &i);
    for (i = 0; i < N;i++)
        Pthread_join(tid[i], NULL);
    exit(0);
}

void *thread(void *vargp)
{
    int myid = *((int*)vargp);
    printf("hello from thread %d\n", myid);
    return NULL;
}

```
问题是由每个对等线程和主线程之间的竞争引起的。你能发现这个竞争吗?下面是发生的情况。当主线程在
第13行创建了一个对等线程，它传递了一个指向本地栈变量之的指针。在此时，竞争出现在下一次在第12行
对之加1和第22行参数的间接引用和赋值之间。如果对等线程在主线程执行第12行对i加1之前就执行了第22行，
那么myid变量就得到正确的ID。否则，它包含的就会是其他线程的ID。令人惊慌的是，我们是否得到正确的
答案依赖于内核是如何调度线程的执行的。在我们的系统中它失败了，但是在其他系统中，它可能就能正确工
作，让程序员“幸福地”察觉不到程序的严重错误。

为了消除竞争，我们可以动态地为每个整数ID分配一个独立的块，并且传递给线程例程一个指向这个块的指针，
如图12-43所示(第12~14行)。请注意线程例程必领释放这些块以避免内存泄漏。
```c
#include "csapp.h"
#define N 4

void *thread(void *vargp);
int main()
{
    pthread_t tid[N];
    int i;
    
    for (i = 0; i < N;i++)
        ptr = Malloc(sizeof(int));
        *ptr = i;
        Pthread_create(&tid[i], NULL, thread, ptr);
    for (i = 0; i < N;i++)
        Pthread_join(tid[i], NULL);
    exit(0);
}

void *thread(void *vargp)
{
    int myid = *((int*)vargp);
    printf("hello from thread %d\n", myid);
    return NULL;
}

```

### 5. 死锁
信号量引人了一种潜在的令人厌恶的运行时错误，叫做`死锁(deadlock)`，它指的是一组线程被阻塞了，
等待一个永远也不会为真的条件。进度图对于理解死锁是一个无价的工具。例如，图12-44展示了一对用
两个信号量来实现互斥的线程的进程因。从这幅图中，我们能够得到一些关于死锁的重要知识:
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/deadlock_0.jpg)

- 程序员使用P和V操作顺序不当，以至于两个信号量的禁止区域重叠。如果某个执行轨迹线碰巧到达了死锁
状态d，那么就不可能有进一步的进展了，因为重叠的禁止区域阻塞了每个合法方向上的进展。换句话说，程
序死锁是因为每个线程都在等待其他线程执行一个根不可能发生的V操作。
- 重叠的禁止区域引起了一组称为`死锁区城(deadlock region)`的状态。如果一个轨迹线碰巧到达了一个
死锁区域中的状态，那么死锁就是不可避免的了。轨迹线可以进人死锁区域，但是它们不可能离开。
- 死锁是一个相当困难的问题，因为它不总是可预测的。一些幸运的执行轨迹线将绕开死锁区域，而其他的将
会陷入这个区域。图12-44展示了每种情况的一个示例。对于程序员来说，这其中隐含的着实令人惊慌。你可
以运行一个程序1000次不出任何问题，但是下一次它就死锁了。或者程序在一台机器上可能运行得很好，但是
在另外的机器上就会死锁。最糟糕的是，错误常常是不可重复的，因为不同的执行有不同的轨迹线。

**互斥锁加锁顺序规则:给定所有互斥操作的一个全序，如果每个线程都是以一种顺序获得互斥锁并
以相反的顺序释放，那么这个程序就是无死锁的**

## 8. 小结
一个并发程序是由在时间上重叠的一组逻辑流组成的。在这一章中，我们学习了三种不同的构建并发程序的
机制:进程、I/0多路复用和线程。我们以一个并发网络服务器作为贯穿全章的应用程序。

进程是由内核自动调度的，而且因为它们有各自独立的虛拟地址空间，所以要实现共享数据，必须要有显式
的IPC机制。事件驱动程序创建它们自己的并发逻辑流，这些逻辑流被模型化为状态机，用I/O多路复用来显
式地调度这些流。因为程序运行在一个单一进程中，所以在流之间共享数据速度很快而且很容易。线程是这
些方法的混合。同基于进程的流一样，线程也是由内核自动调度的。同基于I/O多路复用的流一样，线程是
运行在一个单一进程的上下文中的，因此可以快速而方便地共享数据。

无论哪种并发机制，同步对共享数据的并发访问都是一个困难的问题。提出对信号量的P和V操作就是为了帮助
解决这个问题。信号量操作可以用来提供对共享数据的互斥访问，也对诸如生产者-消费者程序中有限级冲区和
读者-写者系统中的共享对象这样的资源访问进行调度。一个并发预线程化的echo服务器提供了信号量使用场景
的很好的例子。

并发也引人了其他一些困难的问题。被线程调用的函数必须具有一种称为线程安全的属性。我们定义了四类线程
不安全的函数，以及一些將它们变为线程安全的建议。可重入函数是线程安全函数的一个真子集，它不访问任何
共享数据。可重入函数通常比不可重入函数更为有效，因为它们不需要任何同步原语。竞争和死锁是并发程序中
出现的另一些困难的向题。当程序员错误地假设逻辑流该如何调度时，就会发生竞争。当一个流等待一个永远不
会发生的事件时，就会产生死锁。