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

### 4. 使用SO_RCVTIMEO套接字选项为recvfrom设置超时
展示SO_RCVTIMEO套接字选项如何设置超时。本选项一旦设置到某个描述符(包括指定超时值)，其超时设置将应用于该
描述符上的所有读操作。本方法的优势就体现在一次性设置选项上，而前两个方法总是要求我们在欲设置时间限制的每个
操作发生之前做些工作。本套接字选项仅仅应用于读操作，类似的SO_SNDTIMEO选项则仅仅应用于写操作，两者都不能
用于为connect设置超时。

- [dg_cli](https://github.com/lsill/unpvnote/blob/main/advio/dgclitimeo2.c?plain=1#L3)

## 3. recv和send函数
这两个函数类似标准的read和write函数，不过需要 一个额外的参数。
```
#include <sys/socket.h>
ssize_t recv( int sockfd, void *buff, size_t nbytes, int flags) ;
ssize_t send(int sockfd, const void *buff. size_t nbytes, int flags); 
//返回:若成功则为读入或写出的字节数，若出错则为-1
```
recv和sena的前了个参数等同于read和write的3个参数。flags参数的值或为0，或为图14-6列出的一个或多个常值的逻辑或。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_6.jpg)

- MSG_DONTROUTE :本标志告知内核目的主机在某个直接连接的本地网络上，因而无需执行路由表查找。我们己随SO_DONTROUTE
套接字选项提供了本特性的额外信息。这个既可以使用MSG_DONTROUTE标志针对单个输出操作开启，也可以使用SO_DONTROUTE套
接字选项针对某个给定套接字上的所有输出操作开启。
- MSG_DONIWAIT :本标志在无需打开相应套接字的非阻塞标志的前提下，把单个I/O操作临时指定为非阻塞，接着执行I/O操作，
然后关闭非阻寒标志。
- MSG_COB :对于send，本标志指明即将发送带外数据。TCP连接上只有一个字节可以作为带外数据发送。对于recv,本标志指明即
将读入的是带外数据而不是普通数据.
- MSG_PEEK :本标志适用于recv和recvfrom，它允许我们查看己可读取的数据，而且系统不在recv或recvfrom返回后丢弃这些
数据。
- MSG_WAITALL :本标志随4.3 BSD Reno引入。它告知内核不要在尚未读入请求数目的字节之前让一个读操作返回。如果系统支持
本标志，我们就可以省掉readn函数，而替之以如下的宏:#define readn(fd , ptr， n)  recv(fd ,ptr, n, MSG_WAITALL)
即使指定了MSG_WAITALL，如果发生下列情况之一:(a)捕获一个信号，(b)连接被终止，(C)套接字发生一个错误，相应的读函数仍有
可能返回比所请求字节数要少的数据。

另有一些标志适用于TCP/P以外的协议族。举例来说，OSI的传输层是基于记录的(不像TCP那样是一个字节流)，其输出操作支持
MSG_EOR标志，指示逻辑记录的结束。

flags参数在设计上存在一个基本问题:它是按值传递的，而不是一个值-结果参数。因此它只能用于从进程向内核传递标志。内核无法
向进程传回标志。对于TCP/IP协议这一点不成问题，因为TCP/IP几乎不需要从内核向进程传回标志。然而随着OSI协议被加到4.3BSDReno
中，却提出了随输入操作向进程返送MSG_EOR标志的需求。4.3BSDReno做出的决定是保持常用输入函数(recv和recvfrom)的参数不变，
而改变recvmsg和sendmsg所用的msghdr结构。我们将在14.5节中看到该结构新增了一个整数msg_flags成员，而且既然该结构按引用
传递，内核就可以在返回时修改这些标志。这个决定同时意味着如果一个进程需要由内核更新标志，它就必须调用recvmsg，而不是调用
recv或recvfrom。

