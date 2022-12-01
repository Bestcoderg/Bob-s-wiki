# Linux 进程控制块PCB

### PCB是什么

每个进程在内核中都有一个**进程控制块（Processing Control Block）**，Linux 内核的进程控制块是 **task_struct** 结构体，用来维护进程相关的信息，主要表示进程状态。其作用是使一个在多道程序环境下不能独立运行的程序，成为一个能独立运行的基本单位或与其它进程并发执行的进程。PCB 通常是系统内存占用区中的一个连续存区，它存放着操作系统用于描述进程情况及控制进程运行所需的全部信息。

进程控制块 (PCB) 是系统为了管理进程设置的一个专门的数据结构。系统用它来记录进程的外部特征，描述进程的运动变化过程。同时，系统可以利用 PCB 来控制和管理进程，所以说，**PCB（进程控制块）是系统感知进程存在的唯一标志**。PCB作为系统控制进程的标志，优先级是高于程序主体的，在创建进程时将先创建PCB控制块后创建程序主体。而在销毁程序时，将先销毁程序主体后销毁PCB控制块。

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/20171126214307033.jpg](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/20171126214307033.jpg)

进程控制块 PCB 的组织方式有：

- **线性表方式：**不论进程的状态如何，将所有的 PCB 连续地存放在内存的系统区。这种方式适用于系统中进程数目不多的情况，不适合频繁的进程调度
- **索引表方式：**该方式是线性表方式的改进，系统按照进程的状态分别建立就绪索引表、阻塞索引表等。其中进程阻塞可能由于 I/O 请求、申请缓冲区失败、等待解锁、获取数据失败等原因造成，将其组成一张表忽略了进程的优先级，不利于进程的唤醒。

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled.png)

- **链接表方式：**系统按照进程的状态将进程的 PCB 组成队列，从而形成就绪队列、阻塞队列、运行队列等

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%201.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%201.png)

接下来我们利用 linux2.6 的源码进行了浅显的分析：

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%202.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%202.png)

### 进程控制块 task_struct（PCB）

