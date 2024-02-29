---
title: "守护进程和inetd超级服务器"
date: 2024-02-29T16:23:25+08:00
draft: true
---

## 1. 概述
守护进程(daemon)是在后台运行且不与任何控制终端关联的进程。Unix系统通常有很多守护进程在后台运行(约在20到50个的量级)，执行不同的管理任务。

守护进程没有控制终端通常源于它们由系统初始化脚本启动。然而守护进程也可能从某个终端由用户在shell提示符下键入命令行启动，这样的守护进程必须
亲自脱离与控制终端的关联，从而避免与作业控制、终端会话管理、终端产生信号等发生任何不期望的交互，也可以避免在后合运行的守护进程非预期地输出到终端。

守护进程有多种启动方法。
1. 在系统启动阶段，许多守护进程由系统初始化脚本启动。这些脚本通常位于/etc目录或以/etc/rc开头的
某个目录中，它们的具体位置和内容却是实现相关的。由这些脚本启动的守护进程一开始时拥有超级用户特权。
有若干个网络服务器通常从这些脚本启动:ineta超級服务器(见下一条)、Web服务器、邮件服务器(经常是sendmail)。
2. 许多网络服务器由将在本章靠后介绍的ineta超级服务器启动。ineta自身由上一条中的某个脚本启动。
ineta监听网络请求(Telnet、FTP等)，每当有一个请求到达时，启动相应的实际服务器(Telnet服务器、FTP服务器等)。
3. cron守护进程按照规则定期执行一些程序，而由它启动执行的程序同样作为守护进程运行。cron自身由第1条启动方法中的某个脚本启动。
4. at命令用于指定将来某个时刻的程序执行。这些程序的执行时刻到来时，通常由cron守护进程启动执行它们，
因此这些程序同样作为守护进程运行。
5. 守护进程还可以从用户终端或在前台或在后台启动。这么做往往是为了测试守护程序或重启因某种原因而终止了的某个
守护进程。

因为守护进程没有控制终端，所以当有事发生时它们得有输出消息的某种方法可用，而这些消息既可能是普通的通告性消息，
也可能是需由系统管理员处理的紧急事件消息。sys1og函数是输出这些消息的标准方法，它把这些消息发送给syslogd守护进程.

## 2. syslogd守护进程
Unix系统中的syslogd守护进程通常由某个系统初始化脚本启动，而且在系统工作期间一直运行。源自Berkeley的syslogd实现在启动时执行以下步骤。

1. 读取配置文件。通常为/etc/syslog.conf的配置文件指定本守护进程可能收取的各种
日志消息(log message)应该如何处理。这些消息可能被添加到一个文件(/dev/console
文件是一个特例，它把消息写到控制台上)，或被写到指定用户的登录窗口(若该用户已登录到本守
护进程所在系统中)，或被转发给另一个主机上的syslogd进程。
2. 创建一个Unix域数据报套接字，给它捆绑路径名/var/run/log(在某些系统上是/dev/log).
3. 创建一个UDP套接字，给它捆绑端口514(syslog服务使用的端口号)。
4. 打开路径名/dev/klog。来自内核中的任何出错消息看着像是这个设备的输入。

此后syslogd守护进程在一个无限循环中运行:调用select以等待它的3个描述符(分别来自上述第2、第3和第4步)之一变为可读，读入日志消息，
并按照配置文件进行处理。如果守护进程收到SIGHUP信号，那就重新读取配置文件。

通过创建一个Unix域数据报套接字，我们就可以从自己的守护进程中通过往syslogd绑定的路径名发送我们的消息达到发送日志消息的目的，
然而更简单的接口是使用将在下一节讲解的syslog函数。另外，我们也可以创建一个UDP套接字，通过往环回地址和端口514发送我们的消息
达到发送日志消息的目的。

## 3. syslog 函数
既然守护进程没有控制终端，它们就不能把消息fprintf到stderr上。从守护进程中登记消息的常用技巧就是调用syslog函数。

```c
#include <syslog.h>
void syslog (int priority, const char *message, ...) ;
```

本函数的priority参数是级別(level)和设施（fracility)两者的组合，分別如图13-1和图13-2所示。
RFC3164还有关于该参数的额外细节。message参数类似printf的格式串，不过增设了%m规范，它将被
替换成与当前errno值对应的出错消息。message参数的末尼可以出现一个换行符，不过并非必需。

如图13-1所示，日志消息的level可从0到7，它们是按从高到低的顺序排列的。如果发送者未指定level值
，那就默认为IOG_NOTICE。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_1.jpg)

日志消息还包含一个用于标识消息发送进程类型的facility。图13-2列出了facility的各种值。如果发送者未指定
facility值，那就默认为LOC_USER。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_2.jpg)

举例来说，当rename函数调用意外失败时，守护进程可以执行以下调用:
syslog(LOG_INFO | LOG_LOCAL2,"rename(%s, %s): %m"，file1,file2);
facility和level的目的在于，允许在/etc/syslog.conf文件中统一配置来自同一给定设施的所有消息，
或者统一配置具有相同级别的所有消息。举例来说，该配置文件可能含有以下两行:

