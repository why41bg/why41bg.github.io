---
title: xv6中trap的执行流（二）
date: 2023-11-09 21:25:29
tags:
- OS
- XV6
---

# uservec函数

回到 XV6 和 RISC-V，现在程序位于 trampoline page 的起始，也是 **uservec** 函数的起始。我们现在需要做的第一件事情就是保存寄存器的内容。在 RISC-V 上，如果不能使用寄存器，基本上不能做任何事情。所以，对于保存这些寄存器，我们有什么样的选择呢？

在一些其他的机器中，我们或许直接就将32个寄存器中的内容写到物理内存中某些合适的位置。但是我们不能在 RISC-V 中这样做，因为在 RISC-V 中，supervisor mode 下的代码不允许直接访问物理内存。所以我们只能使用 page table 中的内容，但是从前面的输出来看，page table 中也没有多少内容。

虽然 XV6 并没有使用，但是另一种可能的操作是，直接将 SATP 寄存器指向 kernel page table，之后我们就可以直接使用所有的 kernel mapping 来帮助我们存储用户寄存器。这是合法的，因为supervisor mode 可以更改 SATP 寄存器。但是在 trap 代码当前的位置，也就是 trap 机制的最开始，我们并不知道 kernel page table 的地址。并且更改 SATP 寄存器的指令，要求写入 SATP 寄存器的内容来自于另一个寄存器。所以，为了能执行更新 page table 的指令，我们需要一些空闲的寄存器，这样我们才能先将 page table 的地址存在这些寄存器中，然后再执行修改 SATP 寄存器的指令。

对于保存用户寄存器，XV6 在 RISC-V 上的实现包括了两个部分。第一个部分是，XV6 在每个 user page table 映射了 trapframe page，这样每个进程都有自己的 trapframe page。这个 page 包含了很多有趣的数据，但是现在最重要的数据是用来保存用户寄存器的32个空槽位。所以，在 trap 处理代码中，现在的好消息是，我们在 user page table 有一个之前由 kernel设置好的映射关系，这个映射关系指向了一个可以用来存放这个进程的用户寄存器的内存位置。这个位置的虚拟地址总是 0x3ffffffe000。

如果你想查看 XV6 在 trapframe page 中存放了什么，这部分代码在 `kernel/proc.h` 中的 trapframe 结构体中。

```c
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
};
```

你可以看到很多槽位的名字都对应了特定的寄存器。在最开始还有5个数据，这些是内核事先存放在trapframe 中的数据。比如第一个数据保存了 kernel page table 地址，这将会是 trap 处理代码将要加载到 SATP 寄存器的数值。

所以，如何保存用户寄存器的一半答案是，内核非常方便的将 trapframe page 映射到了每个 user page table。

另一半的答案在于我们之前提过的 SSCRATCH 寄存器。这个由 RISC-V 提供的 SSCRATCH 寄存器，就是为接下来的目的而创建的。在进入到 user space 之前，内核会将 trapframe page 的地址保存在这个寄存器中，也就是0x3fffffe000这个地址。更重要的是，RISC-V 有一个指令允许交换任意两个寄存器的值。而 SSCRATCH 寄存器的作用就是保存另一个寄存器的值，并将自己的值加载给另一个寄存器。如果我查看 trampoline.S 代码，

``` assembly
        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)
```

第一件事情就是执行 csrrw 指令，这个指令交换了 a0 和 sscratch 两个寄存器的内容。因此，a0 现在的值是 0x3fffffe000，这是 trapframe page 的虚拟地址。它之前保存在 SSCRATCH 寄存器中，但是我们现在交换到了 a0 中。而 SSCRATCH 中保存的就是 a0 寄存器的值，a0 寄存器保存的是 write 函数的第一个参数，在这个场景下，是 shell 传入的文件描述符2。所以我们现在将 a0 的值保存起来了，并且我们有了指向 trapframe page 的指针。现在我们正在朝着保存用户寄存器的道路上前进。实际上，这就是 trampoline.S 中接下来30多个奇怪指令的工作。这些指令就是的执行 sd，将每个寄存器保存在 trapframe 的不同偏移位置。因为 a0 在交换完之后包含的是 trapframe page 地址，也就是 0x3fffffe000。所以，每个寄存器被保存在了偏移量+ a0 的位置。这些存储的指令比较无聊，我就不介绍了。