## 4. readv和writev函数
这两个函数类似read和write，不过readv和writev允许单个系统调用读入到或写出自一个或多个缓冲区。这些操作分别称为分散读
(scatter read)和集中写(gather write)，因为来自读操作的输入数据被分散到多个应用缓冲区中，而来自多个应用缓冲区的
输出数据则被集中提供给单个写操作。
```
#include<sys/uio.n>
ssize_t readv(int filedes,const struct iovec* iov,int iovcnt);
ssize_t write(int filedes,const struct iovec* iov,int iovcnt);//返回:若成功则为读入或写出的字节数，若出错则为-1
```
这两个函数的第二个参数都是指向某个iovec结构数组的一个指针，其中iovec结构在头文件<sys/uio.h>中定义:
```
struct iovec{
    void *iovbase; /*starting address of buffer*/
    size_t iov_len; /*sizeof buffer*/
};
```
iovec结构数组中元素的数目存在某个限制，具体取决于实现。举例来说，4.3BSD和Linux均最多允许1024个，而HP-UX最多允许2100个。
POSIX要求在头文件<sys/uio.h>中定义IOV_MAX常值，而且其值全少为16。

readv和writev这两个函数可用于任何描述符，而不仅限于套接字。另外writev是一个原子操作，意味着对于一个基于记录的协议
(例如UDP)而言，一次writev调用只产生单个UDP数据报。

## 5. recvmsg 和 sendmsg函数
这两个函数是最通用的I/O函数。实际上我们可以把所有read、readv、recv和recvfrom调用替换成recvmsg调用。
类似地，各种输出函数调用也可以替换成sendmsg调用。
```
#include <sys/socket.h>
ssize_t recvmsg(int sockfd,struct msghdr* msg,int flags);
ssize_t sendmsg(int sockfd,struct msghdr* msg,int flags);
// 返回:若成功则为读入或写出的字节数，若出错則为-1
```
这两个函数把大部分参数封装到一个msghdr结构中:
```
structmsghar{
    void*   msg_name;   /*protocol address*/
    socklen_t   msg_namelen; /*sizeof protocol address*/
    struct iovec* msg_iov; /* scatter / gather array */
    int msg_iovlen; /* # elements in msg_iov*/
    void    *msg_control;   /*ancillary data (cmsghdr struct)*/
    socklen_t   msg_controllen; /*length of ancillary data*/
    int msg_flags; /*flags returned by recvmsg()*/
};
```
这里给出的msghdr结构符合POSIX规范。有些系统仍然使用本结构源自4.2BSD的较旧版本。这个较旧的结构没有
msg_flags成员，而且msg_control和msg_controllen成员分别被称为msg_accrights和msg_accrightslen。
这个较旧结构唯一支持的辅助数据形式用子传递文件描述符(称为访问权限)。

msg_name和msg_namelen这两个成员用于套接字未连接的场合(譬如未连接UDP套接字)。它们类似recvfrom和
sendto的第五个和第六个参数:msg_name指向一个套接字地址结构，调用者在其中存放接收者(对于sendmsg调用)
或发送者(对于recvmsg调用)的协议地址。如果无需指明协议地址(例如对于TCP套接字或己连接UDP套接字)，
msg_name应置为空指针。msg_namelen对于sendmsg是一个值参数，对于recvmsg却是一个值-结果参数。

msg_iov和msg_iovlen这两个成员指定输入或输出缓冲区数组(即iovec结构数组)，类似readv或writev的
第二个和第三个参数。msg_control和msg_controllen这两个成员指定可选的辅助数据的位置和大小。
msg_controllen对于recvmsg是一个值-结果参数。我们将在14.6节讲解辅助数据。

对于recvmsg和sendmsg，我们必须区别它们的两个标志变量，一个是传递值的flags参数，另一个是所传递
msghar结构的msg_flags成员，它传递的是引用，因为传递给函数的是该结构的地址。

- 只有recvmsg使用msg_flags成员。recvmsg被调用时，flags参数被复制到msg_flags成员，并由内核使用其
值驱动接收处理过程。内核还依据recvmsg的结果更新msg_flags成员的值。
- sendmsg则忽略msg_flags成员，因为它直接使用flags参数驱动发送处理过程。这一点意味着如果想在某个
sendmsg调用中设置MSG_DONTWAII标志，那就把flags参数设置为该值，把msg_flags成员设置为该值不起作用。

