---
title: "系统级I/O"
date: 2023-12-01T15:45:23+08:00
draft: true
---

# 系统级I/O
输入/输出(I/O)是在主存和外部设备(例如磁盘驱动器、终端和网络)之间复制数据的过程。输人操作是
从I/O设备复制数据到主存，而输出操作是从主存复制数据到I/O设备。

## 1. UNIX I/O
一个Linux文件就是一个m个字节的序列:
Bo，B1，...，Bk，...，Bm-1
所有的I/O设备(例如网络、磁盘和终端)都被模型化为文件，而所有的输人和输出都被当作对相应文件的
读和写来执行。这种将设备优雅地映射为文件的方式，允许Linux内核引出一个简单、低级的应用接口，
称为Unix I/O，这使得所有的输入和输出都能以一种统一且一致的方式来执行:
- 打开文件。一个应用程序通过要求内核打开相应的文件，来宣告它想要访问一个I/O设备。内核返回一
个小的非负整数，叫做`描述符`，它在后续对此文件的所有操作中标识这个文件。内核记录有关这个打开
文件的所有信息。应用程序只需记住这个描述符。
- Linux shell创建的每个进程开始时都有三个打开的文件:标准输入(描述符为0)、标准输出(描述符为1)
和标准错误(描述符为2)。头文件<unistd.h>定义了常量STDIN_FILENO、STDOUT_FILENO和STDERR_FILENO，
它们可用来代替显式的描述符值。
- 改变当前的文件位置。对于每个打开的文件，内核保持着一个文件位置k，初始为0。这个文件位置是从文件
开头起始的字节偏移量。应用程序能够通过执行seek操作，显式地设置文件的当前位置为k。
- 读写文件。一个读操作就是从文件复制n>0个字节到内存，从当前文件位置k开始，然后将k增加到k+n。给定
一个大小为m字节的文件，当k>=m时执行读操作会触发一个称为end-of-file(EOF)的条件，应用程序能检测到
这个条件。在文件结尾处并没有明确的“EOF符号”。类似地，写操作就是从内存复制n>0个字节到一个文件，从当
前文件位置反开始，然后更新人。
- 关闭文件。当应用完成了对文件的访问之后，它就通知内核`关闭`这个文件。作为响应，内核释放文件打开时
创建的数据结构，并将这个描述符恢复到可用的描述符池中。无论一个进程因为何种原因终止时，内核都会关闭
所有打开的文件并释放它们的内存资源.

