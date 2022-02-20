# Exercise 13

修改 kern/trapentry.S 和 kern/trap.c 以初始化 IDT 中的相应条目并为 IRQ 0 到 15 提供处理程序。然后修改 kern/env.c 中 env_alloc() 中的代码以确保用户环境始终在启用中断的情况下运行。

还要取消注释 sched_halt() 中的 sti 指令，以便空闲 CPU 取消屏蔽中断。

处理器在调用硬件中断处理程序时从不推送错误代码。此时，您可能需要重新阅读 80386 参考手册的第 9.2 节或 IA-32 英特尔架构软件开发人员手册第 3 卷的第 5.8 节。

做完这个练习后，如果你用任何运行了很长时间的测试程序（例如，spin）来运行你的内核，你应该会看到内核打印硬件中断的陷阱帧。虽然现在在处理器中启用了中断，但 JOS 尚未处理它们，因此您应该看到它错误地将每个中断分配给当前正在运行的用户环境并销毁它。最终它应该用完环境来破坏并掉入显示器。



答：



模仿原先设置默认中断向量即可，在kern/trapentry.S中定义IRQ0-15的处理例程。

kern/trapentry.S

```assembly
TRAPHANDLER_NOEC(handler32, IRQ_OFFSET + IRQ_TIMER)
TRAPHANDLER_NOEC(handler33, IRQ_OFFSET + IRQ_KBD)
TRAPHANDLER_NOEC(handler36, IRQ_OFFSET + IRQ_SERIAL)
TRAPHANDLER_NOEC(handler39, IRQ_OFFSET + IRQ_SPURIOUS)
TRAPHANDLER_NOEC(handler46, IRQ_OFFSET + IRQ_IDE)
TRAPHANDLER_NOEC(handler51, IRQ_OFFSET + IRQ_ERROR)
```

　然后在IDT中注册，修改trap_init，由于先前已经实现简化，故此无需做处理。

kern/trap.c

```c
void handler32();
void handler33();
void handler36();
void handler39();
void handler46();
void handler51();



void
trap_init(void)
{
		...
        SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);

        SETGATE(idt[IRQ_OFFSET+IRQ_TIMER], 0, GD_KT, handler32, 0);
        SETGATE(idt[IRQ_OFFSET+IRQ_KBD], 0, GD_KT, handler33, 0);
        SETGATE(idt[IRQ_OFFSET+IRQ_SERIAL], 0, GD_KT, handler36, 0);
        SETGATE(idt[IRQ_OFFSET+IRQ_SPURIOUS], 0, GD_KT, handler39, 0);
        SETGATE(idt[IRQ_OFFSET+IRQ_IDE], 0, GD_KT, handler46, 0);
        SETGATE(idt[IRQ_OFFSET+IRQ_ERROR], 0, GD_KT, handler51, 0);
        // Per-CPU setup 
        trap_init_percpu();
}

```

最后在env_aloc函数中打开中断。

```c
int
env_alloc(struct Env **newenv_store, envid_t parent_id)
{
		...
        // Enable interrupts while in user mode.
        // LAB 4: Your code here.
        e->env_tf.tf_eflags |= FL_IF;
        ...
}
```

kern/sched.c

```c
void
sched_halt(void)
{
    ...
		asm volatile (
                "movl $0, %%ebp\n"
                "movl %0, %%esp\n"
                "pushl $0\n"
                "pushl $0\n"
                // Uncomment the following line after completing exercise 13
                "sti\n"
                "1:\n"
                "hlt\n"
                "jmp 1b\n"
        : : "a" (thiscpu->cpu_ts.ts_esp0));
}

```

