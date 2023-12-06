---
title: "网络编程"
date: 2023-12-04T17:26:23+08:00
draft: true
---

# 网络编程
## 1. 客户端-服务器编程模型
每个网络应用都是基于`客户端一服务器模型`的。采用这个模型，一个应用是由一个`服务器进程`
和一个或者多个`客户端进程`组成。服务器管理某种资源，并且通过操作这种资源来为它的客户端
提供某种服务。例如，一个Web服务器管理着一组磁盘文件，它会代表客户端进行检索和执行。一
个FTP服务器管理着一组磁盘文件，它会为客户端进行存储和检素。相似地，一个电子邮件服务器
管理着一些文件，它为容户端进行读和更新。

客户端-服务器模型中的基本操作是`事务(transaction)`。一个客户端-服务器事务由以下四步组成。
1. 当一个客户端需要服务时，它向服务器发送一个请求，发起一个事务。例如，当Web浏览器需要一个
文件时，它就发送一个请求给Web服务器。
2. 服务器收到请求后，解释它，并以适当的方式操作它的资源。例如，当Web服务器收到浏览器发出的
请求后，它就读一个磁盘文件。
3. 服务器给客户端发送一个响应，并等待下一个请求。例如，Web服务器将文件发送回客户端。 
4. 客户端收到响应并处理它。例如，当Web浏览器收到来自服务器的一页后，就在屏幕上显示此页。

## 2. 网络
对主机而言，网络只是又一种I/O设备，是数据源和数据接收方。

一个插到I/O总线扩展槽的适配器提供了到网络的物理接口。从网络上接收到的数据从适配器经过I/O
和内存总线复制到内存，通常是通过DMA传送。相似地，数据也能从内存复制到网络。

物理上而言，网络是一个按照地理远近组成的层次系统。最低层是LAN(Local Area Network，局域网)，
在一个建筑或者校园范围内。迄今为止，最流行的局域网技术是以太网(Ethernet)，它是由施乐公司帕
洛阿尔托研究中心(Xerox PARC)在20世纪70年代中期提出的。以太网技木被证明是适力极强的，
从3Mb/s演変到10Gb/s。

一个以太网段(Ethernet segment)包括一些电缆(通常是双绞线)和一个叫做集线器的小盒子，以太网段
通常跨越一些小的区域，例如某建筑物的一个房向或者一个楼层。每根电缆都有相同的最大位带宽，通常是
100Mb/s或者1Gb/s。一端连接到主机的适配器，而另一端则连接到集线器的一个`端口`上。集线器不加分
辨地将从一个端口上收到的每个位复制到其他所有的端又口。因此，每台主机都能看到每个位。

集线器每个以太网适配器都有一个全球唯一的48位地址，它存储在这个适配器的非易失性存储器上。一台主
机可以发送一段位(称为`帧(frame)`)到这个网段内的其他任何主机。每个帧包括一些固定数量的头部
(header)位，用来标识此帧的源和目的地址以及此帧的长度，此后紧随的就是数据位的`有效载荷(payload)`
。每个主机适配器都能看到这个帧，但是只有目的主机实际读取它。

使用一些电缆和叫做`网桥(bridge)`的小盒子，多个以太网段可以连接成较大的局域网，称为桥接
以太网(bridged Ethernet)。桥接以太网能够跨越整个建筑物或者校区。在一个桥接以太网里，
一些电缆连接网桥与网桥，而另外一些连接网桥和集线器。这些电缆的带宽可以是不同的。

网桥比集线器更充分地利用了电缆带宽。利用一种聪明的分配算法，它们随着时间自动学习哪个主机
可以通过哪个端口可达，然后只在有必要时，有选择地将帧从一个端又复制到另一个端口。例如，如
果主机A发送一个帧到同网段上的主机B，当该帧到达网桥X的输入端口时，X就将丢弃此帧，因而节省
了其他网段上的带宽。然而，如果主机A发送一个帧到一个不同网段上的主机C，那么网桥Y只会把此帧
复制到和网桥Y相连的端口上，网桥Y会只把此帧复制到与主机C的网段连接的端口。

在层次的更高级别中，多个不兼容的局域网可以通过叫做`路由器(router)`的特殊计算机连接起来，
组成一个`internet(互联网络)`。每台路由器对于它所连接到的每个网络都有一个适配器(端口)。
路由器也能连接高速点到点电话连接，这是称为`WAN(Wide-Area Network，广域网)`的网络示例，
之所以这么叫是因为它们覆盖的地理范围比局域网的大。一般而言，路由器可以用来由各种局域网和
广域网构建互联网络。

