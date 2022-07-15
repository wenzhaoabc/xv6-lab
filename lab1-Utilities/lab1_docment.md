# sleep
>Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.
## **一些提示**
>- 阅读xv6参考书籍第一章
>- 查看`user/`下的文件如`echo.c`,`grep.c`,`rm.c`,了解如何将获得的命令行参数传递给程序
>- 在用户未输入参数时，给出错误提示
>- 命令行参数以字符串的形式传递，通过函数`atoi`(`user/ulib.c`)转为整数
>- 使用系统调用`sleep`
>- 查看内核代码`kernel/sysproc.c`了解系统调用`sleep`的实现(`sys_sleep`函数)，查看`user/user.h`了解可从用户程序调用的`sleep`定义，查看`user/usys.S`了解从用户代码跳转到内核代码的汇编程序
>- 在`main`函数中调用`exit()`来终止你的程序
>- 添加`sleep`程序到Makefile文件的`UPROGS`中，输入`make qemu`运行
## 相关知识
### int main(int argc,char *argv[ ])中的两个参数
`argc`是命令行中输入的总的参数个数

`argv[]`是字符串形式的各个参数，默认第0个参数是程序的全名，以后的参数为命令行用户输入的参数

例如：

    /* test.c */
    #include<stdio.h>
    int main(int argc, char* argv[])
    {
        for(int i = 0;i < argc;i++)
          printf("%s  ",argv[i]);
        return 0;
    }

在命令行运行输出：