## 2. 文件
每个Linux文件都有一个类型(type)来表明它在系统中的角色:
- 普通文件(regular file)包含任意数据。应用程序常常要区分`文本文件(text file)1和
`二进制文件(binary file)`，文本文件是只含有ASCII或Unicode字符的普通文件;二进制文
件是所有其他的文件。对内核而言，文本文件和二进制文件没有区别。
Linux文本文件包含了一个文本行(text line)序列，其中每一行都是一个字符序列，以一个新
行符(“\n”)结束。新行符与ASCII的换行符(LF)是一样的，其数字值为OxOa。
- 目录(directory)是包含一组`链接(link)`的文件，其中每个链接都将一个文件名(filename)
映射到一个文件，这个文件可能是另一个目录。每个目录至少含有两个条目:“.”是到该目录自身的链
接，以及“..”是到目录层次结构(见下文)中`父目录(parent directory)`的链接。你可以用mkdir
命令创建一个目录，用ls查看其内容，用rmdir删除该目录。
- 套接字(socket)是用来与另一个进程进行跨网络通信的文件。

其他文件类型包含`命名通道(named pipe)`、`符号链接(symbolic link)`，以及`字符和
块设备(character and block device)`，

Linux内核将所有文件都组织成一个`目录层次结构(directory hierarchy)`，由名为/(斜杠)的根
目录确定。系统中的每个文件都是根目录的直接或间接的后代。

作为其上下文的一部分，每个进程都有一个当前`工作目录(current working directory)`来确定
其在目录层次结构中的当前位置。你可以用cd命令来修改shell中的当前工作目录。

目录层次结构中的位置用`路径名(pathname)`来指定。路径名是一个字符串，包括一个可选斜杠，其后
紧跟一系列的文件名，文件名之间用斜杠分隔。路径名有两种形式:
- 绝对路径名(absolute pathname)以一个斜杠开始，表示从根节点开始的路径。
- 相对路径名(relative pathname)以文件名开始，表示从当前工作目录开始的路径。

## 3. 打开和关闭文件
进程是通过调用open两数来打开一个已存在的文件或者创建一个新文件的：
```c
#include <sys/types.h> 
#include <sys/stat.h> 
#include <fcntl.h>
int open (char *filename, int flags , mode_t mode); 
// 返回:若成功則为新文件描迷符，若出错为一1
```
open函数将filename转换为一个文件描述符，并且返回描述符数字。返回的描述符总是在进程中当前没有
打开的最小描述符。flags参数指明了进程打算如何访问这个文件:
- O_RDONLY:只读。
- O_WRONLY:只写。
- O_RDWR:可读可写。

例入，下面的代码说明如何以读的方式打开一个已存在的文件:
```
fd = open("foo.txt" , O_RDONLY, 0);
```
flags参数也可以是一个或者更多位掩码的或，为写提供给一些额外的指示:
- O_CREAT:如果文件不存在，就创建它的一个`截断的(truncated)`(空)文件。
- O_TRUNC:如果文件已经存在，就截断它。
- O_APPEND:在每次写操作前，设置文件位置到文件的结尾处。

例如，下面的代码说明的是如何打开一个已存在文件，并在后面添加一些数据:
```
fd=open("foo.txt" ,O_WRONLY | O_APPEND，0);
```
mode参数指定了新文件的访问权限位。

作为上下文的一部分，每个进程都有一个umask，它是通过调用umask函数来设置的。当进程通过
带某个mode参数的open函数调用来创建一个新文件时，文件的访问权限位被设置为mode&~umask。

例如，假设我们给定下面的mode和umask默认值:
```
#define DEF_MODE S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH
#define DEF_UMASK S_IWGRP|S_IWOTH
```
接下来，下面的代码片段创建一个新文件，文件的拥有者有读写权限，而所有其他的用户都有读权限:
```
    umask(DEF_UMASK);
    fd = open("foo.txt", O_CREAT | O_TRUNC | O_WRONLY, DEF_MODE);
