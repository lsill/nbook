---
title: "SCTP套接字选项"
date: 2023-12-26T17:10:43+08:00
draft: true
---

getsockopt和setsockopt函数的第二个参数，IPPROTO_SCTP.

## SCTP_ADAPTION_LAYER 
在关联初始化期间，任何一个端点都可能指定一个适配层指示(adaption layer indication)。这个指示是一个32位
无符号整数，可由两端的应用进程用来协调任何本地应用适配层。

本选项允许调用者获取或没置将由本端提供给对端的适配层指示。获取本选项的值时，调用者得到的是本地套接字将提供
给所有未来对端的值。要获取对端的适配层指示，应用进程必须预订适配层事件。

## SCTP_ASSOCINFO
本套接字选项可用于以下三个目的:(a)获取关于某个现有关联的信息，(b)改变某个已有关联的參数，
(c)为未来的关联设置默认信息。

在获取关于某个现有关联的信息时，应该使用sctp_opt_info函数而不是getsockopt函数数。作为本
选项的输入的是sctp_assocparams结构 。
```c
struct sctp_assocparams
{
    sctp_assoc_t sasoc_assoc_id;
    u_int16_t sasoc_asocmaxrxt;
    u_int16_t sasoc_number_peer_destinations;
    u_int32_t sasoc_peer_rwnd; 
    u_int32_t sasoc_local_rwnd;
    u_int32_t sasoc_cookie_life; 
);
```
- sasoc_assoc_id 存放待访问关联的标识(即关联ID)。如果在调用setsockopt时置0本字段，那么sasoc_asocmaxrxt
和sasoc_cookie_life字段代表将作为默认信息设置在相应套接字上的值。如果在调用getsockopt时提供关联ID，返回的
就是特定于该关联的信息，否则如果置0本字段，返回的就是默认的端点设置信息。
- sasoc_asocmaxrxt 存放的是某个关联在己发送数据没有得到确认的情况下尝试重传的最大次数。达到这个次数后SCTP
放弃重传，报告用户对端不可用，然后关闭该关联。
- sasoc_number_peer_destinations 存放对端目的地址数。它不能设置，只能获取。
- sasoc_peer_rwnd 存放对端的当前接收窗口。该值表示还能发送给对端的数据字节总数。本字段是动态的，本地端点发送数
据时其值减小，外地应用进程读取已经收到的数据时其值增大。它不能设置，只能获取。
- sasoc_local_rwnd 存放本地SCTP协议栈当前通告对端的接收窗口。本字段也是动态的，并受SO_SNDBUF套接字选项影响。
它不能设置，只能获取。
- sasoc_cookie_life 存放送给对端的状态cookie以毫秒为单位的有效期。为了防护重放(replay)攻击，每个随INIT-ACK块
送给对端的状态cookie都关联有一个生命期。原本为60000毫秒的生命期默认值可以通过置sasoc_assoc_id为0并设置本选项加以修改。

## SCTP_AUTOCLOSE 
本选项允许我们获取或设置一个SCTP端点的自动关闭时间。自动关闭时间是一个SCTP关联在空闲时保持打开的秒数。SCTP协议栈把
空闲定义为一个关联的两个端点都没有在发送或接收用户数据的状态。自动关闭功能默认是禁止的。

自动关闭选项意在用于一到多式SCTP接口。当设置本选项时，传递给它的整数值为某个空闲关联被自动关闭前的持续秒数，值为0表示
禁止自动关闭。本选项仅仅影响由相应本地端点将来创建的关联，己有关联保持它们的现行设置不变。

自动关闭功能可由服务器用来强制关闭空闲的关联，服务器无需为此维护额外的状态。使用本特性的服务器应该仔细估算它的所有关联
预期的最长空闲时间。自动关闭时间设置过短会导致关联的过早关闭。