![image.png](https://s2.loli.net/2022/07/12/s95tBc2SFQ6HUMC.png)

据此可以将命令行参数传递给程序

### 将命令行输入的字符串形式的参数转为整数

`user/ulib.c`文件中包含`atoi`函数

    int atoi(const char *s)
    {
        int n;
        n = 0;
        while('0' <= *s && *s <= '9')
            n = n*10 + *s++ - '0';
        return n;
    }

### 系统调用`sleep`

`user.h`中有`sleep`函数，在`usys.S`中有调用`sleep`的汇编代码

    .global sleep
    sleep:
     li a7, SYS_sleep
     ecall
     ret

据此`sleep.c`如下：

    #include "kernel/types.h"
    #include "kernel/stat.h"
    #include "user/user.h"

    int main(int argc,char *argv[])
    {
        if(argc < 2){
        fprintf(2,"sleep:time outs");
        exit(1);
        }

        int time = atoi(argv[1]);
        sleep(time);
        exit(0);
    }

# pingpong

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "\<pid>: received ping", where \<pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "\<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

## 一些提示

>- 使用系统调用`pipe`创建管道
>- 使用系统调用`fork`创建子进程
>- 使用`read`向管道写入数据，使用`write`从管道中读取数据
>- 使用`getid`获取调用进程的PID
>- 在Makefile的`UPROGS`中添加程序
>- 在xv6上用户程序可以使用一些已有的库函数，在`user/user.h`中查看函数声明，除系统调用以外的函数源码可在`user/ulib.c`,`user/ulib.c`,`user/umalloc.c`中查看

## 相关知识

**文件描述符**

文件描述符表示了一个可以被内核读写的对象，进程涉及到的I/O对象都组织在进程的一张表中，文件描述符就是这张表的下标，在xv6中，0代表标准输入，1为标准输出，2为标准错误输出，`write(fd,buf,n)`/`read(fd,buf,n)`从指定的文件描述符中读写n个字节

**管道**

管道是一段内存缓冲区，向外暴露读/写文件描述符，管道提供了进程间交互的一种方式，在xv6中利用系统调用`pipe`可以设置一段管道

据此pingpong.c设置如下：

    #include "kernel/types.h"
    #include "kernel/stat.h"
    #include "user/user.h"

    int main(int argc, char *argv[])
    {
        int pfc[2]; // pipe for father to child
        int pcf[2]; // pipe for child to father
        char byte = 'M';
        char buf[2];
        if (pipe(pfc) < 0)
        {
            fprintf(2, "pingpong:create pipe pfc failed\n");
            exit(1);
        }
        if (pipe(pcf) < 0)
        {
            fprintf(2, "pingpong:create pipe pcf failed\n");
            exit(1);
        }

        if (fork() == 0)
        {
            write(pcf[1], &byte, 1);
            read(pfc[0], buf, 1);
            if (buf[0] == byte)
                printf("%d:received ping\n", getpid());
            else
                printf("receive from father error\n");
        }
        else
        {
            write(pfc[1], &byte, 1);
            read(pcf[0], buf, 1);
            if (buf[0] == byte)
                printf("%d:received pong\n", getpid());
            else
                printf("receive from child error\n");
        }

        close(pfc[0]);
        close(pfc[1]);
        close(pcf[0]);
        close(pcf[1]);
        wait(0);
        exit(0);
    }

# primes

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down this page and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

使用管道设置一个筛选素数的并发版本，使用系统调用`pipe`/`fork`设置线性管道，第一个进程向管道输入2-35，对于每一个素数，创建一个进程从左边管道读取并向右边管道写入

## 一些提示

>- 注意及时关闭程序不再用到的文件描述符，否则程序可能在到达35之前用尽xv6所提供的资源
>- 当地一个进程到达35后，应该等待直到整个线性管道终止，因此主进程应该在所有的输出已经呈现，所有其它进程终止以后退出
>- `read`在管道写入端关闭时返回0
>- 直接在管道中写入4字节的`int`是非常简单的，不要使用格式化的ASCII
>- 仅在管道中创建需要的进程
>- 把程序添加到Makefile的`UPROGS`中

用筛选法求素数的基本思想是在2-N的整数中，从2开始，筛选掉2的倍数，再筛选掉3的倍数 ··· 直至$\sqrt N$，则剩下的数为素数，从进程和管道出发大体如下：

![image.png](https://s2.loli.net/2022/07/14/vmTAqt8Ef7W9wIN.png)

每个进程各自有不同的用户地址空间,任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核,在内核中开辟一块缓冲区,进程A把数据从用户空间拷到内核缓冲区,进程B再从内核缓冲区把数据读走,内核提供的这种机制称为进程间通信。

管道是进程间通信的一种方式，其内部提供同步的机制，可保证数据访问的一致性，当一个进程正在对管道执行读/写操作时，其它进程必须等待，一个进程将一定量的数据写入，然后就去睡眠等待，直到读进程将数据取走，再去唤醒，读进程与之类似。可以通过系统调用`pipe()`创建管道

    int fd[2];
    pipe(fd);

创建成功`pipe()`返回0，失败返回1，`fd`返回两个文件描述符，`fd[0]`指向管道的读端，即从管道中读取数据的端口，`fd[1]`是管道的写入端，即向管道写入数据的端口，`fd[1]`的输入是`fd[0]`的输出。

父进程创建管道，得到指向管道两端的文件描述符，父进程创建子进程，子进程得到同样的两个文件描述符，由于管道只支持单向通信，当父进程关闭`fd[0]`，向`fd[1]`写入数据(`write(fd[1],,)`)，子进程关闭`fd[1]`,从`fd[0]`读出数据(`read(fd[0],,)`)，子进程便可接收到来自父进程的信息。

读写管道的特点

    ① 读管道：	1. 管道中有数据，read返回实际读到的字节数。
                2. 管道中无数据：
                    (1) 管道写端被全部关闭，read返回0 (好像读到文件结尾)。这个也是我们判断对方断开连接的方式。
                    (2) 写端没有全部被关闭，read阻塞等待(不久的将来可能有数据递达，此时会让出cpu)
    ② 写管道：	1. 管道读端全部被关闭， 进程异常终止(也可使用捕捉SIGPIPE信号，使进程不终止)
                2. 管道读端没有全部关闭： 
                    (1) 管道已满，write阻塞。
                    (2) 管道未满，write将数据写入，并返回实际写入的字节数。

资料参考自[https://blog.csdn.net/weixin_44517656/article/details/112412375]

# find

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`.

## 一些提示

>- 查看`user/ls.c`了解如何读取目录
>- 使用递归让查找下降到子目录
>- 不要递归进入文件夹'.'和'..'
>- 通过qemu可以更改文件系统，运行`make clean`清理无关文件
>- 需要使用到C字符串，可以参阅C语言书籍K&R，例如5.5节
>- 比较字符串不能像python那样使用`==`，应该用`strcmp()`
>- 把程序添加到Makefile的`UPROGS`中

在xv6的文件系统中，结构体`stat`包含有关文件物理存储位置，大小，引用数等信息，在结构体`dirent`包含文件名及其长度，利用系统调用`open()`可以根据路径名获取指向文件的文件描述符，`fstat()`可以根据文件描述符获取`stat`结构体，利用`read()`系统调用可以根据文件描述符获取`dirent`结构体。

    /* stat.h */
    #define T_DIR     1   // Directory
    #define T_FILE    2   // File
    #define T_DEVICE  3   // Device

    struct stat {
    int dev;     // File system's disk device
    uint ino;    // Inode number
    short type;  // Type of file
    short nlink; // Number of links to file
    uint64 size; // Size of file in bytes
    };

    /* dirent.h */
    // Directory is a file containing a sequence of dirent structures.
    #define DIRSIZ 14

    struct dirent {
    ushort inum;
    char name[DIRSIZ];
    };

    int fd;
    struct stat st;
    struct dirent de;
    fd = open (path,0);
    fstat(fd,&st);
    read(fd,&de,sizeof(de));

采用类似于`ls`的实现方式，设置了递归函数`int find(char *path,const chat filename)`，函数返回在path路径下查找到的文件数量，根据`stat`的`type`属性，对文件和目录分别处理，文件直接比对是否为查找目标，目录则生成再一层的路径，打开进入，递归`find()`函数，同样设置了`fmtname`函数，根据路径名获取末尾的文件名。具体实现见[`find.c`](./find.c)

