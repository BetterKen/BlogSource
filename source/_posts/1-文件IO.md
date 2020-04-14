---
title: 1.文件系统IO
date: 2020-04-14 20:00:00
tags:
    - Linux
    - 系统编程
categories:
    - 基础
    - Linux系统编程
---

## 1.1 C标准函数与系统函数的区别

C函数进行文件操作的时候,是调用的系统内核函数,中间经过一层缓冲区

![](http://base422.oss-cn-beijing.aliyuncs.com/sysfile.png)

### 1.1.1 I/O缓冲区

`每一个FILE文件流都有一个缓冲区buffer，默认大小8KB`。

### 1.1.2 区别

然write系统调用位于C标准库I/O缓冲区的底层，但在write的底层也可以分配一个内核I/O缓冲区，所以write也不一定是直接写到文件的，也可能写到内核I/O缓冲区中，**至于究竟写到了文件中还是内核缓冲区中对于进程来说是没有差别的，如果进程A和进程B打开同一文件，进程A写到内核I/O缓冲区中的数据从进程B也能读到，而C标准库的I/O缓冲区则不具有这一特性**



## 1.2 PCB概念

广义上，所有的进程信息被放在一个叫做进程控制块的数据结构中，可以理解为进程属性的集合。

**每个进程在内核中都有一个进程控制块来维护进程的相关信息，Linux内核的进程控制块是task_struct结构体**，PCB结构体所在位置如下:

```c
/usr/src/linux-headers/include/linux/sched.h
```

## 1.3 文件描述符

**一个进程默认打开3个文件描述符**

```shell
STDIN_FILENO 0  　 标准输入
STDOUT_FILENO 1　　标准输出
STDERR_FILENO 2　　错误输出
```

**新打开文件返回文件描述符表中未使用的最小文件描述符。**
open函数可以打开或创建一个文件。



## 1.4 open/close函数

### 1.4.1 open函数

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
int open(const char *pathname, int flags, ...);
返回值：成功返回新分配的文件描述符，出错返回-1并设置errno

```

#### 1.4.1.1 pathname

pathname参数是要打开或创建的文件名，和fopen一样，pathname既可以是相对路径也可以是绝对路径。

#### 1.4.1.2 flags

flags参数有一系列常数值可供选择，可以同时选择多个常数用按位或运算符连接起来，所以这些常数的宏定义都以O_开头，表示or。

**必选项**：以下三个常数中必须指定一个，且仅允许指定一个。

- **O_RDONLY 只读打开**
- **O_WRONLY 只写打开**
- **O_RDWR 可读可写打开**

以下**可选项可以同时指定0个或多个**，和必选项按位或起来作为flags参数。可选项有很多，
这里只介绍常用部分:

- **O_APPEND 表示追加**。如果文件已有内容，这次打开文件所写的数据附加到文件的末尾而不覆盖原来的内容。
- **O_CREAT 若此文件不存在则创建它**。使用此选项时需要提供第三个参数mode，表示该文件的访问权限。
- **O_EXCL** 如果同时指定了O_CREAT，并且文件已存在，则出错返回
- **O_TRUNC** 如果文件已存在，并且以只写或可读可写方式打开，则将其长度截断（Truncate）为0字节。
- **O_NONBLOCK 对于设备文件，以O_NONBLOCK方式打开可以做非阻塞I/O（Nonblock I/O）**

#### 1.4.1.4 mask

第三个参数mode指定文件权限，可以用八进制数表示，比如0644表示-rw-r-r–，也可以用S_IRUSR、S_IWUSR等宏定义按位或起来表示，详见open(2)的Man Page。要注意的是，文件权限由open的mode参数和当前进程的umask掩码共同决定。

```shell
$ umask 
0002 
```

可以看到umask值为0002，其中第一个0与特殊权限有关，可以暂时不用理会，后三位002则与普通权限(rwx)有关

- 其中002中第一个0与用户(user)权限有关，表示从用户权限减0，也就是权限不变，所以文件的创建者的权限是默认权限(rw)

- 第二个0与组权限（group）有关，表示从组的权限减0，所以群组的权限也保持默认权限（rw）

- 最后一位2则与系统中其他用户（others）的权限有关，由于w=2，所以需要从其他用户默认权限（rw）减去2，也就是去掉写（w）权限，则其他人的权限为rw - w = r

则创建文件的最终默认权限为 -rw-rw-r-- 。同理，目录的默认权限为 drwxrwxrwx ，则d rwx rwx rwx - 002 = (d rwx rwx rwx)

用touch命令创建一个文件时，创建权限是0666，而touch进程继承了Shell进程的umask
掩码，所以最终的文件权限是0666&∼022=0644。



#### 1.4.1.5 open函数与fopen区别

注意**open函数与C标准I/O库的fopen函数有些细微的区别**：

- 以可写的方式fopen一个文件时，如果文件不存在会自动创建，而open一个文件时必须明确指定O_CREAT才会创建文件，否则文件不存在就出错返回。
- 以w或w+方式fopen一个文件时，如果文件已存在就截断为0字节，而open一个文件时必须明确指定O_TRUNC才会截断文件，否则直接在原来的数据上改写。



### 1.4.2 close函数

close函数关闭一个已打开的文件：

```c
#include <unistd.h>
int close(int fd);
```

**返回值：成功返回0，出错返回-1并设置errno**

#### 1.4.2.1 fd

**参数fd(file descriptor)是要关闭的文件描述符。需要说明的是，当一个进程终止时，内核对该进程所有尚未关闭的文件描述符调用close关闭，所以即使用户程序不调用close，在终止时内核也会自动关闭它打开的所有文件**



由**open返回的文件描述符一定是该进程尚未使用的最小描述符**。由于程序启动时自动打开文件描述符0、1、2，因此第一次调用open打开文件通常会返回描述符3，再调用open就会返回4。可以利用这一点在标准输入、标准输出或标准错误输出上打开一个新文件，实现重定向的功能。

例如，**首先调用close关闭文件描述符1，然后调用open打开一个常规文件，则一定会返回文件描述符1，这时候标准输出就不再是终端，而是一个常规文件了，再调用printf就不会打印到屏幕上，而是写到这个文件中了**



### 1.4.3 最大打开文件个数

**可以打开多少个文件 跟电脑内存有直接关系**

查看当前系统允许打开最大文件个数

```shell
cat /proc/sys/fs/file-max
```

 当前默认设置最大打开文件个数1024

```shell
ulimit -a
```

修改默认设置最大打开文件个数为4096

```shell
ulimit -n 4096
```



## 1.5 read/write函数

### 1.5.1 read

read函数从打开的设备或文件中读取数据。

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```

**返回值：成功返回读取的字节数，出错返回-1并设置errno，如果在调read之前已到达文件末尾，则这次read返回0**

**参数count是请求读取的字节数，读上来的数据保存在缓冲区buf中，同时文件的当前读写位置向后移**

注意这个读写位置和使用C标准I/O库时的读写位置有可能不同，这个读写位置是记在内核中的，而使用C标准I/O库时的读写位置是用户空间I/O缓冲区中的位置。比如用fgetc读一个字节，fgetc有可能从内核中预读1024个字节到I/O缓冲区中，再返回第一个字节，这时该文件在内核中记录的读写位置是1024，而在FILE结构体中记录的读写位置是1。

**注意返回值类型是ssize_t，表示有符号的size_t，这样既可以返回正的字节数、0（表示到达文件末尾）也可以返回负值-1（表示出错）**。

read函数返回时，返回值说明了buf中前多少个字节是刚读上来的。**有些情况下，实际读到的字节数（返回值）会小于请求读的字节数count**，例如：

- 读常规文件时，在读到count个字节之前已到达文件末尾。例如，距文件末尾还有30个字节而请求读100个字节，则read返回30，下次read将返回0。
- 从终端设备读，通常以行为单位，读到换行符就返回了。
- 从网络读，根据不同的传输层协议和内核缓存机制，返回值可能小于请求的字节数，后面socket编程部分会详细讲解

### 1.5.2 write

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);
```

**返回值：成功返回写入的字节数，出错返回-1并设置errno**

写常规文件时，write的返回值通常等于请求写的字节数count，而向终端设备或网络写则不一定。



## 1.6 阻塞和非阻塞

**明确一下阻塞（Block）这个概念。当进程调用一个阻塞的系统函数时，该进程被置于睡眠（Sleep）状态，这时内核调度其它进程运行，直到该进程等待的事件发生了（比如网络上接收到数据包，或者调用sleep指定的睡眠时间到了）它才有可能继续运行。**

**读常规文件是不会阻塞的，不管读多少字节，read一定会在有限的时间内返回。**

**从终端设备或网络读则不一定，如果从终端输入的数据没有换行符，调用read读终端设备就会阻塞，如果网络上没有接收到数据包，调用read从网络读就会阻塞，至于会阻塞多长时间也是不确定的，如果一直没有数据到达就一直阻塞在那里。**

**同样，写常规文件是不会阻塞的，而向终端设备或网络写则不一定。**

非阻塞I/O有一个缺点，如果所有设备都一直没有数据到达，调用者需要反复查询做无用功，如果阻塞在那里，操作系统可以调度别的进程执行，就不会做无用功了。在使用非阻塞I/O时，通常不会在一个while循环中一直不停地查询（这称为Tight Loop），而是每延迟等待一会儿来查询一下，以免做太多无用功，在延迟等待的时候可以调度其它进程执行。

```c
while(1) {
	非阻塞read(设备1);
	if(设备1有数据到达)
		处理数据;
	非阻塞read(设备2);
	if(设备2有数据到达)
		处理数据;
	...
	sleep(n);
}
```



### 1.6.1 demo

```c
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>