## SCTP_DEFAULT_SEND_PARAM
SCTP有许多可选的发送参数，它们通常作为辅助数据传递，或者由sctp_sendmsg函数使用(sctp_sendmsg通常作为库函数实现，它
替用户传递辅助数据)。希望发送大量消息且所有消息具有相同发送参数的应用进程可以使用本选项设置默认参数，从而避免使用辅助
数据或执行sctp_sendmsg调用。本选项接受sctp_sndrcvinfo结构作为输入。
```c
struct sctp_sndrcvinfo 
{
    u_int16_t sinfo_stream; 
    u_int16_t sinfo_ssn; 
    u_int16_t sinfo_flags; 
    u_int32_t sinfo ppid;
    u_int32_t sinfo_context; 
    u_int32_t sinfo_timetolive; 
    u_int32_t sinfo_tsn; 
    u_int32_t sinfo_cumtsn; 
    sctp_assoc_t sinfo_assoc_id;
};
```
这些字段的含义如下所述。
- sinfo_stream 指定新的默认流，所有外出消息将被发送到该流中。
- sinfo_ssn 在设置默认发送参数时被忽略。当使用recvmsg或sctp_recvmsg函数接收消息时，本字段将存放
由对端置于SCTP DATA块的流序号(stream sequence number,SSN)字段中的值。
- sinto_flags 指定新的默认标志，它们将应用于所有消息发送。
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/7_16.jpg)
- sinfo_ppid 指定将置于所有外出消息中的SCTP净荷协议标识(payload protocol identifier)字段的默认值。
- sinfo_context 指定新的默认上下文。本字段是个本地标志，用于检索无法发送到对端的消息。
- sinfo_timetolive 指定新的默认生命期，它将应用于所有消息发送。SCTP协议栈使用本字段判定何时丢弃(尚未执行首
次传送就)因过度拖延而失效的外出消息。如果同一关联的两个端点都支持部分可靠性(partial reliability)选项，那么
本生命期也用于指定完成首次传送后的消息的继续有效期。
- sinfo_tsn 在设置默认发送参数时被忽略。当使用recvmsg或sctp_recvmsg函数接收消息时，本字段将存放由对端置于
SCTP DATA块的传输序号(transport sequence number,TSN)字段中的值。
- sinfo_cumtsn 在设置默认发送参数时被忽略。当使用recvmsg或sctp_recvmsg函数接收消息时，本字段将存放本地
SCTP协议栈己与对端桂钩的当前累积TSN。
- sinfo_assoc_ia 指定请求者希望对其设置默认参数的关联标识。对于一到一式套接字，本字段被忽略。

注意，所有默认设置只能影响没有指定sctp_sndrcvinfo结构的消息发送。指定了该结构的消息发送(例如带辅助数据的
sctp_sendnsg或sendmsg函数调用)将覆写默认设置。除了进行默认设置，通过使用sctp_opt_info函数，本选项也可
用于获取当前的默认设置。

## SCTP_DISABLE_FRAGMENTS
SCTP通常把太大而不适合置于单个SCTP分组中的用户消息分割成多个DATA块。开启本选项将在发送端禁止这种行为。
被禁止后，SCTP将为此向用户返送EMSGSIZE错误，并且不发送用户消息。SCTP的默认行为与本选项被禁止等效，
也就是说，SCTP通常会对用户消息执行分片。

那些希望自己控制消息大小的应用进程可以使用本选项，以使确保每个用户应用消息都适合置于单个IP分组中。开启了
本选项的应用进程必须准备好处理出错情况(即消息过大)，它们既可以提供应用层的消息分片机制，也可以改用较小的消息。

## SCTP_EVENTS 
本套接字选项允许调用者获取、开启或禁止各种SCTP通知。SCTP通知是由SCTP协议栈发送给应用进程的消息。这种消息
就像普通消息那么读取，只需把recvmsg函数的msghdr结构参数中的msg_flags字段设置为MSG_NOTIFICATION。
不准各使用recvmsg或sctp_recvmsg函数的应用进程不应该开启事件通知功能。使用本选项传递一个sctp_event_subscribe
结构就可以预订8类事件的通知。该结构的格式如下，其中各个字段的值为0表示不预订，为1表示预订
```c
struct sctp_event_subscribe{
    u_int8_t sctp_data_io_event; 
    u_int8_t sctp_association_event; 
    u_int8_t sctp_address_event; 
    u_int8_t sctp_send_failure_event;
    u_int8_t sctp_peer_error_event;
    u_int8_t sctp_shutdown_event;
    u int8_t sctp_partial_delivery_event;
    u_int8_t sctp_adaption_layer_event; 
};
```
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/7_17.jpg)

