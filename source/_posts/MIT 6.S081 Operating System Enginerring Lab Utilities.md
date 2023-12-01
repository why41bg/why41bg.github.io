---
title: MIT 6.S081 Operating System Enginerring Lab Utilities
date: 2023-10-10 17:48:36
tags:
- OS
- XV6
categories:
- MIT 6.S081
---

> This lab will familiarize you with xv6 and its system calls.
>
> 本实验旨在实现几个 unix 实用工具，帮助熟悉 XV6 的开发环境以及系统调用。
>
> 实验的具体要求在[这里](https://pdos.csail.mit.edu/6.828/2021/labs/util.html)。

# 修改测评程序

测评脚本grade-lab-util是python程序，需要进行一些小修改，将第一行修改为：

```python
#!/usr/bin python
```

在实现功能（例如sleep）之后，需要更新Makefile的UPROGS部分，如：

```bash
$U/_sleep\
```

另外测评命令修改为：

```bash
sudo python3 grade-lab-util sleep
```



# argc, argv是什么？
> 在编写系统调用之前，首先来看看如何给程序传递参数。

在 `Linux` 下C语言编程中的main函数，常常长成下面这个样子：

```c
#include<stdio.h>
int main(int argc, char* argv[])
{
    
}
```

这里的 `argc` 和 `argv` 是用来给应用程序传递参数的。`argc` 是传递给应用程序的参数个数，`argv` 是传递给应用程序的参数。例如：

```c
#include<stdio.h>
int main(int argc, char* argv[])
{
    for(int idx = 0; idx<argc; idx++){
        printf("argc:%d, argv[%d]:%s\n", idx, idx, argv[idx]);
    }
}
```

测试如下：

```bash
$ ./main
argc:0, argv[0]:./main
$ ./main -p
argc:0, argv[0]:./main
argc:1, argv[1]:-p
$ ./main -p 80
argc:0, argv[0]:./main
argc:1, argv[1]:-p
argc:2, argv[2]:80
```



# 如何解析命令行参数

> Linux是以交互式命令接口的方式工作的，那么如何该如何解析命令行参数呢？答案就是通过getopt()函数，它是用于解析命令行参数的工具，它可以帮助我们方便地获取命令行参数并进行相应的处理。

```c
#include <unistd.h>
int getopt(int argc, char * const argv[],
          const char *optstring);
extern char *optarg;
extern int optind, opterr, optopt;
```

- 该函数用来分析命令行参数：
  - `argc` 和 `argv` 是由 main() 传递的参数个数和内容
  - `optstring` 是选项字符串（选项字符串不做解释）
  - `optarg` 用来保存选项的参数
  - `optind` 记录下一个检索位置
  - `opterr` 是否将错误信息输出到 `stderr`，为0时表示不输出
  - `optopt` 保存的是无法识别的选项

示例如下：

```c
#include <stdio.h>
#include <unistd.h>
int main(int argc,char*argv[])
{
    int optch = 0;
	while((optch = getopt(argc, argv, "a:b:cd::e")) != -1)
	{
        // printf("optind: %d\n", optind); //下一个检索位置，即选项的参数位置
		switch (optch)
		{
        case 'a':
            printf("-a %s\n", optarg);
            break;
        case 'b':
            printf("-b %s\n", optarg);
            break;
        case 'C':
        case 'c':
            printf("-c %s\n", optarg);
            break;
        case 'd':
            printf("d:%s\n", optarg);
            break;
        case 'e':
            printf("-e %s\n", optarg);
            break;
        case '?':
        printf("Unknown option: %c\n",(char)optopt);    //表示不在选项字符串optstring中的选项
            break;
        default:
            break;
		}
	}
    // printf("opterr：%d\n",opterr);  //opterr表示是否将错误信息输出到stderr，为0时表示不输出
}
```

结果如下：

```bash
$ ./main -a test
-a test
$ ./main -b 
./main: option requires an argument -- 'b'
Unknown option: b
```



# sleep([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))😃

Lab1 的第一个实验是实现**sleep 系统调用**，这个比较简单，理解了上面的内容就行了。完整过评测代码如下：

```python
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
        int n;
        if(argc != 2){
                fprintf(2, "Please enter a number\n");
                exit(1);
        }else{
                n = atoi(argv[1]);
                sleep(n);
                exit(0);
        }
}
```

注意一下导入的包，user.h 为 `XV6` 提供的系统函数，types.h 为其提供的变量类型。

需要补充的知识点有：

1. 在 Unix 系统里面，默认情况下 `0` 代表 `stdin`，`1 `代表 `stdout`，`2 `代表 `stderr` 。这3个文件描述符在进程创建时就已经打开了的（从父进程复制过来的），可以直接使用
2. 第 11 行的 `atoi()` 函数，该函数用于将字符串类型转化为整型。在 `XV6` 系统中对于该函数的实现如下：

```c
int
atoi(const char *s)
{
  int n;
  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
```



# pingping([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))😃

Lab1 的第二个实验主要考察对 `fork系统调用` 和 `pipe` 的理解。该实验的流程如下：

- 父进程：父进程向子进程传递信息，等待子进程结束，读取子进程传递的消息
- 子进程：读取父进程传递的消息，传递给父进程消息

对于 `fork系统调用` 需要知道，`fork` 会拷贝当前进程的内存并创建一个新的进程，这里的内存包括指令和数据两个方面。在原始的进程，也就是父进程中，会返回一个大于0的整数，即新创建的进程的 PID。而在新创建的进程，也就是子进程中，会返回0。因此，`fork` 返回的值可以用来区分当前进程是父进程还是子进程，这是关键。

对于 `pipe` ，即管道，又叫做无名管道。以下几点特别重要：

1. 半双工，数据在同一时刻只能在一个方向流动；
2. 管道传送的数据不是无格式，必须事先约定好；
3. 管道不是普通的文件，不属于某个文件系统，只存在于内存中；
4. 管道只能在具有公共祖先的进程之间使用。

创建一个管道的示例如下：

```c
#include<unistd.h>
int pipe(int fds[2]);
```

pipe函数定义中的 fds 参数是一个大小为2的数组类型指针。通过pipe函数创建的这两个文件描述符 fds[0] 和 fds[1] 分别构成管道的两端，往 fds[1] 写入的数据可以从 fds[0] 读出，并且 fds[1] 一端只能进行写操作，fds[0] 一端只能进行读操作，不能反过来使用。

对于 **read系统调用**，一共接受3个参数：

1. 第一个参数是文件描述符，指向一个之前打开的文件。Shell会确保默认情况下，当一个程序启动时，文件描述符0连接到console的输入，文件描述符1连接到了console的输出。这里的0，1文件描述符是非常普遍的Unix风格，许多的Unix系统都会从文件描述符0读取数据，然后向文件描述符1写入数据
2. 第二个参数是指向某段内存的指针，程序可以通过指针对应的地址读取内存中的数据。read系统调用将把读到的数据写入该指针指向的内存空间
3. 第三个参数是代码想读取的最大长度

返回值可能是读到的字节数，但是如果到达了文件的结尾没有更多的内容了，read会返回0。**通常来说，系统调用通常是通过返回-1来表示错误，了解这点很关键**。

对于 **write系统调用**，也接受3个参数，每个参数对应的逻辑和 **read系统调用** 区别不大。

关于 read系统调用 和 write系统调用 还有一个值得注意的点就是：**它们并不关心读写的数据格式，它们就是单纯的读写，而copy程序会按照8bit的字节流处理数据，你怎么解析它们，完全是用应用程序决定的。**

了解了上述这些之后，该实验也就很好解决了。完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char* argv[]){
        int fds[2];
        char buf[2];
        char *msg = "o";
        pipe(fds);

        int pid = fork();
        if(pid == 0){
                if(read(fds[0], buf, 1) != 1){
                        fprintf(2, "Can't read from parent\n");
                        exit(1);
                }
                close(fds[0]);
                printf("%d: received ping\n", getpid());

                if(write(fds[1], msg, 1) != 1){
                        fprintf(2, "Can't write to parent\n");
                        exit(1);
                }
                close(fds[1]);
                exit(0);

        }else{
                if(write(fds[1], msg, 1) != 1){
                        fprintf(2, "Can't write to child\n");
                        exit(1);
                }
                close(fds[1]);
                wait(0);

                if(read(fds[0], buf, 1) != 1){
                        fprintf(2, "Can't read from child\n");
                        exit(1);
                }
                close(fds[0]);
                printf("%d: received pong\n", getpid());
                exit(0);
        }

}
```



# find([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))😃

该实验主要仿照 **ls.c** 的实现思路完成，额外的注意点如下：

1. 当前路径为文件，直接检查是否是要查找的文件名
2. 当前路径为文件夹，对该目录下的所有文件递归，**\. 和 .. 除外**

这里还需要用的 **open系统调用**，一共接受2个参数：

1. 第一个参数是想要打开的文件路径
2. 第二个参数是一些标志位，用来告诉open系统调用在内核中的实现

open系统调用会返回一个新分配的文件描述符。

那么文件描述符到底是什么呢？

当我们打开文件时，操作系统在内存中要创建相应的数据结构来描述目标文件。于是就有了file结构体。表⽰⼀个已经打开的文件对象。而进程执行open系统调⽤，所以必须让进程和文件关联起来。每个进程都有⼀个指针*files, 指向⼀张表files_struct,该表最重要的部分就是包涵一个指针数组，每个元素都是一个指向打开文件的指针！所以，本质上，文件描述符就是该数组的下标。所以，只要拿着文件描述符，就可以找到对应的文件。

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void
compare(char *path, char *name)
{
    int fp = 0;
    int cp = 0;
    while(path[fp] != 0){
        cp = 0;
        int tp = fp;
        while(name[cp] != 0){
            if(path[tp] != name[cp]) break;
            cp++;
            tp++;
        }
        if(name[cp] == 0){
            printf("%s\n", path);
            return;
        }
        fp++;
    }
}

void
find(char *path, char *name)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_FILE:
            compare(path, name);
            break;
        
        case T_DIR:
            // Checks if the total length of the path and the directory entry name exceeds the size of the buffer buf
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0) continue;
                if(de.name[0] == '.' && de.name[1] == 0) continue;
                if(de.name[0] == '.' && de.name[1] == '.' && de.name[2] == 0) continue;
                memmove(p, de.name, DIRSIZ);
                // Set EOF
                p[DIRSIZ] = 0;
                if(stat(buf, &st) < 0){
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }
                find(buf, name);
            }
            break;
    }
    close(fd);
}

int
main(int argc, char *argv[])
{
    if(argc<3){
        fprintf(2, "Usage: find [path] [filename]\n");
        exit(-1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```



