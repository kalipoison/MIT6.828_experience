# Exercise 6

修改trap_dispatch()使断点异常发生时，能够触发kernel monitor。修改完成后运行 make grade，运行结果应该是你修改后的 JOS 能够正确运行 breakpoint 测试程序。



答：

　　　　这个练习其实和上一个练习是类似的，只不过是在这里我们需要处理断点中断 (T_BRKPT)，kernel monitor 就是定义在 kern/monitor.c 文件中的 monitor 函数，所以修改后的程序如下

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
                case (T_BRKPT):
                        monitor(tf);
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

