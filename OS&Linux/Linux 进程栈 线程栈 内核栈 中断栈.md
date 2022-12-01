# Linux 进程栈|线程栈|内核栈|中断栈

栈是计算机科学种最为常见的一种数据结构。计算机便是利用栈结构“先进后出”的特性，实现了函数的调用。

## 函数调用与栈

让我们来考虑一下如何实现一个函数的调用。要实现某个函数的调用，首先最为重要的便是知道这个函数的入口地址。毕竟函数的调用到最后是一句`call xxxxxx` ，函数入口在编译过程中会被确定下来，所以在编译后的程序中，所有的函数地址不需要我们太多的考虑。在找到了函数入口后，我们还需要考虑函数参数的传递。函数的调用必须是高效的，而数据存放在 **CPU通用寄存器** 或者 **RAM 内存** 中无疑是最好的选择。假设我们选择使用 CPU通用寄存器 来存放调用函数时的参数，摆在我们面前首要的问题就是通用寄存器的数目是非常有限的，当出现函数嵌套调用时，子函数很有可能没有足够的通用寄存器来存储所有的函数参数。所以我们选择将 函数参数 保存在RAM内存中。

在搞定了函数的入口与参数后，我们便可以很好地运行某段函数地代码了。但此时我们如何在结束函数的调用后回到原来的执行地址呢？这就引出了另一个问题，我们如何保存调用函数时的上下文。

这种情况下，栈无疑提供很好的解决办法。一、对于通用寄存器传参的冲突，我们可以再调用子函数前，将通用寄存器临时压入栈中；在子函数调用完毕后，在将已保存的寄存器再弹出恢复回来。二、而局部变量的空间申请，也只需要向下移动下栈顶指针；将栈顶指针向回移动，即可就可完成局部变量的空间释放；三、对于函数的返回，也只需要在调用子函数前，将返回地址压入栈中，待子函数调用结束后，将函数返回地址弹出给 PC 指针，即完成了函数调用的返回；

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled.png)

可以看到，栈结构只需要通过栈指针的简单操作，就完整保存了我们函数调用所需的上下文。

## 多任务支持

现在，我们的计算机已经能够支持程序函数的调用了！但是对于一个SMP(Symmetric Multi Processing)系统（对称多处理系统），我们需要面临多进程与多线程的切换问题。

因为 **进程的切换 = 资源切换 + 指令执行序列切换**。将资源和指令序列分开看，如果只是从一个执行指令序列切换到另一个执行指令序列，那么这就是线程的切换。所以后面的内容会将执行序列作为切换的单位。

对于**多个执行序列的切换**，一个栈能够保证上下文的完整吗？我们不妨来做一个模拟(以下是两个执行序列100、300的函数调用)：

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%201.png)

### 两个执行序列一个栈

假设现在仅有一个栈，从 100 开始执行，在 A 函数中遇到 B 函数的调用，B 的返回地址即下一句指令的地址 104 压栈，又在 B 中遇到 Yield() 调用，Yield() 的返回地址 204 压栈。在 B 中的 Yield()jmp 到 300，执行 C 函数，在 C 中遇到 D() 调用，304 压栈，执行 D(), 遇到 D 中的 Yield() 调用，Yield() 的返回地址 404 压栈。

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%202.png)

D 中的 Yield() 执行后，跳到 204。

204 开始执行遇到”}”，会弹栈，栈顶元素 404 被弹出执行 404。这里就出现问题了，现在我们在 100  的执行序列中，如果函数 B() 执行结束，我们期望的结果是回到 104 即函数 A()中继续执行，但当前的栈会回来 404 继续执行。我们刚才已经回到 100 那个线程了，现在却又切到了 300 那个线程。所以两个序列一个栈是不行的。

### 两个执行序列两个栈

假设此时有两个栈，即一个执行序列对应一个栈，每次 yield() 会先切换栈。两个执行序列栈的状态将如下图所示：

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%203.png)

```cpp
//D中的Yield
void Yield()
{
   TCB2.esp=esp;
   esp=TCB1.esp;
}
```