> 学生提问：当与 a0 寄存器进行交换时，trapframe 的地址是怎么出现在 SSCRATCH 寄存器中的？
>
> Robert 教授：在内核前一次切换回用户空间时，内核会执行 set sscratch 指令（在 userret 函数中），将这个寄存器的内容设置为 0x3fffffe000，也就是 trapframe page 的虚拟地址。所以，当我们在运行用户代码，比如运行 shell 时，SSCRATCH 保存的就是指向 trapframe 的地址。之后，shell 执行了 ecall 指令，跳转到了 trampoline page，这个 page 中的第一条指令会交换 a0 和 SSCRATCH 寄存器的内容。所以，SSCRATCH 中的值，也就是指向 trapframe 的指针现在存储与 a0 寄存器中。你或许会好奇，a0 是如何有 trapframe page 的地址。我们可以查看 `kernel/trap.c` 中 usertrapret 函数的代码
> ``` c
> void
> usertrapret(void)
> {
>   // ...
>     
>   // jump to trampoline.S at the top of memory, which 
>   // switches to the user page table, restores user registers,
>   // and switches to user mode with sret.
>   uint64 fn = TRAMPOLINE + (userret - trampoline);
>   ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
> }
> ```
>
> 这是内核返回到用户空间的最后的C函数。C函数做的最后一件事情是调用 fn 函数，传递的参数是 TRAMFRAME 和 user page table。在C代码中，当你调用函数，第一个参数会存在 a0，这就是为什么 a0 里面的数值是指向 trapframe 的指针。
>
> 学生提问：当你启动一个进程，之后进程在运行，之后在某个时间点进程执行了 ecall 指令，那么你是在什么时候执行上一个问题中的 fn 函数呢？因为这是进程的第一个 ecall 指令，所以这个进程之前应该没有调用过 fn 函数吧。
>
> Robert 教授：好的，或许对于这个问题的一个答案是：一台机器总是从内核开始运行的，当机器启动的时候，它就是在内核中。 任何时候，不管是进程第一次启动还是从一个系统调用返回，进入到用户空间的唯一方法是就是执行 **sret** 指令。sret 指令是由 RISC-V 定义的用来从 supervisor mode 转换到 user mode。所以，在任何用户代码执行之前，内核会执行 fn 函数，并设置好所有的东西，例如 SSCRATCH，STVEC 寄存器。

程序现在仍然在 trampoline 的最开始，也就是 uservec 函数的开始部分，我们基本上除了保存用户寄存器的值外，还没有执行任何内容。我在寄存器拷贝的结束位置设置了一个断点，我们在 gdb 中让代码继续执行，现在我们停在了下面这条ld（load）指令。

![寄存器拷贝结束后的断点](/images/寄存器拷贝结束后的断点.png)

这条指令正在将 a0 指向的内存地址往后数的第8个字节开始的数据加载到 Stack Pointer 寄存器。a0 的内容现在是 trapframe page 的地址，从本节第一张图中，trapframe 的格式可以看出，第8个字节开始的数据是内核的 Stack Pointer（kernel_sp）。trapframe 中的 kernel_sp 是由 kernel 在进入用户空间之前就设置好的，它的值是这个进程的 kernel stack。所以这条指令的作用是初始化 Stack Pointer 指向这个进程的 kernel stack 的最顶端。

下一条指令是向tp寄存器写入数据。因为在 RISC-V 中，没有一个直接的方法来确认当前运行在多核处理器的哪个核上，XV6 会将 CPU 核的编号也就是 hartid 保存在 tp 寄存器。在内核中好几个地方都会使用了这个值，例如，内核可以通过这个值确定某个CPU核上运行了哪些进程。