## SCTP_GET_EER_ADDR_INFO
本选项仅用于获取菜个给定对端地址的相关信息，包括拥塞窗口、平滑化后的RTT和MTU等。作为本选项的输入的是sctp_paddrinfo结构。
调用者在其中的spinfo_address字段填入待查询的对端地址，并且为了便于移植，应该使用sctp_opt_info函数而不是getsockopt函数。
sctp_paddrinfo结构的格式如下:
```c
struct sctp_paddrinto 
{
    sctp_assoc_t spinfo_assoc_id;
    struct sockaddr_storage spinfo_address; 
    int32_t spinfo_state;
    u_int32_t spinfo_cwnd;
    u_int32_t spinfo_srtt;
    u_int32_t spinfo_rto;
    u_int32_t spinfo_mtu;
};
```
返回给调用者的该结构中各个字段的含义如下所述。
- spinfo_assoc_id 存放关联标识，它和“communication up”(通信开始)即SCTP_COMM_UP通知中提供的信息一致。几乎所有SCTP
操作都可以使用这个唯一的值作为相应关联的简明标识。
- spinfo_address由调用者设置，用于告知SCTP套接字想要获取哪一个对端地址的信息。调用返回时其值不应该改变。
- spinfo_state存放图7-18所示的一个或多个常值
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/7_18.jpg)
其中未证实地址(unconfirmed address)是一个对端己作为有效地址列出，而本地SCTP尚不能证实对端确实特有它的地址。
当送往某个地址的心跳或用户数据得到对端确认时，本地SCTP端点就可以证实该地址确实为对端所有了。注意，未证实的地址
并没有有效的重传超时(retransmission timeout,RTO)值。活跃地址则表示被认为是可用的地址。
- spinfo_cwnd 表示为所指定对端地址维护的当前拥塞窗口。
- spinfo_srtt 表示就所指定对端地址而言的平滑化后RTT的当前估计值。
- spinfo_rto 表示用于所指定对端地址的当前重传超时值。
- spinfo_mtu 表示由路径MTU发现功能发现的通往所指定对端地址的路径MTU的当前值。
  
本选项的一个有意思的用途是:把一个IP地址结构转换成一个可用于其他调用的关联标识。另一个可能用途是:由应用进程跟踪一
个多宿对端主机每个地址的性能，并把相应关联的主目的地址更新为其中性能最佳的一个。这些值也同样有利于日志记录和程序调试。

## SCTP_I_WANT_MAPPED_V4_ADDR
这个标志套接字选项用于为AF_INET6类型的套接字开启或禁止Iv4映射地址，其默认状态为开启。注意，本选项开启时，所有
IPv4地址在送往应用进程之前将被映射成一个IPV6地址。本选项禁止时，SCTP套接字不会对IPv4地址进行映射，而是作为
sockaddr_in结构直接传递.

## SCTP_INITMSG
本套接字选项用于获取或设置某个SCTP套接字在发送INIT消息时所用的默认初始参数。作为本选项的输入的是sctp_initmsg结构，其定义如下:
```c
struct sctp_initmsg 
{
    uint16_t sinit_num_ostreams; 
    uint16_t sinit_max_instreams; 
    uint16_t sinit_max_attempts; 
    uint16_c sinit_max_init_timeo;
};
```
这些字段的含义如下所述。
- sinit_num_ostreams 表示应用进程想要请求的外出SCTP流的数目。该值要等到相应关联完成初始握手后才得到确认，而且可能因为对端的限制而向下协调。
- sinit_max_instreams 表示应用进程准备允许的外来SCTP流的最大数目。如果该值大于SCIP协议栈所文持的最大允许流数，那么它将被改为这个最大数。
- sinit_max_attempts 表示SCTP协议栈应该重传多少次初始INIT消息才认为对端不可达。
- sinit_max_init_timeo 表示用于INIT定时器的最大RTO值。在初始定时器进行指数退避期间，该值将替代RTO.max作为重传RTO极限。该值以毫秒为单位。

注意，当设置这些字段时，SCTP將忽略其中的任何0值。一到多式套接字的用户在关联隐性建立期间也可能在辅助数据中传递一个sctp_initmsg结构。