图14-7汇总了内核为输入和输出函数检查的flags参数值以及recvmsg可能返回的msg_flag成员值。其中没有
sendmsg msg_flags一栏，因为我们己提及本组合无效。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_7.jpg)
这些标志中，内核只检查而不返回前4个标志，既检查又返回接下来的2个标志，不检查而只返回后4个标志。recvmsg返回的7个标志解释如下。
- MSG_BCAST :本标志随BSD/OS引入，相对较新。它的返回条件是本数据报作为链路层广播收取或者其目的IP地址是一个广播地址。
与IP_RECVDSTADDR套接字选项相比，本标志是用于判定一个UDP数据报是否发往某个广播地址的更好方法。
- MSG_MCAST :本标志随BSD/OS引入，相对较新。它的返回条件是本数据报作为链路层多播收取。
- MSG_TRUNC :本标志的返回条件是本数据报被截断，也就是说，内核预各返回的数据超过进程事先分配的空间(所有iov_len成员之和)。
- MSG_CTRUNC :本标志的返回系件是本数据报的辅助数据被截所，也就是说，内核预备返回的辅助数据超过进程事先分配的空间(msg_controllen)。
- MSG_EOR :本标志的返回条件是返回数据结束一个逻辑记录。TCP不使用本标志，因为它是一个字节流协议。
- MSG_OOB :本标志绝不为TCP带外数据返回。它用于其他协议族(例如OSI协议族)。
- MSG_NOTIFICATION :本标志由SCTP接收者返回，指示读入的消息是一个事件通知，而不是数据消息。

具体实现可能会在msg_flags成员中返回一些输入flags参数值，因此我们应该只检查那些感兴趣的标志值(例如图14-7中的后6个标志)

图14-8展示了一个msghar结构以及它指向的各种信息。图中假设进程即将对一个UDP套接字调用recvmsg。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_8.jpg)
图中给协议地址分配了16个字节，给辅助数据分配了20个字节。为缓冲数据初始化了一个由3个iovec结构构成的数组:
第一个指定一个100字节的缓冲区，第二个指定一个60字节的绶冲区，第三个指定一个80字节的缓冲区。我们还假设己
为这个套接字设置了IP_RECVDSTADDR套接字选项，以接收所读取UDP数据报的目的IP地址。

接着假设从198.6.38.100端口2000到达一个170字节的UDP数据报，它的目的地是我们的UDP套接字，目的IP地址为
206.168.112.96。图14-9展示了recvmsg返回时msghdr结构中的所有信息。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_9.jpg)
图中被recvmsg修改过的字段标上了阴影。从图14-8到图14-9的变动包括以下几点。
- 由msg_name成员指向的缓冲区被填以一个网际网套接字地址结构，其中有所收到数据报的源IP地址和源UDP端口号。
- msg_namelen成员(一个值-结果参数)被更新为存放在msg_name所指缓冲区中的数据量。本成员并无变化，因为
recvmsg调用前和返回后其值均为16。
- 所收取数据报的前100字节数据存放在第一个缓冲区，中60字节数据存放在第二个缓冲区，后10字节数据存放在第
三个缓冲区。最后那个缓冲区的后70字节没有改动。recvmsg函数的返回值(即170)就是该数据报的大小。
- 由msg_control成员指向的缓冲区被填以一个cmsghar结构。该cmsghdr结构中，cmsg_len成员值为16，
cmsg_level成员值为IPPROTO_IP，cmsg_type成员值为IP_RECVDSTADDR，随后4个字节存放所收到UDP
数据报的IP地址。这个20字节缓冲区的后4个字节没有改动。
- msg_controllen成员被更新为所存放辅助数据的实际数据量。本成员也是一个值-结果參数，recvmsg返回时其结果为16。
- msg_flags成员同样被recvmsg更新，不过没有标志返回给进程。

图14-10汇总了我们己讲述的5组I/O函数之间的差异。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_10.jpg)


