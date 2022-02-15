# Exercise 7

给中断向量T_SYSCALL编写一个中断处理函数。你需要去编辑kern/trapentry.S和kern/trap.c中的trap_init()函数。你也需要去修改trap_dispatch()函数，使他能够通过调用syscall()（在kern/syscall.c中定义的）函数处理系统调用中断。最终你需要去实现kern/syscall.c中的syscall()函数。确保这个函数会在系统调用号为非法值时返回-E_INVAL。你应该充分理解lib/syscall.c文件。我们要处理在inc/syscall.h文件中定义的所有系统调用。

　　　　通过make run-hello指令来运行 user/hello 程序，它应该在控制台上输出 "hello, world" 然后出发一个页中断。如果没有发生的话，代表你编写的系统调用处理函数是不正确的。



答：

　　　　我们需要了解一下系统调用的整个流程，如果现在运行的是内核态的程序的话，此时调用了一个系统调用，比如 sys_cputs 函数时，此时不会触发中断，那么系统会直接执行定义在 lib/syscall.c 文件中的 sys_cputs，我们可以看一下这个文件，可以发现这个文件中定义了几个比较常用的系统调用，包括 sys_cputs, sys_cgetc 等等。我们还会发现他们都是统一调用一个 syscall 函数，通过这个函数的代码发现其实它是执行了一个汇编指令。所以最终是这个函数完成了系统调用。

　　　　以上是运行在内核态下的程序，调用系统调用时的流程。

　　　　但是如果是用户态程序呢？这个练习就是让我们编写程序使我们的用户程序在调用系统调用时，最终也能经过一系列的处理最终去执行 lib/syscall.c 中的 syscall 指令。

　　　　让我们看一下这个过程，当用户程序中要调用系统调用时，比如 sys_cputs，从它的汇编代码中我们会发现，它会执行一个 int $0x30 指令，这个指令就是软件中断指令，这个中断的中断号就是 0x30，即 T_SYSCALL，所以题目中让我们首先为这个中断号编写一个中断处理函数，我们首先就要在 kern/trapentry.S 文件中为它声明它的中断处理函数，即TRAPHANDLER_NOEC，就像我们为其他中断号所做的那样。



```assembly
 ...
 64 TRAPHANDLER_NOEC(t_fperr, T_FPERR)
 65 TRAPHANDLER(t_align, T_ALIGN)
 66 TRAPHANDLER_NOEC(t_mchk, T_MCHK)
 67 TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)
 68 
 69 TRAPHANDLER_NOEC(t_syscall, T_SYSCALL)
 70 
 71 /*
 72  * Lab 3: Your code here for _alltraps
 73  */
 74 _alltraps:
 ...

```

　

然后在trap.c 文件中声明 t_syscall() 函数。并且在 trap_init() 函数中为它注册

```c
 ...
void t_fperr();
void t_align();
void t_mchk();
void t_simderr();

void t_syscall();
...

void
trap_init(void)
{
        extern struct Segdesc gdt[];
		...
		SETGATE(idt[T_FPERR], 0, GD_KT, t_fperr, 0);
        SETGATE(idt[T_ALIGN], 0, GD_KT, t_align, 0);
        SETGATE(idt[T_MCHK], 0, GD_KT, t_mchk, 0);
        SETGATE(idt[T_SIMDERR], 0, GD_KT, t_simderr, 0);

        SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);
        // Per-CPU setup 
        trap_init_percpu();
}

```

此时当系统调用中断发生时，系统就可以捕捉到这个中断了，中断发生时，系统会调用 _alltraps 代码块，并且最终来到 trap() 函数处，进入trap函数后，经过一系列处理进入 trap_dispatch 函数。题目中要求此时我们需要去调用 kern/syscall.c 中的syscall函数，这里注意，这个函数可不是 lib/syscall.c 中的 syscall 函数，但是通过阅读 kern/syscall.c 中的 syscall 程序我们发现，它的输入和 lib/syscall.c 中的 syscall 很像，如下

