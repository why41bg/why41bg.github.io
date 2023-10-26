---
title: 对xv6系统shell的分析
date: 2023-10-26 23:21:20
tags:
- OS
- XV6
---
> xv6源码文档中关于shell的实现代码足足有11页之多，遂写下此文章来记录学习xv6过程中，中shell实现方法的分析，从而为学习其他更加复杂的操作系统建立一个基础框架。

shell 是一个最基础的用户程序，它的工作原理为主循环通过循环读取命令行的输入，然后使用 `fork` 系统调用生成一个 shell 进程的子进程。子进程将调用 `exec` 执行用户命令，而父进程调用 `wait` 等待子进程完成。这也就对应了我们在命令行中输入一条命令后，命令行会不断打印命令执行过程中的输出，这个过程其实就是 shell 子进行调用 `exec` 执行用户命令的过程，命令执行完毕后，父 shell 进程结束 `wait` 等待，此时我们才能继续输入下一条命令。

从 `main` （xv6代码文档第8000行）代码入手，先来看看 `main` 代码块中的第一个部分：

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