## SCTP_MAXBURST
本套接字选项允许应用进程获取或设置用于分组发送的最大猝发大小(maximum burstsize)。当SCTP向对端发送数据时，一次不能发送至于这个数目
的分组，以免网络被分组淹没。具体的SCTP实现有两种方法应用这个限制:
(1)把拥塞窗口缩减为当前飞行大小(current flight size)加上最大猝发大小与路径MTU的乘积；
(2)把该值作为一个独立的微观控制量，在任意一个发送机会最多只发送这个数目的分组。

## SCTP_MAXSEG
本套接字选项允许应用进程获取或设置用于SCTP分片的最大片段大小(maximum fragment size)。当某个SCTP发送端从其应用进程收到一个大于这个
大小的消息时，它将把该消息分割成多个块，以便分别传送到对端。SCTP发送端通常使用的这个大小是通达它的对端的所有路径各自的MTU中的最小值
(每条路径对应一个对端地址)。设置本选项可以把这个大小降低到所指定的值。注意，SCTP可能以比本选项所请求的值更小的边界分割消息。当通达对
端的某条路径的MTU变得比本选项所请求的值还要小时，这种偏小的分割就会发生。最大片段大小是一个端点范围的设置，在一到多式接又中，它可能影
响不止一个关联。

## SCTP_NODELAY
开启本选项将禁止SCTP的Nagle算法。本选项默认关闭(也就是说默认情况下Nagle算法是启动的)。SCTP的Nagle算法与TCP的Nagle算法工作
原理相同，区别在于前者对付多个DATA块，后者对付单个流上的字节。

## SCTP_PEER_ADDR_PARAMS
本套接字选项允许应用进程获取或设置关于某个关联的对端地址的各种参数。作为本选项的输入的是sctp_paddrparams结构。调用者必须在该结构
中填写关联标识和/或一个对端地址，其定义如下:
```c
struct sctp_paddrparams 
{
    setp_assoc_t spp_assoc_id;
    struct sockaddr_storage spp_address; 
    u_int32_t spp_hbinterval;
    u_int16_t spp_pathmaxrxt;
};
```
这些字段的含义如下所述。
- spp_assoc_id 存放在其上获取或设置参数信息的关联标识。如果该值为0，那么所访问的是端点默认参数，市不是特定于关联的参数。
- spp_address 指定其参数待获取或待设置的对端IP地址。如果spp_assoc_ia字段值为0，那么本字段被忽略。
- spp_hbinterval 表示心跳间隔时间。设置该值为SCTP_NO_HB将禁止心跳，为SCTP_ISSUE_HB将按请求心跳，为其他值则将把心跳
间隔重置为以毫秒为单位的新值。设置端点默认参数时，不能使用SCTP_ISSUE_HB这个值。
- spp_pathmaxrxt 表示在声明所指定对端地址为不活跃之前将尝试的重传次数。当主目的地址被声明为不活跃时，另外一个对端地址将被选为主目的地址。

## SCTP_PRIMARY_ADDR
本套接字选项用于获取或设置本地端点所用的主目的地址。主目的地址是本端发送给对端的所有消息的默认目的地址。
作为本选项的输入的是sctp_setprim结构。调用者必须在该结构中填写关联标识，若是设置主目的地址则再填写一
个将用作主目的地址的对端地址，其定义如下:
```c
struct sctp_setprim {
    sctp_assoc_t ssp_assoc_id; 
    struct sockaddr_storage ssp_addr;
};
```
这些字段的含义如下所述。
- ssp_assoc_id 存放在其上获取或设置当前主目的地址的关联标识。对于一到一式套接字，本字段被忽略。
- ssp_addr 指定主目的地址(主目的地址必须是一个属于对端的地址)。使用setsockopt函数设置本选项时，
本字段为请求者要求设置的主目的地址的新值，使用getsockopt函数获取本选项时，本字段为当前所用主目的
地址的值。

注意，在只有一个本地地址与之关联的一到一式套接字上获取本选项的值跟直接调用getsockname是一样的。

