### EX1:分配并初始化一个进程控制块（需要编码）

alloc_proc 函数（位于 kern/process/proc.c 中）负责分配并返回一个新的 struct proc_struct 结构，用于存储新建立的内核线程的管理信息。ucore 需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。 

【提示】在 alloc_proc 函数的实现中，需要初始化的 proc_struct 结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/ffags/name。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

* *请说明 proc_struct 中 struct context context 和 struct trapframe tf 成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）*

```
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 YOUR CODE
    /*
     * below fields in proc_struct need to be initialized
     *       enum proc_state state;                      // Process state
     *       int pid;                                    // Process ID
     *       int runs;                                   // the running times of Proces
     *       uintptr_t kstack;                           // Process kernel stack
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
     *       struct proc_struct *parent;                 // the parent process
     *       struct mm_struct *mm;                       // Process's memory management field
     *       struct context context;                     // Switch here to run process
     *       struct trapframe *tf;                       // Trap frame for current interrupt
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
     *       uint32_t flags;                             // Process flag
     *       char name[PROC_NAME_LEN + 1];               // Process name
     */
	proc->state = PROC_UNINIT;//设置进程为未初始化状态
	proc->pid = -1;//设置进程id
	proc->runs = 0;//设置进程运行次数
	proc->kstack = 0;//设置内核栈地址
	proc->need_resched = 0;//设置不需要重新调度
	proc->parent = NULL;//设置父进程为空
	proc->mm = NULL;//设置内存管理字段为空
	memset(&(proc->context), 0, sizeof(struct context));//初始化上下文信息
	proc->tf = NULL;//设置trapframe为空
	proc->cr3 = boot_cr3;//设置cr3寄存器的值，页目录表（PDT）的基地址
	proc->flags = 0;//设置进程标志
	memset(proc->name, 0, PROC_NAME_LEN + 1);//初始化进程名为0

    }
    return proc;
}
```
#### struct context context 成员变量含义和在本实验中的作用是啥？
含义：struct context 结构体用于保存进程的上下文信息，包含了在进行进程切换（context switch）时所需保存的寄存器状态。这些寄存器状态包括程序计数器、栈指针、通用寄存器等。

作用：保存和恢复进程的寄存器上下文，以支持进程切换。在内核进行调度（调度器切换进程）时，当前进程的上下文会被保存到 proc->context 中，调度器会加载下一个进程的上下文，以便恢复其运行状态。

当进程被切换出去时，CPU 寄存器状态会保存在 proc->context 中。
当进程被调度回来时，proc->context 中保存的寄存器状态会被恢复，继续执行该进程的代码。

#### struct trapframe tf 成员变量含义和在本实验中的作用是啥？
含义：struct trapframe 是用来保存中断/异常发生时的 CPU 状态的结构体，它记录了 CPU 的寄存器值、程序计数器（EIP）、栈指针（ESP）等信息，主要用于在中断、异常或系统调用发生时保存用户态和内核态之间的状态信息。

作用：中断帧的指针，保存中断或系统调用发生时的 CPU 状态，以便在中断处理完成后，能够恢复进程的运行状态。

当进程从用户模式切换到内核模式时，用户模式下的状态(如寄存器)会被保存在 trapframe 中。内核完成处理后，可以使用这些信息来恢复进程的状态并继续用户模式下的执行。
### EX2
创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。因此，我们实际需要"fork"的东西就是stack和trapframe。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：

调用alloc_proc，首先获得一块用户信息块。

为进程分配一个内核栈。

复制原进程的内存管理信息到新进程（但内核线程不必做此事）

复制原进程上下文到新进程

将新进程添加到进程列表

唤醒新进程

返回新进程号

请在实验报告中简要说明你的设计实现过程。

####'do_fork'函数实现

**1.分配并初始化进程控制块**
```
proc = alloc_proc();

if (proc == NULL) {
   goto fork_out;
}
```
调用`alloc_proc`函数分配一个新的进程控制块并初始化。如果分配失败（返回NULL），则跳转到`fork_out`，返回错误码。

**2.分配并初始化内核栈**
```
 proc->parent = current;

if (setup_kstack(proc) != 0) {
   goto bad_fork_cleanup_kstack;
}
```
将新进程的父进程设置为当前进程（current）,调用`setup_kstack`函数为新进程分配内核栈。如果分配失败，则跳转到`bad_fork_cleanup_kstack`标签进行清理.