#define MSG_TRY "try again\n"
#define MSG_TIMEOUT "timeout\n"

int main(void) {
    char buf[10];
    int fd, n, i;
    fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
    if (fd < 0) {
        perror("open /dev/tty");
        exit(1);
    }
    for (i = 0; i < 5; i++) { //保证有数据到达时处理延迟较小
        n = read(fd, buf, 10);
        if (n >= 0)
            break;
        if (errno != EAGAIN) {
            perror("read /dev/tty");
            exit(1);
        }
        sleep(1);
        write(STDOUT_FILENO, MSG_TRY, strlen(MSG_TRY));
    }
    if (i == 5)//超时退出的逻辑
        write(STDOUT_FILENO, MSG_TIMEOUT, strlen(MSG_TIMEOUT));
    else
        write(STDOUT_FILENO, buf, n);
    close(fd);
    return 0;
}
```



## 1.7 lseek

每个打开的文件都记录着当前读写位置，打开文件时读写位置是0，表示文件开头，通常读写多少个字节就会将读写位置往后移多少个字节。但是有一个例外，如果以O_APPEND方式打开，每次写操作都会在文件末尾追加数据，然后将读写位置移到新的文件末尾。

**lseek和标准I/O库的fseek函数类似，可以移动当前读写位置（或者叫偏移量）。**

```c
#include <sys/types.h>
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

