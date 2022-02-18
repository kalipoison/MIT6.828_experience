# Exercise 9

实现 kern/trap.c 中 page_fault_handler 中的代码，以将页面错误分派到用户模式处理程序。 写入异常堆栈时，请务必采取适当的预防措施。 （如果用户环境的异常堆栈空间不足会怎样？）



答：

如果当前已经在用户错误栈上了，那么需要留出4个字节，否则不需要，具体和跳转机制有关系。简单说就是在当前的错误栈顶的位置向下留出保存UTrapframe的空间，然后将tf中的参数复制过来。修改当前进程的程序计数器和栈指针，然后重启这个进程，此时就会在用户错误栈上运行中断处理程序了。当然，中断处理程序运行结束之后，需要再回到用户运行栈中，这个就是异常处理程序需要做的了。

```c
void
page_fault_handler(struct Trapframe *tf)
{    
    // LAB 4: Your code here.
    if (curenv->env_pgfault_upcall) {
        struct UTrapframe *utf;
        if (tf->tf_esp >= UXSTACKTOP-PGSIZE && tf->tf_esp <= UXSTACKTOP-1) {
            utf = (struct UTrapframe *)(tf->tf_esp - sizeof(struct UTrapframe) - 4);
        } else {
            utf = (struct UTrapframe *)(UXSTACKTOP - sizeof(struct UTrapframe));
        }

        user_mem_assert(curenv, (void*)utf, 1, PTE_W);
        utf->utf_fault_va = fault_va;
        utf->utf_err = tf->tf_err;
        utf->utf_regs = tf->tf_regs;
        utf->utf_eip = tf->tf_eip;
        utf->utf_eflags = tf->tf_eflags;
        utf->utf_esp = tf->tf_esp;

        curenv->env_tf.tf_eip = (uintptr_t)curenv->env_pgfault_upcall;
        curenv->env_tf.tf_esp = (uintptr_t)utf;
        env_run(curenv);
    }
    // Destroy the environment that caused the fault.
    cprintf("[%08x] user fault va %08x ip %08x\n",
            curenv->env_id, fault_va, tf->tf_eip);
    print_trapframe(tf);
    env_destroy(curenv);
}
```

如果异常栈发生了overflow怎么办？看一下memlayout.h就知道了。用户异常栈就一页的大小，一旦溢出，访问的就是内核都没有访问权限的空间，会发生内核空间中的page fault，此时会直接panic，不会造成更严重的后果。