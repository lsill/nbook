---
title: "并发"
date: 2023-12-15T15:08:28+08:00
draft: true
---

## fork和exec函数
```
#include<unistd.h>
pid_tfork(void);
// 返回:在子进程中为0，在父进程中为子进程1，若出错則为-1
```

fork最困难之处在于调用它一次，它却返回两次。它在调用进程(称为父进程)中返回一次，返回值是新派生
进程(称为子进程)的进程id号:在子进程又返回一次，返回值为0。因此，返回值本身告知当前进程是子进程
还是父进程。

fork在子进程返回0而不是父进程的进程ID的原因在于:任何子进程只有一个父进程，而且子进程总是可以通
过调用getppia取得父进程的进程ID。相反，父进程可以有许多子进程，而且无法获取各个子进程的进程ID。
如果父进程想要跟踪所有子进程的进程D，那么它必须记录每次调用fork的返回值。

父进程中调用fork之前打开的所有描述符在fork返回之后由子进程分享。将看到网络服务器利用了这个特性:
父进程调用accept之后调用fork。所接受的已连接套接字随后就在父进程与子进程之间共享。通常情况下，
子进程接着读写这个己连接套接字，父进程则关闭这个己连接套接字。

fork有两个典型用法。
1. 一个进程创建一个自身的副本，这样每个副本都可以在另一个副本执行其他任务的同时处理各自的某个操作。
这是网络服务器的典型用法。
2. 一个进程想要执行另一个程序。既然创建新进程的唯一办法是调用fork，该进程于是首先调用fork创建一个
自身的副本，然后其中一个副本(通常为子进程)调用exec(接下去介绍)把自身替换成新的程序。这是诸如shell
之类程序的典型用法。

存放在硬盘上的可执行程序文件能够被Unix执行的唯一方法是:由一个现有进程调用六个exec函数中的某一个。
(当这6个函数中是哪一个被调用并不重要时，我们往往把它们统称为exec函数。)exec把当前进程映像替换成新
的程序文件，而且该新程序通常从main函数开始执行。进程ID并不改变。我们称调用exec的进程为调用进程
(calling process)，称新执行的程序为新程序(new program)。

这6个exec函数之间的区别在于:
(a)待执行的程序文件是由文件名(filename)还是由路径名(pathname)指定;
(b)新程序的参数是一一列出还是由一个指针数组来引用;
(c)把调用进程的环境传递给新程序还是给新程序指定新的环境。

```c
#include <unistd.h>
int execl(const char *pathname, const char *arg0, ... /* (char *) 0 */ ); 
int execv(const char *pathname, char *const *argvll);
int execle(const char*pathname, const char* arg0, ... /*(char*)0,char*const envp[]*/);
int execve(const char*pathname, char* const argv[],char* const envp[]);
int.execlp(const char*filename, const char* arg0,.../* (char*)0 */);
int execvp(const char*filename, char* const argv[]);
```
这些函数只在出错时才返回到调用者。否则，控制将被传递给新程序的起始点，通常就是main函数。

这6个函数间的关系如图4-12所示。一般来说，只有execve是内核中的系统调用，其他5个都是调用execve的库函数。

![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/unix/exec_six_func.jpg)

注意这6个函数的下列区别。
1. 上面那行的3个函数把新程序的每个参数字符串指定成exec的一个独立参数，并以一个空指针结束可变数量的
这些参数。下面那行的3个函数都有一个作为exec参数的argv数组，其中含有指向新程序各个参数字符串的所有
指针。既然没有指定参数字符串的数目，这个argv数组必须含有一个用于指定末尾的空指针。
2. 左列2个函数指定一个filename参数。exec将使用当前的PATH环境变量把该文件名参数转换为一个路径名。
然而一旦这2个函数的filename參数中含有一个斜杠(/)，就不再使用PATH环境变量。右两列4个函数指定一个全
限定的pathname参数。
3. 左两列4个函数不显式指定一个环境指针。相反，它们使用外部变量environ的当前值米构造一个传递给新程序
的环境列表。右列2个函数显式指定一个环境列表，其envp指针数组必须以一个空指针结束。

进程在调用exec之前打开着的描述符通常跨exec继续保持打开。我们使用限定词“通常”是因为本默认行为可以使用
fcnt1设置PD_CLOEXEC描述符标志禁止掉。ineta服务器就利用了这个特性。


## 并发服务器

典型的并发服务器程序轮廓
```c
int main(int argc, char **argv)
{
    pid_t pid;
    int listenfd, connfd;
    listenfd = Socket( ... );
    // fill in sockaddr_in() with server's well-known port
    Bind(listenfd, ...);
    Listen(listenfd, LISTENQ);
    for (; ;)
    {
        // 等待连接建立
        connfd = Accept(listenfd, ...); // probably blocks
        if ((pid = Fork()) == 0)
        {
            // 子进程处理doit服务
            Close(listenfd);    // child closes listening socket
            doit(connfd);       // process the request
            Close(connfd);      // done with this client
            exit(0);            // child terminates
        }
        // 子进程已经对connfd提供服务，父进程就可以关闭连接套接字
        Close(connfd);          // parent closes connected socket
    }
}
```

对一个TCP套接字调用close会导致发送一个FIN，随后是正常的TCP连接终止序列。为什么上面代码中父进程对
connfd调用close没有终止它与客户的连接呢?为了便于理解，必须知道每个文件或套接字都有一个引用计数。
引用计数在文件表项中维护，它是当前打开着的引用该文件或套接字的描述符的个数。上面代码中，
socket返回后与listenfd关联的文件表项的引用计数值为1。accept返回后与connfd关联的文件表项的引用
计数值也为1。然而fork返回后，这两个描述符就在父进程与子进程间共享(也就是被复制)，因此与这两个套接
字相关联的文件衣项各自的访问计数值均为2。这么一来，当父进程关闭connfd时，它只是把相应的引用计数值
从2减为1。该套接字真正的清理和资源释放要放到其引用计数值到达0时才发生。这会在稍后子进程也关闭connfd
时发生。

## getsockname和getpeername函数
这两个函数或者返回与某个套接字关联的本地协议地址(getsockname)，或者返回与某个套接字关联的
外地协议地址(getpeername)。















