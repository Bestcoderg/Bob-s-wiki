# Linux 上下文切换消耗

**用户态转去内核态：200ns+ （**使用快速系统调用**）**

**进程上下文切换：一般线程都会有几MB，时间需要3-5us，线程会稍微快一些相差不多**

**有栈协程上下文切换：一般协程只会有几KB，时间需要120-200ns.**

**软中断**：**软中断CPU开销大约3.4us左右**

## 线程与进程切换消耗

进程切换与线程切换的一个最主要区别就在于进程切换涉及到**虚拟地址空间的切换**而线程切换则不会。因为每个进程都有自己的虚拟地址空间，而线程是共享所在进程的虚拟地址空间的，因此同一个进程中的线程进行线程切换时不涉及虚拟地址空间的转换。而进程则不同，两个不同进程位于不同的屋檐下，即进程位于不同的虚拟地址空间，因此进程切换涉及到虚拟地址空间的切换。为了切换进程的虚拟地址空间，会更新 cr3 寄存器切换对应的**页表**，这也是为什么进程切换要比线程切换慢的原因。还有就是进程的上下文是多于线程上下文的，如文件句柄等。

那么**进程上下文切换**的时候，开销都具体有哪些呢？开销分成两种，一种是直接开销、一种是间接开销。 直接开销就是在切换时，cpu 必须做的事情，包括：

1、切换页表全局目录

2、切换内核态堆栈

3、切换硬件上下文（进程恢复前，必须装入寄存器的数据统称为硬件上下文）

ip(instruction pointer)：指向当前执行指令的下一条指令

bp(base pointer): 用于存放执行中的函数对应的栈帧的栈底地址

sp(stack poinger): 用于存放执行中的函数对应的栈帧的栈顶地址

cr3: 页目录基址寄存器，保存页目录表的物理地址

......

4、TLB(TLB 是一种高速缓存，内存管理硬件使用它来改善虚拟地址到物理地址的转换速度) 实例需要重新加载 - 这个对性能影响非常大不说，整个进程的执行都会停止

5、系统调度器的代码执行将，将下一个进程的 PCB 信息改为 “运行态”

间接开销主要指的是虽然切换到一个新进程后，由于各种缓存并不热，速度运行会慢一些。如果进程始终都在一个 CPU 上调度还好一些，如果跨 CPU 的话，之前热起来的 TLB、L1、L2、L3 因为运行的进程已经变了，所以以局部性原理 cache 起来的代码、数据也都没有用了，导致新进程穿透到内存的 IO 会变多。 其实我们上面的实验并没有很好地测量到这种情况，所以实际的上下文切换开销可能比 3.5us 要大

**cache miss的情况：**

lmbench 用于评价系统综合性能的多平台开源 benchmark，能够测试包括文档读写、内存操作、进程创建销毁开销、网络等性能。使用方法简单，但就是跑有点慢，感兴趣的同学可以自己试一试。

这个工具的优势是是进行了多组实验，每组 2 个进程、8 个、16 个。每个进程使用的数据大小也在变，充分模拟 cache miss 造成的影响。我用他测了一下结果如下：

lmbench 显示的进程上下文切换耗时从 2.7us 到 5.48 之间。

大约每次线程切换开销大约是 3.8us 左右。**从上下文切换的耗时上来看，Linux 线程（轻量级进程）其实和进程差别不太大**。

虽然线程与进程切换消耗差别是不大的，但是并不意味着线程存在就没有意义。首先进程创建的开销是比线程大的，进程创建相较线程慢十倍，空间上也是同样，进程通常是2M+而线程是十几KB。其次，线程是共享虚拟地址空间的，所以在数据的交互上相较进程优势明显。

## 协程与线程切换消耗差距

协程消耗小于线程的原因主要有以下三点：

1、 **协程相比线程而言，切换更少**(io阻塞/sleep等才会切换，CPU计算并不会切换)，而**线程是按照时间片来切换**的。所以为了更好的性能，那么就少做切换，切换本身比较耗时