# primes([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))😃

管道筛素数的过程类似于一个递归调用的过程：

1. primes() 函数首先会创建一个自己的管道，接着从管道中读出一个数A，这里有一个注意点就是该数一定是素数（可参考埃氏筛法）。然后继续读取管道中剩下的数B，判断B是否是A的倍数，如果不是就写入管道；
2. primes() 函数还会创建一个子进程用于递归处理自己的管道，从管道中拿出第一个数（一定是素数），然后接着读取管道并作比较。

特别需要注意的点就是：一定要关闭不需要的文件描述符，因为文件描述符是有限的。

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void
prime(int rd)
{
    int num, cnum;
    if(read(rd, &num, 4) != 4){
        fprintf(2, "Cann't read from pipe\n");
        exit(1);
    }
    printf("prime %d\n", num);

    int cfds[2];
    pipe(cfds);
    if(fork() == 0){
        close(cfds[1]);
        prime(cfds[0]);
        close(cfds[0]);
    }else{
        close(cfds[0]);
        while(read(rd, &cnum, 4)){
            if(cnum % num != 0){
                if(write(cfds[1], &cnum, 4) != 4){
                    fprintf(2, "Cann't write to pipe\n");
                    exit(1);
                }
            }
        }
        close(rd);
        close(cfds[1]);
        wait(0);
    }
    exit(0);
}