互联网络至关重要的特性是，它能由采用完全不同和不兼容技术的各种局域网和广域网组成。每台主机
和其他每台主机都是物理相连的，但是如何能够让某台源主机跨过所有这些不兼容的网络发送数据位到
另一台目的主机呢?

解决办法是一层运行在每台主机和路由器上的协议软件，它消除了不同网络之间的差异。这个软件实现
一种协议，这种协议控制主机和路由器如何协同工作来实现数据传输。这种协议必须提供两种基本能力:
- 命名机制。不同的局域网技术有不同和不兼容的方式来为主机分配地址。互联网络协议通过定义一种
一致的主机地址格式消除了这些差异。每台主机会被分配至少一个这种互联网络地址(internet addr
-ess)，这个地址唯一地标识了这台主机。
- 传送机制。在电缆上编码位和将这些位封装成帧方面，不同的联网技术有不同的和不兼容的方式。互
联网络协议通过定义一种把数据位捆扎成不连续的片(称为`包`)的统一方式，从而消除了这些差异。一
个包是由`包头`和`有效载荷`组成的，其中包头包括包的大小以及源主机和目的主机的地址，有效载荷
包括从源主机发出的数据位。

主机和路由器如何使用互联网络协议在不兼容的局域网间传送数据的一个示例。这个互联网络示例由两
个局域网通过一台路由器连接而成。一个客户端运行在主机A上，主机A与LAN1相连，它发送一串数据
字节到运行在主机B上的服务器端，主机B则连接在LAN2上。这个过程有8个基本步骤:
1. 运行在主机A上的客户端进行一个系统调用，从客户端的虛拟地址空间复制数据到内核缓冲区中。
2. 主机A上的协议软件通过在数据前附加互联网络包头和LAN1帧头，创建了一个LAN1的帧。互联网
络包头寻址到互联网络主机B。LAN1帧头寻址到路由器。然后它传送此帧到适配器。注意，LAN1帧的
有效载荷是一个互联网络包，而互联网络包的有效载荷是实际的用户数据。这种封装是基本的网络互
联方法之一。
3. LAN1适配器复制该帧到网络上。
4. 当此帧到达路由器时，路由器的LAN1适配器从电缆上读取它，并把它传送到协议软件。
5. 路由器从互联网络包头中提取出目的互联网络地址，并用它作为路由表的素引，确定向哪里转发这
个包，在本例中是LAN2。路由器剥落旧的LAN1的帧头，加上寻址到主机B的新的LAN2帧头，并把得到
的帧传送到适配器。
6. 路由器的LAN2适配器复制该帧到网络上。
7. 当此帧到达主机B时，它的适配器从电缆上读到此帧，并将它传送到协议软件。
8. 最后，主机B上的协议软件剥落包头和帧头。当服务器进行一个读取这些数据的系统调用时，协议软
件最终将得到的数据复制到服务器的虛拟地址空间。

## 3. 全球IP因特网
每台因特网主机都运行实现 `TCP/IP 协议(Transmission Control Protocol/Internet 
Protocol，传输控制协议/互联网络协议)`的软件，几乎每个现代计算机系统都支持这个协议。
因特网的客户端和服务器混合使用`套接字接又口`函数和UnixI/O函数来进行通信。通常将套接
字函数实现为系统调用，这些系统调用会陷入内核，并调用各种内核模式的TCP/IP函数。

TCP/IP实际是一个协议族，其中每一个都提供不同的功能。例如，IP协议提供基本的命名方法和
递送机制，这种递送机制能够从一台因特网主机往其他主机发送包，也叫做数据报(datagram)。
IP机制从某种意义上而言是不可靠的，因为，如果数据报在网络中丢失或者重复，它并不会试图
恢复。`UDP(Unreliable Datagram Protocol，不可靠数据报协议)`稍微扩展了IP协议，
这样一来，包可以在进程间而不是在主机间传送。TCP是一个构建在IP之上的复杂协议，提供了
进程间可靠的全双工(双向的)连接。为了简化讨论，我们将TCP/IP看做是一个单独的整体协议。
我们将不讨论它的内部工作，只讨论TCP和IP为应用程序提供的某些基本功能。我们将不讨论UDP。