下一条指令是向 t0 寄存器写入数据。这里写入的是我们将要执行的第一个C函数的指针，也就是函数usertrap 的指针。我们在后面会使用这个指针。

下一条指令是向 t1 寄存器写入数据。这里写入的是 kernel page table 的地址。实际上严格来说，t1的内容并不是 kernel page table 的地址，而是你需要向 SATP 寄存器写入的数据。它包含了 kernel page table的地址，但是移位了，并且包含了各种标志位。

下一条指令是交换 SATP 和 t1 寄存器。这条指令执行完成之后，**当前程序会从 user page table 切换到 kernel page table。**现在我们在QEMU中打印 page table，可以看出与之前的 page table 完全不一样。

![kernel页表](/images/kernel页表.webp)

现在这里输出的是由内核设置好的巨大的 kernel page table。所以现在我们成功的切换了 page table，我们在这个位置进展的很好，Stack Pointer 指向了 kernel stack；我们有了 kernel page table，可以读取 kernel data。我们已经准备好了执行内核中的C代码了。

这里还有个问题，为什么代码没有崩溃？毕竟我们在内存中的某个位置执行代码，程序计数器保存的是虚拟地址，如果我们切换了 page table，为什么同一个虚拟地址不会通过新的 page table 寻址走到一些无关的 page 中？看起来我们现在没有崩溃并且还在执行这些指令。

我不知道你们是否还记得 user page table 的内容，**trampoline page 在 user page table 中的映射与kernel page table 中的映射是完全一样的。**这两个 page table 中其他所有的映射都是不同的，只有trampoline page 的映射是一样的，因此我们在切换 page table时，寻址的结果不会改变，我们实际上就可以继续在同一个代码序列中执行程序而不崩溃。这是 trampoline page 的特殊之处，它同时在 user page table 和 kernel page table 都有相同的映射关系。

而它之所以叫 trampoline page，是因为你某种程度在它上面“弹跳”了一下，然后从用户空间走到了内核空间。

最后一条指令是 *jr t0*。执行了这条指令，我们就要从 trampoline 跳到内核的C代码中。这条指令的作用是跳转到 t0 指向的函数中（usertrap 函数）。

# usertrap函数

usertrap 函数是位于 `kernel/trap.c` 文件的一个函数。有很多原因都可以让程序运行进入到 usertrap 函数中来，比如系统调用，运算时除以0，使用了一个未被映射的虚拟地址，或者是设备中断。usertrap 在某种程度上存储并恢复硬件状态，但是它也需要检查触发 trap 的原因，以确定相应的处理方式，我们在接下来执行 usertrap 的过程中会同时看到这两个行为。