kern.* /dev/console
local7.debug /var/log/cisco.1og

这两行指定所有内核消息登记到控制台，来自local7设施的所有debug消息添加到文件/var/1og/cisco.log的末尾。

当syslog被应用进程首次调用时，它创建一个Unix域数据报套接字，然后调用connect连接到由syslogd守护进程
创建的Unix域数据报套接字的众所周知路径名(臂如/var/run/1og)。这个套接字一直保持打开，直到进程终止为止。
作为替换，进程也可以调用openlog和closelog。

```c
#include <syslog.h>
void openlog (const char *ident, int options, int facility) ; 
void closelog (void);
```
openlog可以在首次调用syslog前调用，closelog可以在应用进程不再需要发送日志消息时调用。

ident参数足一个由syslog冠于每个日志消息之前的字符串。它的值通常是程序名。options參数由
图13-3所示的一个或多个常值的逻辑或构成。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_3.jpg)

openlog被调用时，通常井不立即创建Unix域套接字。相反，该套接字直到首次调用syslog时才打开。
TOG_NDELAY选项迫使该套接字在openlog被调用时就创建。

openlog的facility参数为没有指定设施的后线syslog调用指定一个默认值。有些守护进程通过调用
openlog指定一个设施(对于一个给定守护进程，设施通常不变)，然后在每次调用syslog时只指定级別(因为級别可随错误性质改变)。

日志消息也可以由logger命令产生。举例来说，logger命令可用在shell脚本中以向syslogd发送消息。

## 4. daemon_init 函数
daemon_init的函数，通过调用它(通常从服务器程序中)，我们能够把一个普通进程转变为守护进程。
该函数在所有Unix变体上都应该适合使用，不过有些Unix变体提供一个名为daemon的C库函数，实现
类似的功能。BSD和Linux均提供这个daemon函数。

[daemon_init](https://github.com/lsill/unpvnote/blob/main/lib/daemon_init?plain=1#L8)

## 5. inetd 守护进程
典型的Unix系统可能存在许多服务器，它们只是等待客户请求的到达，例如FTP、Telnet、Rlogin、TFTP等等。
4.3BSD面世之前的系统中，所有这些服务都有一个进程与之关联。这些进程都是在系统自举阶段从/etc/rc文件中
启动，而且每个进程执行几乎相同的启动任务:创建一个套接字，把本服务器的众所周知端口捆绑到该套接字，等待
一个连接(若是TCP)或一个数据报(若是UDP)，然后派生子进程。子进程为容户提供服务，父进程则继续等待下一个
客户请求。这个模型存在两个问题。
1. 所有这些守护进程含有几乎相同的启动代码，既表现在创建套接字上，也表现在演变成守护进程上(类似daemon_init函数)。
2. 每个守护进程在进程表中占据一个表项，然而它们大部分时间处于睡眠状态。

4.3BSD版本通过提供一个因特网超级服务器(即inetd守护进程)使上述问题得到简化。基于TCP或UDP的服务器都
可以使用这个守护进程。它是这样解决上述两个问题的。
1. 通过由inetd处理普通守护进程的大部分启动细节以简化守护程序的编写。这么一来每个服务器不再有调用
daemon_init两数的必要。
2. 单个进程(inetd)就能为多个服务等待外来的客户请求，以此取代每个服务一个进程的做法。这么做减少了
系统中的进程总数。

inetd进程使用我们随daemon_init函数讲解的技巧把自己演变成一个守护进程。它接着读入并处理自己的配置文件。
通常是/etc/inetd.conf的配置文件指定本超级服务器处理哪些服务以及当一个服务请求到达时该怎么做。该文件中
每行包含的字段如图13-6所示

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_6.jpg)

当inetd调用exec执行某个服务器程序时，该服务器的真实名字总是作为程序的第一个参数传递。

图13-7展示了ineta守护进程的工作流程。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_7.jpg)

