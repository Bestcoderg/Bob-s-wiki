# 进程上下文切换 context_switch()

## context_switch()

linux 中进程调度时, 内核在选择新进程之后进行抢占时, 通过 context_switch 完成进程上下文切换.

context_switch 其实是一个分配器, 他会调用所需的特定体系结构的方法

- 调用 switch_mm(), 把虚拟内存从一个进程映射切换到新进程中
  
    switch_mm 更换通过 task_struct->mm 描述的内存管理上下文, 该工作的细节取决于处理器, 主要包括加载页表, 刷出地址转换后备缓冲器 (部分或者全部), 向内存管理单元(MMU) 提供新的信息
    
- 调用 switch_to(), 从上一个进程的处理器状态切换到新进程的处理器状态。这包括保存、恢复栈信息和寄存器信息
  
    switch_to 切换处理器寄存器的呢内容和内核栈 (虚拟地址空间的用户部分已经通过 switch_mm 变更, 其中也包括了用户状态下的栈, 因此 switch_to 不需要变更用户栈, 只需变更内核栈), 此段代码严重依赖于体系结构, 且代码通常都是用汇编语言编写.
    

context_switch 函数建立 next 进程的地址空间。进程描述符的 active_mm 字段指向进程所使用的内存描述符，而 mm 字段指向进程所拥有的内存描述符。对于一般的进程，这两个字段有相同的地址，但是，内核线程没有它自己的地址空间而且它的 mm 字段总是被设置为 NULL

context_switch( ) 函数保证：如果 next 是一个内核线程, 它使用 prev 所使用的地址空间

由于不同架构下地址映射的机制有所区别, 而寄存器等信息弊病也是依赖于架构的, 因此 switch_mm 和 switch_to 两个函数均是体系结构相关的

```cpp
/*
 * context_switch - switch to the new MM and the new thread's register state.
 */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
           struct task_struct *next)
{
    struct mm_struct *mm, *oldmm;

    /*  完成进程切换的准备工作  */
    prepare_task_switch(rq, prev, next);

    mm = next->mm;
    oldmm = prev->active_mm;
    /*
     * For paravirt, this is coupled with an exit in switch_to to
     * combine the page table reload and the switch backend into
     * one hypercall.
     */
    arch_start_context_switch(prev);

    /*  如果next是内核线程，则线程使用prev所使用的地址空间
     *  schedule( )函数把该线程设置为懒惰TLB模式
     *  内核线程并不拥有自己的页表集(task_struct->mm = NULL)
     *  它使用一个普通进程的页表集
     *  不过，没有必要使一个用户态线性地址对应的TLB表项无效
     *  因为内核线程不访问用户态地址空间。
    */
    if (!mm)        /*  内核线程无虚拟地址空间, mm = NULL*/
    {
        /*  内核线程的active_mm为上一个进程的mm
         *  注意此时如果prev也是内核线程,
         *  则oldmm为NULL, 即next->active_mm也为NULL  */
        next->active_mm = oldmm;
        /*  增加mm的引用计数  */
        atomic_inc(&oldmm->mm_count);
        /*  通知底层体系结构不需要切换虚拟地址空间的用户部分
         *  这种加速上下文切换的技术称为惰性TBL  */
        enter_lazy_tlb(oldmm, next);
    }
    else            /*  不是内核线程, 则需要切切换虚拟地址空间  */
        switch_mm(oldmm, mm, next);

    /*  如果prev是内核线程或正在退出的进程
     *  就重新设置prev->active_mm
     *  然后把指向prev内存描述符的指针保存到运行队列的prev_mm字段中
     */
    if (!prev->mm)
    {
        /*  将prev的active_mm赋值和为空  */
        prev->active_mm = NULL;
        /*  更新运行队列的prev_mm成员  */
        rq->prev_mm = oldmm;
    }
    /*
     * Since the runqueue lock will be released by the next
     * task (which is an invalid locking op but in the case
     * of the scheduler it's an obvious special-case), so we
     * do an early lockdep release here:
     */
    lockdep_unpin_lock(&rq->lock);
    spin_release(&rq->lock.dep_map, 1, _THIS_IP_);

    /* Here we just switch the register state and the stack. 
     * 切换进程的执行环境, 包括堆栈和寄存器
     * 同时返回上一个执行的程序
     * 相当于prev = witch_to(prev, next)  */
    switch_to(prev, next, prev);

    /*  switch_to之后的代码只有在
     *  当前进程再次被选择运行(恢复执行)时才会运行
     *  而此时当前进程恢复执行时的上一个进程可能跟参数传入时的prev不同
     *  甚至可能是系统中任意一个随机的进程
     *  因此switch_to通过第三个参数将此进程返回
     */

    /*  路障同步, 一般用编译器指令实现
     *  确保了switch_to和finish_task_switch的执行顺序
     *  不会因为任何可能的优化而改变  */
    barrier();  

    /*  进程切换之后的处理工作  */
    return finish_task_switch(prev);
}
```

