---
layout: post
title: Linux 进程
date: 2022-11-30
author: 来自第一世界
tags: [Linux]
comments: false
---
记录 Linux 学习过程中需要总结的点

# Linux 基本操作

## 文件操作

### open 函数

Linux编程下 `open()` 函数的用法

定义函数：

```c
int open( const char * pathname, int flags);
int open( const char * pathname,int flags, mode_t mode);
```

返回值：返回 0 值，表示成功，只要有一个权限被禁止则返回 -1。

一般的写法是

```c
if((fd=open("/dev/ttys0", O_RDWR | O_NOCTTY | O_NDELAY)<0)
{
	perror("open");
}
```

第二参数 flags 所用参数：

* O_RDONLY 只读打开。
* O_WRONLY 只写打开。
* O_RDWR 读、写打开。
* O_APPEND 每次写时都加到文件的尾端。
* O_CREAT 若此文件不存在则创建它。使用此选择项时，需同时说明第三个参数mode，用其说明该新文件的存取许可权位。
* O_EXCL 如果同时指定了O_CREAT，而文件已经存在，则出错。这可测试一个文件是否存在，如果不存在则创建此文件成为一个原子操作。
* O_TRUNC 如果此文件存在，而且为只读或只写成功打开，则将其长度截短为 0。
* O_NOCTTY 如果 pathname 指的是终端设备，则不将此设备分配作为此进程的控制终端。
* O_NONBLOCK 如果 pathname 指的是一个 FIFO、一个块特殊文件或一个字符特殊文件，则此选择项为此文件的本次打开操作和后续的 I/O 操作设置非阻塞方式。
* O_NDELAY所产生的结果使 I/O 变成非阻塞模式(non-blocking)，在读取不到数据或是写入缓冲区已满会马上 return，而不会阻塞等待。
* O_SYNC 使每次 write 都等到物理 I/O 操作完成。
* O_APPEND 当读写文件时会从文件尾开始移动，也就是所写入的数据会以附加的方式加入到文件后面。
* O_NOFOLLOW 如果参数pathname 所指的文件为一符号连接，则会令打开文件失败。
* O_DIRECTORY 如果参数pathname 所指的文件并非为一目录，则会令打开文件失败。

这些控制字都是通过“或”符号分开。

### write 函数

`write()` 会把参数 buf 所指的内存写入 count 个字节到参数 fd 所指的文件内。

返回值：如果顺利 `write()` 会返回实际写入的字节数（len）。当有错误发生时则返回 -1，错误代码存入errno 中。

```c
#include <unistd>
ssize_t write(int filedes, void *buf, size_t nbytes);
// 返回：若成功则返回写入的字节数，若出错则返回-1
// filedes：文件描述符
// buf:待写入数据缓存区
// nbytes:要写入的字节数
```

### read 函数

函数从打开的设备或文件中读取数据。

```c
#include <unistd.h>  
ssize_t read(int fd, void *buf, size_t count);  
```

返回值：成功返回读取的字节数，出错返回-1 并设置 errno，如果在调 read 之前已到达文件末尾，则这次 read 返回0

## Linux 进程

### 进程状态

* D：不可中断的深度睡眠 状态 ，处于这种状态 的进程 不 能响应异步信号；
* R：进程处于运行 态 或就绪状态 只有在该状态的进程才可能在 CPU 上运行。而同一时刻可能有多个进程处于可执行状态；
* S：可中断 的睡眠状态，处于这个状态的进程因为等待某种 事件的发生而被挂起；
* T：暂停状态或跟踪状态
* X：退出状态，进程即将被销毁
* Z：退出状态，进程成为僵尸进程 。

### 进程操作

#### 创建进程

fork() 函数用于创建进程，将运行着的进程分裂出另一个子进程，它通过拷贝父进程的方式创建子进程。如果成功创建了进程，会对父子进程各返回一次，其中对父进程返回子进程的 PID，对子进程返回 0；失败返回小于0的错误码。

#### 终止进程

进程终止可分为正常终止和异常终止两大类，其中常见正常终止方式的有：

* 从 `main() `函数 return 返回；
* 调用类 `exit()` 函数。

常见的异常终止方式有：

* 调用 `abort()` 函数；
* 接收到一个信号终止。

#### exec 函数