假设 D() 执行完 yield() 时，此时栈指针已经完成了切换 `esp=1000;` 然后遇到 yield() 结尾的 "}" ，弹栈。此时弹栈弹出栈顶元素 204，执行 204 (“}” 相当于 ret)，再 ret，弹栈就是 104 了。

可以看到两个执行序列在两个栈的情况下是能够正常工作的。

所以两个线程的样子：两个 TCB、两个栈、切换的 PC 在栈中。

而 ThreadCreate 的核心就是申请一个栈，一个 TCB(线程控制块)，再将栈与 TCB 关联。

```cpp
void ThreadCreate(A)
{
   TCB* tcb=malloc();
   *stack=malloc();
   *stack=A;//100 执行线程的起始地址
   tcb.esp=stack;//栈和TCB关联
}
```

## 进程栈|线程栈|内核栈|中断栈

内核根据切换目的的不同将栈分为四种：进程栈、线程栈、内核栈、中断栈。

### 进程栈

进程栈是属于用户态栈，和进程 虚拟地址空间 (Virtual Address Space) 密切相关。那我们先了解下什么是虚拟地址空间，在 32 位机器下，虚拟地址空间大小为 4G。这些虚拟地址通过页表 (Page Table) 映射到物理内存，页表由操作系统维护，并被处理器的内存管理单元 (MMU) 硬件引用。**每个进程都拥有一套属于它自己的页表，因此对于每个进程而言都好像独享了整个虚拟地址空间。**

Linux 内核将这 4G 字节的空间分为两部分，将最高的 1G 字节（0xC0000000-0xFFFFFFFF）供内核使用，称为 **内核空间**。而将较低的3G字节（0x00000000-0xBFFFFFFF）供各个进程使用，称为 **用户空间**。每个进程可以通过系统调用陷入内核态，因此内核空间是由所有进程共享的。虽然说内核和用户态进程占用了这么大地址空间，但是并不意味它们使用了这么多物理内存，仅表示它可以支配这么大的地址空间。它们是根据需要，将物理内存映射到虚拟地址空间中使用。

请看这篇：[虚拟内存(内核空间&用户空间)](%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98(%E5%86%85%E6%A0%B8%E7%A9%BA%E9%97%B4&%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4)%20919e35135f674d998286e9d031aa158a.md) 以及程序在内存中的布局：[程序在内存中的存储位置](%E7%A8%8B%E5%BA%8F%E5%9C%A8%E5%86%85%E5%AD%98%E4%B8%AD%E7%9A%84%E5%AD%98%E5%82%A8%E4%BD%8D%E7%BD%AE%20ca4bdf90aeab41ae963add936a0a0241.md) 

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%204.png)

Linux 对进程地址空间有个标准布局，地址空间中由各个不同的内存段组成 (Memory Segment)，主要的内存段如下：

- 程序段 (Text Segment)：可执行文件代码的内存映射
- 数据段 (Data Segment)：可执行文件的已初始化全局变量的内存映射
- BSS段 (BSS Segment)：未初始化的全局变量或者静态变量（用零页初始化）
- 堆区 (Heap) : 存储动态内存分配，匿名的内存映射
- 栈区 (Stack) : 进程用户空间栈，由编译器自动分配释放，存放函数的参数值、局部变量的值等
- 映射段(Memory Mapping Segment)：任何内存映射文件

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%205.png)

而上面进程虚拟地址空间中的栈区，正指的是我们所说的进程栈。进程栈的初始化大小是由编译器和链接器计算出来的，但是栈的实时大小并不是固定的，Linux 内核会根据入栈情况对栈区进行动态增长（其实也就是添加新的页表）。但是并不是说栈区可以无限增长，它也有最大限制 RLIMIT_STACK (一般为 8M)，我们可以通过 ulimit 来查看或更改 RLIMIT_STACK的值。

【扩展阅读】：如何确认进程栈的大小

我们要知道栈的大小，那必须得知道栈的起始地址和结束地址。**栈起始地址**获取很简单，只需要嵌入汇编指令获取栈指针 esp 地址即可。**栈结束地址**的获取有点麻烦，我们需要先利用递归函数把栈搞溢出了，然后再 GDB 中把栈溢出的时候把栈指针 esp 打印出来即可。代码如下：

