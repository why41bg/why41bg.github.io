---
title: MIT 6.S081 Operating System Enginerring Lab1
date: 2023-10-10 17:48:36
tags:
- OS
- XV6
---

> 本 Lab1 的具体实验要求在[这里](https://pdos.csail.mit.edu/6.828/2021/labs/util.html)。

# 修改测评程序

测评脚本grade-lab-util是python程序，需要进行一些小修改，将第一行修改为：

```python
#!/usr/bin python
```

在实现功能（例如sleep）之后，需要更新Makefile的UPROGS部分，如：

```
$U/_sleep\
```

另外测评命令修改为：

```ba
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

```
# ./main
argc:0, argv[0]:./main
# ./main -p
argc:0, argv[0]:./main
argc:1, argv[1]:-p
# ./main -p 80
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

```
# ./main -a test
-a test
# ./main -b 
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

需要补充的知识点可能就是第 11 行的 `atoi()` 函数了，该函数用于将字符串类型转化为整型。在 `XV6` 系统中对于该函数的实现如下：

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