参数offset和whence的含义和fseek函数完全相同。只不过第一个参数换成了文件描述符。和fseek一样，偏移量允许超过文件末尾，这种情况下对该文件的下一次写操作将延长文件，中间空洞的部分读出来都是0。

## 1.8 fcntl

**可以用fcntl函数改变一个已打开的文件的属性，可以重新设置读、写、追加、非阻塞等标志（这些标志称为File Status Flag），而不必重新open文件。**

```c
#include <unistd.h>
#include <fcntl.h>
int fcntl(int fd, int cmd);
int fcntl(int fd, int cmd, long arg);
int fcntl(int fd, int cmd, struct flock *lock);
```



### 1.8.1 用fcntl改变File Status Flag

```c
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>

#define MSG_TRY "try again\n"

int main(void) {
    char buf[10];
    int n;
    int flags;
    //获取文件属性
    flags = fcntl(STDIN_FILENO, F_GETFL);
    //增加非阻塞属性
    flags |= O_NONBLOCK;
    if (fcntl(STDIN_FILENO, F_SETFL, flags) == -1) {
        perror("fcntl");
        exit(1);
    }
    tryagain:
    n = read(STDIN_FILENO, buf, 10);
    if (n < 0) {
        if (errno == EAGAIN) {
            sleep(1);
            write(STDOUT_FILENO, MSG_TRY, strlen(MSG_TRY));
            goto tryagain;
        }
        perror("read stdin");
        exit(1);
    }
    write(STDOUT_FILENO, buf, n);
    return 0;
}
```



















