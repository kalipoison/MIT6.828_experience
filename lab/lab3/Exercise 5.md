# Exercise 5

   　　修改一下 trap_dispatch 函数，使系统能够把缺页异常引导到 page_fault_handler() 上执行。在修改完成后，运行 make grade，出现的结果应该是你修改后的 JOS 可以成功运行 faultread，faultreadkernel，faultwrite，faultwritekernel 测试程序。

　   答：

　　　 根据 trapentry.S 文件中的 TRAPHANDLER 函数可知，这个函数会把当前中断的中断码压入堆栈中，再根据 inc/trap.h 文件中的 Trapframe 结构体我们可以知道，Trapframe 中的 tf_trapno 成员代表这个中断的中断码。所以在 trap_dispatch 函数中我们需要根据输入的 Trapframe 指针 tf 中的 tf_trapno 成员来判断到来的中断是什么中断，这里我们需要判断是否是缺页中断，如果是则执行 page_fault_handler 函数，所以我们可以这么修改代码：

```c
static void
trap_dispatch(struct Trapframe *tf)
{
        // Handle processor exceptions.
        // LAB 3: Your code here.
        int32_t ret_code;
        switch(tf->tf_trapno) {
                case (T_PGFLT):
                        page_fault_handler(tf);
                        break;
                default:
                        // Unexpected trap: The user process or the kernel has a bug.
                        print_trapframe(tf);
                        if (tf->tf_cs == GD_KT)
                                panic("unhandled trap in kernel");
                        else {
                                env_destroy(curenv);
                                return;
                        }
        }
}
```

　我们之后还会对这个中断处理函数进一步改进。