```cpp
void *orig_stack_pointer;

void blow_stack() {
    blow_stack();
}

int main() {
    __asm__("movl %esp, orig_stack_pointer");

    blow_stack();
    return 0;
}

$ g++ -g stacksize.c -o ./stacksize
$ gdb ./stacksize
(gdb) r
Starting program: /home/home/misc-code/setrlimit

Program received signal SIGSEGV, Segmentation fault.
blow_stack () at setrlimit.c:4
4       blow_stack();
(gdb) print (void *)$esp
$1 = (void *) 0xffffffffff7ff000
(gdb) print (void *)orig_stack_pointer
$2 = (void *) 0xffffe340
(gdb) print 0xffffe340-0xff7ff000
$3 = 8385344    // Current Process Stack Size is 8M
```

内核使用内存描述符来表示进程的地址空间，该描述符表示着进程所有地址空间的信息。内存描述符由 mm_struct 结构体表示，下面给出内存描述符结构中各个域的描述，请大家结合前面的 进程内存段布局 图一起看，具体请看[Linux 进程控制块PCB](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4.md) ：

```cpp
struct mm_struct {
    struct vm_area_struct *mmap;           /* 内存区域链表 */
    struct rb_root mm_rb;                  /* VMA 形成的红黑树 */
    ...
    struct list_head mmlist;               /* 所有 mm_struct 形成的链表 */
    ...
    unsigned long total_vm;                /* 全部页面数目 */
    unsigned long locked_vm;               /* 上锁的页面数据 */
    unsigned long pinned_vm;               /* Refcount permanently increased */
    unsigned long shared_vm;               /* 共享页面数目 Shared pages (files) */
    unsigned long exec_vm;                 /* 可执行页面数目 VM_EXEC & ~VM_WRITE */
    unsigned long stack_vm;                /* 栈区页面数目 VM_GROWSUP/DOWN */
    unsigned long def_flags;
    unsigned long start_code, end_code, start_data, end_data;    /* 代码段、数据段 起始地址和结束地址 */
    unsigned long start_brk, brk, start_stack;                   /* 栈区 的起始地址，堆区 起始地址和结束地址 */
    unsigned long arg_start, arg_end, env_start, env_end;        /* 命令行参数 和 环境变量的 起始地址和结束地址 */
    ...
    /* Architecture-specific MM context */
    mm_context_t context;                  /* 体系结构特殊数据 */

    /* Must use atomic bitops to access the bits */
    unsigned long flags;                   /* 状态标志位 */
    ...
    /* Coredumping and NUMA and HugePage 相关结构体 */
};
```

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%206.png)

- **进程栈的动态增长实现**
    
    进程在运行的过程中，通过不断向栈区压入数据，当超出栈区容量时，就会耗尽栈所对应的内存区域，这将触发一个**缺页异常 (page fault)**。通过异常陷入内核态后，异常会被内核的 expand_stack() 函数处理，进而调用 acct_stack_growth() 来检查是否还有合适的地方用于栈的增长。
    
    如果栈的大小低于 RLIMIT_STACK（通常为8MB），那么一般情况下栈会被加长，程序继续执行，感觉不到发生了什么事情，这是一种将栈扩展到所需大小的常规机制。然而，如果达到了最大栈空间的大小，就会发生**栈溢出（stack overflow）**，进程将会收到内核发出的**段错误（segmentation fault）**信号。
    
    动态栈增长是唯一一种访问未映射内存区域而被允许的情形，其他任何对未映射内存区域的访问都会触发页错误，从而导致段错误。一些被映射的区域是只读的，因此企图写这些区域也会导致段错误。
    

### 线程栈

从 Linux 内核的角度来说，其实它并没有线程的概念。Linux 把所有线程都当做进程来实现，它将线程和进程不加区分的统一到了 task_struct 中。**线程仅仅被视为一个与其他进程共享某些资源的进程，而是否共享地址空间几乎是进程和 Linux 中所谓线程的唯一区别**。线程创建的时候，加上了 CLONE_VM 标记，这样 **线程的内存描述符 将直接指向 父进程的内存描述符**。

