### 练习0：填写已有实验
本实验依赖实验2/3/4。请把你做的实验2/3/4的代码填入本实验中代码中有“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意：为了能够正确执行lab5的测试应用程序，可能需对已完成的实验2/3/4的代码进行进一步改进。
#### `alloc_proc`函数的lab4部分与lab5添加
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
	
     //LAB5 YOUR CODE : (update LAB4 steps)
     /*
     * below fields(add in LAB5) in proc_struct need to be initialized  
     *       uint32_t wait_state;                        // waiting state
     *       struct proc_struct *cptr, *yptr, *optr;     // relations between processes
     */
        proc->wait_state = 0;
        proc->cptr = NULL;//没有子进程
        proc->optr = NULL;//没有老兄弟进程
        proc->yptr = NULL;//没有年轻兄弟进程
        
    }
    return proc;
}
```
#### `proc_run`函数的lab4部分
```
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        // LAB4:EXERCISE3 YOUR CODE
        /*
        * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
        * MACROs or Functions:
        *   local_intr_save():        Disable interrupts
        *   local_intr_restore():     Enable Interrupts
        *   lcr3():                   Modify the value of CR3 register
        *   switch_to():              Context switching between two processes
        */
        
       bool intr_flag;
       //之前执行的进程是current，即将执行的是proc
       struct proc_struct *prev = current, *next = proc;
       //禁用中断
       local_intr_save(intr_flag);
       //正在运行的进程修改为proc
        current = proc;
        //将 CR3 寄存器指向目标进程的页目录表
        lcr3(next->cr3);
        //切换上下文
        switch_to(&(prev->context), &(next->context));
       //恢复中断状态
       local_intr_restore(intr_flag);        

    }
}

```

#### `do_fork`函数的lab4部分与lab5添加
```
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
 
    //LAB5 YOUR CODE : (update LAB4 steps)
    //TIPS: you should modify your written code in lab4(step1 and step5), not add more code.
   /* Some Functions
    *    set_links:  set the relation links of process.  ALSO SEE: remove_links:  lean the relation links of process 
    *    -------------------
    *    update step 1: set child proc's parent to current process, make sure current process's wait_state is 0
    *    update step 5: insert proc_struct into hash_list && proc_list, set the relation links of process
    */
    
    proc = alloc_proc();
    
    if (proc == NULL) {
        goto fork_out;
    }
    
    current->wait_state = 0; //lab5添加 设置当前进程的wait_state为0

    proc->parent = current;

    if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }

    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_proc;
    }

    copy_thread(proc, stack, tf);

    bool intr_flag;
    local_intr_save(intr_flag);
    
    proc->pid = get_pid();
    hash_proc(proc);
    //set_links里已经做了,下两行注释掉
    //list_add(&proc_list, &(proc->list_link));
    //nr_process++;
    set_links(proc); //lab5添加 链接进程间的关系

    local_intr_restore(intr_flag);

    wakeup_proc(proc);

    ret = proc->pid;
 
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```



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

### 练习2: 父进程复制自己的内存空间给子进程（需要编码）

创建子进程的函数`do_fork`在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过`copy_range`函数（位于kern/mm/pmm.c中）实现的，请补充`copy_range`的实现，确保能够正确执行。

#### `copy_range`的实现

`copy_range`函数的调用过程：do_fork()---->copy_mm()---->dup_mmap()---->copy_range()。

1.`do_fork`:过程和lab4一样，都是创建一个进程，并放入CPU中调度。

2.`copy_mm`：`copy_mm`函数在LAB4中没有实现,本次重点实现如何完成从父进程把内存复制到子进程中。使用互斥锁（用于避免多个进程同时访问内存）。进下一层的调用`dup_mmap`函数。

3.`dup_mmap`接受两个参数，前一个mm是待复制的内存，而复制的源内容在oldmm（父进程）内容中。只是完成了新进程中的段创建。具体行为最终落到`copy_range`中。

4.`copy_range`:用于在内存页级别上复制一段地址范围的内容。

首先，它通过 `get_pte` 函数获取源页表中的页表项，并检查其有效性。然后它在目标页表中获取或创建新的页表项，并为新的页分配内存。最后，它确保源页和新页都成功分配，并准备进行复制操作，也就是我们需要完成的。主要分为四步：

首先通过刚开始获取的page即源page通过宏page2kva转换为源虚拟内存地址。

同样的将要复制过去的n个page转换为目的虚拟内存地址。

通过memcpy将虚拟地址进行复制，复制其内容。

最后我们再使用前面的参数（to是目标进程的页目录地址，npage是页，start是起始地址，perm是提取出的页目录项ptep中的`PTE_USER`即用户级别权限相关的位）调用`page_insert`函数。

```c
void *src_kvaddr = page2kva(page); 

