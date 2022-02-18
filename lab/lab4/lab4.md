# Lab 4: Preemptive Multitasking



## **Introduction**

在本实验中，您将在多个同时处于活动状态的用户模式环境中实现抢占式多任务处理。

在 A 部分中，您将为 JOS 添加多处理器支持，实现循环调度，并添加基本的环境管理系统调用（创建和销毁环境以及分配/映射内存的调用）。

在 B 部分中，您将实现一个类 Unix `fork()`，它允许用户模式环境创建自身的副本。

最后，在 C 部分中，您将添加对进程间通信 (IPC) 的支持，允许不同的用户模式环境显式地相互通信和同步。您还将添加对硬件时钟中断和抢占的支持。

Lab 4添加的源文件：

1. kern/cpu.h 对多处理器支持的内核私有定义
2. kern/mpconfig.c 读取多处理器配置的代码
3. kern/lapic.c 引导每个CPU上的Local APIC的内核代码
4. kern/mpentry.S 非启动CPU的入口汇编代码
5. kern/spinlock.h 内核自旋锁定义
6. kern/spinlock.c 自旋锁实现的内核代码
7. kern/sched.c 调度器代码框架



## Part A: Multiprocessor Support and Cooperative Multitasking

在本实验的第一部分，您将首先扩展 JOS 以在多处理器系统上运行，然后实现一些新的 JOS 内核系统调用以允许用户级环境创建额外的新环境。您还将实现*协作*式循环调度，允许内核在当前环境自愿放弃 CPU（或退出）时从一个环境切换到另一个环境。稍后在 C 部分中，您将实现*抢占式*调度，它允许内核在经过一定时间后从环境中重新控制 CPU，即使环境不合作也是如此。



#### Multiprocessor Support

我们将让 JOS 支持“对称多处理”（SMP），这是一种多处理器模型，其中所有 CPU 对系统资源（例如内存和 I/O 总线）具有同等访问权限。虽然在 SMP 中所有 CPU 在功能上都是相同的，但在引导过程中它们可以分为两种类型：引导处理器 (BSP) 负责初始化系统和引导操作系统；只有在操作系统启动并运行后，BSP 才会激活应用处理器 (AP)。哪个处理器是 BSP 是由硬件和 BIOS 决定的。至此，您现有的所有 JOS 代码都在 BSP 上运行。

在 SMP 系统中，每个 CPU 都有一个随附的本地 APIC (LAPIC) 单元。LAPIC 单元负责在整个系统中传递中断。LAPIC 还为其连接的 CPU 提供唯一标识符。在本实验中，我们使用了 LAPIC 单元（在`kern/lapic.c`中）的以下基本功能：

- 读取 LAPIC 标识符 (APIC ID) 以了解我们的代码当前在哪个 CPU 上运行（请参阅 参考资料`cpunum()`）。
- 从 BSP 向 AP发送`STARTUP`处理器间中断 (IPI) 以启动其他 CPU（请参阅 参考资料 `lapic_startap()`）。
- 在 C 部分中，我们对 LAPIC 的内置定时器进行编程以触发时钟中断以支持抢先式多任务处理（请参阅 参考资料 `apic_init()`）。

处理器使用内存映射 I/O (MMIO) 访问其 LAPIC。在 MMIO 中，一部分*物理*内存被硬连线到一些 I/O 设备的寄存器，因此通常用于访问内存的相同加载/存储指令可用于访问设备寄存器。您已经在物理地址 `0xA0000`处看到了一个 IO 孔（我们使用它来写入 VGA 显示缓冲区）。LAPIC 存在于一个从物理地址 `0xFE000000`开始的洞中（4GB 短 32MB），因此对于我们使用 KERNBASE 中常用的直接映射来访问它来说太高了。JOS 虚拟内存映射在`MMIOBASE处留下 4MB 的空白`所以我们有一个地方可以映射这样的设备。由于后面的实验会引入更多的 MMIO 区域，因此您将编写一个简单的函数来从该区域分配空间并将设备内存映射到该区域。



**Exercise 1**



#### **Application Processor Bootstrap**

在启动 AP 之前，BSP 应该首先收集有关多处理器系统的信息，例如 CPU 的总数、它们的 APIC ID 和 LAPIC 单元的 MMIO 地址。`kern/mpconfig.c`中的`mp_init()`函数 通过读取位于 BIOS 内存区域中的 MP 配置表来检索此信息。``