2、协程又可以理解成“用户态线程”，也就意味着协程切换上下文时候，**并不会做陷入内核态，也就没有内核态时候的上下文切换，内核态的堆栈寄存器切换就没有了**。 切换本身上，协程做的事情更少，也就更快

3、 起一个协程只需要几kb的内存空间， 但是一个线程需要2M 的空间。系统资源上更节省

协程切换只涉及基本的**CPU上下文切换**，所谓的 CPU 上下文，就是一堆寄存器，里面保存了 CPU运行任务所需要的信息：

**从哪里开始运行**（%rip：指令指针寄存器，标识 CPU 运行的下一条指令），栈顶的位置（**%rsp：** 是堆栈指针寄存器，通常会指向栈顶位置）

**当前栈帧在哪**（%rbp 是栈帧指针，用于标识当前栈帧的起始位置）

**以及其它的CPU的中间状态或者结果**（%rbx，%r12，%r13，%14，%15 等等）

而且**完全在用户态进行,无用户态与内核态的切换**，一般来说一次协程上下文切换最多就是**几十ns** 这个量级。

下面给出 libco 的协程切换的汇编代码，也就是二十来条汇编指令，完成当前协程 CPU 寄存器的保存，并恢复调度进来的 CPU 寄存器状态，类似的也可以参考 boost context 里面的切换汇编代码，大同小异。

- ***context switch***
  
    ```cpp
    #elif defined(__x86_64__)
    	leaq 8(%rsp),%rax
    	leaq 112(%rdi),%rsp
    	pushq %rax
    	pushq %rbx
    	pushq %rcx
    	pushq %rdx
    
    	pushq -8(%rax) //ret func addr
    
    	pushq %rsi
    	pushq %rdi
    	pushq %rbp
    	pushq %r8
    	pushq %r9
    	pushq %r12
    	pushq %r13
    	pushq %r14
    	pushq %r15
    	
    	movq %rsi, %rsp
    	popq %r15
    	popq %r14
    	popq %r13
    	popq %r12
    	popq %r9
    	popq %r8
    	popq %rbp
    	popq %rdi
    	popq %rsi
    	popq %rax //ret func addr
    	popq %rdx
    	popq %rcx
    	popq %rbx
    	popq %rsp
    	pushq %rax
    	
    	xorl %eax, %eax
    	ret
    #endif
    ```
    

而**线程的调度只有拥有最高权限的内核空间才可以完成**，所以线程的切换涉及到**用户空间和内核空间的切换**，也就是特权模式切换，然后需要操作系统调度模块完成**线程调度，**而且除了和协程相同基本的 CPU 上下文，还有线程私有的栈和寄存器等，说白了就是上下文比协程多一些。**线程用户态和内核态的切换是最主要的开销**。

- **当然，协程也有着自己的缺陷：**
  
    毕竟是用户态，比如如果遇到一个密集运算的，一直遇不到io阻塞，就会一直占用线程，你只能蹩脚的手动插入代码切换上下文，要解决这个问题会用一些畸形的办法，比如计时器。
    
    协程引以为傲的占用内存少，能开几万十万几百万的协程，那就是最优解吗？并不然，相反的没有协程 以事件驱动的模型并发和占用资源更少，以为协程的切换代价是真的小吗？不想想你可是有几万十万的协程。
    

平均每次协程切换的开销是（655035993-415197171)/2000000=**120ns**。相对于前面文章测得的进程切换开销大约 3.5us，大约是其的三十分之一。比系统调用的造成的开销还要低。

协程由于是在用户态来完成上下文切换的，所以切换耗时只有区区100ns多一些，比**进程切换要快30倍。单个协程需要的栈内存也足够小，只需要2KB。**所以，近几年来协程大火，在互联网后端的高并发场景里大放光彩。

扩展：由于go的协程调用起来太方便了，所以一些go的程序员就很随意地go来go去。要知道go这条指令在切换到协程之前，得先把协程创建出来。而一次创建加上调度开销就涨到400ns，差不多相当于一次系统调用的耗时了。虽然协程很高效，但是也不要乱用，否则go祖师爷Rob Pike花大精力优化出来的性能，被你随意一go又给葬送掉了。