- **详解 task_struct (PCB):**
  
    ```cpp
    PCB 包含信息：
    进程 id、用户 id 和组 id
    程序计数器
    进程的状态 (有就绪、运行、阻塞)
    进程切换时需要保存和恢复的 CPU 寄存器的值
    描述虚拟地址空间的信息
    描述控制终端的信息
    当前工作目录
    文件描述符表，包含很多指向 file 结构体的指针
    进程可以使用的资源上限 (ulimit –a 命令可以查看)
    输入输出状态：配置进程使用 I/O 设备，如磁带机。
    ```
    
    ```cpp
    struct task_struct {
        //表示进程当前运行状态
        //volatile避免了读取到缓存在寄存器中的脏数据，而是直接从内存中取
        //可以看到state基本有三种，但大于0还分为很多不同的状态
        //#define TASK_RUNNING		0
        //#define TASK_INTERRUPTIBLE	1
        //#define TASK_UNINTERRUPTIBLE	2
        //#define TASK_STOPPED		4
        //#define TASK_TRACED		8
        //#define EXIT_ZOMBIE		16
        //#define EXIT_DEAD		32
    	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
        //这个结构体保存了进程描述符中中频繁访问和需要快速访问的字段，内核依赖于该数据结构来获得当前进程的描述符。通过源码可以看到，该struct内拥有指向task_struct的指针
    	struct thread_info *thread_info;
        
    	atomic_t usage;
    	//进程标志，描述进程当前的状态（不是运行状态）如：PF_SUPERPRIV表示进程拥有超级用户特权。
    	unsigned long flags;	/* per process flags, defined below */
      //系统调用跟踪
    	unsigned long ptrace;
    	//内核锁标志（判断是否被上锁）
    	int lock_depth;		/* Lock depth */
    	//进程优先级
    	int prio, static_prio;
    	struct list_head run_list;
      //进程调度队列
    	prio_array_t *array;
    	//进程平均等待时间
    	unsigned long sleep_avg;
      //timestamp:进程最近插入运行队列的时间或最近一次进程切换的时间
      //last_ran:最近一次替换本进程的进程切换时间
    	unsigned long long timestamp, last_ran;
      //进程被唤醒时所使用的条件代码，就是从什么状态被唤醒的。
    	int activated;
    	//进程的调度类型
    	unsigned long policy;
    	cpumask_t cpus_allowed;
      //time_slice:进程的剩余时间片
      //first_time_slice:创建后首次获取的时间片，为1表示当前的时间片是从父进程分来的
    	unsigned int time_slice, first_time_slice;
    
    #ifdef CONFIG_SCHEDSTATS
    	struct sched_info sched_info;
    #endif
    
    	struct list_head tasks;
    	/*
    	 * ptrace_list/ptrace_children forms the list of my children
    	 * that were stolen by a ptracer.
    	 */
    	struct list_head ptrace_children;
    	struct list_head ptrace_list;
    	**//mm：内存描述符，其下有程地址空间下的虚拟内存信息
      //actvie_mm：内核线程所借用的地址空间
    	struct mm_struct *mm,active_mm;**
    
    	/* task state */
    	struct linux_binfmt *binfmt;
      //进程的退出状态，大于0表示僵死
    	long exit_state;
      //exit_code:存放进程的退出码
      //exit_signal:当进程退出时发给父进程的信号，如果是轻量级进程为-1
    	int exit_code, exit_signal;
    	int pdeath_signal;  /*  The signal sent when the parent dies  */
    	/* ??? */
    	unsigned long personality;
      //记录是否执行execve系统调用
    	unsigned did_exec:1;
      **//进程id
    	pid_t pid;
      //所在线程组领头进程的PID
    	pid_t tgid;**
    	**/* 
    	 * pointers to (original) parent process, youngest child, younger sibling,
    	 * older sibling, respectively.  (p->father can be replaced with 
    	 * p->parent->pid)
    	 */
    	struct task_struct *real_parent; /* real parent process (when being debugged) */**
      **//指向P的当前父进程
    	struct task_struct *parent;	/* parent process */**
    	/*
    	 * children/sibling forms the list of my children plus the
    	 * tasks I'm ptracing.
    	 */
      //链表的头部，链表中所有元素都是P创建的子进程
    	struct list_head children;	/* list of my children */
    	//兄弟进程之间相连接的链表
      struct list_head sibling;	/* linkage in my parent's children list */
    	//P所在进程组的领头进程
      struct task_struct *group_leader;	/* threadgroup leader */
    	**//每个进程有四个PID，把这四个PID挂到PID HASH表里的不同位置，这样从PID到task就很快了
    	//PIDTYPE_PID 进程的 PID, PIDTYPE_TGID 线程组 ID, PIDTYPE_PGID 进程组 ID, PIDTYPE_SID    会话 ID
    	struct pid pids[PIDTYPE_MAX];**
      //为vfork()用来等待子进程的队列
    	struct completion *vfork_done;		/* for vfork() */
    	int __user *set_child_tid;		/* CLONE_CHILD_SETTID */
    	int __user *clear_child_tid;		/* CLONE_CHILD_CLEARTID */
    	//进程的实时优先级
    	unsigned long rt_priority;
        //以下为一些时间与定时信息
    	unsigned long it_real_value, it_real_incr;
    	cputime_t it_virt_value, it_virt_incr;
    	cputime_t it_prof_value, it_prof_incr;
    	struct timer_list real_timer;
    	cputime_t utime, stime;
    	unsigned long nvcsw, nivcsw; /* context switch counts */
    	struct timespec start_time;
    /* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */
    	unsigned long min_flt, maj_flt;
    	/* process credentials */
    	uid_t uid,euid,suid,fsuid;
    	gid_t gid,egid,sgid,fsgid;
    	struct group_info *group_info;
    	kernel_cap_t   cap_effective, cap_inheritable, cap_permitted;
    	unsigned keep_capabilities:1;
    	struct user_struct *user;
    #ifdef CONFIG_KEYS
    	struct key *session_keyring;	/* keyring inherited over fork */
    	struct key *process_keyring;	/* keyring private to this process (CLONE_THREAD) */
    	struct key *thread_keyring;	/* keyring private to this thread */
    #endif
    	int oomkilladj; /* OOM kill score adjustment (bit shift). */
    	char comm[TASK_COMM_LEN];
    	/* file system info */
    	int link_count, total_link_count;
    	/* ipc stuff */
    	struct sysv_sem sysvsem;
    	/* CPU-specific state of this task */
    	struct thread_struct thread;
    	/* filesystem information */
        //进程的可执行映象所在的文件系统
    	struct fs_struct *fs;
    	/* open file information */
      **//进程打开的文件
    	struct files_struct *files;**
    	/* namespace */
    	struct namespace *namespace;
    	/* signal handlers */
    	struct signal_struct *signal;
    	struct sighand_struct *sighand;
    
    	sigset_t blocked, real_blocked;
    	struct sigpending pending;
    
    	unsigned long sas_ss_sp;
    	size_t sas_ss_size;
    	int (*notifier)(void *priv);
    	void *notifier_data;
    	sigset_t *notifier_mask;
    	
    	void *security;
    	struct audit_context *audit_context;
    
    	/* Thread group tracking */
      u32 parent_exec_id;
      u32 self_exec_id;
    	/* Protection of (de-)allocation: mm, files, fs, tty, keyrings */
    	spinlock_t alloc_lock;
    	/* Protection of proc_dentry: nesting proc_lock, dcache_lock, write_lock_irq(&tasklist_lock); */
    	spinlock_t proc_lock;
    	/* context-switch lock */
    	spinlock_t switch_lock;
    
    	/* journalling filesystem info */
    	void *journal_info;
    
    	/* VM state */
    	struct reclaim_state *reclaim_state;
    
    	**struct dentry *proc_dentry;**
    	struct backing_dev_info *backing_dev_info;
    
    	struct io_context *io_context;
    
    	unsigned long ptrace_message;
    	siginfo_t *last_siginfo; /* For ptrace use.  */
    	/*
    	 * current io wait handle: wait queue entry to use for io waits
    	 * If this thread is processing aio, this points at the waitqueue
    	 * inside the currently handled kiocb. It may be NULL (i.e. default
    	 * to a stack based synchronous wait) if its doing sync IO.
    	 */
    	wait_queue_t *io_wait;
    	/* i/o counters(bytes read/written, #syscalls */
    	u64 rchar, wchar, syscr, syscw;
    #if defined(CONFIG_BSD_PROCESS_ACCT)
    	u64 acct_rss_mem1;	/* accumulated rss usage */
    	u64 acct_vm_mem1;	/* accumulated virtual memory usage */
    	clock_t acct_stimexpd;	/* clock_t-converted stime since last update */
    #endif
    #ifdef CONFIG_NUMA
      	struct mempolicy *mempolicy;
    	short il_next;
    #endif
    };
    ```
    