## SCTP_PTOINFO
本套接字选项用于获取或设置各种RTO信息，它们既可以是关于某个给定关联的设置，也可以是用于本地端点的默认设置。
为了便于移植，当获取信息时，调用者应该使用sctp_opt_info函数而不是getsockopt函数。作为本选项的输入的是
sctp_rtoinfo结构，其定义如下:
```c
struct sctp_rtoinfo 
{
    sctp_assoc_t srto_assoc_id; 
    uint32_t srto_initial; 
    uint32_t srto_max; 
    uint32_t srto_min;
};
```
这些字段的含义如下所述。
- srto_assoc_id 存放感兴趣关联的标识或0。若值为0，当前函数调用会对系统的默认参数产生影响。
- srto_initial 存放用于对端地址的初始RTO值。初始RTO值在向对端发送INIT块时使用。该值以毫秒为单位且默认值为3000。
- srto_max 存放在更新重传定时器时使用的最大RTO值。如果更新后的RTO值大于这个RTO最大值，那就把这个最大值作为新的
RTO值。该值默认为60000。
- srto_min 存放在启动重传定时器时使用的最小RTO值。任何时候RTO定时器一旦更改，就对照这个RTO最小值检查新值。如果
新值小于最小值，那就把这个最小值作为新的RTO值。该值默认为1000。

srto_initial、srto_max或srto_min值为0表示当前设定的默认值不应改变。所有时间值都以亳秒为单位。

## SCTP_SET_PEER_PRIMARY_ADDR
设置本套接字选项导致发送一个消息:请求对端把所指定的本地地址作为它的主目的地址。作为本选项的输入的是sctp_setpeerprim结构。
调用者必须在该结构中填写关联标识和一个请求对端标为其主目的地址的本地地址。这个本地地址必须己经绑定在本地端点。
sctp_setpeerprim结构的定义如下:
```c
struct sctp_setpeerprim (
    sctp_assoc_t sspp_assoc_id; 
    struct sockaddr_storage sspp_addr;
};
```
这些字段的含义如下。
- sspp_assoc_id 指定在其上想要设置主目的地址的关联标识。对于一到一式套接字，本字段被忽胳。
- sspp_addr 存放想要对端设置为主目的地址的本地地址。本特性是可选的，只有两端均支持才能运作。如果本地端点不支持本特性，
那就给调用者EOPNOTSUPP返回错误。如果远程端点不支持本特性，那就返回调用者EINVAL错误。另外注意，本套接字选项只能设置，不能获取。

## SCTP_STATUS 
本套接字选项用于获取某个SCTP关联的状态。为了便于移植，调用者应该使用sctp_opt_info函数而不是getsockopt函数。
作为本选项的输入的是sctp_status结构。调用者必须在该结构中填写关联标识，关于这个关联的信息将在返回时被填写到该结
构的其他字段中。scLp_status结构的格式如下:
```c
struct sctp_status
{
    sctp_assoc_t sstat_assoc_id; 
    int32_t stat_state;
    u_int32_t setat_rwnd; 
    u_int16_t sstat_unackdata;
    u_int16_t sstat_penddata;
    u_int16_t stat_instrms;
    u_int16_t sstat_outstrms;
    u_int32_t sstat_fragmentation_point; 
    struct sctp paddrinfo stat_ primary;
};
```
这些字段的含义如下。
- sstat_assoc_id存放关联标识。
- sstat_state存放图7-19所示常值之一，指出关联的总体状态。
- sstat_rwnd 存放本地端点对于对端接收窗口的当前估计。
- sstat_unackdata 存放等着对端处理的未确认DATA块数目。
- sstat_pendata 存放本地端点暂存并等着应用进程读取的末读DATA块数目。
- sstat_instrms 存放对端用于向本端发送数据的流的数目。
- sstat_outstrms 存放本端可用于向对端发送数据的流的数目。
- sstat_fragmentation_point 存放本地SCTP端点将其用作用户消息分割点的当前值。该值通常是所有目的地址的
最小MTU，或者是由本地应用进程使用SCTP_MAXSEG套接字选项设置的更小的值。
- sstat_primary 存放当前主目的地址。主目的地址是向对瑞发送数据时使用的默认目的地址。

这些值可用于诊断或确定会话的特征。举偏低的sstat_rwnd值和/或偏高的sstat_unackdata值可用于判定对端的接收套
接字缓冲区正在变满，这一点又可用作让应用进程尽可能降低发送速率的信号。有些应用进程使用sstat_fragmentation_point
减少SCTP不得不创建的片段数量，办法就是发送较小的应用消息。




















