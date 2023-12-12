---
title: "套接字编程简介"
date: 2023-12-08T17:32:28+08:00
draft: true
---

# 通用套接字地址结构sockaddr_storage
```
struct sockaddr_storage {
    sa_family_t  ss_family;  // 地址族

    // 实现特定的填充，确保结构体的大小足够容纳所有类型的地址
    // 并且对齐到适当的边界。
    char         __ss_pad1[_SS_PAD1SIZE];
    int64_t      __ss_align;
    char         __ss_pad2[_SS_PAD2SIZE];
};
```
在实际的网络编程中，sockaddr_storage 结构体通常用于以下情况：
- 接收任意类型的地址：当你的程序需要接受不同类型的套接字地址时（例如，同时处理 IPv4 
和 IPv6 连接），可以使用 sockaddr_storage 结构体来通用地存储地址信息。
- 与 sockaddr 类型兼容：sockaddr_storage 可以被安全地强制转换为 sockaddr 类型
，用于那些需要 sockaddr * 类型参数的系统调用（如 bind(), connect(), accept() 等）。

这种结构体的使用提高了网络程序的灵活性和兼容性，使其能够适应不同的网络环境和地址类型。

在不同的操作系统和网络编程环境中，sockaddr_storage 结构的具体实现可能会有所不同。在某些系统
中，会存在一个名为 ss_len 的字段，用于表示地址的长度。然而，在 POSIX 标准中，sockaddr_storage 
结构并没有定义这样一个字段。

# 套接字结构比较
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/socket_struct_com.jpg)



