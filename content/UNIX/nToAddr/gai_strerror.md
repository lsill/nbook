---
title: "gai_strerror 函数"
date: 2023-12-29T16:36:33+08:00
draft: true
---

gai_strerror以这些值为它的唯一参数，返回一个指向对应的出错信息串的指针。
```c
#include <netdb.h>
const char *gai_strerror (int error);
// 返回:指向错误描述消息字符串的指针
```
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/11_7.jpg)