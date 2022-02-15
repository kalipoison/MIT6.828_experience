# Exercise 4

编辑一下trapentry.S 和 trap.c 文件，并且实现上面所说的功能。宏定义 TRAPHANDLER 和 TRAPHANDLER_NOEC 会对你有帮助。你将会在 trapentry.S文件中为在inc/trap.h文件中的每一个trap加入一个入口指， 你也将会提供_alttraps的值。

　　你需要修改trap_init()函数来初始化idt表，使表中每一项指向定义在trapentry.S中的入口指针，SETGATE宏定义在这里用得上。

　　　　你所实现的 _alltraps 应该：

　　　　1. 把值压入堆栈使堆栈看起来像一个结构体 Trapframe

　　　　2. 加载 GD_KD 的值到 %ds, %es寄存器中

　　　　3. 把%esp的值压入，并且传递一个指向Trapframe的指针到trap()函数中。

　　　　4. 调用trap

　　考虑使用pushal指令，他会很好的和结构体 Trapframe 的布局配合好。





答：

　　首先看一下 trapentry.S 文件，里面定义了两个宏定义，TRAPHANDLER，TRAPHANDLER_NOEC。他们的功能从汇编代码中可以看出：声明了一个全局符号name，并且这个符号是函数类型的，代表它是一个中断处理函数名。其实这里就是两个宏定义的函数。这两个函数就是当系统检测到一个中断/异常时，需要首先完成的一部分操作，包括：中断异常码，中断错误码(error code)。正是因为有些中断有中断错误码，有些没有，所以我们采用利用两个宏定义函数。

```assembly
 46 /*
 47  * Lab 3: Your code here for generating entry points for the different traps    .
 48  */
 49 TRAPHANDLER_NOEC(divide_entry, T_DIVIDE);
 50 TRAPHANDLER_NOEC(debug_entry, T_DEBUG);
 51 TRAPHANDLER_NOEC(nmi_entry, T_NMI);
 52 TRAPHANDLER_NOEC(brkpt_entry, T_BRKPT);
 53 TRAPHANDLER_NOEC(oflow_entry, T_OFLOW);
 54 TRAPHANDLER_NOEC(bound_entry, T_BOUND);
 55 TRAPHANDLER_NOEC(illop_entry, T_ILLOP);
 56 TRAPHANDLER_NOEC(device_entry, T_DEVICE);
 57 TRAPHANDLER(dblflt_entry, T_DBLFLT);
 58 TRAPHANDLER(tss_entry, T_TSS);
 59 TRAPHANDLER(segnp_entry, T_SEGNP);
 60 TRAPHANDLER(stack_entry, T_STACK);
 61 TRAPHANDLER(gpflt_entry, T_GPFLT);
 62 TRAPHANDLER(pgflt_entry, T_PGFLT);
 63 TRAPHANDLER_NOEC(fperr_entry, T_FPERR);
 64 TRAPHANDLER(align_entry, T_ALIGN);
 65 TRAPHANDLER_NOEC(mchk_entry, T_MCHK);
 66 TRAPHANDLER_NOEC(simderr_entry, T_SIMDERR);
 67 TRAPHANDLER_NOEC(syscall_entry, T_SYSCALL);
```

然后就会调用 _alltraps，_alltraps函数其实就是为了能够让程序在之后调用trap.c中的trap函数时，能够正确的访问到输入的参数，即Trapframe指针类型的输入参数tf。

```assembly
 70 /*
 71  * Lab 3: Your code here for _alltraps
 72  */
 73 _alltraps:
 74         pushl %ds
 75         pushl %es
 76         pushal
 77 
 78         movl $GD_KD, %eax
 79         movl %eax, %ds
 80         movl %eax, %es
 81 
 82         push %esp
 83         call trap
```

最后在trap.c中实现trap_init函数，即在idt表中插入中断向量描述符，可以使用SETGATE宏实现：
　　SETGATE宏的定义：
　　#define SETGATE(gate, istrap, sel, off, dpl)
　　其中gate是idt表的index入口，istrap判断是异常还是中断，sel为代码段选择符，off表示对应的处理函数地址，dpl表示触发该异常或中断的用户权限。

```c
void
trap_init(void)
{
        extern struct Segdesc gdt[];

        // LAB 3: Your code here. 
        SETGATE(idt[T_DIVIDE], 0, GD_KT, divide_entry, 0);
        SETGATE(idt[T_DEBUG], 0, GD_KT, debug_entry, 0);
        SETGATE(idt[T_NMI], 0, GD_KT, nmi_entry, 0); 
        SETGATE(idt[T_BRKPT], 0, GD_KT, brkpt_entry, 3);
        SETGATE(idt[T_OFLOW], 0, GD_KT, oflow_entry, 0);
        SETGATE(idt[T_BOUND], 0, GD_KT, bound_entry, 0);
        SETGATE(idt[T_ILLOP], 0, GD_KT, illop_entry, 0);
        SETGATE(idt[T_DEVICE], 0, GD_KT, device_entry, 0);
        SETGATE(idt[T_DBLFLT], 0, GD_KT, dblflt_entry, 0);
        SETGATE(idt[T_TSS], 0, GD_KT, tss_entry, 0); 
        SETGATE(idt[T_SEGNP], 0, GD_KT, segnp_entry, 0);
        SETGATE(idt[T_STACK], 0, GD_KT, stack_entry, 0);
        SETGATE(idt[T_GPFLT], 0, GD_KT, gpflt_entry, 0);
        SETGATE(idt[T_PGFLT], 0, GD_KT, pgflt_entry, 0);
        SETGATE(idt[T_FPERR], 0, GD_KT, fperr_entry, 0);
        SETGATE(idt[T_ALIGN], 0, GD_KT, align_entry, 0);
        SETGATE(idt[T_MCHK], 0, GD_KT, mchk_entry, 0);
        SETGATE(idt[T_SIMDERR], 0, GD_KT, simderr_entry, 0);
        SETGATE(idt[T_SYSCALL], 0, GD_KT, syscall_entry, 3);
        // Per-CPU setup 
        trap_init_percpu();
}
```