## 6. 辅助数据
辅助数据(ancillary data)可通过调用sendmsg和recvmsg这两个函数，使用msghar结构中的msg_control和
msg_control_len这两个成员发送和接收。辅助数据的另一-个称调是控制信息(control information)。

图14-11汇总了辅助数据的各种用途。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_11.jpg)
辅助数据由一个或多个辅助数据对象(ancillary data object)构成，每个对象以一个定义在头文件<sys/socket.hs中
的cmsghdr结构开头。
```
struct cmsghdr{
    socklen_t   cmsglen;    /* length in bytes,including this structure*/
    int cmsg_level; /*originating   protocol*/
    int cmsg_type;  /*  protocol-specific   type */
/*  followed    by  unsigned    char    cmsg_data[]*/
};
```

我们己在图14-9中见识过这个结构，当时它由IP_RECVDSTADDR套接字选项用来返回所收取UDP数据报的目的們地址。
由msg_control指向的辅助数据必须为csmghar结构适当地对齐。我们将在图15-11中展示一个对齐方法。

图14-12展示了在一个控制缓冲区中出现2个辅助数据对象的例子。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_12.jpg)
msg_control指向第一个辅助数据对象，辅助数据的总长度则由msg_controllen指定。每个对象开头都是一个描述该对象的
cmsghdr结构。在cmsg_type成员和实际数据之间可以有填充字节，从数据结尾处到下一个辅助数据对象之前也可以有填充字节。

图14-13展示了通过一个Unix域套接字传递描述符或传递凭证时所用cmsghdr结构的格式。

图中我们假设cmsghar结构的每个成员(总共3个)都占用4个字节，而且在cmsghar结构和实际数据之间没有填充字节。当传递描述
符时，cmsg_data数组的内容是真正的描述符值。图中只展示了一个待传递的描达符，然而一般总能传递多个描达符(这种情况下
cmsg_len的值为12加上4乘以描述符的数目，这里假设每个描述符占据4个字节)。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_13.jpg)
既然由recvmsg返回的辅助数据可含有任意数目的辅助数据对象，为了对应用程序屏蔽可能出现的填充宇节，头文件
<sys/socket.h>中定义了以下5个宏，以简化对辅助数据的处理。
```
#include <sys/socket.h>
#include <sys/param.h>
/* for ALIGN macro on many implementations */ 
struct cmsghdr *CMSG_FIRSTHDR (struct msghdr *mhdrptr) ; // 返回:指向第一个cmsghdx结构的指针，若无辅助數据则为NULI 
struct cmsghdr “CMSG_NXTHDR(struct msghdr *mhdpir, struct cmsghdr *cmseper); // 返回:指向下一个cmsghdr 结构的指针，若不再有辅助數据对象则为NULL
unsigned char *MSG_DATA (struct cmsghdr *cmsgptr) ; // 返回:指向与cmsghdr结构关联的数据的第一个字节的指针 
unsighed int CMSG_LEN (unsigned int length);    // 返回:给定数据量下存放到cmeg_1en中的值
unsigned int CMSG_SPACE (unsigned int length);  //  返回:给定数据量下一个辅助数据对象总的大小
```