从程序员的角度，我们可以把因特网看做一个世界范围的主机集合，满足以下特性:
- 主机集合被映射为一组32位的IP地址。
- 这组IP地址被映射为一组称为`因特网域名(Internet domain name)`的标识符。
- 因特网主机上的进程能够通过连接(connection)和任何其他因特网主机上的进程通信

### 1. IP地址
一个IP地址就是一个32位无符号整数。网络程序将IP地址存放在如下所示的IP地址结构中。
```
struct in_addr {
    uint32_t s_addr; /* Address in network byte order (big-endian) */
};
```
把一个标量地址存放在结构中，是套接字接口早期实现的不幸产物。为IP地址定义一个标量类型应该
更有意义，但是现在更改已经太迟了，因为已经有大量应用是基于此的。因为因特网主机可以有不同
的主机字节顺序，TCP/IP为任意整数数据项定义了统一的`网络字节顺序(network byte order)
(大端字节顺序))`，例如IP地址，它放在包头中跨过网络被携带。在IP地址结构中存放的地址总是
以(大端法)网络字节顺序存放的，即使主机字节顺序(host byte order)是小端法。Unix提供了
下面这样的函数在网络和主机字节顺序间实现转换。
```
#include <arpa/inet.h>
uint32_t hton1 (uint32_t hostlong); 
uint16_t htons (uint16_t hostshort);
// 返回:按照网络字节顺序的值。 
uint32_t ntohl (uint32_t netlong); 
uint16_t ntohs (unit16_t netshort);
// 返回:按照主机字节顺序的值 
```

hotnl函数将32位整数由主机字节顺序转换为网络字节顺序。ntohl函数将32位整数从网络字节顺序转换
为主机字节。htons和ntohs函数为16位无符号整数执行相应的转换。注意，没有对应的处理64位值的函数。

IP地址通常是以一种称为`点分十进制表示法`来表示的，这里，每个字节由它的十进制值表示，并且用句点
和其他字节间分开。例如，128.2.194.242就是地址0x8002c2£2的点分十进制表示。在Linux系统上，
你能够使用HOSTNAME命令来确定你自己主机的点分十进制地址:

linux>hostname-i
128.2.210.175

应用程序使用inet_pton和inet_ntop函数来实现IP地址和点分十进制串之间的转换。
```
#include <arpa/inet.h>
int inet_pton(AF_INET, const char *src, void *dst);
// 返回:若成功则为 1，若src 为非法点分十进制地址则为0，若出錯則为一1。
const char *inet_ntop(AF_INET, const void *src, char *dst , socklen_t size);
// 返回:若成功則指向点分 十进制宇符串的指針，若出错则为NULL.
```
在这些函数名中，“n”代表网络，“p”代表表示。它们可以处理32位IPv4地址(AF_INET)(就像这里展
示的那样)，或者128位IPv6地址(AE_INETG)(这部分我们不讲)。

inet_pton函数将一个点分十进制串(src)转换为一个二进制的网络字节顺序的IP地址(dst)。如果
src没有指向一个合法的点分十进制字符串，那么该函数就返回0。任何其他错误会返回一1，并设置
errno。相似地，inet_ntop函数将一个二进制的网络字节顺序的IP地址(src)转换为它所对应的
点分十进制表示，并把得到的以null结尾的字符串的最多size个字节复制到dst。

### 2. 因特网域名

### 3. 因特网连接

## 4. 套接字接口
套接字接口(socket interface)是一组函数，它们和UnixI/O函数结合起来，用以创建网络应用。
大多数现代系统上都实现套接字接口，包括所有的Unix变种、Windows和Macintosh系统。图11-12
给出了一个典型的客户端-服务器事务的上下文中的套接字接口概述。当讨论各个函数时，可以使用
这张图来作为向导图。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/socketinterfacefun.jpg)

### 1. 套接字地址结构
从Linux内核的角度来看，一个套接字就是通信的一个端点。从Linux程序的角度来看，套接字就是
一个有相应描述符的打开文件。

