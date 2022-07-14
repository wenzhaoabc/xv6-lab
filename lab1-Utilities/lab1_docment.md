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

