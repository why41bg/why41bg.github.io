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

首先要运行下面的指令生成 `user/call.asm` 文件，需要重点关注其中的 g、f 和 main 三个函数。

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