## switch_to()

```cpp
#define switch_to(prev, next, last)      \
do {         \
/*        \
  * Context-switching clobbers(彻底击败) all registers, so we clobber \
  * them explicitly, via unused output variables.  \
  * (EAX and EBP is not listed because EBP is saved/restored \
  * explicitly for wchan access and EAX is the return value of \
  * __switch_to())      \
  */        \
unsigned long ebx, ecx, edx, esi, edi;    \
         \
asm volatile("pushfl\n\t"  /* save    flags */ \
       "pushl %%ebp\n\t"  /* save    EBP   */ \
       "movl %%esp,%[prev_sp]\n\t" /* save    ESP   */ \
       "movl %[next_sp],%%esp\n\t" /* restore ESP   */ \
       "movl $1f,%[prev_ip]\n\t" /* save    EIP   */ \
       "pushl %[next_ip]\n\t" /* restore EIP   */ \
       "jmp __switch_to\n" /* regparm call  */ \
       "1:\t"      \
       "popl %%ebp\n\t"  /* restore EBP   */ \
       "popfl\n"   /* restore flags */ \
         \
       /* output parameters */                       \
       : [prev_sp] "=m" (prev->thread.sp),  \
       /*m表示把变量放入内存，即把[prev_sp]存储的变量放入内存，最后再写入prev->thread.sp*/\
         [prev_ip] "=m" (prev->thread.ip),  \
         "=a" (last),                                           \
         /*=表示输出,a表示把变量last放入ax,eax = last*/         \
         \
         /* clobbered output registers: */  \
         "=b" (ebx), "=c" (ecx), "=d" (edx),  \
         /*b 变量放入ebx,c表示放入ecx，d放入edx,S放入si,D放入edi*/\
         "=S" (esi), "=D" (edi)    \
                \
         /* input parameters: */    \
       : [next_sp]  "m" (next->thread.sp),  \
       /*next->thread.sp 放入内存中的[next_sp]*/\
         [next_ip]  "m" (next->thread.ip),  \
                \
         /* regparm parameters for __switch_to(): */ \
         [prev]     "a" (prev),    \
         /*eax = prev  edx = next*/\
         [next]     "d" (next)    \
         \
       : /* reloaded segment registers */   \
   "memory");     \
} while (0)   
```

首先简单提一下这个宏和函数的被调用关系：

 **schedule() --> context_switch() --> switch_to --> __switch_to()**

这里面，schedule 是主调度函数，涉及到一些调度算法，这里不讨论。当 schedule() 需要暂停 A 进程的执行而继续 B 进程的执行时，就发生了进程之间的切换。

进程切换主要有两部分：

1、切换全局页表项；

2、切换内核堆栈和硬件上下文。

**这个切换工作由 context_switch() 完成。其中 switch_to 和__switch_to() 主要完成第二部分。更详细的，__switch_to() 主要完成硬件上下文切换，switch_to 主要完成内核堆栈切换。**

阅读 **switch_to 时请注意：这是一个宏，不是函数，它的参数 prev, next, last 不是值拷贝，而是它的调用者 context_switch() 的局部变量。局部变量是通过 %ebp 寄存器(栈基址寄存器)来索引的，也就是通过 n(%ebp)**，n 是编译时决定的，在不同的进程的同一段代码中，同一局部变量的 n 是相同的。在 switch_to 中，发生了堆栈的切换，即 ebp 发生了改变，所以要格外留意在任一时刻的局部变量属于哪一个进程。**关于__switch_to() 这个函数的调用，并不是通过普通的 call 来实现，而是直接 jmp，函数参数也并不是通过堆栈来传递，而是通过寄存器来传递。**

在下文中提到一些局部变量和寄存器值，为了不引起混淆，在名字后面加上_X，表示是 X 进程的成员。如 esp_A 表示进程 A 的 esp 的值，prev_B，表示进程 B 中的 prev 变量，等等。

switch_to 切换主要有以下三部分：

[switch_to](%E8%BF%9B%E7%A8%8B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2%20context_switch()%20915df4bde5d14763bf39abde79d302cd/switch_to%20b6e43d6ed13f4f09902a105430668641.csv)