```c
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

它做的第一件事情是更改 STVEC 寄存器。取决于 trap 是来自于用户空间还是内核空间，实际上 XV6 处理 trap 的方法是不一样的。目前为止，我们只讨论过当trap 是由用户空间发起时会发生什么。如果 trap 从内核空间发起，将会是一个非常不同的处理流程，因为从内核发起的话，程序已经在使用 kernel page table。所以当 trap 发生时，程序执行仍然在内核的话，很多处理都不必存在。

在内核中执行任何操作之前，usertrap 中先将 STVEC 指向了 kernelvec 变量，这是内核空间 trap 处理代码的位置，而不是用户空间 trap 处理代码的位置。出于各种原因，我们需要知道当前运行的是什么进程，我们通过调用 myproc 函数来做到这一点。myproc 函数实际上会查找一个根据当前 CPU 核的编号索引的数组，CPU 核的编号是 hartid，如果你还记得，我们之前在 uservec 函数中将它存在了 tp 寄存器。这是 myproc 函数找出当前运行进程的方法。接下来我们要保存用户程序计数器（PC），它仍然保存在 SEPC 寄存器中，但是可能发生这种情况：当程序还在内核中执行时，我们可能切换到另一个进程，并进入到那个程序的用户空间，然后那个进程可能再调用一个系统调用进而导致 SEPC 寄存器的内容被覆盖。所以，我们需要保存当前进程的 SEPC 寄存器到一个与该进程关联的内存中，这样这个数据才不会被覆盖。这里我们使用 trapframe 来保存这个程序计数器。

接下来我们需要找出我们现在会在 usertrap 函数的原因。根据触发 trap 的原因，RISC-V 的 SCAUSE 寄存器会有不同的数字。数字8表明，我们现在在trap代码中是因为系统调用。所以，我们可以进到 `if(r_scause() == 8)` 语句中。接下来第一件事情是检查是不是有其他的进程杀掉了当前进程，但是我们的 shell 没有被杀掉，所以检查通过。

在 RISC-V 中，存储在 SEPC 寄存器中的程序计数器，**是用户程序中触发 trap 的指令的地址**。但是当我们恢复用户程序时，我们希望在下一条指令恢复，也就是ecall 之后的一条指令。所以对于系统调用，我们对于保存的用户程序计数器加4，这样我们会在 ecall 的下一条指令恢复，而不是重新执行 ecall 指令。

XV6 会在处理系统调用的时候使能中断，这样中断可以更快的服务，有些系统调用需要许多时间处理。中断总是会被 RISC-V 的 trap 硬件关闭（原子操作，关中断），所以在这个时间点，我们需要显式的打开中断，即 `intr_on()`。下一行代码中，我们会调用 syscall 函数。这个函数定义在 `kernel/syscall.c` 中。

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

它的作用是从 syscall 表单中，根据系统调用的编号查找相应的系统调用函数。如果你还记得之前的内容，shell 调用的 write 函数将 a7 设置成了系统调用编号，对于write 来说就是16。所以 syscal l函数的工作就是获取由 trampoline 代码保存在 trapframe 中 a7 的数字，然后用这个数字索引找到对应的内核程序（这在 Lab2 实验中涉及到了）。接下来内核程序干得事情我们之前都已经介绍过了，在这节课中，我只对进入和跳出内核感兴趣。

这里有件有趣的事情，系统调用需要找到它们的参数。你们还记得 write 函数的参数吗？分别是文件描述符2，写入数据缓存的指针，写入数据的长度2。syscall 函数直接通过 trapframe 来获取这些参数，就像这里刚刚可以查看 trapframe中 的 a7 寄存器一样，我们可以查看 a0 寄存器，这是第一个参数，a1 是第二个参数，a2 是第三个参数。

![查看write函数传递到参数](/images/查看write函数传递到参数.png)

现在 syscall 执行了真正的系统调用，之后 sys_write 返回了。 `p->trapframe->a0 = syscalls[num]()`。这里向 trapframe 中的 a0 赋值的原因是：所有的系统调用都有一个返回值，比如 write 会返回实际写入的字节数，而 RISC-V 上的C代码的习惯是函数的返回值存储于寄存器 a0，所以为了模拟函数的返回，我们将返回值存储在 trapframe 的 a0 中。之后，当我们返回到用户空间，trapframe 中的a0槽位的数值会写到实际的 a0 寄存器，shell 就会认为 a0 寄存器中的数值是write 系统调用的返回值。

从 syscall 函数返回之后，我们回到了 trap.c 中的 usertrap 函数。我们需要再次检查当前用户进程是否被杀掉了，因为我们不想恢复一个被杀掉的进程。当然，在我们的场景中，shell 没有被杀掉。最后，usertrap 调用了一个函数 usertrapret。

# usertrapret函数

usertrap 函数的最后调用了 usertrapret 函数，来设置好我之前说过的，在返回到用户空间之前内核要做的工作。我们可以查看这个函数的内容。

``` c
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  w_stvec(TRAMPOLINE + (uservec - trampoline));

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```

它首先关闭了中断。我们之前在系统调用的过程中是打开了中断的，这里关闭中断是因为我们将要更新 STVEC 寄存器来指向用户空间的 trap 处理代码（原子操作），而之前在内核中的时候，我们指向的是内核空间的 trap 处理代码。我们关闭中断因为当我们将 STVEC 更新到指向用户空间的 trap 处理代码时，我们仍然在内核中执行代码。如果这时发生了一个中断，那么程序执行会走向用户空间的 trap 处理代码，即便我们现在仍然在内核中，出于各种各样具体细节的原因，这会导致内核出错。所以我们这里关闭中断。

在下一行（`w_stvec(TRAMPOLINE + (uservec - trampoline))`）我们设置了 STVEC 寄存器指向 trampoline 代码，在那里最终会执行 sret 指令返回到用户空间。位于 trampoline 代码最后的 sret 指令会重新打开中断。这样，即使我们刚刚关闭了中断，当我们在执行用户代码时中断是打开的。

接下来的几行填入了 trapframe 的内容，这些内容对于执行 trampoline 代码非常有用。这里的代码就是：

- 存储了 kernel page table 的指针。
- 存储了当前用户进程的 kernel stack。
- 存储了 usertrap 函数的指针，这样 trampoline 代码才能跳转到这个函数。
- 从 tp 寄存器中读取当前的 CPU 核编号，并存储在 trapframe 中，这样 trampoline 代码才能恢复这个数字，因为用户代码可能会修改这个数字。

现在我们在 usertrapret 函数中，我们正在设置 trapframe 中的数据，这样下一次从用户空间转换到内核空间时可以用到这些数据。

接下来我们要设置 SSTATUS 寄存器，这是一个控制寄存器。这个寄存器的 SPP bit 位控制了 sret 指令的行为，该 bit 为0表示下次执行 sret 的时候，我们想要返回 user mode 而不是 supervisor mode。这个寄存器的 SPIE bit 位控制了在执行完 sret 之后，是否打开中断。因为我们在返回到用户空间之后，我们的确希望打开中断，所以这里将 SPIE bit 位设置为1。修改完这些 bit 位之后，我们会把新的值写回到 SSTATUS 寄存器。接下来我们将 SEPC 寄存器的值设置成之前保存的用户程序计数器的值。在不久之前，我们在 usertrap 函数中将用户程序计数器保存在 trapframe 中的 epc 字段。

接下来 `uint64 satp = MAKE_SATP(p->pagetable)`，我们根据 user page table 地址生成相应的 SATP 值，这样我们在返回到用户空间的时候才能完成 page table 的切换。实际上，我们会在汇编代码 trampoline 中完成 page table 的切换，并且也只能在 trampoline 中完成切换，因为只有 trampoline 中代码是同时在用户和内核空间中映射。但是我们现在还没有在 trampoline 代码中，我们现在还在一个普通的C函数中，所以这里我们将 page table 指针准备好，并将这个指针作为第二个参数传递给汇编代码，这个参数会出现在 a1 寄存器。

倒数第二行的作用是计算出我们将要跳转到汇编代码的地址。我们期望跳转的地址是 tampoline 中的 userret 函数，这个函数包含了所有能将我们带回到用户空间的指令。所以这里我们计算出了 userret 函数的地址。倒数第一行，将 fn 指针作为一个函数指针，执行相应的函数（也就是 userret 函数）并传入两个参数，两个参数存储在 a0，a1 寄存器中。

# userret函数

现在程序执行又到了 trampoline 代码。

``` ass
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

