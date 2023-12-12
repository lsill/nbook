---
title: "STCP"
date: 2023-12-08T16:31:28+08:00
draft: true
---
流控制传输协议，Stream Control Transmission Protocol
# 1. 分组交换图
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/stcptanstate.jpg)
- INIT：包含IP地址清单、初始化序列号、用于表示本关联中所有分组的起始标记、请求的外出流以及
能支持外来流数目
- cookie ：包含确信本关联所需的所有状态，它是数组化签名过得，确保其有效性
- cookie echo:回射服务器的状态cookie
- Ta:客户研验证标记
- Tz:服务验证标记
- 