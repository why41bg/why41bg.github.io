---
title: MIT 6.S081 Operating System Enginerring Lab2
date: 2023-10-22 11:56:28
tags:
- OS
- XV6
categories:
- MIT 6.S081
---

> In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.
>
> 本实验主要就是自己实现几个简单的系统调用并添加到XV6的内核当中。
>
> 实验的具体要求在[这里](https://pdos.csail.mit.edu/6.828/2021/labs/syscall.html)。

# System call tracing ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))😔

此系统调用的许多函数都已经给我们写好了，我们需要做的仅仅只是完善系统调用的流程。因此，了解如何从用户空间到内核空间，从而执行系统调用的流程至关重要。系统调用的全流程如下：

1. 在用户空间，在 `user/user.h` 中声明一个跳板函数 `trace()`。
2. 在用户空间，在 `user/usys.S` 中，跳板函数 `trace()` 使用 **ecall** 指令，把控制权交给 CPU，进入到内核态。
3. 在内核空间，在 `kernel/syscall.c` 中，到达内核态统一系统调用处理函数 `syscall()`，这是所有系统调用的统一入口。由该函数分析用户程序想要执行的系统调用后，调用内核中对应的内核程序。
4. 在内核空间，在 `kernel/syscall.c` 中，`syscall()` 函数查找 `syscalls[]` 表，找到对应的内核程序。系统调用编号和内核程序指针的映射关系定义在 `kernel/syscall.h` 中。
5. 在内核空间，在 `sysproc.c` 中，到达指定的内核程序。

**上述系统调用执行的流程反映了用户态和内核态的强隔离性。**

根据上述系统调用的完成，一步一步完善从用户空间到内核空间的代码：

1. 首先在 `user/user.h` 中声明 trace() 函数，该函数处于用户空间，与内核空间的 trace() 函数相对应。
   ```c
   // ……
   char* sbrk(int);
   int sleep(int);
   int uptime(void);
   int trace(int);  // trace syscall
   ```

2. 在 `user/usys.S` 中，进入到内核的 ecall 内核统一入口处。
   ```perl
   entry("sbrk");
   entry("sleep");
   entry("uptime");
   entry("trace");
   ```

3. 接下来就进入到了内核空间，在 `kernel/syscall.c` 中分析用户程序想要执行的内核程序。在该文件中要声明内核程序，以及内核程序对应的系统调用编号。
   ```c
   // ……
   extern uint64 sys_wait(void);
   extern uint64 sys_write(void);
   extern uint64 sys_uptime(void);
   extern uint64 sys_trace(void);  // 内核程序声明
   
   static uint64 (*syscalls[])(void) = {
   // ……
   [SYS_unlink]  sys_unlink,
   [SYS_link]    sys_link,
   [SYS_mkdir]   sys_mkdir,
   [SYS_close]   sys_close,
   [SYS_trace]   sys_trace,  // 内核程序与编号的映射
   };
   ```

   还需要添加一个数组用来保存系统调用的名字，以便打印输出。
   ```c
   char* syscalls_name[24] = {"", "fork", "exit", "wait", "pipe", "read", "kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep", "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close", "trace"};
   ```

   修改 syscall() 函数，当系统调用号与mask匹配时打印系统调用的名字。

   ```c
   void
   syscall(void)
   {
     int num;
     struct proc *p = myproc();
   
     num = p->trapframe->a7;
     if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
       p->trapframe->a0 = syscalls[num]();
   
       // 当系统调用号和mask匹配时打印
       if((1 << num) & p->syscall_trace){
         printf("%d: syscall %s -> %d\n", 
                 p->pid, syscalls_name[num], p->trapframe->a0);
       }
     } else {
       printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
       p->trapframe->a0 = -1;
     }
   }
   ```

4. 在 `kernel/syscall.c` 的 `syscalls[]` 表中找到了要执行的内核程序的编号后，到达`sysproc.c` 中的指定的内核程序处。因此，需要在 `sysproc.c` 定义内核程序。
   ```c
   uint64
   sys_trace(void)
   {
     int mask;
   
     if(argint(0, &mask) < 0)
         return -1;
       
     myproc()->syscall_trace = mask;
     return 0;
   }
   ```

5. 上述流程仅仅只是完成了系统调用的整个流程，还需要修改内核中的一些结构体，以记录一些必要的信息。首先是 `kernel/proc.h` 中的 **proc** 结构体，该结构体存储了进程的一些必要信息，需要在该结构体中添加一个掩码。
   ```c
   // Per-process state
   struct proc {
     struct spinlock lock;
   
     // p->lock must be held when using these:
     enum procstate state;        // Process state
     void *chan;                  // If non-zero, sleeping on chan
     int killed;                  // If non-zero, have been killed
     int xstate;                  // Exit status to be returned to parent's wait
     int pid;                     // Process ID
   
     // wait_lock must be held when using this:
     struct proc *parent;         // Parent process
   
     // these are private to the process, so p->lock need not be held.
     uint64 kstack;               // Virtual address of kernel stack
     uint64 sz;                   // Size of process memory (bytes)
     pagetable_t pagetable;       // User page table
     struct trapframe *trapframe; // data page for trampoline.S
     struct context context;      // swtch() here to run process
     struct file *ofile[NOFILE];  // Open files
     struct inode *cwd;           // Current directory
     char name[16];               // Process name (debugging)
     uint64 syscall_trace;        // Mask syscall trace
   };
   ```

   其次就是修改 fork 函数，使子进程能够继承父进程的掩码。在 `kernel/proc.c` 中的 fork() 函数中，添加下面的代码。
   ```c
   int
   fork(void)
   {
     // something else
     safestrcpy(np->name, p->name, sizeof(p->name));
   
     // 子进程复制父进程的mask
     np->syscall_trace = p->syscall_trace;
   
     pid = np->pid;
     // something else
   }
   ```

至此，完整的Lab代码就已经全部完成🌼🌼