## 7. 排队的数据量
有时候我们想要在不真正读取数据的前提下知道一个套接字上己有多少数据排队等着读取。
有3个技术可用于获悉己排队的数据量。
1. 如果获悉已排队数据量的目的在于避免读操作阻塞在内核中(因为没有数据可读时我们还有其他事情可做)，
那么可以使用非阻塞式I/O。
2. 如果我们既想查看数据，又想数据仍然留在接收队列中以供本进程其他部分稍后读取，那么可以使用
MSG_PEEK标志(图14-6)。如果我们想这样做，然而不能肯定是否真有数据可读，那么可以结合非阻塞套接字
使用该标志，也可以组合使用MSG_DONTWAIT标志和NSG_PEEK标志。需注意的是，就一个字节流套接字而言，
其接收队列中的数据量可能在两次相继的recv调用之间发生变化。举例来说，假设指定MSG_PEEK标志以一个
长度为1024字节的缓冲区对一个TCP套接字调用recv，而且其返回值为100。如果再次调用同一个recv，返
回值就有可能超过100(假设指定的缓冲区长度大于100)，因为在这两次调用之间TCP可能又收到了一些数据。
就一个UDP套接字而言，假设其接收队列中己有一个数据报，如果我们指定MSG_PEEK标志调用recvfrom一次，
稍后不指定该标志再调用recvfrom一次，那么即使另有数据报在这两次调用之间加入该套接字的接收队列，
这两个调用的返回值(数据报大小、内容及发送者地址)也完全相同。(当然这里假设没有其他进程共享该套接字
并从中读取数据。)
3. 一些实现支持ioctl的FIONREAD命令。该命令的第三个ioctl参数是指向某个整数的一个指针，内核通过
该整数返回的值就是套接字接收队列的当前字节数。该值是己排队字节的总和，对于UDP套接字而言包括所有己
排队的数据报。还要注意的是，在源自Berkcley的实现中，为UDP套接字返回的值还包括一个套接字地址结构
的空间，其中含有发送者的IP地址和端又号(对于IPv4为16个字节，对于IPV6为24个字节)。

## 8. 套接字和标准I/O
到目前为止的所有例子中，我们一直使用也称为UnixI/O 包括read、write这两个函数及它们的变体(recv、
send等等)——的函数执行I/O。这些函数围绕描述符(descriptor)工作，通常作为Unix内核中的系统调用实现。

执行I/O的另一个方法是使用标准I/O函数库(standardI/O library)。这个函数库由ANSIC标准规范，意在
便于移植到支持ANSIC的非Unix系统上。标准I/O函数库处理我们直接使用UnixI/O函数时必须考志的一些细节，
臂如自动缓冲输入流和输出流。不幸的是，它对于流的缓冲处理可能导致我们同样必须考虑的一组新的问题。

标准I/O 函数库可用于套接字，不过需要考虑以下几点。
- 通过调用fdopen，可以从任何一个描述符创建出一个标准I/O流。类似地，通过调用fileno，可以获取一个给定标准
I/O流对应的描述符。select只能用于描述符，因此我们不得不获取那个标准I/O流的描述符。
- TCP和UDP套接字是全双工的。标准I/O流也可以是全双工的:只要以r+类型打开流即可，r+意味着读写。然而在这样的
流上，我们必领在调用一个输出函数之后插入一个fflush、fseek、fsetpos或rewind调用才能接着调用一个输入函数。
类似地，调用一个输入函数后也必须插入一个fseek、fsetpos或rewind调用才能调用一个输出函数，除非输入函数遇到
一个EOF。fseek、fsetpos和rewind这3个函数的问题是它们都调用lseek, 而lseek用在套接字上只会失败。
- 解决上述读写问题的最简单方法是为一个给定套接字打开两个标准I/O流:一个用于读，一个用于写。