1. 在启动阶段，读入/etc/inetd.conf文件并给该文件中指定的每个服务创建一个适当类型(字节流或数据报)的套接字。
inetd能够处理的服务器的最大数目取决于inetd能够创建的描述符的最大数目。新创建的每个套接字都被加入到將由某个
select调用使用的一个描述符集中。
2. 为每个套接字调用bind，指定捆绑相应服务器的众所周知端口和通配地址。这个TCP或UDP端口号通过调用getservbyname
获得，作为函数参数的是相应服务器在配置文件中的service-name字段和protocol字段。
3. 对于每个TCP套接字，调用listen以接受外来的连接请求。对于数据报套接字则不执行本步骤。
4. 创建完毕所有套接字之后，调用select等待其中任何一个套接字变为可读。TCP监听套接字将在有一个新连接准备好可被接受
时变为可读，UDP套接字将在有一个数据报到达时变为可读。inetd的大部分时间花在阻塞于select调用内部，等待某个套接字变为可读。
5. 当select返回指出某个套接字己可读之后，如果该套接字是一个TCP套接字，而且其服务器的wait-flag值为nowait，那就调用
accept接受这个新连接。
6. inetd守护进程调用fork派生进程，并由子进程处理服务请求。这一点类似标准的并发服务器。子进程关闭除要处理的套接字描述符
之外的所有描述符:对于TCP服务器来说，这个套接字是由accept返回的新的己连接套接字，对于UDP服务器来说，这个套接字是父进程
最初创建的UDP套接字。子进程调用dup2三次，把这个待处理套接字的描述符复制到描述符0、1和2(标准输入、标准输出和标准错误输出)，
然后关闭原套接字描述符。子进程打开的描述符于是只有0、1和2。子进程自标准输入读实际是从所处理的套接字读，往标准输出或
标准错误输出写实际上是往所处理的套接字写。子进程根据它在配置文件中的login-name字段值，调用getpwnam获取对应的保密字文件表项。
如果login-name字段值不是root，子进程就通过调用setgid和setuid把自身改为指定的用户。(既然inetd进程以值为0的用户ID运行，
其子进程将跨fork调用继承这个用户ID，因而能够变成所选定的任何用户。)子进程然后调用exec执行由相应的server-program字段指定
的程序来具体处理请求，相应的server-program-arguments字段值则作为命令行参数传递给该程序。
7. 如果第5步中select返回的是一个字节流套接字，那么父进程必须关闭已连接套接字(就像标准并发服务器那样)。父进程再次调用
select，等待下一个变为可读的套接字。

让我们更仔细地查看ineta中发生的描述符处理。图13-8展示了当有一个来自某个FTP客户的新连接请求到达时inetd中的描述符。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_8.jpg)
这个连接请求指向TCP端口21，不过accept为它创建了一个新的已连接套接字。

图13-9展示了在调用过fork，并关闭了除这个已连接套接字描述符之外的所有描述符之后，子进程中的描述符。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_9.jpg)

下一步是子进程把这个已连接套接字描述符复制到描述符0、1和2，然后关闭原描述符。 图13-10展示了此时的描述符 。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/13_10.jpg)

子进程接着调用exec。通常情况下所有描述符跨exec保持打开，因此exec加载的实际服务器程序使用描述符0、1或
2之一与客户通信。服务器中应该只打开这些描述符。

上述情形处理的是配置文件中指定了nowait标志的服务器。对于TCP服务这是典型的设置，意味着inetd不必等待某
个子进程终止就可以接受对于该子进程所提供之服务的另一个连接。如果对于某个子进程所提供之服务的另一个连接
确实在该子进程终止之前到达，那么一旦父进程再次调用select，这个连接就立即返回到父进程。前面列出的第4、
第5和第6个步骤再次被执行，于是派生出另一个子进程来处理这个新请求。

给一个数据报服务指定wait标志导致父进程执行的步骤发生变化。这个标志要求inetd必须在这个套接字再次成为
select调用的候选套接字之前等待当前服务该套接字的子进程终止。发生的变化有以下几点。
1. fork返回到父进程时，父进程保存子进程的进程ID。这么做使得父进程能够通过查看由waitpid返回的值确定
这个子进程的终止时间。
2. 父进程通过使用FD_CIR宏关闭这个套接字在select所用描述符集中对应的位，达成在将来的select调用中
禁止这个套接字的目的。这一点意味着子进程将接管该套接字，直到自身终止为止。
3. 当子进程终止时，父进程被通知以一个SIGCHLD信号，而父进程的信号处理函数将取得这个子进程的进程ID。
父进程通过打开相应的套接字在select所用描述符集中对应的位，使得该套接字重新成为select的候选套接字。

数据报服务器必须接管其套接字直至自身终止，以防inetd在此期间让select检查该套接字的可读性(也就是等待
来自任何客户的另一个数据报)，这是因为每个数据报服务器只有一个套接字，而不像每个TCP服务器那样既有一个
监听套接字，对于每个客户又各有一个己连接套接字字。如果inetd不关闭对于某个数据报套接字的可读条件检查，
而且父进程(inetd)先于服务该套接字的子进程执行，那么引发本次fork的那个数据报仍然在套接字接收缓冲区中，
导致select再次返回可读条件，致使inetd再次fork另一个(不必要的)子进程。inetd必须在得知子进程己从
套接字接收队列中读走该数据报之前忽略这个数据报套接字。inetd得知子进程何时使用完其套接字的手段是通过
接收表明子进程己终止的SIGCHLD信号。。

既然替一个TCP服务器调用accept的进程是inetd，由inetd启动的真正服务器通常通过调用getpeername获取
客户的IP地址和端口号。fork和exec发生之后(就如inetd)，真正的服务器获悉客户身份的唯一方法是调用getpeername。
 
inetd通常不适用于服务密集型服务器，其中值得注意的有邮件服务器和Web服务器。举例来说，sendmail通常作为
一个标准的并发服务器来运行。这种模式下每个客户连接的进程控制开销仅仅是一个fork，而由inetd启动的每个
TCP服务器的开销是一个fork加一个exec。而web服务器则使用多种技术把每个客户连接的进程控制开销降低到最小.