该`boot_aps()`函数（在`kern/init.c`中）驱动 AP 引导过程。AP 以实模式启动，就像引导加载程序在`boot/boot.S`中启动的方式一样，因此`boot_aps()` 将 AP 入口代码 ( `kern/mpentry.S` ) 复制到在实模式下可寻址的内存位置。与引导加载程序不同，我们可以控制 AP 开始执行代码的位置；我们将入口代码复制到`0x7000` ( `MPENTRY_PADDR`)，但是任何低于 640KB 的未使用的、页面对齐的物理地址都可以使用。

之后`boot_aps()`，通过向`STARTUP`相应 AP 的 LAPIC 单元发送 IPI 以及`CS:IP`AP 应该开始运行其入口代码的初始地址（`MPENTRY_PADDR`在我们的例子中），一个接一个地激活 AP。`kern/mpentry.S`中的入口代码与`boot/boot.S`中的入口代码非常相似。经过一些简短的设置后，它将 AP 置于启用分页的保护模式，然后调用 C 设置例程`mp_main()`（也在`kern/init.c 中`）。 在继续唤醒下一个之前， `boot_aps()`等待 AP 在其字段中发出`CPU_STARTED`标志信号。

总结：

![image-20220217155827967](../../pic/image-20220217155827967.png)

在i386init函数中进行BSP启动的一些配置，经由lab2的 mem_init，lab3的env_init和trap_init，lab4的mp_init和lapic_init，然后boot_aps函数启动所有的CPU。
　　 多核处理器的初始化都在mp_init函数中完成，首先是调用mpconfig函数，主要功能是寻找一个MP 配置条目，然后对所有的CPU进行配置，找到启动的处理器。

![image-20220217155641262](../../pic/image-20220217155641262.png)

 在启动过程中，mp_init和lapic_init是和硬件以及体系架构紧密相关的，通过读取某个特殊内存地址（当然前提是能读取的到，所以在mem_init中需要修改进行相应映射），来获取CPU的信息，根据这些信息初始化CPU结构。
　　 在boot_aps函数中首先找到一段用于启动的汇编代码，该代码和上一章实验一样是嵌入在内核代码段之上的一部分，其中mpentry_start和mpentry_end是编译器导出符号，代表这段代码在内存（虚拟地址）中的起止位置，接着把代码复制到MPENTRY_PADDR处。随后调用lapic_startap来命令特定的AP去执行这段代码。

![image-20220217155916051](../../pic/image-20220217155916051.png)



**Exercise 2**



Qusetion 1：
　　 仔细比较kern/mpentry.S与boot/boot.S，想想kern/mpentry.S是被编译链接来运行在KERNBASE之上的，那么定义MPBOOTPHYS宏的目的是什么？为什么在kern/mpentry.S中是必要的，在boot/boot.S中不是呢？换句话说，如果我们在kern/mpentry.S中忽略它，会出现什么错误？
　　 回答：
　　 #define MPBOOTPHYS(s) ((s) - mpentry_start + MPENTRY_PADDR))))
　　 MPBOOTPHYS is to calculate symobl address relative to MPENTRY_PADDR. The ASM is executed in the load address above KERNBASE, but JOS need to run mp_main at 0x7000 address! Of course 0x7000’s page is reserved at pmap.c.
　　 在AP的保护模式打开之前，是没有办法寻址到3G以上的空间的，因此用MPBOOTPHYS是用来计算相应的物理地址的。
但是在boot.S中，由于尚没有启用分页机制，所以我们能够指定程序开始执行的地方以及程序加载的地址；但是，在mpentry.S的时候，由于主CPU已经处于保护模式下了，因此是不能直接指定物理地址的，给定线性地址，映射到相应的物理地址是允许的。



#### Per-CPU State and Initialization

​	在编写多处理器操作系统时，区分每个处理器专用的每个 CPU 状态和整个系统共享的全局状态非常重要。 `kern/cpu.h`定义了大多数 per-CPU 状态，包括`struct CpuInfo`，它存储 per-CPU 变量。 `cpunum()`总是返回调用它的 CPU 的 ID，它可以用作数组的索引，如 `cpus`. 或者，宏`thiscpu`是当前 CPU 的简写`struct CpuInfo`。

以下是您应该注意的每个 CPU 的状态：

