---
title: 对xv6系统shell的分析
date: 2023-10-26 23:21:20
tags:
- OS
- XV6
---
> xv6源码文档中关于shell的实现代码足足有11页之多，遂写下此文章来记录学习xv6过程中，中shell实现方法的分析，从而为学习其他更加复杂的操作系统建立一个基础框架。

shell 是一个最基础的用户程序，它的工作原理为主循环通过循环读取命令行的输入，然后使用 `fork` 系统调用生成一个 shell 进程的子进程。子进程将调用 `exec` 执行用户命令，而父进程调用 `wait` 等待子进程完成。这也就对应了我们在命令行中输入一条命令后，命令行会不断打印命令执行过程中的输出，这个过程其实就是 shell 子进行调用 `exec` 执行用户命令的过程，命令执行完毕后，父 shell 进程结束 `wait` 等待，此时我们才能继续输入下一条命令。

# 文件描述符

shell 启动的时候，一般只会保留标准输入、标准输出、标准错误输出三个文件描述符。shell 程序在 `main` （xv6代码文档第8000行）一开始就会检查是否存在其它的文件描述符，如果存在就要关闭这些额外的文件描述符。检查并处理文件描述符的代码如下：

```c
int
main(void)
{
    static char buf[100];
    int fd;
    
    // Assumes three file descriptors open.
    while((fd = open("console", 0_RDWR)) >= 0){
        if(fd >= 3)}{
        	close(fd);
        	break;
    	}
    }
	
	// ......
}
```

这段代码不断尝试打开名为 "console" 的文件，如果成功打开并且获得的文件描述符大于2，即不属于标准输入、标准输出和标准错误输出，那么就关闭该文件并跳出循环。通过这段代码就能保证 shell 启动时只保留了标准输入、标准输出和标准错误输出三个文件描述符，不存在任何其他不必要的文件描述符。

# 获取命令

在处理了文件描述符之后，`main` 程序就不断通过 `getcmd` 函数（7983）读取用户输入的命令。`getcmd` 函数的定义如下：

```c
int
getcmd(char *buf, int nbuf)
{
    printf(2, "$ ");
    memset(buf, 0, nbuf);
    gets(buf, nbuf);
    if(buf[0] == 0) // EOF
        return -1;
    return 0;
}
```

`getcmd` 函数接受一个字符缓冲区，在程序一开始打印一个"$ "，正如我们在终端中输入命令时常常见到的那样，这表示命令提示符，用来提示用户输入命令。然后便使用 `memset` 函数将缓冲区前 nbuf 个字节设置为0，保证缓冲区为空，以便存储用户输入的命令。`gets` 函数用于从标准输入中读取用户输入的命令，将其存储到缓冲区中，nbuf 参数表示最大读取的字符数，防止缓冲区溢出。接下来是一个 if 条件语句，判断缓冲区中的第一个字符是否为0。在某些操作系统中，用户可以通过键入 Ctrl+D 来表示输入的结束，此时缓冲区中的第一个字符会被设置为0。如果检测到输入的结束，函数返回 -1 表示终止此次输入。

# 执行命令

`main` 函数通过不断执行 `getcmd` 函数获取用户输入，`main` 函数中的有关代码如下：

```c
int
main()
{
    // ......
    
    // Read and run input commands
    while(getcmd(buf, sizeof(buf)) >= 0){
        if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
            // Clumsy but will have to do for now.
            // Chdir has no effect on the parent if run in the chile.
            buf[strlen(buf) - 1] = 0;  // chop \n
            if(chdir(buf+3)<0)
                printf(2, "cannot cd %s\n", buf+3);
            continue;
        }
        if(fork1() == 0)
            runcmd(parsecmd(buf));
        wait();
    }
    exit();
}
```

buf 用来存储用户输入的命令，如果成功将用户输入的命令写入到了 buf 中，则进入循环体中。第一个 if 条件语句用来判定用户输入的命令是否是 `cd`。“xv6 关于文件系统的操作都被实现为用户程序，诸如 `mkdir`，`ln`，`rm` 等等。这种设计允许任何人都可以通过用户命令拓展 shell 。有一个例外，那就是 `cd`”。`cd` 必须在 shell 中实现，因为如果 `cd` 作为一个普通命令执行，那么 shell 会 `fork` 一个子进程来运行 `cd`，这只会改变子进程的工作目录，而父进程的工作目录保持原样。因此，这里的 if 条件语句就是在 shell 中实现 `cd` 的入口。

第二个 if 条件语句为进入子进程的入口，执行除 `cd` 以外的用户程序，`fork1` 函数创建一个子进程并返回进程号。子进程中首先通过 `parsecmd` 函数（8217）解析用户命令，该函数使用了 `struct cmd` 结构体（7865），该结构体有一个 int 变量 cmd 表示用户命令（7856）