### switch_to()细节解析

下面来具体看 switch_to 从 A 进程切换到 B 进程的步骤。

- **step1**: 复制两个变量到寄存器：
  
    [prev] "a" (prev)
    
    [next] "d" (next)
    
    即:
    
    eax <== prev_A 或 eax <== %p(%ebp_A)
    
    edx <== next_A 或 edx <== %n(%ebp_A)
    
    这里 prev 和 next 都是 A 进程的局部变量。
    
    现在 eax 中保存 prev，ebx 中保存 next。其中 eax 中始终都保持 prev，最后把该值交给 last
    
- **step2**: 保存进程 A 的 ebp 和 eflags
  
    pushfl
    
    pushl %ebp
    
    注意，因为现在 esp 还在 A 的堆栈中，所以这两个东西被保存到 A 进程的内核堆栈中。
    
- **step3**: 保存当前 esp 到 A 进程内核描述符中：
  
    "movl %%esp,%[prev_sp]\n\t"    /* save    ESP   */
    
    它可以表示成： prev_A->thread.sp <== esp_A
    
    在调用 switch_to 时，prev 是指向 A 进程自己的进程描述符的。
    
- **step4**: 从 next（进程 B）的描述符中取出之前从 B 切换出去时保存的 esp_B。
  
    "movl %[next_sp],%%esp\n\t" /* restore ESP */
    
    它可以表示成：esp_B <== next_A->thread.sp
    
    注意，在 A 进程中的 next 是指向 B 的进程描述符的。
    
    从这个时候开始，**CPU 当前执行的进程已经是 B 进程了**，**因为 esp 已经指向 B 的内核堆栈。**但是，现在的 ebp 仍然指向 A 进程的内核堆栈，所以所有局部变量仍然是 A 中的局部变量，比如 next 实质上是 %n(%ebp_A)，也就是 next_A，即指向 B 的进程描述符。
    
- **step5:** 把标号为 1 的指令地址保存到 A 进程描述符的 ip 域：
  
    "movl $1f,%[prev_ip]\n\t"    /* save    EIP   */
    
    它可以表示成：prev_A->thread.ip <== %1f，当 A 进程下次被 switch_to 回来时，会从这条指令开始执行。具体方法看后面被切换回来的 B 的下一条指令。
    
- **step6:** 将返回地址保存到堆栈，然后调用__switch_to() 函数，__switch_to() 函数完成硬件上下文切换。
  
    "pushl %[next_ip]\n\t"    /* restore EIP   */
    
    "jmp __switch_to\n"    /* regparm call  */
    
    这里，如果之前 B 也被 switch_to 出去过，那么 [next_ip] 里存的就是下面这个 1f 的标号，但如果进程 B 刚刚被创建，之前没有被 switch_to 出去过，那么 [next_ip] 里存的将是 ret_ftom_fork（参看 copy_thread()函数）。这就是这里为什么不用 call __switch_to 而用 jmp，因为 call 会导致自动把下面这句话的地址 (也就是 1:) 压栈，然后__switch_to()就必然只能 ret 到这里，而无法根据需要 ret 到 ret_from_fork。
    
    另外请注意，这里__switch_to() 返回时，将返回值 prev_A 又写入了 %eax，这就使得在 switch_to 宏里面 eax 寄存器始终保存的是 prev_A 的内容，或者，更准确的说，是指向 A 进程描述符的 “指针”。这是有用的，下面 step8 中将会看到。
    
- **step7:** 从__switch_to() 返回后继续从 1: 标号后面开始执行，修改 ebp 到 B 的内核堆栈，恢复 B 的 eflags：
  
    "popl %%ebp\n\t"        /* restore EBP   */
    
    "popfl\n"            /* restore flags */
    
    如果从__switch_to() 返回后从这里继续运行，那么说明在此之前 B 肯定被 switch_to 调出过，因此此前肯定备份了 ebp_B 和 flags_B(step2中备份)，这里执行恢复操作。
    
    注意，**这时候 ebp 已经指向了 B 的内核堆栈，所以上面的 prev,next 等局部变量已经不是 A 进程堆栈中的了，而是 B 进程堆栈中的 (B 上次被切换出去之前也有这两个变量，所以代表着 B 堆栈中 prev、next 的值了)，因为 prev == %p(%ebp_B)，**而在 B 上次被切换出去之前，该位置保存的是 B 进程的描述符地址。如果这个时候就结束 switch_to 的话，在后面的代码中（即 context_switch() 函数中 switch_to 之后的代码）的 prev 变量是指向 B 进程的，因此，进程 B 就不知道是从哪个进程切换回来。而从 context_switch() 中 switch_to 之后的代码中，我们看到 finish_task_switch(this_rq(), prev) 中需要知道之前是从哪个进程切换过来的，因此，我们必须想办法保存 A 进程的描述符到 B 的堆栈中，这就是 last 的作用。
    