**3.根据clone_flags决定是复制还是共享内存管理系统**
```
if (copy_mm(clone_flags, proc) != 0) {
    goto bad_fork_cleanup_proc;
}
```
根据`clone_flags`调用`copy_mm`函数，决定是复制进程的内存管理结构还是与之共享。如果是`CLONE_VM`,则共享。如果失败，则跳转到`bad_fork_cleanup_proc`标签进行清理。因为我们创建的是内核线程，因此在这个程序里，我们跳过了内存管理信息的复制。

**4.设置进程的中断帧和上下文**
```
copy_thread(proc, stack, tf);
```
调用`copy_thread`函数，在新进程的内核栈顶设置`trapframe`，并配置内核入口点和栈。此步骤不检查返回值，因为即使失败也应继续执行清理操作。

**5.把设置好的进程加入链表**
```
bool intr_flag;
local_intr_save(intr_flag);//禁用中断

proc->pid = get_pid();//分配PID
hash_proc(proc);
list_add(&proc_list, &(proc->list_link));
nr_process++;

local_intr_restore(intr_flag);
```
使用`local_intr_save`函数保存当前中断状态，并禁用中断。这是为了防止在修改进程列表时发生中断，从而确保操作的原子性。调用`get_pid`函数为新进程分配一个唯一的PID。调用`hash_proc`函数将新进程添加到进程哈希表中。使用`list_add`函数将新进程添加到进程列表中。增加全局进程计数`nr_process`。使用`local_intr_restore`函数恢复之前保存的中断状态。

**6.将新建的进程设为就绪态**
```
wakeup_proc(proc);
```
调用`wakeup_proc`函数，将新进程的状态设置为`PROC_RUNNABLE`，使其可以被调度执行。

**7.将返回值设为线程id**
```
ret = proc->pid;
```
将返回值ret设置为新进程的PID。

#### ucore是否做到给每个新fork的线程一个唯一的id？
是。在`get_pid`函数中，`next_safe` 下一个可安全分配的PID，即不会与现有进程冲突的PID。`last_pid` 上次分配的PID，作为本次分配的起点。在PID分配逻辑中，每次`get_pid`调用时，从`last_pid+1`开始寻找下一个可分配的PID，并检查是否需要重置 `next_safe` 和跳转到 `repeat`。如果 `last_pid `达到 `MAX_PID`，则重置为1，准备从头开始查找可用的PID，如果当前进程的PID大于 `last_pid `且 `next_safe `大于当前进程的PID，则更新 `next_safe`。

### EX3
proc_run用于将指定的进程切换到CPU上运行。它的大致执行步骤包括：

检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。
禁用中断。你可以使用/kern/sync/sync.h中定义好的宏local_intr_save(x)和local_intr_restore(x)来实现关、开中断。
切换当前进程为要运行的进程。
切换页表，以便使用新进程的地址空间。/libs/riscv.h中提供了lcr3(unsigned int cr3)函数，可实现修改CR3寄存器值的功能。
实现上下文切换。/kern/process中已经预先编写好了switch.S，其中定义了switch_to()函数。可实现两个进程的context切换。
允许中断。


#### `proc_run` 函数分析：

 1. **检查是否切换进程**
```c
if (proc != current) 
```
如果目标进程 `proc` 不是当前进程 `current`，才会进行上下文切换。如果目标进程就是当前进程，不需要切换，直接返回即可。

2. **禁用中断**
```c
local_intr_save(intr_flag);
```
禁用本地的中断并保存当前中断状态，保存在 `intr_flag` 变量中）。禁用中断的目的是防止在上下文切换期间发生中断，从而避免引发竞态条件或数据不一致的问题。

3. **保存当前进程并设置新的当前进程**
```c
struct proc_struct *prev = current, *next = proc;
current = proc;
```
在上下文切换前，保存当前进程 `prev` 和即将切换到的目标进程 `next`。然后将 `current` 设置为目标进程 `proc`。

4. **更新 CR3 寄存器**
```c
lcr3(next->cr3);
```
`lcr3` 函数修改了 CR3 寄存器的值。CR3 寄存器指向当前进程的页目录表（Page Directory Table，PDT）。切换到新的进程时，需要将 CR3 寄存器指向目标进程的页目录表，确保内存访问使用的是目标进程的虚拟地址空间。

 5. **执行上下文切换**