```

| 掩码      | 描述                |
|---------|-------------------|
| S_IRUSR | 使用者 (拥有者)能够读这个文件  |
| S_IWUSR | 使用者 (拥有者)能够写这个文件  |
| S_IXUSR | 使用者 (拥有者)能够执行这个文件 |
| S_IRGRP | 拥有者所在组的成员能够读这个文件  |
| S_IWGRP | 拥有者所在组的成员能够写这个文件  |
| S_IXGRP | 拥有者所在组的成员能够执行这个文件 |
| S_IROTH | 其他人 (任何人)能够读这个文件  |
| S_IWOTH | 其他人 (任何人)能够写这个文件  |
| S_IXOTH | 其他人 (任何人)能够执行这个文件 |
访问权限位。在sys/stat.h中定义

最后，进程通过调用close 函数关闭一个打开的文件
```
#include sunista.h> 
int close (int fd);
// 逅回:若成功則为0，若出错則为一1
```

## 读和写文件
应用程序是通过分别调用 read 和write 函数来执行输人和输出的。
```
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t n);
//返回:若成功則为读的宇节数，若 EOF 则为0 ，若出错为一1。
ssize_t write(int fd, const void *buf, size_t n);
// 返回:若成功則为写的字节数，若出錯則为一1。
```
read函数从描述符为fd的当前文件位置复制最多n个字节到内存位置buf。返回值-1表示一个错误，
而返回值0表示EOF。否则，返回值表示的是实际传送的字节数量。

write函数从内存位置buf复制至多n个字节到描述符fd的当前文件位置。

通过调用lseek函数，应用程序能够显示地修改当前文件的位置

在某些情况下，read和write传送的字节比应用程序要求的要少。这些`不足值(short count)`不表示
有错误。出现这样情况的原因有:
- 读时遇到EOF。假设我们准备读一个文件，该文件从当前文件位置开始只含有20多个字节，而我们以
50个字节的片进行读取。这样一来，下一个read返回的不足值为20，此后的read将通过返回不足值。
来发出EOF信号。
- 从终端读文本行。如果打开文件是与终端相关联的(如键盘和显示器)，那么每个read函数将一次传送
一个文本行，返回的不足值等于文本行的大小。
- 读和写网络套接宇(socket)。如果打开的文件对应于网络套接字，那么内部缓冲约束和较长的网络延迟
会引起read和write返回不足值。对`Linux管道(pipe)`调用read和write时，也有可能出现不足值，
这种进程间通信机制不在我们讨论的范围之内。

实际上，除了EOF，当你在读磁盘文件时，将不会遇到不足值，而且在写磁盘文件时，也不会遇到不足值。
然而，如果你想创建健壮的(可靠的)诸如Web服务器这样的网络应用，就必须通过反复调用read和write
处理不足值，直到所有需要的字节都传送完毕。

## 5. 用RIO包健壮地读写
在像网络程序这样容易出现不足值的应用中，RIO包提供了方便、健壮和高效的I/O。RIO提供了两类不同的函数:
- 无缓冲的输入输出函数。这些函数直接在内存和文件之间传送数据，没有应用级缓冲。它们对将二进制数据
读写到网络和从网络读写二进制数据尤其有用。
- 带缓冲的输入函数。这些函数允许你高效地从文件中读取文本行和二进制数据，这些文件的内容缓存在
应用级缓冲区内，类似于为printf这样的标准I/O函数提供的缓冲区。与[110]中讲述的带缓冲的I/O
例程不同，带缓冲的RIO输入函数是线程安全的，它在同一个描述符上可以被交错地调用。例如，你可以
从一个描述符中读一些文本行，然后读取一些二进制数据，接着再多读取一些文本行

### 1. RIO的无缓冲的输入输出函数


## 6. 读取文件元数据
应用程序能够通过调用stat和fstat函数，检素到关于文件的信息(有时也称为文件的元数据(metadata))。

```
#include sunistd.h> 
#include <sys/stat.h>
int stat(const char *filename, struct stat *buf); 
int fstat (int fd, struct stat *buf);
// 返回:若成功則为0 ，若出错則为一1
```

## 7. 读取目录内容

## 8. 

```c
// 返回:若成功，則为处理的指針;若出錯，則为NULL.函数opendir以路径名为参数，
// 返回指向目录流(directorystream)的指针。
// 流是对条目有序列表的抽象，在这里是指目录项的列表。
#include <sys/types.h> 
#include sdirent .b>
DIR *opendir(const char *name);
// 返回:若成功，則为处理的指針;若出錯，則为NULL.

#include <dirent.h>
struct dirent *readdir(DIR *dirp);
// 返回:若成功，則为指向 下一个目录项的指针;若没有更多的目录項或出錯，則为NULL

