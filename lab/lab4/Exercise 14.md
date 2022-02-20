# Exercise 14

修改内核的 trap_dispatch() 函数，以便在发生时钟中断时调用 sched_yield() 来查找并运行不同的环境。

您现在应该能够让用户/旋转测试工作：父环境应该分叉子，sys_yield() 给它几次，但在每种情况下，在一个时间片后重新获得对 CPU 的控制，最后杀死 子环境并优雅地终止。



答：

修改trap_dispatch函数，当发生时钟中断时调用sched_yield函数来调度下一个进程。



```c
if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
                lapic_eoi();
                sched_yield();
                return;
 }
```