因特网的套接字地址存放在如图下所示的类型为sockaddr_in的16字节结构中。对于因特网应用，
sin_family成员是AF_INET，sin_port成员是一个16位的端口号，而sin_addr成员就是一个
32位的IP地址。IP地址和端口号总是以网络字节顺序(大端法)存放的。
```
/* IP socket address structure */ 
struct sockaddr_in
{
    uint16_t sin_family; /* Protocol family (always AF_INET) */ 
    uint16_t sin_port; /* Port number in network byte order */ 
    struct in_addr sin_addr; /* IP address in network byte order */
    unsigned char sin_zero[8]; /* Pad to sizeof (struct sockaddr) */ 
};

/* Generic socket address structure (for connect, bind, and accept) */
struct sockaddr 
{
    uint16_t sa_family; /* Protocol family*/
    char sa_data[14];   /* Address data */
}
```
connect、bind和accept函数要求一个指向与协议相关的套接字地址结构的指针。套接字接口
的设计者面临的问题是，如何定义这些函数，使之能接受各种类型的套接字地址结构。今天我们
可以使用通用的void*指针，但是那时在C中并不存在这种类型的指针。解决办法是定义套接字
函数要求一个指向通用sockaddr结构(图上)的指针，然后要求应用程序將与协议特定的结构的
指针强制转换成这个通用结构。为了简化代码示例，我们跟随Steven的指导，定义下面的类型:

typedef struct sockaddr SA;

然后无论何时需要将sockaddr_in结构强制转换成通用sockaddr结构时，我们都使用这个类型。

### 2. socket函数
客户端和服务器使用socket函数来创建一个套接宇描述符(socket descriptor)。
```c
#include <sys/types.h> 
#include ssys/socket.h>
int socket(int domain, int type, int protocol);
// 返回:若成功則为非负描述符，若出錯則 为一1
```
如果想要使套接字成为连接的一个端点，就用如下硬编码的参数来调用socket函数:

clientfd = Socket(AF_INET,SOCK_STREAM, 0);

其中，AF_INET表明我们正在使用32位IP地址，而SOCK_STREAM表示这个套接字是连接
的一个端点。不过最好的方法是用getaddrinfo函数来自动生成这些参数，这样代码就与
协议无关了。

socket返回的clientfd描述符仅是部分打开的，还不能用于读写。如何完成打开套接字
的工作，取决于我们是客户端还是服务器。

### 3. connect函数
客户端通过调用connect函数来建立和服务器的连接。
```c
#include <svs/socket .h>
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen);
// 返回:若成功則为0，若出错則为一1。
```
connect函数试图与套接字地址为addr的服务器建立一个因特网连接，其中addrlen
是sizeof(sockaddr_in)。connect函数会阻塞，一直到连接成功建立或是发生错误。
如果成功，clientfd描述符现在就准备好可以读写了，并且得到的连接是由套接宇对
(x:y,addr.sin_addr:addr.sin_port)
刻画的，其中×表示客户端的IP地址，而y表示临时端口，它唯一地确定了客户端主机上的
客户端进程。对于socket，最好的方法是用getaddrinfo来为connect提供参数。

### 4. bind函数
剩下的套接字函数————bind、listen和accept，服务器用它们来和客户端建立连接。
```
#include <sys/socket .h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 返回:若成功則为0，若出錯則为一1
```
bind函数告诉内核将addr中的服务器套接宇地址和套接字描述符sockfd联系起来。参数
addrlen就是sizeof(sockaddr_in)。对于socket和connect，最好的方法是用
getaddrinfo来为bind提供参数。

### 5. listen函数
客户端是发起连接请求的主动实体。服务器是等待来自客户端的连接请求的被动实体。默认情况下，
内核会认为socket函数创建的描述符对应于`主动套接字(active socket)`，它存在于一个连接
的客户端。服务器调用listen函数告诉内核，描述符是被服务器而不是客户端使用的。
```c
#include sys/socket.h>
int listen(int sockfd, int backlog);
返回:若成功則为0，若出错则为-1 
```
listen函数将sockfd从一个主动套接字转化为一个`监听套接宇(listening socket)`，该套接字
可以接受来自客户端的连接请求。backlog参数暗示了内核在开始拒绝连接请求之前，队列中要排队的
未完成的连接请求的数量。backlog参数的确切含义要求对TCP/IP协议的理解，这超出了我们讨论的
范围。通常我们会把它设置为一个较大的值，比如1024。