#include sdirent .h>
int closedir (DIR *dirp);
// 返回:成功为0 ;錯误为一1
```

## 8. 共享文件
可以用许多不同的方式来共享Linux文件。除非你很清楚内核是如何表示打开的文件，否则文件
共享的概念相当难懂。内核用三个相关的数据结构来表示打开的文件:
- 描述符表(descriptor table)。每个进程都有它独立的描述符表，它的表项是由进程打开的文
件描述符来索引的。每个打开的描述符表项指向文件表中的一个表项。
- 文件表(file table)。打开文件的集合是由一张文件表来表示的，所有的进程共享这张表。每个
文件表的表项组成(针对我们的目的)包括当前的文件位置、`引用计数(reference count)`(即当前
指向该表项的描述符表项数)，以及一个指向v-node表中对应表项的指针。关闭一个描述符会减少相
应的文件表表项中的引用计数。内核不会删除这个文件表表项，直到它的引用计数为零。
- v-node表(v-node table)。同文件表一样，所有的进程共享这张v-node表。每个表项包含
stat结构中的大多数信息，包括st_mode和st_size成员。

- 不同（描述符表，每个进程一张表）描述符通过打开文件表项引用两个不同的文件，
两个打开文件（所有进程共享）指向不同的v-node表（所有进程共享）
- 不同（描述符表，每个进程一张表）描述符通过打开文件表项引用两个相同的文件，
两个打开的相同文件（描述符都有自己的文件位置）（所有进程共享）指向相同的v-node表（
两个打开的文件表项共享同一个磁盘文件，所有进程共享）
- 父子进程共享相同的打开文件表集合，因此共享相同的文件位置。一个很重要的结果就是，在
内核删除相应文件表表项之前，父子进程必须都关闭了它们的描述符。

## 9. I/O重定向
Linux shell提供了I/O 重定向操作符，允许用户将磁盘文件和标准输人输出联系起 来。例如，键入

linux>ls >foo.txt
```c
#include <unistd.h>
int dup2(int oldfd, int newfd);
// 返回:若成功則为非负的描迷符，若出错則为一1。
```

## 10. 标准I/O
C语言定义了一组高级输入输出函数，称为标准I/O库，为程序员提供了UnixI/O的较高级别的替代。
这个库(libc)提供了打开和关闭文件的函数(fopen和fclose)、读和写字节的函数(fread和
fwrite)、读和写字符串的函数(fgets和fputs)，以及复杂的格式化的I/O函数(scanf和printf)。

标准I/O库将一个打开的文件模型化为一个流。对于程序员而言，一个流就是一个指向FILE类型的
结构的指针。每个ANSI C程序开始时都有三个打开的流stdin,stdout和stderr，分别对应于
标准输人、标准输出和标准错误:
```
#include<stdio.h>
extern FILE*stdin;   /*Standard input(descriptor 0)*/
extern FILE*stdout;  /*Standard output(descriptor 1)*/
extern FILE*stderr;  /*Standard error(descriptor 2)*/