## 用户态/内核态切换消耗

我们从上文知道了线程相较协程(用户态线程)切换的主要消耗在于内核态与用户态的切换，所以接下来让我们看看用户态与内核态之间切换具体的消耗，首先用户态切换到内核态有三种方式：

- 系统调用
  
    这是用户态进程主动要求切换到内核态的一种方式，用户态进程通过系统调用申请使用操作系统提供的服务程序完成工作，比如前例中 fork() 实际上就是执行了一个创建新进程的系统调用。**而系统调用的机制其核心还是使用了操作系统为用户特别开放的一个中断来实现，例如 Linux 的 int 80h 中断**。
    
- 异常
  
    当 CPU 在执行运行在用户态下的程序时，发生了某些事先不可知的异常，这时会触发由当前运行进程切换到处理此异常的内核相关程序中，也就转到了内核态，比如缺页异常。
    
- 外围设备的中断
  
    当外围设备完成用户请求的操作后，会向 CPU 发出相应的中断信号，这时 CPU 会暂停执行下一条即将要执行的指令转而去执行与中断信号对应的处理程序，如果先前执行的指令是用户态下的程序，那么这个转换的过程自然也就发生了由用户态到内核态的切换。比如硬盘读写操作完成，系统会切换到硬盘读写的中断处理程序中执行后续操作等。
    

这 3 种方式是系统在运行时由用户态转到内核态的最主要方式，其中系统调用可以认为是用户进程主动发起的，异常和外围设备中断则是被动的。

从触发方式上看，可以认为存在前述 3 种不同的类型，但是从最终实际完成由用户态到内核态的切换操作上来说，涉及的关键步骤是完全一致的，没有任何区别，都相当于执行了一个中断响应的过程，因为系统调用实际上最终是中断机制实现的，而异常和中断的处理机制基本上也是一致的。

涉及到由用户态切换到内核态的步骤主要包括：

- 从当前进程的描述符中提取其**内核栈的 ss0 及 esp0** 信息
- 使用 ss0 和 esp0 指向的内核栈将当前进程的 **SS、ESP、EFLAGS、CS 和 EIP 寄存器全部都需要进行切换** ，这个过程也完成了由用户栈到内核栈的切换过程，同时保存了被暂停执行的程序的下一条指令
- 将先前由中断向量检索得到的中断处理程序的 cs,eip 信息装入相应的寄存器，开始执行中断处理程序，这时就转到了内核态的程序执行了
- 内核态程序执行完毕时如果要从内核态返回用户态，可以通过执行指令 iret 来完成，指令 iret 会将先前压栈的进入内核态前的 SS、ESP、EFLAGS、CS 和 EIP  信息从栈里弹出，加载到各个对应的寄存器中，重新开始执行用户态的程序。

实际上用户态切换内核态看起来操作只是切换了一些寄存器，消耗并不大，但实际上这就涉及到了两种状态下处理方式的不同：

对于普通的函数调用来说，**一般只需要进行几次寄存器操作**，如果有参数或返回函数的话，再进行几次用户栈操作而已。而且**用户栈早已经被 CPU cache 接住**，也并不需要真正进行内存 IO。

但是对于系统调用来说，这个过程就要麻烦一些了。系统调用时需要从用户态切换到内核态。由于内核态的栈用的是内核栈，因此还需要进行栈的切换。**SS、ESP、EFLAGS、CS 和 EIP 寄存器全部都需要进行切换**。

而且栈切换后还可能有一个隐性的问题，那就是 CPU 调度的指令和数据一定程度上破坏了局部性原来，导致一二三级数据缓存、**TLB 页表缓存的命中率一定程度上有所下降**。

除了上述堆栈和寄存器等环境的切换外，系统调用由于特权级别比较高，也还需要**进行一系列的权限校验、有效性等检查相关操作**。所以系统调用的开销相对函数调用来说要大的多。