第一步是切换 page table。在执行 *csrw satp, a1* 之前，page table 应该还是巨大的 kernel page table。这条指令会将 user page table（在 usertrapret 中作为第二个参数传递给了这里的 userret 函数，所以存在 a1 寄存器中）存储在 SATP 寄存器中。执行完这条指令之后，page table 就变成了小得多的 user page table。但是幸运的是，user page table 也映射了 trampoline page，所以程序还能继续执行而不是崩溃。

在 uservec 函数中，第一件事情就是交换 SSRATCH 和 a0 寄存器。而这里，我们将 SSCRATCH 寄存器恢复成保存好的用户的 a0 寄存器。在这里 a0 是 trapframe 的地址，因为C代码 usertrapret 函数中将 trapframe 地址作为第一个参数传递过来了。112 是 a0 寄存器在 trapframe中的位置。（注，这里有点绕，本质就是通过当前的 a0 寄存器找出存在 trapframe 中的 a0 寄存器）我们先将这个地址里的数值保存在 t0 寄存器中，之后再将 t0 寄存器的数值保存在SSCRATCH 寄存器中。为止目前，所有的寄存器内容还是属于内核。而接下来的这些指令将 a0 寄存器指向的 trapframe 中，之前保存的寄存器的值加载回对应的各个寄存器中。之后，我们离能真正运行用户代码就很近了。