例子：
[str_echo](https://github.com/lsill/unpvnote/blob/main/advio/str_echo_stdio02.c?plain=1#L3)
命令行输入：
```
hpux % topc110 2206.168.112.96
hello,world         键入本行，但无回射输出
and hi              再键入本行，仍无回射输出
hello??             再键入本行，仍无回射输出
^D                  键入EOF字符
hello,              至此才输出那三个回射行
world
andhihello?
```
服务器直到我们键入EOF字符才回射所有文本行的原因在于这里存在一个缓冲问题。以下是实际发生的步骤。
- 我们键入第一行输入文本，它被发送到服务器。
- 服务器用fgets读入本行，再用fputs回射本行。
- 服务器的标准I/O流被标准I/O函数库完全缓冲。这意味者该函数库把回射行复制到输出流的标准I/O缓冲区，
但是不把该缓冲区中的内容写到描述符，因为该级冲区未满。
- 我们键入第二行输入文本，它被发送到服务器。
- 服务器用fgets读入本行，再用fputs回射本行。
- 服务器的标准I/O函数库再次把回射行复制到输出流的标准I/O缓冲区，但是不把该绶冲区中的内容写到描述符，因为该缓冲区仍未满。
- 同样的情形发生在我们键入的第三行文本上。
- 我们键入EOF字符，致使我们的str_cli函数调用shutdown，从而发送一个FIN到服务器。
- 服务器TCP收取这个FIN，它被fgets读入，致使fgets返回一-个空指针。
- str_echo函数返回到服务器的main函数，子进程通过调用exit终止。
- C库函数exit调用标准I/O清理函数(APUE第162~164页)。之前由我们的fputs调用填入输出级冲区中的未满内容现被输出。
- 服务器子进程终止，致使它的己连接套接字被关闭，从而发送-个FIN到客户，完成TCP的四分组终止序列。
- 我们的str_cli函数收取并输出由服务器回射的三行文本。
- str-cli接着在其套按字上收到一个EOF，客户于是终止。

这里的问题出在服务器中由标准I/O函数库自动执行的缓冲之上。标准I/O函数库执行以下三类缓冲。
1. 完全缓冲(fully buffering)意味着只在出现下列情况时才发生I/O:缓冲区满，进程显式调用
fflush，或进程调用exit终止自身。标准I/O缓冲区的通常大小为8192字节。
2. 行缓冲(line buffering)意味着只在出现下列情况时才发生I/O:碰到一个换行符，进程调用fflush，
或进程调用exit终止自身。
3. 不缓冲(unbuffering)意味着每次调用标准I/O输出函数都发生I/O。标准IO函数库的大多数Unix实现使用如下规则。
- 标准错误输出总是不缓冲。
- 标准输入和标准输出完全缓冲，除非它们指代终端设备(这种情况下它们行级冲)。
- 所有其他I/O流都是完全缓冲，除非它们指代终端设备（这种情况下它们行缓冲)。

既然套接字不是终端设备，图14-14中的str_echo函数的上述问题就在于输出流(fpout)是完全缓冲的。本问题有两个解决
办法。第一个办法是通过调用setvbuf迫使这个输出流变为行缓冲。第二个办法是在每次调用fputs之后通过调用fflush强制
输出每个回射行。然而在现实使用中，这两种办法都易于犯错，与Nagle算法(如7.9节所述)的交互可能也成问题。大多数情况
下，最好的解决办法是彻底避免在套接字上使用标准I/O函数库，并且如3.9节所述在缓冲区而不是文本行上执行操作。当标准
I/O流的便利性大过对缓冲带来的bug的担优时，在套接字上使用标准I/0流也可能可行，但这种情况很罕见。

## 9. 高级轮询技术

### 1. /dev/poll接口
Solaris上名为/dev/poll的特殊文件提供了一个可扩展的轮询大量描述符的方法。select和poll存在的一个问题是，每次调用
它们都得传递待查询的文件描述符。轮询设备能在调用维持状态，因此轮询进程可以预先设置好待查询描述符的列表，然后进入一个
循环等待事件发生，每次循环回来时不必再次设置该列表。
   
