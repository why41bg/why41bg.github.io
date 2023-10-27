---
title: 对xv6系统shell的分析
date: 2023-10-26 23:21:20
tags:
- OS
- XV6
---
> xv6源码文档中关于 shell 的实现代码足足有11页之多，遂写下此文章来记录学习xv6过程中，对 shell 实现方法的分析，从而为学习其他更加复杂的操作系统建立一个基础框架。

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

第二个 if 条件语句（8024）为进入子进程的入口，执行除 `cd` 以外的用户程序，`fork1` 函数创建一个子进程并返回进程号。在子进程中，首先通过 `parsecmd` 函数（8217）解析用户命令，该函数返回一个指向 `struct cmd` 结构体的指针，该结构体记录了一个 int 类型的 cmd 变量表示用户命令（7856）。然后通过 `runcmd` 函数执行具体的用户命令。

# 解析命令

xv6中负责解析用户命令的代码部分从8150开始，下面按顺序依次分析一下这部分代码，中间会穿插一些其他部分的函数，不要担心，都很简单。从8150开始，首先定义了两个字符数组，whitespace 数组包含了表示空白字符的字符，symbols 包含了一些特殊符号。定义如下：

```c
char whitespace[] = "\t\r\n\v";
char symbols[] = "<|>&;()";
```

## gettoken

首先是一个解析命令获得其中第一个标记的函数 `gettoken`，该函数实现如下：

```c
int
gettoken(char **ps, char *es, char **q, char **eq)
{
    char *s;
    int ret;
    
    s = *ps;
    while(s < es && strchr(whitespace, *s))
        s++;
    if(q)
        *q = s;
    ret = *s;
    switch(*s){
    case 0:
        break;
    case '|':
    case '(':
    case ')':
    case ';':
    case '&':
    case '<':
        s++;
        break;
    case '>':
        s++;
        if(*s == '>'){
            ret = '+';
            s++;
        }
        break;
    default;
        ret = 'a';
        while(s < es && !strchr(whitespace, *s) && !strchr(symbols, *s))
            s++;
        break;
    }
    if(eq)
        *eq = s;
    
    while(s < es && strchr(whitespace, *s))
        s++;
    *ps = s;
    return ret;
}
```

`gettoken` 函数接受4个参数，指向字符串指针的指针 `ps`，指向字符串结束位置指针 `es`，以及两个指向指针的指针 `q` 和 `eq`。`char *s` 用于遍历字符串，`int ret` 用于存储返回的标记。第一个 while 循环将遍历字符串，直到 `char *s` 指向字符串的末尾或者遇到非空白字符，如果遇到非空白字符，则将 `s` 指向空白字符的后一位。`if(q) *q = s;` 这段代码判断 `q` 是否为空，如果是则将 `q` 指向的指针设置为 `s`，即保存当前标记的起始位置。然后将当前当前字符作为标记，存储在 `ret` 变量中。接下来通过 switch 对当前字符进行判断：

1. 如果当前字符为字符串结束位置，则跳出 switch。
2. 如果当前字符属于除 > 以外的特殊字符，则将 `s` 指向下一位，然后跳出 switch。
3. 如果当前字符为 >，则将 `s` 指向下一位，并接着判定是不是也是 >，若是，则将标记设置为 + 号，然后跳出 switch。
4. 如果不满足上述条件则进入默认分支：将标记设置为 a，然后再次遍历字符串，直到 `s` 指向字符串的末尾或者遇到空白字符或特殊字符。

接下来判断 `eq` 是否为空指针，如果为空，则将 `eq` 指向 `s`，即保存当前标记的结束位置。再次遍历字符串，直到 `s` 指向字符串的末尾或者遇到非空白字符，如果遇到空白字符，将 `s` 指向空白字符的下一位，然后更新指针 `ps` 的值，使其指向当前位置，以便下次调用函数时从正确的位置开始解析。最后将 `ret` 标记返回。

## peek

下面是一个 `peek` 函数，该函数用来查看第一个标记（如果有）是否属于给定的一组标记字符，该函数定义如下：

```c
int
peek(char **ps, char *es, char *toks)
{
    char *s;
    
    s = *ps;
    while(s < es && strchr(whitespace, *s))
        s++;
    *ps = s;
    return *s && strchr(toks, *s);
}
```

该函数接受3个参数，第3个参数是一组标记字符的字符串指针 `char *toks`。该函数的代码逻辑和 `gettoken` 函数大致类似。目的在于获得命令字符串中的第一个匹配到的标记（如果存在），然后判定是否属于给定的一组标记字符，如果同时满足上述两个条件，则返回1（True），否则返回0（False）。

## execcmd，pipecmd，listcmd，backcmd

在继续分析后续的代码之前，要先分析一下 Constructors 部分（8050）中的4个函数：`execcmd`，`pipecmd`，`listcmd`，`backcmd`。以 `execcmd` 函数举例说明，其函数定义如下：

```c
struct cmd {
	int type;
}

struct execcmd {
	int type;
	char *argv[MAXARGS];
	char *eargv[MAXARGS];
}

struct cmd*
execcmd(void)
{
    struct execcmd *cmd;
    
    cmd = malloc(sizeof(*cmd));
    memset(cmd, 0, sizeof(*cmd));
    cmd->type = EXEC;
    return (struct cmd*) cmd;
}
```

`struct cmd` 定义了命令对象的基本结构，`struct execcmd` 定义了一个特定类型的命令对象。

该函数用于创建一个 `struct execcmd` 对象，其中关键的部分在于 `cmd->type = EXEC`，表示这个命令对象的指令类型为 EXEC，返回时将类型强制转换为 `struct cmd*`，这样做可以方便地创建和处理不同类型的命令对象。其余函数的逻辑和作用大致都是相同的。

## parseline，parsepipe，parseredirs，parseblock，parseexec

接下来回到之前的介绍顺序中来，下面一个函数是在 shell 主函数中调用的 `parsecmd` 函数，但是这里先跳过它，介绍它下面的其他几个函数。在它之后有 `parseline`，`parsepipe`，`parseredirs`，`parseblock`，`parseexec` 5个函数，接下来按照其调用栈的顺序依次介绍。

### parseline

### parsepipe

### parseredirs

### parseblock

### parseexec