如果非要扒到内核的实现上，我建议大家参考一下《深入理解 LINUX 内核 - 第十章系统调用》。最初系统调用是通过汇编指令 int（中断）来实现的，当用户态进程发出 int $0x80 指令时，CPU 切换到内核态并开始执行 system_call 函数。 只不过后来大家觉得系统调用实在是太慢了，因为 int 指令要执行一致性和安全性检查。后来 Intel 又提供了 “快速系统调用” 的 sysenter 指令。

相比较函数调用时的不到 1ns 的耗时，系统调用确实开销蛮大的。虽然使用了 “快速系统调用” 指令，但**耗时仍大约在 200ns+**，多的可能到十几 us。每个系统调用内核要进行许多工作，大约需要执行 1000 条左右的 CPU 指令，所以确实应该尽量减少系统调用。但是即使是 10us，仍然是 1ms 的百分之一，所以还没到了谈系统调用色变的程度，能理性认识到它的开销既可。

## 软中断消耗

CPU正常情况下都是专心处理用户的进程的，当外部的硬件或软件有消息想要通知CPU，就会通过**中断请求（interrupt request，IRQ）**的方式来进行。比如当你的鼠标有了点击产生，再比如磁盘设备完成了数据的读取的时候，都会通过中断通知CPU工作已完成。

但是当中断机制应用到网络IO的时候，就产生了一点点问题。网络包收到后的处理工作，不像鼠标、键盘、磁盘IO读取完成那样简单，而是要进行大量的内核协议栈的处理，最终才能放到进程的接收缓存区中。假如只用一种中断（硬终端）的方式来处理网络IO，由于硬中断的优先级又比较高，这样CPU就会忙于处理大量的网络IO而不能及时响应键盘鼠标等事情，导致操作系统实时性变差，你会感觉机器以卡一卡的。

所以现代的Linux又发明了软件中断，配合硬中断来处理网络IO。 硬中断你可以理解只是个收包的，把包收取回来放到“家里”就完事，很快就能完成，这样不耽误CPU响应其它外部高优先级的中断。而软中断优先级较低，负责将包进行各种处理，完成从驱动层、到网络协议栈，最终把处理出来的数据放到socker的接收buffer中。

**软中断消耗的CPU周期相对比硬中断要多不少**，我们重点关注软中断的开销。

**从实验数据来看，一次软中断CPU开销大约3.4us左右**

这个时间里其实包含两部分，一是上下文切换开销，二是软中断内核执行开销。 其中上下文切换和系统调用、进程上下文切换有很多相似的地方。让我们将他们进行一个简单的对比：

### 软中断与系统调用开销对比

《深入理解Linux内核-第五章》开头的一句话，很形象地把中断和系统调用两个不相关的概念联系了起来，巧妙地找到了这二者之间的相似处。“你可以把内核看做是不断对请求进行响应的服务器，这些请求可能来自在CPU上执行的进程，也可能来自发出中断的外部设备。老板的请求相当于中断，而顾客的请求相当于用户态进程发出的系统调用”。

**软中断和系统调用一样，都是CPU停止掉当前用户态上下文，保存工作现场，然后陷入到内核态继续工作。二者的唯一区别是系统调用是切换到同进程的内核态上下文，而软中断是则是切换到了另外一个内核进程ksoftirqd上。**

而事实上，早期的系统调用也还真的是通过汇编指令int（中断）来实现的，当用户态进程发出**int $0x80**指令时，CPU切换到内核态并开始执行system_call函数。后来大家觉得**系统调用实在是太慢了，因为int指令要执行一致性和安全性检查**。后来内核又该用了**Intel提供的“快速系统调用”的sysenter指令，才算是和中断脱离了一点点干系**。**而软中断必须得进行这些检查，所以从这点上来看，中断的开销应该是比系统调用的开销要多的。**

> 根据前文的实验结果，系统调用开销是200ns起步。
> 

### 软中断**和进程上下文切换开销对比**

和进程上下文切换比较起来，进程上下文切换是从用户进程A切换到了用户进程B。而软中断切换是从用户进程A切换到了内核线程ksoftirqd上。 而ksoftirqd作为一个内核控制路径，其处理程序比一个用户进程要轻量，所以上下文切换开销相对比进程切换要小一些。

> 根据前文的实验结果，进程上下文切换开销是3us-5us。