```cpp
if (clone_flags & CLONE_VM) {
    /*
     * current 是父进程而 tsk 在 fork() 执行期间是共享子进程
     */
    atomic_inc(¤t->mm->mm_users);
    tsk->mm = current->mm;
}
```

虽然线程的地址空间和进程一样，但是对待其地址空间的 stack 还是有些区别的。对于 **Linux 进程或者说主线程**，其 stack 是在 fork 的时候生成的，实际上就是复制了父亲的 stack 空间地址，然后写时拷贝 (cow) 以及动态增长。然而对于**主线程生成的子线程而言**，其 stack 将不再是这样的了，而是**事先固定下来的**，使用 mmap 系统调用，它不带有 VM_STACK_FLAGS 标记。这个可以从 glibc 的nptl/allocatestack.c 中的 allocate_stack() 函数中看到：

```cpp
mem = mmap (NULL, size, prot,
            MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
```

线程的栈空间的最大值在线程创建的时候就已经定下来了，如果栈的大小超过个了个值，系统将访问未授权的内存块，毫无疑问，再来的肯定是一个段错误。pthread_create() 创建线程时，若不指定分配堆栈大小，系统会分配默认值，默认值可以通过 `ulimit -a`  查看，一般默认线程栈大小为8M.

由于线程的 mm->start_stack 栈地址和所属进程相同，所以线程栈的起始地址并没有存放在 task_struct 中，应该是使用 pthread_attr_t 中的 stackaddr 来初始化 task_struct->thread->sp（sp 指向 struct pt_regs 对象，该结构体用于保存用户进程或者线程的寄存器现场）。这些都不重要，重要的是，**线程栈不能动态增长，一旦用尽就没了，这是和生成进程的 fork 不同的地方**。由于线程栈是从进程的地址空间中 map 出来的一块内存区域，原则上是线程私有的。但是同一个进程的所有线程生成的时候浅拷贝生成者的 task_struct 的很多字段，其中包括所有的 vma，如果愿意，其它线程也还是可以访问到的，于是一定要注意。

### 进程内核栈

在每一个进程的生命周期中，必然会通过到系统调用陷入内核。在执行系统调用陷入内核之后，这些**内核代码所使用的栈并不是原先进程用户空间中的栈，而是一个单独内核空间的栈**，这个称作进程内核栈。进程内核栈在进程创建的时候，通过 slab 分配器从 thread_info_cache 缓存池中分配出来，其大小为 THREAD_SIZE，一般来说是一个页大小 4K；

```cpp
union thread_union {                                   
        struct thread_info thread_info;                
        unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

thread_union 进程内核栈 和 task_struct 进程描述符有着紧密的联系。由于内核经常要访问 task_struct，高效获取当前进程的描述符是一件非常重要的事情。**因此内核将进程内核栈的头部一段空间，用于存放 thread_info 结构体**，而此结构体中则记录了对应进程的描述符，两者关系如下图（对应内核函数为 dup_task_struct()）：

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%207.png)

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%208.png)

有了上述关联结构后，内核可以先获取到栈顶指针 esp，然后通过 esp 来获取 thread_info。这里有一个小技巧，直接将 esp 的地址与上 ~(THREAD_SIZE - 1) 后即可直接获得 thread_info 的地址。由于 thread_union 结构体是从 thread_info_cache 的 Slab 缓存池中申请出来的，而 thread_info_cache 在 kmem_cache_create 创建的时候，保证了地址是 THREAD_SIZE 对齐的。因此只需要对栈指针进行 THREAD_SIZE 对齐，即可获得 thread_union 的地址，也就获得了 thread_union 的地址。成功获取到 thread_info 后，直接取出它的 task 成员就成功得到了 task_struct。其实上面这段描述，也就是 current 宏的实现方法：

```cpp
register unsigned long current_stack_pointer asm ("sp");

static inline struct thread_info *current_thread_info(void)  
{                                                            
        return (struct thread_info *)                        
                (current_stack_pointer & ~(THREAD_SIZE - 1));
}                                                            

#define get_current() (current_thread_info()->task)