- **step8:** 将 eax 写入 last，以在 B 的堆栈中保存正确的 prev 信息。
  
    "=a" (last)  即  last_B <== %eax
    
    而从 context_switch() 中看到的调用 switch_to 的方法是：
    
    switch_to(prev, next, prev);
    
    所以，**这里面的 last 实质上就是 prev，因此在 switch_to 宏执行完之后，prev_B 就是正确的 A 的进程描述符了**。
    
    至此，switch_to 已经执行完成，A 停止运行，而开始了 B。在以后，可能在某一次调度中，进程 A 得到调度，就会出现 switch_to(C, A) 这样的调用，这时，A 再次得到调度，得到调度后，A 进程从 context_switch() 中 switch_to 后面的代码开始执行，这时候，它看到的 prev_A 将指向 C 的进程描述符。
    
    为什么不用两个参数而是三个参数，我个人认为这样做是为了便于理解，不然的话，switch_to(prev, next); 也可以实现上述功能，直接把 prev 这个参数即当输入参数，又当输出参数就可以了。last 的作用就是让当前进程知道上一个进程是什么。
    
    这里有两个堆栈，在这个过程中，有一个时期 esp 和 ebp 并不在同一个堆栈上，要格外注意这个时期里所有涉及堆栈的操作分别是在哪个堆栈上进行的。记住一个简单的原则即可，**pop/push 这样的操作，都是对 esp 所指向的堆栈进行的，这些操作同时也会改变 esp 本身，除此之外，其它关于变量的引用，都是对 ebp 所指向的堆栈进行的。**
    

### 总结

为了便于理解，我们首先忽略 switch_to 中的具体细节，仅仅把它当作一个普通的指令。对 A 进程来说，它始终没有感觉到自己被打断过，它认为自己一直是不间断执行的。switch_to 这条 “指令”，除了改变了 A 进程中的 prev 变量外，对 A 没有其它任何影响。在系统中任何进程看到的都是这个样子，所有进程都认为自己在不间断的独立运行。然而，实际上 switch_to 的执行并不是一瞬间完成的，switch_to 执行花了很长很长的时间，但是，在执行完 switch_to 之后，这段时间被从 A 的记忆中抹除，所以 A 并没有觉察到自己被打断过。

接着，我们再来看这个 “神奇” 的 switch_to。switch_to 是从 A 进程到 B 进程的过渡，我们可以认为在 switch_to 这个点上，A 进程被切出，B 进程被切入。但是，如果把粒度放小到 switch_to 里面的单个汇编语句，这个界限就不明显了。进入 switch_to 的宏里面之后，首先 pushfl 和 pushl ebp 肯定仍然属于进程 A，之后把 esp 指向了 B 的堆栈，严格的说，从此时开始的指令流都属于 B 进程了。但是，这个时候 B 进程还没有完全准备好继续运行，因为 ebp、硬件上下文等内容还没有切换成 B 的，剩下的部分宏代码就是完成这些事情。

另外需要格外强调的是，这部分代码是内核代码，它们跟用户代码不在同一个代码段，所有进程在内核态共用这一段内核代码。这里涉及到的所有堆栈都是内核堆栈，而不涉及用户堆栈。进程切换时需要的页表项的切换不是在这里面做的。

我们现在再向 “上 “看，从一个高级语言程序员的角度看，内核态的东西就好比这里的 switch_to 一样，对高级语言程序员是透明的。高级语言程序员始终认为自己的进程在不间断连续执行，而调度点的语句以及调度点之后的整个过程对该程序是完全没有影响的。

关于内核进程切换就讲这么多吧。switch_to 只是个普通的宏，但是却能实现进程的切换，很多人对此比较费解。为了正确的理解，大家需要注意：

这些代码是所有进程共用的，代码本身不属于某一个特定的进程，所以判定当前在哪一个进程不是通过看执行的代码是哪个进程的，而是通过 esp 指向哪个进程的堆栈来判定的。所以，对于上面图中的切换点也可以这样理解，在这一点处，esp 指向了其它进程的堆栈，当前进程即被挂起，等待若干时间，当 esp 指针再次指回这个进程的堆栈时，这个进程又重新开始运行。