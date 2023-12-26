---
title: "Ipv4套接字选项"
date: 2023-12-26T16:02:23+08:00
draft: true
---

getsockopt和setsockopt函数的第二个参数为 IPPROTO_IP。

## IP_HDRINCL 套接字选项
如果本选项是给一个原始IP套接字设置的，那么我们必须为所有在该原始套接字上发送的数据报构造自己的IP首部。一般情况下，
在原始套接字上发送的数据报其IP首部是由内核构造的，不过有些应用程序(特別是路由跟踪程序traceroute)需要构造自己的
IP首部以取代IP置于该首部中的某些字段。

当本选项开启时，我们构造完整的IP首部，不过下列情况例外。
- IP总是计算并存储IP首部校验和。
- 如果我们将IP标识字段置为0，内核将设置该字段。
- 如果源IP地址是INADDR_ANY，IP将把它设置为外出接口的主IP地址。
- 如何设置IP选项取决于实现。有些实现取出我们预先使用IP_OPTIONS套接字选项设置的任何IP选项，把它们添加到我们构造
的首部中，而其他实现则要求我们亲自在首部指定任何期望的IP选项。
- IP首部中有些字段必须以主机字节序填写，有些字段必须以网络字节序填写，具体取决于实现。这使得利用本套接字选项编排
原始分组的代码不像期待的那样便于移植。

## IP_OPTIONS 套接字选项
本选项的设置允许我们在IPv4首部中设置IP选项。

## IP_RECVDSTADDR 套接字选项
(ip recv dit addr)

本套接字选项导致所收到UDP数据报的目的IP地址由recvmsg函数作为辅助数据返回。

## IP_RECVIF 套接字选项
(ip recv if)

本套接字选项导致所收到UDP数据报的接收接口素引由recvmsg函数作为辅助数据返回

## IP_TOS 套接字选项
本套接字选项允许我们为TCP、UDP或SCTP套接字设置IP首部中的服务类型字段(该字段包含DSCP和ECN子字段)。如果我们给本选项
调用getsockopt，那么用于放入外出IP数据报首部的DSCP和ECN字段中的TOS当前值(默认为0)将返回。我们没有办法从接收到的
IP数据报中取得该值。

应用进程可以把DSCP设置成用户和网络业务供应商预先协商好的某个值，以便接受预定的服务，例如对IP电话的低延迟服务，对海量
数据传送的高吞吐量服务。把IP_TOS设置成<netinet/ip.h>中定义的某个常值(例如IPTOS_LOWDELAY和IPTOS_THROUGHPUT)
的应用程序应该改为使用由用户指定的某个DSCP值。区分服务存留的TOS值只有优先权级别6(“internet work control”，网间
控制)和7(“network control”，网内控制)，这意味着把IP_TOS设置成IPTOS_PREC_NETCONTROL或IPTOS_PREC_INTERNETCONTROL
的应用程序在区分服务网络中可以继续工作。

应用进程通常应该把ECN字段的设置留给内核，也就是把由IP_TOS设置的值中的低两位指定为0。

## IP_TTL 套接字选项
可以使用本选项设置或获取系统用在从某个给定套接字发送的单播分组上的默认TTL值。(多播TTL值使用IP_MULTICAST_ITL套接字选项设置)
例如4.4BSD对TCP和UDP套接字使用的默认值都是64(这由IANA的“IP Option Numbers”注册处规定)，对原始套接字使用的默认值则是255
。跟TOS字段一样，调用getsockopt返回的是系统将用于外出数据报的字段的默认值。我们没有办法从接收到的IP数据报中取得该值。



