### 6. accept函数
服务器通过调用accept函数来等待来自容户端的连接请求。
```
#include <sys/socket .h>
int accept(int listenfd, struct sockaddr *addr, int *addr1en);
// 返回 :若成功則为非负连接描述符，若出错則为-1 。
```
accept两数等待来自客户端的连接请求到达侦听描述符listenfd，然后在addr中填写客户端的
套接字地址，并返回一个`已连接描述符(connected descriptor)`，这个描述符可被用来利用
UnixI/O函数与客户端通信。

监听描述符和已连接描述符之间的区别使很多人感到迷惑。监听描述符是作为客户端连接请求的一个端点。
它通常被创建一次，并存在于服务器的整个生命周期。已连接描述符是客户端和服务器之间己经建立起来了
的连接的一个端点。服务器每次接受连接请求时都会创建一次，它只存在于服务器为一个客户端服务的过程中。

### 7. 主机和服务的转换
Linux提供了一些强大的函数(称为getaddrinfo和getnameinfo)实现二进制套接字地址结构和主机名
、主机地址、服务名和端口号的字符串表示之间的相互转化。当和套接字接口一起使用时，这些函数能使
我们编写独立于任何特定版本的IP协议的网络程序。 
#### 1. getaddrinfo函数
getaddrinfo。函数将主机名、主机地址、服务名和端口号的字符串表示转化成套接字地址结构。
它是已弃用的gethostbyname和getservbyname函数的新的替代品。和以前的那些函数不同，
这个函数是可重入的，适用于任何协议。
```c
#include <sys/types.h> 
#include <sys/socket.h> 
#include <netdb.h>

int getaddrinfo(const char *host, const char *service, const struct addrinfo *hints,
struct addrinfo **result);
// 返回:如果成功則为又，如果错误則为非零的错误代码。
void freeaddrinfo(struct addrinfo *result); 
// 返回:无
const char *gai_strerror (int errcode);
// 返回:错误消息 。
```
给定host和service(套接字地址的两个组成部分)，getaddrinfo返回result,result一个指向addrinfo
结构的链表，其中每个结构指向一个对应于host和service的套接字地址结构(图11-15)。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/getaddrinfo_result.jpg)
在客户端调用了getaddrinfo之后，会遍历这个列表，依次尝试每个套接字地址，直到调用socket和connect
成功，建立起连接。类似地，服务器会尝试遍历列表中的每个套接字地址，直到调用socket和bind成功，描述
符会被绑定到一个合法的套接字地址。为了避免内存泄漏，应用程序必须在最后调用freeaddrinfo，释放该链表。
如果getaddrinfo返回非零的错误代码，应用程序可以调用gai_streeror，将该代码转换成消息字符串。

getaddrinfo的host参数可以是域名，也可以是数字地址(如点分十进制IP地址)。service参数可以是
服务名(如http)，也可以是十进制端口号。如果不想把主机名转换成地址，可以把host设置为NULL。对
service来说也是一样。但是必须指定两者中至少一个。

可选的参数hints是一个addrinfo结构，它提供对getaddrinfo返回的套接字地址列表的更好的控制。如果要
传递hints参数，只能设置下列字段:ai_family、ai_socktype、ai_protocol和ai_flags字段。其他字
段必领设置为0(或NULL)。实际中，我们用memset将整个结构清零，然后有选择地设置一些字段:
- getaddrinfo默认可以返回IPv4和IPv6套接字地址。ai_family设置为AF_INFT会将列表限制为IPv4地址;
设置为AF_INET6则限制为IPv6地址。
- 对于host关联的每个地址，getaddrinfo函数默认最多返回三个addrinfo结构，每个的ai_socktype字段
不同:一个是连接，一个是数据报(末讲述)，一个是原始套接宇(末讲述)。ai_socktype设置为SOCK_STREAM
将列表限制为对每个地址最多一个addrinfo结构，该结构的套接字地址可以作为连接的一个端点。这是所有示
例程序所期望的行为。
- ai_flags宇段是一个位掩码，可以进一步修改默认行为。可以把各种值用OR组合起来得到该掩码。下面是一些
我们认为有用的值:`AL_ADDRCONFIG`。如果在使用连接，就推荐使用这个标志。它要求只有当本地主机被配置
为IPV4时，getaddrinfo返回IPV4地址。对IPV6也是类似。`AL_CANONNAME`。ai_canonname字段默认为
NULL。如果设置了该标志，就是告诉getaddrinfo将列表中第一个addrinfo结构的ai_canonname字段指向
host的权威(官方)名字(见图11-15)。`AL_NUMERICSERV`。参数service默认可以是服务名或端口号。这个
标志强制参数service为端口号。`AL_PASSIVE`。getaddrinfo默认返回套接字地址，客户端可以在调用
connect时用作主动套接字。这个标志告诉该函数，返回的套接字地址可能被服务器用作监听套接字。在这种情
况中，参数host应该为NULL。得到的套接字地址结构中的地址字段会是通配符地址(wildcard address)，
告诉内核这个服务器会接受发送到该主机所有IP地址的请求。这是所有示例服务器所期望的行为。

