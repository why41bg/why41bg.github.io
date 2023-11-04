---
title: MIT 6.S081 Operating System Enginerring Lab3
date: 2023-11-03 23:52:42
tags:
- XV6
- OS
categories:
- MIT 6.S081
---

> 
# Speed up system calls ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))
> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. 
>
> 本实验就是通过页表在用户空间和内核间共享只读区域的数据，这样就能在无需切换内核的情况加速系统调用，快速获得用户进程的PID。

每个进程创建时，都会创建一个属于它自己的虚拟地址空间（页表），该地址空间的结构图在教材中有说明，主要有三个部分：TRAMPOLINE、TRAPFRAME、USYSCALL。而 `USYSCALL` 就是映射的与内核共享的**只读**区域。用户进程的虚拟地址空间布局在 `kernel/memlayout.h` 文件中，这里面同时还定义了一个 `usyscall` 结构体用来存储用户进程的PID。相关定义如下：

```c
// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   USYSCALL (shared with kernel)
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
#endif
```

按照提示就能够完成该实验，详细的步骤说明如下：

1. 创建用户进程的页表，创建页表的函数在 `kernel/proc.c` 中，函数名为 `proc_pagetable`。
   仿照 TRAMPOLINE 和 TRAPFRAME 的样子为 USYSCALL 分配PTE：

   ``` c
   pagetable_t
   proc_pagetable(struct proc *p)
   {
         // ...
         // map the trapframe just below TRAMPOLINE, for trampoline.S.
         if(mappages(pagetable, TRAPFRAME, PGSIZE,
                     (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
           uvmunmap(pagetable, TRAMPOLINE, 1, 0);
           uvmfree(pagetable, 0);
           return 0;
         }
   
         // map the USYSCALL just below trapframe
         if(mappages(pagetable, USYSCALL, PGSIZE, 
                     (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
           uvmunmap(pagetable, TRAMPOLINE, 1, 0);
           uvmunmap(pagetable, TRAPFRAME, 1, 0);
           uvmfree(pagetable, 0);
           return 0;
         }
   
         return pagetable;
   }
   ```

   

2. 为用户进程分配实际的物理内存，同时初始化。相关函数为 `kernel/proc.c` 中的 `allocproc`。
   仿照为 TRAPFRAME 分配内存的方式为 USYSCALL 分配实际的物理内存，**不要忘记对 `USYSCALL` 初始化**：

   ```c
   static struct proc*
   allocproc(void)
   {
       // ...
       // Allocate a trapframe page.
       if((p->trapframe = (struct trapframe *)kalloc()) == 0){
         freeproc(p);
         release(&p->lock);
         return 0;
       }
   
       // Allocate a usyscall page.
       if((p->usyscall = (struct usyscall *)kalloc()) == 0){
         freeproc(p);
         release(&p->lock);
         return 0;
       }
       p->usyscall->pid = p->pid;
       // ...
   }
   ```

   

3. 完善 `kernel/proc.c` 中与释放物理内存有关的函数 `freeproc`。
   ```c
   static void
   freeproc(struct proc *p)
   {
       // ...
       if(p->usyscall)
       	kfree((void*)p->usyscall);
     	p->usyscall = 0;
       // ...
   }
   ```

   

4. 完善 `kernel/proc.c` 中释放页表有关的函数 `proc_freepagetable`。
   ```c
   void
   proc_freepagetable(pagetable_t pagetable, uint64 sz)
   {
     uvmunmap(pagetable, TRAMPOLINE, 1, 0);
     uvmunmap(pagetable, TRAPFRAME, 1, 0);
     uvmunmap(pagetable, USYSCALL, 1, 0);
     uvmfree(pagetable, sz);
   }
   ```

该实验最后还提出了一个问题，**Which other xv6 system call(s) could be made faster using this shared page? ** 答案是：fork 系统调用。只需要在 `usyscall` 结构体中加入父进程的 `parent` 结构体，即可在 fork 系统调用复制父进程内存空间时，无需切换到内核即可找到父进程实际的物理内存地址，从而完成父进程内存空间的复制。