打开/dev/poll之后，轮询进程必须先初始化一个pollfd结构(即poll函数使用的结构，不过本机制不使用其中的revents成员)
数组，再调用write往/dev/poll设备上写这个结构数组以把它传递给内核，然后执行ioctl的DP_POLL命令阻塞自身以等待事件
发生。传递给ioctl调用的结构如下:
```
struct dvpoll{
   struct pollfd* dp_fds;
   int dp_nfds;
   int dp_timeout;
}
```
其中dp_fds成员指向一个缓冲区，供ioctl在返回时存放一个pollfd结构数组。dp_nfds成员指定该缓冲区的大小。ioctl调用将
一直阻塞到任何一个被轮询描述符上发生所关心的事件，或者流逝时间超过经由dp_timeout成员指定的毫秒数为止。dp_timeout
指定为0将导致ioctl立即返回，从而提供了使用本接口的非阻塞手段。dp_timeout指定为-1表示没有超时设置。
[/etc/poll](https://github.com/lsill/unpvnote/blob/main/advio/str_cli_poll03.c?plain=1#L4)

### 1. kqueue 接口
FreeBSD随4.1版本引入了kqueue接口。本接又允许进程向内核注册描述所关注kqueue事件的事件过滤器(event filter)。事件
除了与select所关注类似的文件I/O和超时外，还有异步I/O、文件修改通知(例如文件被删除或修改时发出的通知)、进程跟踪(例如
进程调用exit或fork时发出的通知)和信号处理。kqueue接口包括如下2个函数和1个宏。
```c
#include <sys/types.h>
#include <sys/event.h>
#include <sys/time.h>
int kgueue(void);
int kevent(int kq,const struct kevent *changelist, int nchanges, struct kevent *eventlist,int nevents, const struct timespec *timeout);
void EV_SET(struct kevent *kev,uintptr_t ident,short filter, u_short flags,u_int flags,intptr_t data,void* udata);
```
kqueue函数返回一个新的kqueue描述符，用于后续的kevent调用中。kevent函数既用于注册所关注的事件，也用于确定是否有所关注
事件发生。changelist和nchanges这两个参数给出对所关注事件做出的更改，若无更改则分别取值NULL和0。如果nchanges不为0，
kevent函数就执行changelist数组中所请求的每个事件过滤器更改。其条件已经触发的任何事件(包括刚在changelist中增设的那些事件)
由kevent函数通过eventlist参数返回，它指向一个由nevents个元素构成的kevent结构数组。kevent函数在eventist中返回的事件数
目作为的数返回值返回，0表示发生超时。超时通过timeout参数设置，其处理类似select:NULL阻塞进程，非0值timespec指定明确的超时
值，0值timespec执行非阻塞事件检查。注意，kevent使用的timespec结构不同于select使用的timeval结构，前者的分辨率为纳秒，后
者的分辦率为微秒。

kevent结构在头文件<sys/event.h>中定义:
```c
struct kevent{
    uintptr_t ident; /*identifier (e.g.,file descriptor) */
    short filter; /*filter type(e.g.,EVFILT_READ) */
    ushort flags;   /*action flags(e.g.,EV_ADD)*/
    u_int fflags;   /*filter-specific flags*/
    intptr_t data;  /*filter-specific sdata*/
    void *udata;    /*opaque user data*/
}
```
其中flags成员在调用时指定过滤器更改行为，在返回时额外给出系件，如图14-16所示
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_16.jpg)
filter成员指定的过滤器类型如图14-17所示
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_17.jpg)
[kqueue](https://github.com/lsill/unpvnote/blob/main/advio/str_cli_kqueue04.c?plain=1#L4)

## 10 T/TCP:事务目的TCP
T/TCP是对TCP进行过略微修改的一个版本，能够避免近来彼此通信过的主机之间的三路握手。

T/TCP能够把SYN、FIN和数据组合到单个分节中，前提是数据的大小小于MSS。图14-19展示最小TCP事务的时间线。第一个分节
是由客户的单个sendto调用产生的SYN、FIN和数据。该分节组合了connect、write和shutdown共三个调用的功能。服务器
执行通常的套接字函数调用步骤:socket、bind、listen和accept，其中后者在客户的分节到达时返回。服务器用send发回其
应答并关闭套接字。这使得服务器在同一个分节中向客户发出SYN、FIN和应答。比较TCP，我们看到不仅需在网络中传输的分节有
所减少(T/TCP需3个，TCP需10个，UDP需2个〉，而且客户从初始化连接到发送一个请求再到读取相应应答所花费的时间也滅少了
一个RTT。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/14_19.jpg)
T/TCP的优势在于TCP的所有可靠性(序列号、超时、重传，等等)得以保留，而不像UDP那样把可靠性推给应用程序去实现。
T/TCP同样维持TCP的慢启动和拥塞避免措施，UDP应用程序却往往缺乏这些特性。

为了处理T/TCP，套接字API需作些变动。在提供T/TCP的系统上TCP应用程序无需任何改动，除非要使用T/TCP的特性。所有现有
TCP应用程序继续使用我们己经讲述过的套接字API工作。
- 客户调用sendto，以便把数据的发送结合到连接的建立之中。该调用替换单独的connect调用和write调用。服务器的协议地址
改为传递给sendto而不是connect。
- 新增一个输出标志MSG_EOF(参见图14-6)，用于指示本套接字上不再有数据待发送。该标志允许我们把shutdown调用结合到
输出操作(send或sendto)之中。给一个sendto调用同时指定本标志和服务器的协议地址有可能导致发送单个含有SYN、FIN和
数据的分节。我们还在图14-19中指出，服务器发送应答使用的是send而不是write，其原因在于为了指定MSG_BOP标志，以便
随应答一起发送FIN。〔不要把这个新标志与己有的MSG_EOR标志混为一谈，后者为面向记录的协议指示记录结束条件)。
- 新定义一个级别为IPPROTO_TCP的套接字选项TCP_NOPUSH。本选项防止TCP只为腾空套接字发送缓冲区而发送分节。当某个
客户准备以单个sendto发送一个请求时，如果该请求大小超过MSS，它就应该为相应套接字设置本选项，以减少所发送分节的数目。
- 想跟一个服务器建立连接并且使用T/TCP发送一个请求的客户应该调用socket、setsockopt(开启TCP_NOPUSH选项)和sendto
(若只有一个请求待发送则指定MSG_EOF标志)。如果setsockopt返回ENVOPROTOOPT错误或者sendto返回ENOICONN错误，那么
本主机不支持T/TCP。这种情况下容户可以干脆调用connect和write，加上可能后跟的shutdown(如果只有一个请求待发送)。
- 服务器所需的唯一变动是，如果服务器想随应答一起发送FIN，它就应该指定MSG_EOF标志调用send以发送应答，而不是调用write以发送应答。
- T/TCP的编译时测试可以使用伪代码#ifdef MSG_FOF。

