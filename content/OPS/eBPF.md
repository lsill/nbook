---
title: "eBPF"
date: 2023-04-27T14:59:48+08:00
draft: true
---

### BBC定义的苦函数和辅助宏定义
- bpf_get_current_pid_tgid 用于获取进程的 TGID 和 PID。因为这儿定义的 data.pid数据类型为 u32，所以高 32 位舍弃掉后就是进程的 PID;
- bpf_ktime_get_ns 用于获取系统自启动以来的时间，单位是纳秒;
- bpf_get_current_comm 用于获取进程名，并把进程名复制到预定义的缓冲区中;
- bpf_probe_read 用于从指定指针处读取固定大小的数据，这里则用于读取进程打开的文件名。