### 虚拟内存描述符 mm_struct

可以看到，**task_struct** 具有很多字段，其包含了**进程状态、内存、调度、文件系统、时间分配等各种信息**，上面我也只是给出了部分的注释，抓住重点的虚拟内存描述符 mm_struct，我们进一步查看其源码：

- **详解 mm_struct:**
  
    ```cpp
    struct mm_struct {
      **//指向线性区对象的链表头
    	struct vm_area_struct * mmap;		/* list of VMAs */
      //指向线性区对象的红黑树
    	struct rb_root mm_rb;**
      //指向最后一个引用的线性区对象
    	struct vm_area_struct * mmap_cache;	/* last find_vma result */
      //在进程地址空间中搜索有效线性地址区间的方法
    	unsigned long (*get_unmapped_area) (struct file *filp,
    				unsigned long addr, unsigned long len,
    				unsigned long pgoff, unsigned long flags);
      //释放线性区时调用的方法
    	void (*unmap_area) (struct vm_area_struct *area);
      // 标识第一个分配的匿名线性区或者是文件内存映射的线性地址
    	unsigned long mmap_base;		/* base of mmap area */
      //内核从这个地址开始搜索进程地址空间中线性地址的空闲区间
    	unsigned long free_area_cache;		/* first hole */
      **//指向页表
    	pgd_t * pgd;**
    	atomic_t mm_users;	/* How many users with user space? */
    	atomic_t mm_count;	/* How many references to "struct mm_struct" (users count as 1) */
    	int map_count;			/* number of VMAs */
    	struct rw_semaphore mmap_sem;
    	//线性区的自旋锁和页表的自旋锁
      spinlock_t page_table_lock;		/* Protects page tables, mm->rss, mm->anon_rss */
    
    	struct list_head mmlist;		/* List of maybe swapped mm's.  These are globally strung
    						 * together off init_mm.mmlist, and are protected
    						 * by mmlist_lock
    						 */
    
      //各个片段的起始地址和终止地址
    	unsigned long start_code, end_code, start_data, end_data;
    	unsigned long start_brk, brk, start_stack;
    	unsigned long arg_start, arg_end, env_start, env_end;
    	unsigned long rss, anon_rss, total_vm, locked_vm, shared_vm;
    	unsigned long exec_vm, stack_vm, reserved_vm, def_flags, nr_ptes;
    
    	unsigned long saved_auxv[42]; /* for /proc/PID/auxv */
    
    	unsigned dumpable:1;
    	cpumask_t cpu_vm_mask;
    
    	/* Architecture-specific MM context */
    	mm_context_t context;
    
    	/* Token based thrashing protection. */
    	unsigned long swap_token_time;
    	char recent_pagein;
    
    	/* coredumping support */
    	int core_waiters;
    	struct completion *core_startup_done, core_done;
    
    	/* aio bits */
    	rwlock_t		ioctx_list_lock;
    	struct kioctx		*ioctx_list;
    
    	struct kioctx		default_kioctx;
    
    	unsigned long hiwater_rss;	/* High-water RSS usage */
    	unsigned long hiwater_vm;	/* High-water virtual memory usage */
    };
    ```
    