## 11. 小结
在套接字操作上设置时间限制的方法有三个:
- 使用alarm函数和SIGALRX信号;
- 使用由select提供的时间限制:
- 使用较新的SO_RCVTIMEO和SO_SNDTIMEO套接字选项。

第一个方法易于使用，不过涉及信号处理，而信号处理可能导致竞争条件。使用select意味着我们阻塞在指定过时间限制的这个函数上，而
不是阻塞在read、write或connect调用上。第三个方法也易于使用，不过并非所有实现都提供。

recvmsg和sendmsg是所提供的5组I/O函数中最为通用的。它们组合了如下能力:指定MSG_xxx标志(出自recv和send)，返回或指定对端的
协议地址(出自recvfrom和sendto)，使用多个缓冲区(出自readv和writev)。此外还增加了两个新的特性:给应用进程返回标志，接收或发送辅助数据。

我们在文中讲述了10种不同格式的辅助数据，其中6种是随IPv6新定义的。辅助数据由一个或多个辅助数据对象构成，每个对象都以一个cmsghdr
结构打头，它指定数据的长度、协议级别及类型。5个以CMSC_打头的函数可用于构建和分析辅助数据。

C标准I/O函数库也可以用在套接字上，不过这么做将在己经由TCP提供的缓冲级别之上新增一级缓冲。实际上，对由标准I/O函数库执行的级冲缺乏了解是
使用这个函数库最常见的问题。既然套接字不是终端设各，这个潜在问题的常用解决办法就是把标准I/O流设置成不缓冲，或者干脆不要在套接字上使用标准I/O。

许多厂家提供轮询大量事件却没有select和poll所需开销的高级方法。尽管应该避免编写不可移植的代码，有时候性能改善的收益会重于不可移植造成的风险。
T/TCP是对TCP的一个简单增强版本，能够在客户和服务器近来彼此通信过的前提下避免三路握手，使得服务器对于客户的请求更快地给出应答。从编程角度看，
客户通过调用sendto而不是通常的connect、write和shutdown调用序列发择T/TCP的优势。































