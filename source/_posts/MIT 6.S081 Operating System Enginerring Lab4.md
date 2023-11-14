---
title: MIT 6.S081 Operating System Enginerring Lab4
date: 2023-11-07 17:09:23
tags:
- XV6
- OS
categories:
- MIT 6.S081
---

> This lab explores how system calls are implemented using traps. 

## RISC-V assembly ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> 对于这个实验内容，关键在于理解 RISC-V 汇编

首先要运行下面的指令生成 `user/call.asm` 文件，需要重点关注其中的 g、f 和 main 三个函数，然后回答下面五个问题。

```bash
make fs.img
```

*Q1*：Which registers contain arguments to functions? For example, which register holds 13 in main's call to `printf` ?

`user/call.ams` 文件中关于 main 函数如下：

```assembly
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7a050513          	addi	a0,a0,1952 # 7c8 <malloc+0xe8>
  30:	00000097          	auipc	ra,0x0
  34:	5f8080e7          	jalr	1528(ra) # 628 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	274080e7          	jalr	628(ra) # 2ae <exit>
```

第 7 行将 13 保存到了 **a2** 寄存器，因此答案为 a2 。



*Q2*：Where is the call to function `f` in the assembly code for main? Where is the call to `g`? (Hint: the compiler may inline functions.)

从代码可以看出，这两个都被内联优化处理了，无函数调用。



*Q3*：At what address is the function `printf` located?

第 1076 行 `0000000000000628 <printf>:`



*Q4*：What value is in the register `ra` just after the `jalr` to `printf` in `main` ?

0x0000000000000038, jalr 指令的下一条汇编指令的地址。



## Backtrace ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> For debugging it is often useful to have a backtrace: a list of the function calls on the stack above the point at which the error occurred.
>
> 简单来说，该实验就是将函数调用堆栈中的返回地址依次打印出来。

每一个栈帧的结构图如下：

![XV6中栈帧的简单示意图](/images/XV6中栈帧的简单示意图.png)

栈帧从栈底（高地址处）开始的第一个8字节存储返回地址，第二个8字节存储上一个栈帧的栈底（fp）。

关于RISC-V中的必要的寄存器说明如下图：

![Regster-info](/images/Regster-info.png)

其中 `s0` 存储堆栈帧指针，在实验说明中的 hints 中也指出了：“ The GCC compiler stores the frame pointer of the currently executing function in the register `s0` ”。整个实验流程大致分为如下几步：

1. 在 `kernel/defs.h` 中添加 `backtrace` 函数的声明。

2. 在 `kernel/ riscv.h` 中添加一个 `r_fp` 函数用于获取当前 `s0` 寄存器中的值，即当前堆栈的帧指针。`r_fp` 函数实现如下：
   ```c
   static inline uint64
   r_fp()
   {
     uint64 x;
     asm volatile("mv %0, s0" : "=r" (x) );
     return x;
   }
   ```

3. 在 `kernel/printf` 中定义 `backtrace` 函数的实现。函数实现如下：
   ```c
   void
   backtrace(void)
   {
     printf("backtrace:\n");
   
     uint64 fp = r_fp();
   
     uint64 base = PGROUNDUP(fp);  // high address
     uint64 top = PGROUNDDOWN(fp);  // low address
   
     while((fp < base) && (top < base)){
       printf("%p\n", *((uint64*)(fp-8)));
       fp = *((uint64*)(fp-16));
     }
   }
   ```

4. 在 `kernel/printf` 中的 `panic` 函数中调用 `backtrace` 函数。

5. 由于该实验需用通过 `sys_sleep` 系统调用来调用 `backtrace` 函数，达到评测的目的。因此需要在 `kernel/sysproc.c` 中的 `sys_sleep` 函数中调用 `backtrace` 函数。

最后通过在系统中执行 `bttest` 或者执行测试程序，前者能正确打印出 function calls list，后者能正确通过。