>  学生提问：现在 trapframe 中的 a0 寄存器（区分 a0 寄存器和 trapframe 中的 a0 寄存器，这是两个意思）是我们执行系统调用的返回值吗？
>
> Robert 教授：是的，系统调用的返回值覆盖了我们保存在 trapframe 中的 a0 寄存器的值。我们希望用户程序 shell 在 a0 寄存器中看到系统调用的返回值。所以，trapframe 中的 a0 寄存器现在是系统调用的返回值2。相应的 SSCRATCH 寄存器中的数值也应该是2。

如果我们打印所有的寄存器，这些寄存器的值就是我们在最最开始看到的用户寄存器的值。但是 a0 寄存器现在还是个例外，它现在仍然是指向 trapframe 的指针，而不是保存了的用户数据。接下来，在我们即将返回到用户空间之前，我们交换 SSCRATCH 寄存器和 a0 寄存器的值。前面我们看过了 SSCRATCH 现在的值是系统调用的返回值2，a0 寄存器是 trapframe 的地址。交换完成之后，a0 持有的是系统调用的返回值，SSCRATCH 持有的是 trapframe 的地址。之后trapframe 的地址会一直保存在 SSCRATCH 中，直到用户程序执行了另一次 trap。现在我们还在 kernel 中。

sret 是我们在 kernel 中的最后一条指令，当我执行完这条指令：

- 程序会切换回 user mode。
- SEPC 寄存器的数值会被拷贝到PC寄存器（程序计数器）。
- 重新打开中断。

现在我们终于回到了用户空间。PC 寄存器中的值如下。

![回到用户空间后PC的值](/images/回到用户空间后PC的值.png)

如果我们查看 sh.asm，可以看到这个地址是 write 函数的 ret 指令地址。所以，现在我们回到了用户空间，执行完 ret 指令之后我们就可以从 write 系统调用返回到 shell中 了。或者更严格的说，是从触发了系统调用的 write 库函数中返回到 shell 中。

# 总结

最后总结一下，系统调用被刻意设计的看起来像是函数调用，但是背后的 user/kernel 转换比函数调用要复杂的多。之所以这么复杂，很大一部分原因是要保持user/kernel 之间的**隔离性**，内核不能信任来自用户空间的任何内容。

另一方面，XV6 实现 trap 的方式比较特殊，XV6 并不关心性能。但是通常来说，操作系统的设计人员和 CPU 设计人员非常关心如何提升 trap 的效率和速度。必然还有跟我们这里不一样的方式来实现 trap，当你在实现的时候，可以从以下几个问题出发：

- 硬件和软件需要协同工作，你可能需要重新设计 XV6，重新设计 RISC-V 来使得这里的处理流程更加简单，更加快速。
- 另一个需要时刻记住的问题是，恶意软件是否能滥用这里的机制来打破隔离性。

（最后我自己也想说一句话，Robert 教授真是我见过讲得最好最细致的一位**professor**）💐💐💐
