---

title: 文件描述符fd到底是什么？
date: 2023-10-24 23:21:42
tags:

- OS

---

> fd 是 File descriptor 的缩写，中文名叫做：文件描述符。文件描述符是一个非负整数，本质上是一个索引值。在 POSIX（可移植操作系统接口，由IEEE发布） 语义中，0，1，2 这三个 fd 值已经被赋予特殊含义，分别是标准输入（ STDIN_FILENO ），标准输出（ STDOUT_FILENO ），标准错误（ STDERR_FILENO ）。

下面进入 Linux 内核当中去看看所谓的文件描述符到底是个啥。

# tast_struct

**tast_struct**称为进程描述符结构，用来完整描述一个进程的所有信息。例如这个进程打开的文件、它的内存空间和进程的状态等等。这里需要着重注意的就是进程打开的文件，在tast_struct中与其有关的关键代码部分如下：

```c
struct task_struct{
    /* open file information */
    struct files_struct *files;
}
```

**files** 是一个指针，指向一个名为 **files_struct** 的结构体，**该结构体用来管理进程打开的所有文件**。

# files_struct

files_struct 结构体的源码如下：

```c
/*
 * Open file table structure
 */
struct files_struct {
    // 读相关字段
    atomic_t count;
    bool resize_in_progress;
    wait_queue_head_t resize_wait;

    // 打开的文件管理结构
    struct fdtable __rcu *fdt;
    struct fdtable fdtab;

    // 写相关字段
    unsigned int next_fd;
    unsigned long close_on_exec_init[1];
    unsigned long open_fds_init[1];
    unsigned long full_fds_bits_init[1];
    struct file * fd_array[NR_OPEN_DEFAULT];
};
```

files_struct 结构体管理进程打开的所有文件的方法就是——将所有打开的文件放进一个数组中，用数组管理的方式。这里这个数组就是 `struct file * fd_array[NR_OPEN_DEFAULT]`，fd_array 是一个二维数组，或者说是一个指针数组，这个数组的元素是指向 file 结构体的指针。不难猜想，这个这个 file 结构体就应该指某个具体的打开文件。

**fd** 的真面目就藏在了 **struct fdtable fdtab** 这句中，关于 **fdtable** 这个结构体的代码如下：

```c
struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;      /* current fd array */
};
```

这里一共有两个字段，第一个是 **fd**（fd 前面的 __rcu，即RCU锁，是一种数据同步机制，这里可以先不用管）是一个二级指针，或者说是一个指向指针数组的指针。这个数组中的每一个元素都是一个指针，指向一个 **struct file**。分析到这里，对 “fd 本质上就是索引”这句话的理解应该也就清晰了，fd 其实就是这个指针数组的索引，根据这个索引，我们就能找到与这个索引对应的指向 **struct file** 的指针。而 **max_fds** 呢，就是这个指针数组的边界了。

# file

fd 是指针数组的一个下标，根据这个下标我们能得到一个指向 **strcut file** 的指针。file 结构体的相关代码如下：

```c
struct file {
    // ...
    struct path                     f_path;
    struct inode                    *f_inode;
    const struct file_operations    *f_op;

    atomic_long_t                    f_count;
    unsigned int                     f_flags;
    fmode_t                          f_mode;
    struct mutex                     f_pos_lock;
    loff_t                           f_pos;
    struct fown_struct               f_owner;
    // ...
}
```

这个结构体就是用来记录打开文件的各种信息的，本文主要说明 fd 的本质，因此不对 file 结构体中的字段进行说明了。

# 小结

文件描述符是一个强大的对象，它为我们屏蔽了连接的一切细节，我们不用知道文件描述符背后连接到的究竟是一个文件，亦或是一个设备，这和 **“Everthing is file”** 的理念相契合。很想多说一点，`fork` 系统调用会复制父进程的文件描述符，`exec` 系统调用会保留文件描述符，这说明文件描述符在一定程度上是可以共享的，这就为 IO 重定向提供了实现途径（我想你知道该怎么做😉）。