void *dst_kvaddr = page2kva(npage); 

memcpy(dst_kvaddr, src_kvaddr, PGSIZE); 

ret = page_insert(to, npage, start, perm); 
```

#### 如何设计实现Copy on Write机制？

1. 判断是否是资源共享：当多个任务读取同一资源时，它们共享对该资源的访问。可以通过`copy_range`的`share`参数实现，在`copy_range`中要根据`share`参数判断一下是应该复制还是共享。

2. 检测写操作：对于共享资源，当一个任务试图写入时，系统需要捕获这个操作。当任务试图写入一个标记为只读的内存区域时，通过内存保护硬件实现，硬件触发一个异常。可以通过定义一个新的`trap`类型，然后到`trap.c`的`exception_handler`中对应处理。

3. 资源复制：对于上述的写操作，分配新的内存或磁盘空间，并将原始资源的内容复制到新分配的空间中。可以调用`copy_range`实现。

4. 更新指针：将写入资源的任务的指针更新为指向复制的资源，然后允许写操作继续进行。这样写操作只影响新复制的资源，而不影响其他任务看到的原始资源。


### 练习3
请分析fork/exec/wait/exit的执行流程。重点关注哪些操作是在用户态完成，哪些是在内核态完成？内核态与用户态程序是如何交错执行的？内核态执行结果是如何返回给用户程序的？

以exit为例
exit(user/lib/ulib.c)

sys_exit(user/lib/syscall)

sys_call(user/lib/syscall)

- 内链汇编里ecall语句，此时提权
- 抛出一个异常

exception_handle(kern/trap)

syscall(kern/syscall)

sys_exit()

do_exit()

返回：tf->gpr->a0
trap返回的时候调用了sret指令

PROC_UNINIT -- alloc_proc --> PROC_RUNNABLE -- do_fork --> PROC_RUNNABLE
                                      |                          |
                                      |                          |
                                      v                          v
                              PROC_RUNNABLE -- do_wait --> PROC_SLEEPING
                                      |                          |
                                      |                          |
                                      v                          v
                              PROC_RUNNABLE -- do_exit --> PROC_ZOMBIE
                                      |                          |
                                      |                          |
                                      v                          v
                              PROC_RUNNABLE -- wakeup_proc --> PROC_RUNNABLE

SPP 位：sstatus 寄存器的第 8 位，用于指示进入 Supervisor 模式之前的特权级别。

0：进入 Supervisor 模式之前处于 User 模式。

1：进入 Supervisor 模式之前处于 Supervisor 模式。

sepc 寄存器用于保存触发异常的指令地址。

在异常处理时，sscratch 用于保存当前的栈指针（sp），以便在内核栈和用户栈之间切换。

    #sp为0，说明之前是内核态，我们刚才把内核栈指针换到了sscratch, 需要再拿回来
    #sp不为0 时，说明之前是用户态，sp里现在存的就是内核栈指针，sscratch里现在是用户栈指针

### 需要理解的点
#### memlayout

---

##### **物理内存映射**
- **KERNBASE (0xFFFFFFFFC0200000)**：所有物理内存映射的起始地址。
- **KMEMSIZE (0x7E00000)**：物理内存的最大大小。
- **KERNEL_BEGIN_PADDR (0x80200000)**：内核物理地址的起始地址。
- **KERNEL_BEGIN_VADDR (0xFFFFFFFFC0200000)**：内核虚拟地址的起始地址。
- **PHYSICAL_MEMORY_END (0x88000000)**：物理内存的结束地址。

---

##### **内核栈和用户栈**
- **内核栈**：
  - **KSTACKPAGE (2)**：内核栈占用的页面数。
  - **KSTACKSIZE (KSTACKPAGE * PGSIZE)**：内核栈的大小。

- **用户栈**：
  - **USTACKPAGE (256)**：用户栈占用的页面数。
  - **USTACKSIZE (USTACKPAGE * PGSIZE)**：用户栈的大小。

---

##### **用户空间定义**
- **USERBASE (0x00200000)**：用户空间的起始地址。
- **UTEXT (0x00800000)**：用户程序通常的起始地址。
- **USTAB (USERBASE)**：用户 STAB 数据结构的位置。

---

##### **访问权限检查**
- **USER_ACCESS(start, end)**：检查地址范围是否在用户空间内。
- **KERN_ACCESS(start, end)**：检查地址范围是否在内核空间内。


---

##### **虚拟内存映射**

1. **4GB 虚拟地址空间**：
   - 整个虚拟地址空间从 `0x00000000` 到 `0xFFFFFFFF`，共 4GB。

2. **空闲内存（Empty Memory）**：
   - 从 `0xFB000000` 到 `0xFFFFFFFF`，这部分内存通常未映射，但用户程序可以根据需要映射页面。

3. **当前页表（Cur. Page Table）**：
   - 从 `0xFAC00000` 到 `0xFB000000`，这部分内存用于存储当前的页表，权限为内核读写（RW/--）。

4. **无效内存（Invalid Memory）**：
   - 从 `0xF8000000` 到 `0xFAC00000`，这部分内存是无效的，内核确保它永远不会被映射。

5. **重映射的物理内存（Remapped Physical Memory）**：
   - 从 `0xC0000000` 到 `0xF8000000`，这部分内存是物理内存的重映射区域，权限为内核读写（RW/--）。

6. **无效内存（Invalid Memory）**：
   - 从 `0xB0000000` 到 `0xC0000000`，这部分内存是无效的，内核确保它永远不会被映射。

7. **用户栈（User Stack）**：
   - 从 `0xB0000000` 开始，这部分内存用于用户栈。

8. **用户程序和堆（User Program & Heap）**：
   - 从 `0x00800000` 开始，这部分内存用于用户程序和堆。

9. **用户 STAB 数据（User STAB Data）**：
   - 从 `0x00200000` 开始，这部分内存用于存储用户程序的调试信息（可选）。

10. **无效内存（Invalid Memory）**：
    - 从 `0x00000000` 到 `0x00200000`，这部分内存是无效的，内核确保它永远不会被映射。

---

#### pmm


操作系统的内存管理，特别是页表操作、页面分配与释放、以及进程间内存的复制。


##### **1. `page_remove_pte` 函数**
- 从页表中移除一个页表项（PTE），并释放相关的物理页面。 
  1. 检查页表项是否有效（`PTE_V`）。
  2. 通过页表项找到对应的物理页面。
  3. 减少页面的引用计数。
  4. 如果引用计数为 0，释放页面。
  5. 清除页表项。
  6. 刷新 TLB，使修改生效。



---

##### **2. `unmap_range` 函数**
- 
  1. 确保 `start` 和 `end` 是页对齐的。
  2. 确保地址范围在用户空间内。
  3. 遍历范围内的每个页面，调用 `page_remove_pte` 移除页表项。

  - 如果页表项为空，直接跳过。
  - 每次处理一个页面（`PGSIZE`），直到处理完整个范围。

---

##### **3. `exit_range` 函数**
- 释放指定范围内的页表和页目录。
- 
  1. 确保 `start` 和 `end` 是页对齐的。
  2. 确保地址范围在用户空间内。
  3. 遍历一级页目录（PD1）和二级页目录（PD0），释放所有无效的页表。
  4. 如果某个页目录的所有页表项都无效，释放该页目录。

  - 这是一个递归的过程，逐层释放页表和页目录。

---

##### **4. `copy_range` 函数**
- 将一个进程的内存范围复制到另一个进程。
- 
  1. 确保 `start` 和 `end` 是页对齐的。
  2. 确保地址范围在用户空间内。
  3. 遍历范围内的每个页面：
     - 获取源进程的页表项。
     - 为目标进程分配新的页表项。
     - 分配新的物理页面，并将源页面的内容复制到目标页面。
     - 建立目标进程的页表映射。



---

##### **5. `page_remove` 函数**
- 移除指定线性地址的页表项。
- 
  1. 获取指定地址的页表项。
  2. 如果页表项存在，调用 `page_remove_pte` 移除它。

---

##### **6. `page_insert` 函数**
- 将一个物理页面插入到指定线性地址的页表中。
- 
  1. 获取指定地址的页表项。
  2. 如果页表项已经存在：
     - 如果页表项指向的页面与当前页面相同，减少引用计数。
     - 否则，移除旧的页表项。
  3. 创建新的页表项，指向目标页面。
  4. 刷新 TLB。

---

##### **7. `tlb_invalidate` 函数**
- ：刷新 TLB，使页表修改生效。
- 
  - 使用汇编指令 `sfence.vma` 刷新指定地址的 TLB 条目。

---

##### **8. `pgdir_alloc_page` 函数**
- 分配一个页面并将其插入到页表中。
- 
  1. 分配一个物理页面。
  2. 调用 `page_insert` 将页面插入到页表中。
  3. 如果启用了交换机制，将页面标记为可交换。

---

##### **10. 辅助函数**
- **`perm2str`**：将页表项的权限转换为字符串表示。
- **`get_pgtable_items`**：在页目录或页表中查找连续的页表项范围。

---
#### vmm
##### mm数据结构


---

 **1. `list_entry_t mmap_list`**
- *链表头，用于管理进程的虚拟内存区域（VMA，Virtual Memory Area）。
- 
  - 链表中的每个节点是一个 `vma_struct` 结构体，表示一个虚拟内存区域。
  - 链表按照 VMA 的起始地址排序。

---

 **2. `struct vma_struct *mmap_cache`**
- 指向最近访问的 VMA。
- 
  - 用于加速对 VMA 的访问，避免每次都遍历链表。
  - 当进程频繁访问某个 VMA 时，可以通过 `mmap_cache` 快速定位。

---

 **3. `pde_t *pgdir`**
- 指向进程的页目录（Page Directory Table，PDT）。
- 
  - 页目录是进程虚拟地址空间的核心数据结构。
  - 通过页目录，操作系统可以管理进程的虚拟内存到物理内存的映射。

---

 **4. `int map_count`**
- 记录进程中 VMA 的数量。
- 
  - 用于统计进程的虚拟内存区域数量。
  - 当进程的内存映射发生变化时（如分配或释放 VMA），`map_count` 会相应更新。

---

 **5. `void *sm_priv`**
- 指向交换管理器的私有数据。
- 
  - 用于实现页面的换入和换出机制。
  - 不同的交换管理器（如 FIFO、LRU）可以使用 `sm_priv` 存储其内部状态。

---

 **6. `int mm_count`**
- 记录共享该 `mm_struct` 的进程数量。
- 
  - 当多个进程共享同一个 `mm_struct` 时（如通过 `fork` 创建的子进程），`mm_count` 会递增。
  - 当最后一个进程退出时，`mm_count` 会递减到 0，此时可以释放 `mm_struct`。

---

 **7. `lock_t mm_lock`**
- 用于保护 `mm_struct` 的互斥锁。
- 
  - 当多个线程或进程同时访问或修改 `mm_struct` 时，需要使用 `mm_lock` 进行同步。
  - 例如，在调用 `dup_mmap` 函数复制内存映射时，需要加锁以避免竞争条件。

---


这段代码实现了与虚拟内存管理相关的几个函数，主要涉及 VMA（虚拟内存区域）的创建、复制和释放。以下是对这些函数的详细解释：

---

##### **1. `mm_map` 函数**
- 在进程的虚拟地址空间中创建一个新的 VMA。


  1. **地址对齐**：
     - `start = ROUNDDOWN(addr, PGSIZE)`：将起始地址向下对齐到页大小（4KB）。
     - `end = ROUNDUP(addr + len, PGSIZE)`：将结束地址向上对齐到页大小。

  2. **检查地址范围**：
     - 确保地址范围在用户空间内（`USER_ACCESS(start, end)`）。

  3. **查找 VMA**：
     - 调用 `find_vma(mm, start)` 查找是否已经存在一个与 `start` 重叠的 VMA。
     - 如果存在且 `end > vma->vm_start`，说明地址范围冲突，返回错误。

  4. **创建 VMA**：
     - 调用 `vma_create(start, end, vm_flags)` 创建一个新的 VMA。
     - 如果创建失败，返回 `-E_NO_MEM`。

  5. **插入 VMA**：
     - 调用 `insert_vma_struct(mm, vma)` 将新创建的 VMA 插入到 `mm_struct` 的 `mmap_list` 链表中。

  6. **返回结果**：
     - 如果 `vma_store` 不为空，将创建的 VMA 指针存储到 `vma_store` 中。
     - 返回成功（`ret = 0`）。


---

##### **2. `dup_mmap` 函数**
- 复制一个进程的 VMA 到另一个进程。

  - `to`：目标进程的 `mm_struct` 结构体。
  - `from`：源进程的 `mm_struct` 结构体。

- 
  1. **遍历源进程的 VMA 链表**：
     - 使用 `list_entry_t` 遍历 `from->mmap_list` 链表。
     - 每次获取一个 VMA（`vma = le2vma(le, list_link)`）。

  2. **创建新的 VMA**：
     - 调用 `vma_create(vma->vm_start, vma->vm_end, vma->vm_flags)` 创建一个新的 VMA。
     - 如果创建失败，返回 `-E_NO_MEM`。

  3. **插入新 VMA**：
     - 调用 `insert_vma_struct(to, nvma)` 将新创建的 VMA 插入到 `to` 的 `mmap_list` 链表中。

  4. **复制内存内容**：
     - 调用 `copy_range(to->pgdir, from->pgdir, vma->vm_start, vma->vm_end, share)` 复制源进程的内存内容到目标进程。
     - 如果复制失败，返回 `-E_NO_MEM`。

  5. **返回结果**：
     - 如果所有 VMA 都成功复制，返回 `0`。


---

##### **3. `exit_mmap` 函数**
- ：释放进程的所有 VMA 并清理页表。
- **参数**：`mm`：目标进程的 `mm_struct` 结构体。

  1. **检查参数**：
     - 确保 `mm` 不为空，且 `mm_count(mm) == 0`，表示没有其他进程共享该 `mm_struct`。

  2. **遍历 VMA 链表**：
     - 使用 `list_entry_t` 遍历 `mm->mmap_list` 链表。
     - 每次获取一个 VMA（`vma = le2vma(le, list_link)`）。

  3. **取消映射 VMA**：
     - 调用 `unmap_range(pgdir, vma->vm_start, vma->vm_end)` 取消映射 VMA 对应的虚拟地址范围。

  4. **释放页表和页目录**：
     - 调用 `exit_range(pgdir, vma->vm_start, vma->vm_end)` 释放 VMA 对应的页表和页目录。


---

##### **1. `copy_from_user` 函数**
- **功能**：从用户空间复制数据到内核空间。

  1. **检查用户空间内存的合法性**：
     - 调用 `user_mem_check(mm, (uintptr_t)src, len, writable)` 检查用户空间的内存是否可以访问。

  2. **复制数据**：
     - 调用 `memcpy(dst, src, len)` 将用户空间的数据复制到内核空间。



---

##### **2. `copy_to_user` 函数**
- **功能**：从内核空间复制数据到用户空间。

  1. **检查用户空间内存的合法性**：
     - 调用 `user_mem_check(mm, (uintptr_t)dst, len, 1)` 检查用户空间的内存是否可以访问。

  2. **复制数据**：
     - 调用 `memcpy(dst, src, len)` 将内核空间的数据复制到用户空间。


---

##### **3. `user_mem_check` 函数**
- 检查用户空间的内存是否可以访问。
- **参数**：
  - `mm`：目标进程的 `mm_struct` 结构体。
  - `addr`：用户空间的起始地址。
  - `len`：要检查的内存长度。
  - `writable`：表示是否需要检查内存的可写权限。

  1. **遍历 VMA 链表**：
     - 检查用户空间的地址范围是否在某个 VMA 的范围内。
     - 如果地址范围不在任何 VMA 范围内，返回 `0`。

  2. **检查权限**：
     - 如果 `writable` 为 `1`，检查 VMA 是否具有写权限（`VM_WRITE`）。
     - 如果权限不足，返回 `0`。

  3. **返回结果**：
     - 如果地址范围和权限都合法，返回 `1`。

---

#### proc

`set_links` 和 `remove_links` 是用于管理进程间关系的函数。它





- **`set_links`**：用于在进程创建时设置进程的父子关系和兄弟关系。
  - 将进程插入全局进程列表。
  - 设置进程的老兄弟进程和年轻兄弟进程。
  - 更新父进程的当前子进程。
  - 更新全局进程计数。

- **`remove_links`**：用于在进程退出时清理进程的父子关系和兄弟关系。
  - 从全局进程列表中删除进程。
  - 更新老兄弟进程和年轻兄弟进程的关系。
  - 更新父进程的当前子进程。
  - 更新全局进程计数。


`setup_pgdir` 和 `put_pgdir` 是用于管理进程页目录表（Page Directory Table，PDT）的函数。


- **`setup_pgdir`**：用于为进程分配一个新的页目录表。
  - 分配一个物理页面作为页目录表。
  - 将启动时的页目录表内容复制到新分配的页目录表中。
  - 设置进程的页目录表。

- **`put_pgdir`**：用于释放进程的页目录表。
  - 将页目录表的内核虚拟地址转换为物理页面。
  - 释放页目录表对应的物理页面。

通过这两个函数，操作系统可以高效地管理进程的页目录表，确保进程的虚拟地址空间在创建和退出时能够正确分配和释放。

`copy_mm` 函数用于在创建新进程或线程时，复制或共享父进程的内存管理结构.
- **`copy_mm`**：用于在创建新进程或线程时，复制或共享父进程的内存管理结构。
  - 如果 `clone_flags` 包含 `CLONE_VM`，则共享父进程的内存管理结构。
  - 否则，创建一个新的内存管理结构，并复制父进程的内存映射。
  - 成功时返回 `0`，失败时返回相应的错误码。


##### load icode
`load_icode` 函数用于将 ELF 格式的二进制程序加载到当前进程的内存空间中。它是操作系统中用于加载用户程序的核心函数。

1 检查当前进程的内存管理结构

- 确保当前进程的内存管理结构 `current->mm` 为空。
- 如果 `current->mm` 不为空，说明当前进程已经有内存映射，无法加载新的程序，触发 `panic`。

**2. 创建新的内存管理结构**
- 调用 `mm_create()` 创建一个新的内存管理结构 `mm`。
 

 **3 创建新的页目录表**

- 调用 `setup_pgdir(mm)` 为新进程分配并初始化页目录表。
- 如果创建失败，跳转到 `bad_pgdir_cleanup_mm` 标签。

 **4 解析 ELF 文件头**

- 将 `binary` 转换为 `elfhdr` 结构体，解析 ELF 文件头。
- 检查 ELF 文件的魔数（`e_magic`）是否为 `ELF_MAGIC`，确保文件是有效的 ELF 格式。
- 如果文件无效，跳转到 `bad_elf_cleanup_pgdir` 标签。

 **5 遍历程序段头**

- 遍历 ELF 文件的程序段头（`proghdr`），找到所有类型为 `ELF_PT_LOAD` 的段。
- 为每个段创建虚拟内存区域（VMA），并设置相应的权限。
- 分配物理页面，并将段的内容从 ELF 文件复制到进程的内存空间。
- 如果段的大小（`p_memsz`）大于文件大小（`p_filesz`），则将剩余部分初始化为零（BSS 段）。

 **6 创建用户栈**
- 为进程创建用户栈，设置栈的虚拟内存区域（VMA）。
- 分配物理页面，初始化用户栈。

 **7 设置当前进程的内存管理结构**

- 增加内存管理结构的引用计数。
- 将当前进程的 `mm` 设置为新创建的 `mm`。
- 将当前进程的 `cr3` 设置为页目录表的物理地址，并加载到 CR3 寄存器。

 **8 设置 Trapframe**

- 设置当前进程的 Trapframe，确保进程可以从内核模式返回到用户模式。
- 设置用户栈指针 `tf->gpr.sp` 为用户栈顶。
- 设置程序入口点 `tf->epc` 为 ELF 文件的入口地址。
- 设置 `tf->status`，确保进程在返回用户模式时启用中断。



#### do wait
`do_wait` 函数是操作系统内核中的一个系统调用处理函数，用于等待子进程结束。它允许父进程等待特定的子进程（通过 `pid` 指定）或任意一个子进程结束，并获取子进程的退出状态。


`do_wait` 函数的主要功能是等待子进程结束。它首先检查用户提供的参数是否合法，然后查找指定的子进程或任意一个子进程，如果找到已结束的子进程（状态为 `PROC_ZOMBIE`），则获取其退出状态并释放相关资源；如果未找到，则将当前进程设置为睡眠状态，等待子进程结束。


1. **获取当前进程的内存管理结构**


2. **检查用户内存是否合法**


3. **查找子进程**

   - 初始化 `haskid` 为 `0`，表示是否存在需要等待的子进程。
   - 如果 `pid` 不为 0，则通过 `find_proc(pid)` 查找指定的子进程。
   - 如果找到的子进程的父进程是当前进程（`proc->parent == current`），则设置 `haskid` 为 `1`。
   - 如果子进程的状态是 `PROC_ZOMBIE`（僵尸进程），则跳转到 `found` 标签。



4. **查找任意一个子进程**

   - 如果 `pid` 为 0，则遍历当前进程的所有子进程（从 `cptr` 开始，通过 `optr` 指针遍历）。
   - 如果找到状态为 `PROC_ZOMBIE` 的子进程，则跳转到 `found` 标签。

5. **等待子进程结束**

   - 如果存在需要等待的子进程（`haskid` 为 `1`），则将当前进程的状态设置为 `PROC_SLEEPING`，并设置等待状态为 `WT_CHILD`。
   - 调用 `schedule()` 函数切换到其他进程执行。
   - 如果当前进程被标记为退出（`current->flags & PF_EXITING`），则调用 `do_exit(-E_KILLED)` 退出当前进程。
   - 重新跳转到 `repeat` 标签，继续查找子进程。

6. **返回错误**

   - 如果没有找到需要等待的子进程，则返回错误码 `-E_BAD_PROC`。

---

##### 找到子进程后的处理

1. **检查特殊进程**

   ```
   - 如果找到的子进程是 `idleproc` 或 `initproc`

