---
title: linux中进程，线程，协程
date: 2021-03-06 23:47:19
tags:
- linux
categories:
- linux
---

### 进程， 线程，协程

#### 进程

进程是操作系统对一个正在运行的程序的一种抽象，进程是资源分配的最小单位。为什么会有 ”进程“ 呢？**说白了还是为了合理压榨 CPU 的性能和分配运行的时间片**，不能 “闲着“。

![进程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2wy8jjr6j31120iyq3f.jpg)

##### 进程的控制结构

在linux操作系统中，进程控制块PCB用来唯一标识一个进程，这意味一个进程 一定会有对应的PCB，进程消失，PCB也会随之消失。在不同的操作系统中对进程的控制和管理机制不同，PCB中的信息多少不一样，通常PCB应包含如下一些信息。

1. **进程 标识符(name)**：每个进程都必须有一个唯一的标识符，可以是字符串，也可以是一个数字。
2. **进程控制和状态管理**: 说明进程当前所处的状态。为了管理的方便，系统设计时会将相同的状态的进程组成一个队列，如就绪进程队列，等待进程队列等。
3. **进程资源清单**：列出所拥有的除CPU外的资源记录，如拥有的[I/O设备](http://baike.sogou.com/lemma/ShowInnerLink.htm?lemmaId=7705644)，打开的文件列表等。
4. **进程优先级priority**：进程的优先级反映进程的紧迫程度，通常由用户指定和系统设置。
5. **CPU相关信息**: 当进程因某种原因不能继续占用CPU时（如等待打印机），释放CPU，这时就要将CPU的各种状态信息保护起来，为将来再次得到处理机恢复CPU的各种状态，继续运行。
6. 进程相应的程序和数据地址，以便把PCB与其程序和数据联系起来。
7. 与进程有关的其他信息。 如进程记账信息，进程占用CPU的时间等。

在linux 中每一个进程都由task_struct 数据结构来定义, task_struct就是我们通常所说的PCB。结构如下

```c
struct task_struct {
	volatile long state; //说明了该进程是否可以执行，还是可中断等信息
	unsigned long flags; // flags 是进程号，在调用fork()时给出
	int sigpending; // 进程上是否有待处理的信号
 
	 mm_segment_t addr_limit;  //进程地址空间,区分内核进程与普通进程在内存存放的位置不同  //0-0xBFFFFFFF for user-thead    //0-0xFFFFFFFF for kernel-thread
	 //调度标志,表示该进程是否需要重新调度,若非0,则当从内核态返回到用户态,会发生调度
	 volatile long need_resched;
	 int lock_depth;    //锁深度
	 long nice;       //进程的基本时间片
 
	 //进程的调度策略,有三种,实时进程:SCHED_FIFO,SCHED_RR, 分时进程:SCHED_OTHER
	 unsigned long policy;
	 struct mm_struct *mm;    //进程内存管理信息
 
	 int processor;
	 //若进程不在任何CPU上运行, cpus_runnable 的值是0，否则是1 这个值在运行队列被锁时更新
	 unsigned long cpus_runnable, cpus_allowed;
	 struct list_head run_list;   //指向运行队列的指针
	 unsigned long sleep_time;   //进程的睡眠时间
 
	 //用于将系统中所有的进程连成一个双向循环链表, 其根是init_task
	 struct task_struct *next_task, *prev_task;
	 struct mm_struct *active_mm;
	 struct list_head local_pages;      //指向本地页面      
	 unsigned int allocation_order, nr_local_pages;
	 struct linux_binfmt *binfmt;      //进程所运行的可执行文件的格式
	 int exit_code, exit_signal;
	 int pdeath_signal;           //父进程终止时向子进程发送的信号
	 unsigned long personality;
	 //Linux可以运行由其他UNIX操作系统生成的符合iBCS2标准的程序
	 int did_exec:1; 
	 pid_t pid;          //进程标识符,用来代表一个进程
	 pid_t pgrp;        //进程组标识,表示进程所属的进程组
	 pid_t tty_old_pgrp;      //进程控制终端所在的组标识
	 pid_t session;             //进程的会话标识
	 pid_t tgid;
	 int leader;    //表示进程是否为会话主管
	 struct task_struct *p_opptr,*p_pptr,*p_cptr,*p_ysptr,*p_osptr;
	 struct list_head thread_group;          //线程链表
	 struct task_struct *pidhash_next;    //用于将进程链入HASH表
	 struct task_struct **pidhash_pprev;
	 wait_queue_head_t wait_chldexit;      //供wait4()使用
	 struct completion *vfork_done;         //供vfork() 使用
 
 
	 unsigned long rt_priority;       //实时优先级，用它计算实时进程调度时的weight值
 
 
	 //it_real_value，it_real_incr用于REAL定时器，单位为jiffies, 系统根据it_real_value
 
	 //设置定时器的第一个终止时间. 在定时器到期时，向进程发送SIGALRM信号，同时根据
 
	 //it_real_incr重置终止时间，it_prof_value，it_prof_incr用于Profile定时器，单位为jiffies。
 
	 //当进程运行时，不管在何种状态下，每个tick都使it_prof_value值减一，当减到0时，向进程发送
 
	 //信号SIGPROF，并根据it_prof_incr重置时间.
	 //it_virt_value，it_virt_value用于Virtual定时器，单位为jiffies。当进程运行时，不管在何种
 
	 //状态下，每个tick都使it_virt_value值减一当减到0时，向进程发送信号SIGVTALRM，根据
 
	 //it_virt_incr重置初值。
 
	 unsigned long it_real_value, it_prof_value, it_virt_value;
	 unsigned long it_real_incr, it_prof_incr, it_virt_value;
	 struct timer_list real_timer;        //指向实时定时器的指针
	 struct tms times;                      //记录进程消耗的时间
	 unsigned long start_time;          //进程创建的时间
 
	 //记录进程在每个CPU上所消耗的用户态时间和核心态时间
	 long per_cpu_utime[NR_CPUS], per_cpu_stime[NR_CPUS]; 
 
 
	 //内存缺页和交换信息:
 
	 //min_flt, maj_flt累计进程的次缺页数（Copy on　Write页和匿名页）和主缺页数（从映射文件或交换
 
	 //设备读入的页面数）； nswap记录进程累计换出的页面数，即写到交换设备上的页面数。
	 //cmin_flt, cmaj_flt, cnswap记录本进程为祖先的所有子孙进程的累计次缺页数，主缺页数和换出页面数。
 
	 //在父进程回收终止的子进程时，父进程会将子进程的这些信息累计到自己结构的这些域中
	 unsigned long min_flt, maj_flt, nswap, cmin_flt, cmaj_flt, cnswap;
	 int swappable:1; //表示进程的虚拟地址空间是否允许换出
	 //进程认证信息
	 //uid,gid为运行该进程的用户的用户标识符和组标识符，通常是进程创建者的uid，gid
 
	 //euid，egid为有效uid,gid
	 //fsuid，fsgid为文件系统uid,gid，这两个ID号通常与有效uid,gid相等，在检查对于文件
 
	 //系统的访问权限时使用他们。
	 //suid，sgid为备份uid,gid
	 uid_t uid,euid,suid,fsuid;
	 gid_t gid,egid,sgid,fsgid;
	 int ngroups;                  //记录进程在多少个用户组中
	 gid_t groups[NGROUPS];      //记录进程所在的组
 
	 //进程的权能，分别是有效位集合，继承位集合，允许位集合
	 kernel_cap_t cap_effective, cap_inheritable, cap_permitted;
 
	 int keep_capabilities:1;
	 struct user_struct *user;
	 struct rlimit rlim[RLIM_NLIMITS];    //与进程相关的资源限制信息
	 unsigned short used_math;         //是否使用FPU
	 char comm[16];                      //进程正在运行的可执行文件名
	 int link_count, total_link_ count;  //文件系统信息
 
	 //NULL if no tty 进程所在的控制终端，如果不需要控制终端，则该指针为空
	 struct tty_struct *tty;
	 unsigned int locks;
	 //进程间通信信息
	 struct sem_undo *semundo;       //进程在信号灯上的所有undo操作
	 struct sem_queue *semsleeping;   //当进程因为信号灯操作而挂起时，他在该队列中记录等待的操作
	 //进程的CPU状态，切换时，要保存到停止进程的task_struct中
	 struct thread_struct thread;
	 struct fs_struct *fs;           //文件系统信息
	 struct files_struct *files;    //打开文件信息
	 spinlock_t sigmask_lock;   //信号处理函数
	 struct signal_struct *sig;   //信号处理函数
	 sigset_t blocked;                //进程当前要阻塞的信号，每个信号对应一位
	 struct sigpending pending;      //进程上是否有待处理的信号
	 unsigned long sas_ss_sp;
	 size_t sas_ss_size;
	 int (*notifier)(void *priv);
	 void *notifier_data;
	 sigset_t *notifier_mask;
	 u32 parent_exec_id;
	 u32 self_exec_id;
 
	 spinlock_t alloc_lock;
	 void *journal_info;
}
```

PCB通过链表的方式进行组织，把具有相同状态的进程链在一起，组成各种队列, 例如将处于就绪状态的进程链在一块，形成就绪队列，将所有因等待某事件而处于等待队列的进程链在一块形成阻塞队列。

![pcb链表](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2xnlh053j31260q2756.jpg)

##### 进程的状态

进程的执行期间，至少具备三种基本状态，即运行态、就绪态、阻塞态。

- 运行态(Runing)：此刻进程占用CPU;
- 就绪态(Ready): 可运行，但因为其他进程正在运行而暂停停止
- 阻塞状态(Blocked)：该进程等待某个事件（比如IO读取）停止运行，这时，即使给它CPU控制权，它也无法运行

![进程状态](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2xshd5c3j311a0iadge.jpg)

如上所示：

1. CPU调度绪态进程执行，进入运行状态，时间片使用完了，回到就绪态，等待 CPU调度
2. CPU调度绪态进程执行，进入运行状态，执行IO请求，进入阻塞态，IO请求完成，CPU收到 中断 信号，进入就绪态，等待 CPU 调度

如果在三态基础上，做进一步细化，出现了另外两个基本状态，创建态和结束态。

- 创建态（new）：进程正在被创建
- 就绪态（Ready）：可运行，但因为其他进程正在运行而暂停停止
- 运行态（Runing）：时刻进程占用 C P U
- 结束态（Exit）：进程正在从系统中消失时的状态
- 阻塞状态（Blocked）：该进程等待某个事件（比如IO读取）停止运行，这时，即使给它CPU控制权，它也无法运行

![进程五态](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2yr36kntj31800jcdgs.jpg)

##### CPU的上下文切换

CPU上下文是指 CPU寄存器和程序计数器

- CPU寄存器是CPU内置的容量小，速度极快的缓存
- 程序计数器是用来存储CPU正在执行的指令位置或即将执行的下一条指令位置

CPU上下文切就是把前一个任务的CPU上下文保存起来，然后在加载当前任务的CPU上下文，最后再跳转到 程序计数器 所指的新位置，运行任务。

上面说到所谓的「任务」，主要包含进程、线程和中断。所以，可以根据任务的不同，把 CPU 上下文切换分成： 进程上下文切换、线程上下文切换和中断上下文切换。

##### 进程的上下文切换

CPU把一个进程切换到另一个进程运行的过程，称为进程上下文切换。

首先进程是由内核管理与调度的，所以进程上下文切换发生在内核态，进程上下文切换的内容包含用户空间资源（虚拟内存、栈、全局变量等）与内核空间资源（内核堆栈、寄存器等）。在做上下文切换的时候，会把前一个 进程 的上下文保存到它的 PCB中，然后加载当前 进程 的 PCB上下文到 CPU中，使得 进程 继续执行。

![进程上下文切换](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2ya3nu9dj310e0d2mxj.jpg)

进程上下文切换的场景：

- 为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，切换到其它正在等待 CPU 的进程运行
- 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。
- 当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，自然也会重新调度。
- 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行
- 发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。

#### 线程

有了多进程，想必在操作系统上可以同时运行多个进程。那么为什么有了进程，还要线程呢？

原因如下：

- 进程间的信息难以共享数据，父子进程并未共享内存，需要通过进程间通信（IPC），在进程间进行信息交换，性能开销较大。
- 创建进程（一般是调用 `fork` 方法）的性能开销较大。

一个进程可以由多个称为线程的执行单元组成。每个线程都运行在进程的上下文中，共享着同样的代码和全局数据。多个进程，就可以有更多的线程。**多线程比多进程之间更容易共享数据，在上下文切换中线程一般比进程更高效**。每个线程都有独立一套的寄存器和栈，这样可以确保线程的控制流是相对独立的。

![线程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2yq46zecj315g0p0dh5.jpg)

引入线程带来的好处有以下几点

- 一个进程中可以同时存在多个线程
- 让进程具备多任务并行处理能力
- 同进程下的各个线程之间可以共享进程资源 （同进程内的多线程通信十分简单高效）
- 更轻量与高效

##### 线程的上下文切换

当进程只有一个线程时，可以认为进程等于线程，线程上下文的切换分两种情况

1. 不同进程的线程，切换的过程就跟进程上下文切换一样
2. 两个线程是属于同一个进程，因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据

![线程上下文](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw2yp0fsrmj311s0fggmb.jpg)

#### 线程模型

线程的实现模型主要有3种：内核级线程模型、用户级线程模型和混合型线程模型。它们之间最大的区别在于线程与内核调度实体KSE(Kernel Scheduling Entity)之间的对应关系上。所谓的内核调度实体KSE 就是指可以被操作系统内核调度器调度的对象实体，有些地方也称其为内核级线程，是操作系统内核的最小调度单元。在linux中，这样一个KSE就是一个轻量级的进程（通过clone系统调用创建出来的）。

##### 内核级线程模型

因为内核线程是由内核空间管理，所以它的 结构线程控制块（Thread Control Block, TCB） 在内核空间，操作系统对 T C B 是可见的。内核线程与KSE是1对1关系(1:1)。大部分编程语言的线程库(如linux的pthread，Java的java.lang.Thread，C++11的std::thread等等)都是对操作系统的线程（内核级线程）的一层封装，创建出来的每个线程与一个不同的KSE静态关联，因此其调度完全由OS调度器来做。这种方式实现简单，直接借助OS提供的线程能力，并且不同用户线程之间一般也不会相互影响。但其创建，销毁以及多个线程之间的上下文切换等操作都是直接由OS层面亲自来做，在需要使用大量线程的场景下对OS的性能影响会很大。

![内核线程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw36lb65p2j315o0lwq44.jpg)

##### 用户级线程模型

因为 用户线程 在用户空间，是由 用户态 通过线程库来管理，所以它的 结构线程控制块（Thread Control Block, TCB） 也是在线程库里面，对于操作系统而言是看不到 T C B 的，它只能看到整个进程的 P C B（内核无法管理用户线程，也感知不到用户线程）。

用户线程与KSE是多对1关系(M:1)，这种线程的创建，销毁以及多个线程之间的协调等操作都是由用户自己实现的线程库来负责，对OS内核透明，一个进程中所有创建的线程都与同一个KSE在运行时动态关联。现在有许多语言实现的 协程 基本上都属于这种方式。这种实现方式相比内核级线程可以做的很轻量级，对系统资源的消耗会小很多，因此可以创建的数量与上下文切换所花费的代价也会小得多。但该模型有个致命的缺点，如果我们在某个用户线程上调用阻塞式系统调用(如用阻塞方式read网络IO)，那么一旦KSE因阻塞被内核调度出CPU的话，剩下的所有对应的用户线程全都会变为阻塞状态（整个进程挂起）。
所以这些语言的协程库会把自己一些阻塞的操作重新封装为完全的非阻塞形式，然后在以前要阻塞的点上，主动让出自己，并通过某种方式通知或唤醒其他待执行的用户线程在该KSE上运行，从而避免了内核调度器由于KSE阻塞而做上下文切换，这样整个进程也不会被阻塞了。

![用户线程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw36lqx5xej316k0qu760.jpg)

##### 混合型线程模型

用户线程与KSE是多对多关系(M:N), 这种实现综合了前两种模型的优点，为一个进程中创建多个KSE，并且线程可以与不同的KSE在运行时进行动态关联，当某个KSE由于其上工作的线程的阻塞操作被内核调度出CPU时，当前与其关联的其余用户线程可以重新与其他KSE建立关联关系。当然这种动态关联机制的实现很复杂，也需要用户自己去实现，这算是它的一个缺点吧。Go语言中的并发就是使用的这种实现方式，Go为了实现该模型自己实现了一个运行时调度器来负责Go中的”线程”与KSE的动态关联。此模型有时也被称为 两级线程模型，即用户调度器实现用户线程到KSE的“调度”，内核调度器实现KSE到CPU上的调度。

![混合级线程模型](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw36nrsfktj316q0rsgnr.jpg)

#### 协程

协程是用户态的线程(go 的协程属于混合型线程)。通常创建协程时，会从进程的堆中分配一段内存作为协程的栈。线程的栈有 8 MB，而协程栈的大小通常只有 KB，而 Go 语言的协程更夸张，只有 2-4KB，非常的轻巧。

协程的优势如下：

- 节省 CPU：避免系统内核级的线程频繁切换，造成的 CPU 资源浪费。好钢用在刀刃上。而协程是用户态的线程，用户可以自行控制协程的创建于销毁，极大程度避免了系统级线程上下文切换造成的资源浪费。
- 节约内存：在 64 位的Linux中，一个线程需要分配 8MB 栈内存和 64MB 堆内存，系统内存的制约导致我们无法开启更多线程实现高并发。而在协程编程模式下，可以轻松有十几万协程，这是线程无法比拟的。
- 稳定性：前面提到线程之间通过内存来共享数据，这也导致了一个问题，任何一个线程出错时，进程中的所有线程都会跟着一起崩溃。
- 开发效率：使用协程在开发程序之中，可以很方便的将一些耗时的IO操作异步化，例如写文件、耗时 IO 请求等。

### 孤儿进程，僵尸进程，守护进程，init进程

#### init进程

进程号为1的进程称为init进程，init进程的好处就是SIGKILL信号对它无效。

#### 孤儿进程

父进程退出了，但是他的子进程还在运行，这种情况下子进程将变成孤儿进程。孤儿进程将被init进程(进程号为1)接管，并由init进程对它完成状态收集(wait/waitpid)工作。由于孤儿进程会被init进程给收养，所以孤儿进程不会对系统造成危害

#### 僵尸进程

一个进程使用fork创建子进程，如果子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中。父进程认为子进程还存活着，这种进程称之为僵死进程。

#### 守护进程

守护进程是运行在后台的一种特殊进程。它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。它不需要用户输入就能运行而且提供某种服务，不是对整个系统就是对某个用户程序提供服务。

### systemd进程

**提到 systemd 不得不提一下 Linux 的启动流程，这样才能清楚 systemd 在 Linux 系统中的地位和作用**

#### linux 启动流程

1. 加电自检(检查硬件是否有问题)
2. GRUB引导
3. 内核加载
4. init 初始化(内核启动第一个**用户空间应用程序**，即 systemd 进程， 通过dmesg能查看)

![dmesg](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goanfk29j1j328m0tcqbm.jpg)

#### systemd简介

*systemd* 是一个 Linux 系统基础组件的集合，提供了一个系统和服务管理器，运行为 PID 1 并负责启动其它程序, 所有的进程都会被挂在这个进程下，如果这个进程退出了，那么所有的进程都被 kill 。 systemd功能包括：

- 支持并行化任务；
- 同时采用 socket 式与 [D-Bus](https://wiki.archlinux.org/index.php/D-Bus_(简体中文)) 总线式激活服务；
- 按需启动守护进程（daemon）；
- 利用 Linux 的 [cgroups](https://wiki.archlinux.org/index.php/Cgroups) 监视进程；
- 支持快照和系统恢复；
- 维护挂载点和自动挂载点；
- 各服务间基于依赖关系进行精密控制。
- systemd* 支持 SysV 和 LSB 初始脚本，可以替代 sysvinit。
- 除此之外，功能还包括日志进程、控制基础系统配置，维护登陆用户列表以及系统账户、运行时目录和设置，可以运行容器和虚拟机，可以简单的管理网络配置、网络时间同步、日志转发和名称解析等。

通过pstree 能够查看进程数状态，用户空间的进程都挂在 PID 为 1 的 systemd 进程下。(似乎systemd进程无法被杀死，kill -9 1似乎无效！)

![pstree 进程数](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goan63vhgqj313s0pygp3.jpg)

#### systemd 体系架构

![systemd体系架构](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goanaecdizj31g40qc1kx.jpg)

![systemd交互](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goannznnthj31bn0u07wh.jpg)

- 最底层：systemd 内核层面依赖 cgroup、autofs、kdbus

- 第二层：systemd libraries 是 systemd 依赖库

- 第三层：systemd Core 是 systemd 自己的库

- 第四层：systemd daemons 以及 targets 是自带的一些基本 unit、target，类似于 sysvinit 中自带的脚本

- 最上层就是和 systemd 交互的一些工具其中：

  1. systemctl 命令控制 `systemd` 的管理系统和服务的命令行工具

  2. journalctl 命令查詢 `systemd` 日志系统

  3. loginctl 命令控制 `systemd` 登入管理器

  4. systemd-analyze 分析系统启动效能

  ![systemd命令大全](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goanl9xhdsj320c08etap.jpg)

#### systemd软连接

当我们使用 reboot 、poweroff 、shutdown 等命令的时候，其实并不是执行该命令本身，背后是调用的 systemctl 命令。systemctl 命令会将 reboot 这些命令作为 $1 参数传递进去。所以执行 reboot 和 systemctl reboot 本质上是一样的。

![systemd软连接](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goanp7lgg9j319q07o75f.jpg)

![systemd system commands](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goansqy13wj317y0ewdib.jpg)

### 参考文档

- https://segmentfault.com/a/1190000040373756   // 煎鱼的博客
- https://segmentfault.com/a/1190000039378412    // 程序员阿星
- [Linux 的小伙伴 systemd 详解](https://blog.k8s.li/systemd.html)   //木子的博客
- http://shareinto.k8s.101.com/2019/01/30/docker-init(1)/   // shareinto
- [LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)  //酷壳