- **每个 CPU 内核堆栈**。
  由于多个 CPU 可以同时陷入内核，因此我们需要为每个处理器设置一个单独的内核堆栈，以防止它们干扰彼此的执行。该数组 `percpu_kstacks[NCPU][KSTKSIZE]`为 NCPU 的内核堆栈保留空间。

  `bootstack` 在实验 2 中，您映射了称为 BSP 的内核堆栈 的物理内存，就在`KSTACKTOP`. 同样，在本实验中，您将把每个 CPU 的内核堆栈映射到这个区域，保护页面充当它们之间的缓冲区。CPU 0 的堆栈仍然会从`KSTACKTOP`;向下增长。CPU 1 的堆栈将从`KSTKGAP`CPU 0 的堆栈底部以下的字节开始，依此类推。`inc/memlayout.h`显示映射布局。

- **每 CPU TSS 和 TSS 描述符**。
  为了指定每个 CPU 的内核堆栈所在的位置，还需要每个 CPU 的任务状态段 (TSS)。CPU *i*的 TSS存储在 中`cpus[i].cpu_ts`，相应的 TSS 描述符在 GDT 条目中定义`gdt[(GD_TSS0 >> 3) + i]`。`kern/trap.c`中定义的全局`ts`变量将不再有用。``

- **每 CPU 当前环境指针**。
  由于每个 CPU 可以同时运行不同的用户进程，我们重新定义了符号`curenv`来引用 `cpus[cpunum()].cpu_env`（或`thiscpu->cpu_env`），它指向*当前*CPU 上*当前*执行 的环境（代码正在运行的 CPU）。

- **每个 CPU 系统寄存器**。
  所有寄存器，包括系统寄存器，都是 CPU 专用的。因此，初始化这些寄存器的指令，如`lcr3()`、 `ltr()`、`lgdt()`、`lidt()`等，必须在每个 CPU 上执行一次。函数`env_init_percpu()` 和`trap_init_percpu()`是为此目的而定义的。

- 除此之外，如果您在解决方案中添加了任何额外的每个 CPU 状态或执行了任何额外的特定于 CPU 的初始化（例如，在 CPU 寄存器中设置新位）以挑战早期实验中的问题，请务必复制它们在这里的每个 CPU 上！



**Exercise 3**

**Exercise 4**



#### Locking
　　 在mp_main函数中初始化AP后，代码就会进入自旋。在让AP进行更多操作之前，我们首先要解决多CPU同时运行在内核时产生的竞争问题。最简单的办法是实现1个大内核锁，1次只让一个进程进入内核模式，当离开内核时释放锁。
　　 在kern/spinlock.h中声明了大内核锁，提供了lock_kernel和unlock_kernel函数来快捷地获得和释放锁。总共有四处用到大内核锁：

`kern/spinlock.h`声明了大内核锁，即 `kernel_lock`. 它还提供了`lock_kernel()` 和`unlock_kernel()`，获取和释放锁的快捷方式。您应该在四个位置应用大内核锁：

- 在`i386_init()`中，在 BSP 唤醒其他 CPU 之前获取锁。
- 中`mp_main()`，初始化AP后获取锁，然后调用`sched_yield()`该AP上开始运行环境。
- 在`trap()`中，从用户模式捕获时获取锁。要确定陷阱是在用户模式还是内核模式中发生，请检查`tf_cs`.
- 在中，在 切换到用户模式*之前*`env_run()`立即释放锁定。不要太早或太晚这样做，否则你会遇到竞争或僵局。



**Exercise 5**



 Question2:
　　 既然大内核锁保证了只有1个CPU能运行在内核，为什么我们还要为每个CPU准备1个内核栈。
　　 回答：
　　 因为不同的内核栈上可能保存有不同的信息，当1个CPU从内核退出来之后，有可能在内核栈中留下了一些将来还有用的数据，所以一定要有单独的栈。

Challenge 1:
　　 大内核锁简单便于应用，但是它取消了内核模式的并行。大多数现代操作系统使用不同的锁来保护共享状态的不同部分，这称之为细粒度锁。细粒度锁能有效地提高性能，但是也更困难地去实现和检测错误。所以你可以去掉大内核锁，在JOS中实现内核并发。
　　 回答：
　　 实验指导中提供了一些JOS内核中的共享结构，具体实现就是在保证在使用这些结构体时保证互斥。





#### 循环调度

您在本实验中的下一个任务是更改 JOS 内核，以便它可以以“循环”方式在多个环境之间交替。JOS 中的循环调度工作如下：