```c
switch_to(&(prev->context), &(next->context));
```
`switch_to` 函数实现了上下文切换。它将当前进程的上下文（寄存器状态、栈指针等）保存，并将目标进程的上下文恢复。

6. **恢复中断**
```c
local_intr_restore(intr_flag);
```
最后，调用 `local_intr_restore` 恢复之前保存的中断状态。


请回答如下问题：

在本实验的执行过程中，创建且运行了几个内核线程？

本次实验共建立了两个内核线程。
idleproc：最初的内核线程，完成内核中各个子线程的创建以及初始化。之后循环执行调度，执行其他进程。
initproc：该线程主要是为了显示实验的完成而打印出字符串"hello world"的内核线程。


完成代码编写后，编译并运行代码：make qemu




### 总结

1. **进程状态：**
   - `PROC_UNINIT`：未初始化状态，通常在 `alloc_proc` 函数中分配进程时使用。
   - `PROC_SLEEPING`：睡眠状态，它正在等待某些事件的发生或某些资源的可用。
   - `PROC_RUNNABLE`：进程处于可运行状态时，表示它已经准备好被调度器调度执行。此时，进程可能正在运行，也可能只是等待调度。
   - `PROC_ZOMBIE`：当进程执行完毕并退出时，进程进入僵尸状态。处于僵尸状态的进程已经完成了所有工作，但仍然存在于进程表中，等待父进程回收其资源。

2. **进程管理相关函数：**
   - `alloc_proc`：分配并初始化一个新的 `proc_struct` 进程结构体。
   - `set_proc_name`：设置进程的名称。
   - `get_proc_name`：获取进程的名称。

3. **进程的生命周期管理：**
   - `proc_run`：将进程设置为可运行状态，并进行上下文切换（切换到该进程）。
   - `do_fork`：创建一个新进程（或线程），并通过复制父进程的状态来初始化子进程。
   - `do_exit`：处理进程退出，释放资源并将进程设置为僵尸状态。
   
4. **进程调度：**
   - `wakeup_proc`：将进程从睡眠状态变为可运行状态，允许调度器调度。
   - `scheduler`：负责进程的调度和上下文切换。

5. **内存管理：**
   - `setup_kstack`：为进程分配内核栈。
   - `put_kstack`：释放进程的内核栈。

6. **哈希表和进程列表：**
   - `hash_proc`：将进程添加到哈希表中，方便根据进程 ID 查找进程。
   - `find_proc`：根据进程 ID 查找进程。
  
- **kernel_thread 函数**
  创建一个内核线程，并使其执行指定的函数 fn，该函数将通过 kernel_thread_entry 作为入口点开始执行。函数内部通过设置 trapframe 结构体来配置线程执行的上下文，然后调用 do_fork 创建线程。


- **`alloc_proc` 函数：**  
  分配一个 `proc_struct` 结构体，并初始化相关字段。这些字段包括进程的状态、PID、父进程、内核栈、上下文等。

- **`do_fork` 函数：**  
  实现了进程的创建。它通过分配新的 `proc_struct`、分配内核栈、复制进程的内存管理信息、设置上下文和 trapframe 来初始化新进程。之后，将新进程加入到进程列表和哈希表中，并将其设置为可运行状态。

- **`proc_run` 函数：**  
  当需要切换到某个进程时，调用 `proc_run` 函数来进行上下文切换。通过 `switch_to` 函数完成从当前进程到目标进程的上下文切换。

- **`do_exit` 函数：**  
  处理进程退出，释放与进程相关的资源，设置进程为僵尸状态，并通知父进程回收子进程资源。

#### 进程和线程的关系：
在 ucore 中，线程是特殊类型的进程，多个线程可以共享同一进程的内存空间。因此，线程和进程的管理机制是类似的，`kernel_thread` 函数用于创建一个新的内核线程。

#### 内核线程的创建：
通过 `kernel_thread` 函数，可以创建一个新的内核线程，并设置该线程的入口函数和参数。这个函数会调用 `do_fork` 来创建进程，并通过 `copy_thread` 设置进程的上下文。