int main(int argc, char *argv[])
{
    if(argc != 1){
        fprintf(2, "Usage: primes\n");
        exit(1);
    }

    int fds[2];
    pipe(fds);

    int pid = fork();
    if(pid != 0){
        close(fds[0]);
        for(int i = 2; i <= 35; i++){
            if(write(fds[1], &i, 4) != 4){
                fprintf(2, "Cann't write to pipe\n");
                exit(1);
            }
        }
        close(fds[1]);
        wait(0);
    }else{
        close(fds[1]);
        prime(fds[0]);
        close(fds[0]);
    }
    exit(0);
}
```



# xargs([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))😃

首先介绍一下什么是**xargs**。简单来说，**xargs 命令** 是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具。它擅长将标准输入数据转换成命令行参数，xargs 能够处理管道或者 stdin 并将其转换成特定命令的命令参数。xargs 也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。xargs 的默认命令是 echo，空格是默认定界符。这意味着通过管道传递给 xargs 的输入将会包含换行和空白，不过通过 xargs 的处理，换行和空白将被空格取代。xargs 是构建单行命令的重要组件之一。

了解了这些之后，就清楚了任务是什么：从stdin中获取到参数，拼接到xargs命令后面。如果stdin中的输出是多行，则需要拼接多行命令执行。

程序代码具体流程如下：

1. 将xargs传入的命令单独保存，将xargs传入的参数单独保存到一个数组中；
2. 将stdin中获取到的数据根据换行符"\n"，拼接到xargs传入参数的数组后面；
3. 对每行参数依据空格进行划分；
4. 调用一个 **fork()系统调用和exec()系统调用** 执行命令。

**这里有个常用的写法要注意，先调用fork，再在子进程中调用exec。尽管有些浪费，但是优化过程后面再提。**

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"
#define MAXLEN 100

int main(int argc, char *argv[])
{
    if(argc < 2){
        fprintf(2, "Usage: xargs [command]\n");
        exit(1);
    }
    // Get the command
    char *cmd = argv[1];
    
    char nargv[MAXARG][MAXLEN];
    char *pnargv[MAXARG];
    char buf;
    
    // Loop lines
    while(1){
        memset(nargv, 0, MAXARG*MAXLEN);
        for(int i = 1; i < argc; i++) strcpy(nargv[i-1], argv[i]);
        int cargc = argc - 1;
        int offset = 0;
        int is_read = 0;
        // Get all params of one line
        while((is_read = read(0, &buf, 1)) > 0){
            if(buf == ' '){
                cargc++;
                offset = 0;
                continue;
            }
            if(buf == '\n'){
                break;
            }
            if(offset == MAXLEN){
                fprintf(2, "xargs: parameter too long\n");
                exit(1);
            }
            if(cargc == MAXARG){
                fprintf(2, "xargs: too many arguments\n");
                exit(1);
            }
            nargv[cargc][offset++] = buf;
        }
        if(is_read <= 0) break;
        for(int i = 0; i <= argc; i++) pnargv[i] = nargv[i];
        if(fork() == 0){
            exec(cmd, pnargv);
            exit(1);
        }
        wait(0);
    }
    exit(0);
}
```