**其中这里要讲到的字段为 mmap（虚拟内存维护），其指向线性区对象的链表头。而对于 pgd（虚拟内存到物理内存的维护），其指向该进程的页表。**

在地址空间中，mmap 为地址空间的内存区域（用 vm_area_struct 结构来表示）链表，mm_rb 用红黑树来存储，链表表示起来更加方便，红黑树表示起来更加方便查找。区别是，当虚拟区较少的时候，这个时候采用单链表，由 mmap 指向这个链表，当虚拟区多时此时采用红黑树的结构，由 mm_rb 指向这棵红黑树。这样就可以在大量数据的时候效率更高。所有的 mm_struct 结构体通过自身的 mm_list 域链接在一个双向链表上，该链表的首元素是 init_mm 内存描述符，代表 init 进程的地址空间。

### 线性区描述符 vm_area_struct （VMA）

- **详解 vm_area_struct:**
  
    ```cpp
    struct vm_area_struct {
      **//指向vm_mm
    	struct mm_struct * vm_mm;	/* The address space we belong to. */**
    	//该虚拟内存空间的首地址
      **unsigned long vm_start;**		/* Our start address within vm_mm. */
    	//该虚拟内存空间的尾地址
      **unsigned long vm_end;**		/* The first byte after our end address within vm_mm. */
    
      //VMA链表的下一个成员
    	/* linked list of VM areas per task, sorted by address */
    	struct vm_area_struct *vm_next;
    
    	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
    	//保存VMA标志位
      unsigned long vm_flags;		/* Flags, listed below. */
    	//将本VMA作为一个节点加入到红黑树中
    	struct rb_node vm_rb;
      ...
    }
    ```
    

至此，我们可以看出，虚拟内存即为由一个个 vm_area_struct 结构体，通过链表组装起来的空间。这三种数据结构之间关系可看下图：

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%202.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%202.png)

进程所拥有的线性区从来不重叠，并且内核尽力把新分配的线性区与紧邻的现有线性区进行合并。如果两个相邻区的访问权限相匹配，就能把他们合并在一起。

进程所有的线性区是通过一个简单的链表链接在一起的，出现在链表中的线性区是按内存地址的升序排列的。每两个线性区可以由未用的内存地址区隔开。内核通过进程的内存描述符的 mmap 字段来查找线性区。同时内存描述符的 map_count 字段存放进程所拥有的线性区数目。

### 内存的组织形式

我们在了解了进程控制块相关的结构在内存中的组织形式后，我们可以结合虚拟内存来看：[页缓存&内存组织方式](%E9%A1%B5%E7%BC%93%E5%AD%98&%E5%86%85%E5%AD%98%E7%BB%84%E7%BB%87%E6%96%B9%E5%BC%8F%20351ab7a6eb64447a9ed2b9178098c8ae.md) ，

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%203.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%203.png)

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%204.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%204.png)

结合上图，最终我们可以总结出一张闭环的内存组织形式图：

![Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%205.png](Linux%20%E8%BF%9B%E7%A8%8B%E6%8E%A7%E5%88%B6%E5%9D%97PCB%202d908aa06c3b4b59bfe309147b0004b4/Untitled%205.png)