#define current get_current()
```

我们知道从内核转到用户态时用户栈的地址是在陷入内核的时候保存在内核栈里面的，但是在陷入内核的时候，我们是如何知道内核栈的地址的呢？

关键在进程从用户态转到内核态的时候，进程的内核栈总是空的。这是因为，当进程在用户态运行时，使用的是用户栈，当进程陷入到内核态时，内核栈保存进程在内核态运行的相关信心，但是一旦进程返回到用户态后，内核栈中保存的信息无效，会全部恢复，因此每次进程从用户态陷入**内核的时候得到的内核栈都是空的**。所以在进程陷入内核的时候，直接把内核栈的栈顶地址给堆栈指针寄存器就可以了。

### 中断栈

进程陷入内核态的时候，需要内核栈来支持内核函数调用。中断也是如此，当系统收到中断事件后，进行中断处理的时候，也需要中断栈来支持函数调用。由于系统中断的时候，系统当然是处于内核态的，所以中断栈是可以和内核栈共享的。但是具体是否共享，这和具体处理架构密切相关。

X86 上中断栈就是独立于内核栈的；独立的中断栈所在内存空间的分配发生在 arch/x86/kernel/irq_32.c 的 irq_ctx_init()函数中(**如果是多处理器系统，那么每个处理器都会有一个独立的中断栈**)，函数使用 __alloc_pages 在低端内存区分配 **2个物理页面**，也就是8KB大小的空间。有趣的是，这个函数还会为 softirq 分配一个同样大小的独立堆栈。如此说来，softirq 将不会在 hardirq 的中断栈上执行，而是在自己的上下文中执行。

![Untitled](Linux%20%E8%BF%9B%E7%A8%8B%E6%A0%88%20%E7%BA%BF%E7%A8%8B%E6%A0%88%20%E5%86%85%E6%A0%B8%E6%A0%88%20%E4%B8%AD%E6%96%AD%E6%A0%88%20af1be01df9c64ca284343cd634cd795d/Untitled%209.png)

而 ARM 上中断栈和内核栈则是共享的；中断栈和内核栈共享有一个负面因素，如果中断发生嵌套，可能会造成栈溢出，从而可能会破坏到内核栈的一些重要数据，所以栈空间有时候难免会捉襟见肘

## Q&A

### 内核态为什么需要独立的内核栈？

原因是**避免高特权（比如 内核态）栈内存不被低特权（比如 用户态）任意修改**。如果将内核态的栈与用户态栈共用，那么当切换到用户态时，用户态就可以修改到内核态的数据。

### 为什么需要单独的进程内核栈？

这个问题实际上在第二节“多任务支持”中已经解释，为了保证任务执行的正确性，一个执行序列需要一个栈。所有进程运行的时候，都可能通过系统调用陷入内核态继续执行。假设第一个进程 A 陷入内核态执行的时候，需要等待读取网卡的数据，主动调用 schedule() 让出 CPU；此时调度器唤醒了另一个进程 B，碰巧进程 B 也需要系统调用进入内核态。那问题就来了，如果内核栈只有一个，那进程 B 进入内核态的时候产生的压栈操作，必然会破坏掉进程 A 已有的内核栈数据；一但进程 A 的内核栈数据被破坏，很可能导致进程 A 的内核态无法正确返回到对应的用户态了；

### 为什么需要单独的线程栈？

看第二节“多任务支持”。

### 进程和线程是否共享一个内核栈？

不共享，可以这么想，对于内核来说进程与线程都是一个task_struct，task_struct中有thread_info*的指针指向内核栈。而thread_info有一个task_struct*指针指向进程描述符。可以看出 tread_info 与 task_struct 必须是一一对应的。如果多个线程共享一个内核栈，在内核栈取进程描述符task_struct将会造成混乱。

线程和进程创建的时候都调用 dup_task_struct 来创建 task 相关结构体，而内核栈也是在此函数中 alloc_thread_info_node 出来的。因此虽然线程和进程共享一个地址空间 mm_struct，但是并不共享一个内核栈。

### 为什么需要单独中断栈？

这个问题其实不对，ARM 架构就没有独立的中断栈