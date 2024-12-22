### 练习1: 加载应用程序并执行（需要编码） 

***do\_execv**函数调用`load_icode`（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序。你需要补充`load_icode`的第6步，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好`proc_struct`结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。*

*请在实验报告中简要说明你的设计实现过程。*
- *请简要描述这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。*

#### 补充代码如下
```
tf->gpr.sp = USTACKTOP;
tf->epc = elf->e_entry;
tf->status = (sstatus & ~SSTATUS_SPP) | SSTATUS_SPIE;
```

#### 设计实现过程
`do_execve` 将用户态的程序加载到内核态，然后执行。

`load_icode`的六个步骤：

1.创建一个新的 mm_struct。

2.创建一个新的 PDT，将 mm 的 pgdir 设置为这个 PDT 的虚拟地址。

3.读取 ELF 格式，检验其合法性，循环读取每一个程序段，将需要加载的段加载到内存中，设置相应段的权限。之后初始化 BSS 段，将其清零。

4.设置用户栈。

5.设置当前进程的 mm，cr3,设置 satp 寄存器。

6.需要保证用户态的程序无论是在用户态发生了什么，都能够从用户态来到特权态，并正常返回。即中断帧能够设置正确。

为了实现第6步，步骤如下：

(1)设置tf的gpr.sp（即存储栈顶指针）指向用户栈顶（USTACKTOP），以便在用户程序运行时可以正确地访问栈。

(2)设置tf的epc为ELF文件的入口地址（存储发生异常或中断时的程序计数器的值为elf的e_entry）。
- elf是在load_icode的前面通过struct elfhdr *elf = (struct elfhdr *)binary;方式定义的二进制ELF文件的文件头，elf的e_entry就是用户程序的入口点地址了，以便在用户程序运行时可以从正确的位置开始执行。

(3)设置tf的status（即存储处理器的状态信息）。
- SPP:用于表示处理器在发生异常或中断之前的特权级别。0表示处理器在发生异常或中断之前处于用户模式，1表示处理器在发生异常或中断之前处于特权模式。又因为统调用 sys_exec 之后， 在 trap 返回的时候调用了 sret 指令。sret指令会根据SPP的值回到中断前的状态。因为如果是在用户态可能发生中断而来到特权态处理，为了最终通过sret返回用户态，所以SPP应该为0。

- SPIE：用于表示处理器在发生异常或中断之前的中断使能状态。0表示处理器在发生异常或中断之前中断被禁用，1表示处理器在发生异常或中断之前中断被启用。为了保证用户态能够正常触发中断，因此应该启用中断，即SPIE应该为1。


#### 用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过

1. 内核线程初始化

- proc_init 初始化：
创建 idle 内核线程（第 0 个内核线程）。
调用 kernel_thread 创建 init 内核线程（第 1 个内核线程）。

2. 执行 init 线程：
- 进入调度循环：
cpu_idle 不断查询 need_resched 标志位。
当 need_resched 为真时，调用 schedule 进行线程调度。

- 切换到 init 线程：
schedule 找到 init 线程。
调用 proc_run 切换到 init 线程，返回到 kernel_thread_entry 执行指定函数。

- 执行 init_main：
在 init_main 中再次调用 kernel_thread 创建 user_main 用户线程。

3. 创建用户线程user_main
- 调用 kernel_execve：
KERNEL_EXECVE(exit) 调用 kernel_execve。
kernel_execve 使用 ebreak 触发断点中断。

- 进入系统调用处理：
中断处理时，调用 syscall 函数。
系统调用 sys_exec 调用 do_execve。
do_execve 调用 load_icode 加载用户程序：
加载二进制程序到用户虚拟内存空间。
设置中断帧，将 SPP 设置为 0，准备切换到用户态。

4. 调度切换到 user_main
- 继续执行 init_main：
init_main 中调用 do_wait：
当前线程状态设为 SLEEPING。
等待状态设为 WT_CHILD。
调用 schedule 进行调度。

- 切换到 user_main：
调度器选择 user_main。
kernel_execve_ret 最终通过 sret 切换到用户态。

5. 用户态执行：
- 进入用户态：
initcode.s 开始执行，进入 umain 函数。
在 umain 中执行 exit.c 的主体函数。
调用 fork 创建子进程。
调用 wait 进行进程切换，回到父进程，完成子进程回收。
  
6. 回到内核
- 父进程退出后，返回到 initproc 内核线程。
调用 do_exit 结束实验。