　　　　kern/syscall.c 中的 syscall ：

　　　　　　syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)

　　　　lib/syscall.c 中的 syscall ：

　　　　　　syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)

　　　　所以我们可以假象一下，是不是 kern/syscall.c 中的 syscall 就是一个外壳函数，它的存在就是为了能够调用 lib/syscall 的呢？ 所以我们按照这个思路继续进行下去，我们再继续观察 kern/syscall.c 中的其他函数，会惊人的发现，kern/syscall.c 中的所有函数居然和 lib/syscall.c 中的所有函数都是一样的！！比如 在这两个文件中都有 sys_cputs 函数，但是我们仔细观察可以发现这两个同名的函数，实现方式却不一样。拿 sys_cputs 函数举例

　　　　在 kern/syscall.c 中的 sys_cputs 是这样的：

```c
static void
sys_cputs(const char *s, size_t len)
{
        // Check that the user has permission to read memory [s, s+len).
        // Destroy the environment if not.

        // LAB 3: Your code here.
        user_mem_assert(curenv, s, len, 0);
        // Print the string supplied by the user.
        cprintf("%.*s", len, s);
}
```

而在 lib/syscall.c 中的 sys_cputs 是这样的

```c
void
sys_cputs(const char *s, size_t len)
{
        syscall(SYS_cputs, 0, (uint32_t)s, len, 0, 0, 0);
}
```

​				可见在 lib/syscall.c 中，是直接调用 syscall 的，但是注意观察 kern/syscall.c 中的 sys_cputs，它调用了 cprintf，这个调用其实就是为了完成输出的功能，但是我们要注意，当我们程序运行到这里时，系统已经工作在内核态了，而cprintf函数其实就是通过调用 lib/syscall.c 中的 sys_cputs 来实现的，由于此时系统已经处于内核态了，所以这个 sys_cputs 可以被执行了！所以 kern/syscall.c 中的 sys_cputs 函数通过调用 cprintf 实现了调用 lib/syscall.c 中的 syscall ！这正是我们一开始要实现的目标！

　　　　所以剩下的就是我们如何在 kern/syscall.c 中的 syscall() 函数中正确的调用 sys_cputs 函数了，当然 kern/syscall.c 中其他的函数也能完成这个功能。所以我们必须根据触发这个系统调用的指令到底想调用哪个系统调用来确定我们该调用哪个函数。

　　　　那么怎么知道这个指令是要调用哪个系统调用呢？答案是根据 syscall 函数中的第一个参数，syscallno，那么这个值其实要我们手动传递进去的，这个值存在哪里？通过阅读 lib/syscall.c 中的syscall函数我们可以知道它存放在 eax寄存器中，大家可以自己思考下这个是为什么。所以我们来最后完成 trap_dispatch 和 kern/syscall.c 中的 syscall 函数的代码。

​	/kern/trap.c

　trap_dispatch:

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
                case (T_SYSCALL):
                        // print_trapframe(tf);
                        ret_code = syscall(
                                tf->tf_regs.reg_eax,
                                tf->tf_regs.reg_edx,
                                tf->tf_regs.reg_ecx,
                                tf->tf_regs.reg_ebx,
                                tf->tf_regs.reg_edi,
                                tf->tf_regs.reg_esi);
                        tf->tf_regs.reg_eax = ret_code;
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

kern/syscall.c 中的 syscall()

```c
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
        // Call the function corresponding to the 'syscallno' parameter.
        // Return any appropriate return value.
        // LAB 3: Your code here.

        //panic("syscall not implemented");

        switch (syscallno) {
                case (SYS_cputs):
                        sys_cputs((const char *)a1, a2);
                        return 0;
                case (SYS_cgetc):
                        return sys_cgetc();
                case (SYS_getenvid):
                        return sys_getenvid();
                case (SYS_env_destroy):
                        return sys_env_destroy(a1);
                default:
                        return -E_INVAL;
        }
}
```