```c
struct addrinfo {
int ai_flags; /* Hints(提示) argument flags */
int ai_family; /* First arg to socket function */
int ai_socktype; /* Second arg to socket function */
int ai_protocol; /* Third arg to socket function * / 
char *ai_canonname; /* Canonical hostname(规范主机名) */
size_t ai_addrlen; /* Size of ai_addr struct */
struct sockaddr *ai_addr; /* Ptr to socket address structure */ 
struct addrinfo *ai_next; /* Ptr to next item in linked list */
};
```
当getaddrinfo创建输出列表中的addrinfo结构时，会填写每个字段，除了ai_flags。ai_addr字段指向
一个套接字地址结构，ai_addrlen字段给出这个套接字地址结构的大小，而ai_next字段指向列表中下一个
addrinfo结构。其他字段描述这个套接宇地址的各种属性。

getaddrinfo一个很好的方面是addrinfo结构中的字段是不透明的，即它们可以直接传递给套接字接口中
的函数，应用程序代码无需再做任何处理。例如，ai_family、ai_socktype和ai_protocol可以直接
传递给socket。类似地，ai_addr和ai_addrlen可以直接传递给connect和bind。这个强大的属性使得
我们编写的客户端和服务器能够独立于某个特殊版本的IP协议。

#### 2. getnameinfo 函数
getnameinfo函数和getaddrinfo是相反的，将一个套接字地址结构转换成相应的主机和服务名字符串。
它是己弃用的gethostbyaddr和getservbyport函数的新的替代品，和以前的那些函数不同，它是可
重入和与协议无关的。
```c
#include <sys/socket.h> 
#include <netdb.h>
int getnameinfo(const struct sockaddr *sa, socklen_t salen, char *host, 
    size_t hostlen, char *service, size_t servlen, int flags);
// 返回:如果成功則为 0， 如果错误則为非零的錯误代码
```
参数sa指向大小为salen字节的套接字地址结构，host指向大小为hostlen字节的缓冲区，service指向
大小为servlen字节的缓冲区。getnameinfo函数将套接字地址结构sa转换成对应的主机和服务名字符串，
并将它们复制到host和servcice缓冲区。如果getnameinfo返回非零的错误代码，应用程序可以调用
gai_strerror把它转化成字符串。

如果不想要主机名，可以把host设置为NULL,hostlen设置为0。对服务字段来说也是一样。不过，两者
必须设置其中之一。

参数flags是一个位掩码，能够修改默认的行为。可以把各种值用OR组合起来得到该掩码。下面是两个有用的值:
- NI_NUMERICHOST。getnameinfo默认试图返回host中的域名。设置该标志会使该函数返回一个数字地址字符串。
- NL_NUMERICSERV。getnameinfo默认会检查/etc/services，如果可能，会返回服务名而不是端口号。
设置该标志会使该函数跳过查找，简单地返回端口号

示例：
```c
int main(int argc, char **argv) {
    struct addrinfo *p, *listp, hints;
    char buf[MAXLINE];
    int rc, flags;
    if (argc != 2) {
        fprintf(stderr, "usage: %s <domain name>\n", argv[0]);
        exit(0);
    }
    // get a list of addrinfo records
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    if ((rc = getaddrinfo(argv[1], NULL, &hints, &listp)) != 0) {
        fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(rc));
        exit(1);
    }
    // Walk the list and display each IP address
    flags = NI_NUMERICHOST; // Display address string instead of domain name
    for (p = listp; p; p = p->ai_next) {
        getnameinfo(p->ai_addr, p->ai_addrlen, buf, MAXLINE, NULL, 0, flags);
        printf("%s\n", buf);
    }
    // clean up
    freeaddrinfo(listp);
    exit(0);
}
```
首先，初始化hints结构，使getaddrinfo返回我们想要的地址。在这里，我们想查找32位的IP地址AF_INET，
用作连接的端点SOCK_STREAM。因为只想getaddrinfo转换域名，所以用service参数为NULL(getaddrinfo
 第二个参数)来调用它。