exec 族函数用来替换调用进程的执行程序。

`fork()` 在创建进程后子进程与父进程有相同的代码空间；而实际应用中，子进程往往需要执行另一个程序，这种情况下，可以在子进程中调用 exec 族函数将此进程的执行程序完全替换为新程序，并从新进程的 main 函数开始执行。

exec族函数有6个不同的exec函数，函数原型分别如下：

```c
#include 
extern char **environ;
int execl(co nst char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg,..., char * const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const arg v[]);
int execvpe(const char *file, char *const argv[],char *const envp[]);
```

* 后缀 p：表示使用 filename 做参数，如果 filename 中包含“/”，则视为路径名，否则在 PATH 环境变量所指定的各个目录中搜索可执行文件，如 `execlp()` 函数。无后缀 p 则使用路径名来指定可执行文件的位置，如 `execl()` 函数。
* 后缀 e：表示可以传递一个指向环境字符串指针数组的指针，环境数组需要以 NULL 结束，如 execvpe() 函数。而无此后缀的函数则使用调用进程中 environ 变量为新程序复制现有的环境，如 `execv()` 函数。
* 后缀 l：表示使用 list 形式来传递新程序的参数，传给新程序的所有参数以可变参数的形式从 exec 给出，最后一个参数需要为NULL以表示结束，如 `execl()` 函数。
* 后缀 v：表示使用 vector 形式来传递新程序的参数，传给新程序的所有参数放入一个字符串数组中，数组以 NULL 结束以表示结束，如 `execv()` 函数

exec族函数只有在出错的时候才会返回，如果成功，该函数无返回，否则返回-1。

#### 进程退出

`wait()` 函数用来帮助父进程获取其子进程的退出状态 。当进程退出时，内核为每一个进程保存了一定量的退出状态信息，父进程可根据此退出信息来判断子进程的运行状况。如果父进程未调用 `wait()` 函数，则子进程的退出信息将一直保存在内存中。

由于进程终止的异步性，可能会出现子进程先终止或者父进程先终止的情况，从而出现两种特殊的进程：

* 僵尸进程：如果子进程先终止，但其父进程未为其调用 `wait()` 函数，那么该子进程就变为僵尸进程。僵尸进程在它父进程为它调用 `wait()` 函数之前将一直占有系统的内存资源。
* 孤儿进程：如果父进程先终止，尚未终止的子进程将会变成孤儿进程。孤儿进程将直接被 init 进程收管，由 init 进程负责收集它们的退出状态。

### 进程间通信

#### 管道

管道是一个进程连接数据流到另一个进程的通道，它通常是用作把一个进程的输出通过管道连接到另一个进程的输入。通过符号“|”将一个进程的输出连接到另一个管道的输入中。

管道分为匿名管道和命名管道两种，匿名管道主要用于两个进程间有父子关系的进程间通信，命名管道主要用于没有父子关系的进程间通信。

##### 匿名管道

匿名管道是不能在文件系统中以任何方式看到的半双工管道。

`pipe()` 函数可以用来创建一条匿名管道，它的原型如下：

```c
#include <unistd.h>
int pipe(int pipefd[2]);
```

函数成功返回0，否则返回-1。

参数 pipefd 是一个文件描述符数组，对应着打开管道的两端，其中 pipefd[0] 为读端，pipefd[1] 为写端，往写端写的数据会被内核缓存起来，直到读端将数据读完。

##### 命名管道

命名管道也被称为FIFO文件，它突破了匿名管道无法在无关进程之间通信的限制，使得同一主机内的所有的进程都可以通信。

同时命名管道是一个特殊的文件类型，它在文件系统中以文件名的形式存在，在 stat 结构中 st_mode 指明一个文件结点是不是命名管道。

mkfifo()函数用来创建一个命名管道，它的原型如下：

```c
#include <sys/types.
#include <sys/stat.
int mkfifo(const char *pathname, mode_t mode);
```

`mkfifo()` 创建一个真实存在于文件系统中的命名管道文件，参数 pathname 指定了文件名，参数 mode 则指定了文件的读写权限。函数成功返回 0，否则返回-1并设置errno

`mkfifo()` 创建命名管道文件后，需要通过命名管道通信的进程需要打开该管道文件，然后通过 `read`、`write` 函数像操作普通文件一样进行通信。