2. **获取子进程的退出状态**

   ```
   - 如果 `code_store` 不为 `NULL`，则将子进程的退出状态（`proc->exit_code`）存储到用户提供的内存中。

3. **禁用中断并释放子进程资源**

   - 禁用中断，确保操作的原子性。
   - 调用 `unhash_proc(proc)` 将子进程从进程哈希表中移除。
   - 调用 `remove_links(proc)` 移除子进程的链接关系（例如父子进程关系）。

4. **释放子进程的内核栈和进程结构**

   - 调用 `put_kstack(proc)` 释放子进程的内核栈。
   - 调用 `kfree(proc)` 释放子进程的进程控制块（`proc_struct`）。

5. **返回成功**

   - 返回 0，表示成功等待子进程结束。

#### do kill
`do_kill` 函数是操作系统内核中的一个系统调用处理函数，用于向指定进程发送终止信号（即杀死进程）。它通过设置目标进程的标志位来标记其为退出状态，并唤醒可能正在等待该进程的父进程或其他相关进程。以
1. **查找目标进程**

   - 调用 `find_proc(pid)` 查找指定进程 ID 的进程。
   - 如果找到目标进程（`proc != NULL`），则继续执行。

2. **检查目标进程的退出标志**
   - 检查目标进程的标志位是否已经包含 `PF_EXITING`（表示进程正在退出或已被杀死）。
   - 如果目标进程尚未被标记为退出状态，则继续执行。

3. **设置目标进程的退出标志**

   - 将目标进程的标志位设置为 `PF_EXITING`，表示该进程已被杀死。

4. **唤醒等待目标进程的进程**
   ```
   - 检查目标进程的等待状态是否包含 `WT_INTERRUPTED`（表示有进程正在等待该进程）。
   - 如果存在等待的进程，则调用 `wakeup_proc(proc)` 唤醒这些进程。

5. **返回成功**
   - 返回 0，表示成功向目标进程发送终止信号。

6. **目标进程已被标记为退出状态**

   - 如果目标进程已经被标记为退出状态（`proc->flags & PF_EXITING`），则返回错误码 `-E_KILLED`，表示目标进程已经被杀死
   - 如果未找到目标进程（`proc == NULL`），则返回错误码 `-E_INVAL`，表示无效的进程 ID。

---