```
类型为FILE的流是对文件描述符和流缓冲区的抽象。流缓冲区的目的和RIO读缓冲区的一样:就是使
开销较高的LinuxI/O系统调用的数量尽可能得小。例如，假设我们有一个程序，它反复调用
标准I/O的getc函数，每次调用返回文件的下一个字符。当第一次调用getc时，库通过调用一次read
函数来填充流缓冲区，然后将缓冲区中的第一个字节返回给应用程序。只要缓冲区中还有末读的宇节，
接下来对getc的调用就能直接从流缓冲区得到服务。

## 11 综合：我该使用哪些I/O函数
I/O包：
![](https://raw.githubusercontent.com/lsill/gitLink/main/document/photo/note/IOpackage.jpg)
UnixI/O模型是在操作系统内核中实现的。应用程序可以通过诸如open、close、iseek、read、write和
stat这样的函数来访问Unix I/0。较高级别的R1O和标准I/O函数都是基于(使用)UnixI/O函数来实现的。
RI0函数是专为本书开发的read和write的健壮的包装函数。它们自动处理不足值，并且为读文本行提供一种
高效的带缓冲的方法。标准I/0函数提供了UnixI/0函数的一个更加完整的带缓冲的替代品，包括格式化的I/0
例程，如printf和scanf 。

那么，在你的程序中该使用这些函数中的哪一个呢?下面是一些基本的指导原则:

- G1:只要有可能就使用标准I/O。对磁盘和终端设备I/O来说，标准I/O函数是首选方法。大多数C程序员
在其整个职业生涯中只使用标准I/O，从不受较低级的UnixI/0函数的困扰(可能stat除外，因为在标准
I/O库中没有与它对应的函数)。只要可能，我们建议你也这样做。
- G2:不要使用scanf或rio_readlineb来读二进制文件。像scanf或rio_readlineb这样的函数是专门
设计来读取文本文件的。学生通常会犯的一个错误就是用这些函数来读取二进制文件，这就使得他们的程序出
现了诡异莫测的失败。比如，二进制文件可能散布着很多0xa字节，而这些字节又与终止文本行无关。
- G3:对网络套接字的I/O使用RIO函数。不幸的是，当我们试着將标准I/O用于网络的输入输出时，出现了
一些令人讨厌的问题。Linux对网络的抽象是一种称为`套接字`的文件类型。就像所有的Linux文件一样，
套接字由文件描述符来引用，在这种情况下称为套接字描述符。应用程序进程通过读写套接字描述符来与
运行在其他计算机的进程实现通信。

标准I/O流，从某种意义上而言是`全双工`的，因为程序能够在同一个流上执行输入和输出。然而，对流的
限制和对套接字的限制，有时候会互相冲突，而又极少有文档描述这些现象:
- `限制一:跟在输出函数之后的输入函数`。如果中间没有插人对fflush、fseek、fsetpos或者rewind
的调用，一个输入函数不能跟随在一个输出函数之后。fflush函数清空与流相关的缓冲区。后三个两数使用
UnixI/O iseek两数来重置当前的文件位置。
- `限制二:跟在输入函数之后的输出函数。`如果中间没有插人对fseek、fsetpos或者rewind的调用，
一个输出函数不能跟随在一个输入函数之后，除非该输人函数遇到了一个文件结束。

这些限制给网络应用带来了一个问题，因为对套接字使用lseek函数是非法的。对流I/O的第一个限制能够
通过采用在每个输入操作前刷新缓冲区这样的规则来满足。然而，要满足第二个限制的唯一办法是，对同
一个打开的套接字描述符打开两个流，一个用来读，一个用来写:
```
FILE* fpin，*fpout;
fpin = fdopen(sockfd,"r");
fpout = fdopen(sockfd,"w");
```

但是这种方法也有问题，因为它要求应用程序在两个流上都要调用fclose，这样才能释放与每个流相关联
的内存资源，避免内存泄漏:
```
fclose(fpin);
fclose(fpout);
```

这些操作中的每一个都试图关闭同一个底层的套接字描述符，所以第二个close操作就会失败。对顺序的程序
来说，这并不是问题，但是在一个线程化的程序中关闭一个已经关闭了的描述符是会导致灾难的。

因此，我们 建议你在网络套接字上不要使用标准I/O两数来进行输人和输出，而要使用健壮的RIO函数。如果你
需要格式化的输出，使用sprintf函数在内存中格式化一个宇行串，然后用rio_writen把它发送到套接口。如
果你需要格式化输入，使用rio_readlineb来读一个完整的文本行，然后用sscanf从文本行提取不同的字段。

## 12. 小结
Linux提供了少量的基于Unix I/O模型的系统级函数，它们允许应用程序打开、关闭、读和写文件，提取文件
的元数据，以及执行I/O重定向。Linux的读和写操作会出现不足值，应用程序必须能正确地预计和处理这种情
况。应用程序不应直接调用Unix I/O函数，而应该使用RI0包，RIO包通过反复执行读写操作，直到传送完所
有的请求数据，自动处理不足值。

Linux内核使用三个相关的数据结构来表示打开的文件。描述符表中的表项指向打开文件表中的表项，而打开
文件表中的表项又指向v-node表中的表项。每个进程都有它自己单独的描述符表，而所有的进程共享同一个
打开文件表和v-node表。理解这些结构的一般组成就能使我们清楚地理解文件共享和I/O重定向。

标淮I/O库是基于Unix I/O实现的，并提供了一组强大的高级I/O例程。对于大多数应用程序而言，
标准I/O更简单，是优于Unix I/0的选择。然而，因为对标准I/O和网络文件的一些相互不兼容的限
制，UnixI/O比之标准I/O更该适用于网络应用程序。