- `sched_yield()`新的`kern/sched.c`中 的函数 负责选择一个新的环境来运行。它以循环方式顺序搜索`envs[]`数组，从先前运行的环境之后开始（或者如果没有先前运行的环境，则从数组的开头开始），选择它找到的第一个环境，状态为`ENV_RUNNABLE` （参见`inc/env. h` )，并调用`env_run()`跳转到该环境。
- `sched_yield()`决不能同时在两个 CPU 上运行相同的环境。它可以判断一个环境当前正在某个 CPU（可能是当前 CPU）上运行，因为该环境的状态将为`ENV_RUNNING`.
- 我们为您实现了一个新的系统调用， `sys_yield()`用户环境可以调用该系统调用来调用内核的`sched_yield()`功能，从而自愿将 CPU 交给不同的环境。



**Exercise 6**



 然后修改kern/syscall.c添加相关的系统调用分发机制。
　　 Question 3：
　　 在lcr3运行之后，这个CPU对应的页表就立刻被换掉了，但是这个时候的参数e，也就是现在的curenv，为什么还是能正确的解引用？
　　 回答：
　　 因为当前是运行在系统内核中的，而每个进程的页表中都是存在内核映射的。每个进程页表中虚拟地址高于UTOP之上的地方，只有UVPT不一样，其余的都是一样的，只不过在用户态下是看不到的。所以虽然这个时候的页表换成了下一个要运行的进程的页表，但是curenv的地址没变，映射也没变，还是依然有效的。
　　 Question 4：
　　 在用户环境进行切换时，为什么旧进程的寄存器一定要被保存以便之后重新装载？在哪里发生这样的操作？
　　 回答：
　　 因为不进行保存，旧进程运行时的状态就丢失了，运行就不正确了。每次进入到内核态的时候，当前的运行状态都是在一进入的时候就保存了的。如果没有发生调度，那么之前trapframe中的信息还是会恢复回去，如果发生了调度，恢复的就是被调度运行的进程的上下文了。



**System Calls for Environment Creation**

尽管您的内核现在能够在多个用户级环境之间运行和切换，但它仍然仅限于*内核*最初设置的运行环境。您现在将实现必要的 JOS 系统调用，以允许*用户*环境创建和启动其他新用户环境。

Unix 提供`fork()`系统调用作为其进程创建原语。Unix`fork()`复制调用进程（父进程）的整个地址空间来创建一个新进程（子进程）。从用户空间观察到的两个可观察对象之间的唯一区别是它们的进程 ID 和父进程 ID（由`getpid`and返回`getppid`）。在父进程中， `fork()`返回子进程 ID，而在子进程中，`fork()`返回 0。默认情况下，每个进程都有自己的私有地址空间，并且两个进程对内存的修改都不可见。

您将提供一组不同的、更原始的 JOS 系统调用来创建新的用户模式环境。`fork()`通过这些系统调用，除了其他风格的环境创建之外，您将能够完全在用户空间中实现类 Unix 。您将为 JOS 编写的新系统调用如下：

- `sys_exofork`：

  这个系统调用创建了一个几乎空白的新环境：在其地址空间的用户部分没有映射任何内容，并且它是不可运行的。新环境在调用时将具有与父环境相同的寄存器状态`sys_exofork`。在父级中，`sys_exofork` 将返回`envid_t`新创建的环境（如果环境分配失败，则返回负错误代码）。然而，在子节点中，它将返回 0。（由于子节点开始标记为不可运行， `sys_exofork`因此在父节点通过使用...标记子节点可运行明确允许此操作之前，不会在子节点中实际返回。）

- `sys_env_set_status`：

  将指定环境的状态设置为`ENV_RUNNABLE`或`ENV_NOT_RUNNABLE`。这个系统调用通常用于标记一个新环境准备好运行，一旦它的地址空间和寄存器状态已经完全初始化。

- `sys_page_alloc`：

  分配一页物理内存并将其映射到给定环境地址空间中的给定虚拟地址。

- `sys_page_map`：

  将页面映射（*不是*页面的内容！）从一个环境的地址空间复制到另一个环境，保留内存共享安排，以便新的和旧的映射都引用物理内存的同一页面。

- `sys_page_unmap`：

  取消映射在给定环境中在给定虚拟地址处映射的页面。

对于以上所有接受环境 ID 的系统调用，JOS 内核都支持值 0 表示“当前环境”的约定。这个约定是`envid2env()` 在`kern/env.c`中实现的。



`fork()` 我们在测试程序`user/dumbfork.c` 中提供了一个非常原始的类 Unix 实现。该测试程序使用上述系统调用来创建和运行具有自己地址空间副本的子环境。`sys_yield` 然后，这两个环境使用前面练习中的方法来回切换。父级在 10 次迭代后退出，而子级在 20 次后退出。



**Exercise 7**