调用getaddrinfo之后，会遍历addrinfo结构，用getnameinfo将每个套接字地址转换成点分十进制地址
字符串。遍历完列表之后，我们调用freeaddrinfo小心地释放这个列表(虽然对于这个简单的程序来说，并
不是严格需要这样做的)。

### 8. 套接字接口的辅助函数
初学时，getnameinfo。函数和套接字接口看上去有些可怕。用高级的辅助函数包装一下会方便很多，
称为open_clientfd和open_listenfd，客户端和服务器互相通信时可以使用这些函数。
#### 1. open_clientfd函数
客户端调用open_clientfd建立与服务器的连接。
```c
#include "csapp.h"
int open_clientfd(char *hostname, char *port);
// 返回 : 若成功則为描述符 ，若出錯則为 -1 
```
```c
int open_clientfd(char *hostname, int port) 
{
    
}
```
open_clientfd函数建立与服务器的连接，该服务器运行在主机hostname上，并在端口号port上监听
连接请求。它返回一个打开的套接字描述符，该描述符准备好了，可以用UnixI/O函数做输入和输出。
上面函数给出了open_clientfd的代码。我们调用getaddrinfo，它返回addrinfo结构的列表，每个结构指向一个套接字地址结构，可用于建立与服务器的连接，该服务器运行在hostname上并监听port端又。然后遍历该列表，依次尝试列表中的每个条目，直到调用socket和connect成功。如果connect失败，在尝试下一个条目之前，要小心地关闭套接字描述符。如果connect成功，我们会释放列表内存，并把套接字描述符返回给容户端，客户端可以立即开始用UnixI/0与服务器通信了。
注意，所有的代码都与任何版本的IP无关。socket和connect的参数都是用getaddrinfo自动产生的，这使得我们的代码干净可移植

openlistenfd函数打开和返回一个监听描述符，这个描述符准备好在端口port上接收连接请求。
openlistenfd的风格类似于openclientfd。调用getaddrinfo，然后遍历结果列表，直到调
用socket和bind成功。使用setsockopt函数(本书中没有讲述)来配置服务器，使得服务器能够
被终止、重启和立即开始接收连接请求。一个重启的服务器默认将在大约30秒内拒绝客户端的连接
请求，这严重地阻碍了调试。因为我们调用getaddrinfo时，使用了AI_PASSIVE标志并将host
参数设置为NULL，每个套接字地址结构中的地址字段会被设置为通配符地址，这告诉内核这个服务
器会接收发送到本主机所有IP地址的请求。

如果listen失败，我们要小心地避免内存泄漏，在返回前关闭描述符。

### 9. echo客户端和服务器的示例

## 5. Web服务器
### 2. web内容
Web服务器以两种不同的方式向客户端提供内容:
- 取一个磁盘文件，并将它的内容返回给客户端。磁盘文件称为弹态内容(static content)，而
返回文件给客户端的过程称为服务静态内容(serving static content)。
- 运行一个可执行文件，并将它的输出返回给客户端。运行时可执行文件产生的输出称为动态内容
(dynamic content)，而运行程序并返回它的输出到客户端的过程称为服务动态内容(serving 
dynamic content)。

关于服务器如何解释一个URL的后缀，有几点需要理解:
- 确定一个URL指向的是静态内容还是动态内容没有标准的规则。每个服务器对它所管理的文件都有
自己的规则。一种经典的(老式的)方法是，确定一组目录，例如cgi-bin，所有的可执行性文件都
必须存放这些目录中。
- 后缀中的最开始的那个“/”不表示Linux的根目录。相反，它表示的是被请求内容类型的主目录。
例如，可以将一个服务器配置成这样:所有的静态内容存放在目录/usr/nttpd/html下，而所有的
动态内容都存放在日录/usr/httpd/cgi-bin下。
- 最小的URL后缀是“/”字符，所有服务器将其扩展为某个默认的主页，例如/index.html。这解释
了为什么简单地在浏览器中键入一个域名就可以取出一个网站的主页。浏览器在URL后添加缺失的“/”，
并将之传递给服务器，服务器又把“/”扩展到某个默认的文件名。

### 3. HTTP事务





























