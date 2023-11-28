---
title: MIT 6.S081 Operating System Enginerring Lab6
date: 2023-11-28 09:37:18
tags:
- XV6
- OS
categories:
- MIT 6.S081
---

> This lab will familiarize you with multithreading. You will implement switching between threads in a user-level threads package, use multiple threads to speed up a program, and implement a barrier.
> 说实话，翻译文档看起来挺混乱的，导致我长时间迷失在了线程切换的复杂的函数调用中。直到我在b站上找到了[翻译视频](https://www.bilibili.com/video/BV1rS4y1n7y1/?p=10&share_source=copy_web&vd_source=648b3a8f2dffc97d9c07c0e073ceb75b)，实际观察到了在线程切换时的函数调用过程，我觉得将视频和翻译文档结合起来，对函数的调用过程才会更加明晰。

## Uthread: switching between threads ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it. 
>
> 这个实验是完成用户级线程的切换（注意区分用户级线程和内核级线程）

实验的提示已经说得很清晰了，我们只需要修改 `user/uthread.c` 中的 `thread_scheduled` 函数和 `thread_create`函数，和 `user/uthread_switch.S` 文件。代码中需要修改的地方及修改目标都已经说得很清楚了，理解了线程切换的原理后，跟着做就很简单了。

首先在 `user/uthread.c` 文件中需要添加一个 context 结构体，然后将这个结构体添加到 thread 结构体中，代码如下：

```c
struct thread_context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char            stack[STACK_SIZE]; /* the thread's stack */
  int             state;             /* FREE, RUNNING, RUNNABLE */
  struct thread_context context;     /* thread's context*/
};
```

然后是修改 `thread_scheduled` 函数，只需要在该函数中调用一下 `thread_switch` 函数，传入两个 context 的指针即可，代码如下：

```c
void 
thread_schedule(void)
{
	// ...
    
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&next_thread->context);
  } else
    next_thread = 0;
}
```

接下来修改 `thread_create` 函数，修改如下：

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)&t->stack[STACK_SIZE - 1];
  t->context.s0 = (uint64)&t->stack[STACK_SIZE - 1];
}
```

最后是 `uthread_switch.S` 文件，这个在课中讲得已经很清楚了，该文件代码如下：

```assembly
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)

	ld ra, 0(a1)
	ld sp, 8(a1)
	ld s0, 16(a1)
	ld s1, 24(a1)
	ld s2, 32(a1)
	ld s3, 40(a1)
	ld s4, 48(a1)
	ld s5, 56(a1)
	ld s6, 64(a1)
	ld s7, 72(a1)
	ld s8, 80(a1)
	ld s9, 88(a1)
	ld s10, 96(a1)
	ld s11, 104(a1)
	
	ret    /* return to ra */
```

