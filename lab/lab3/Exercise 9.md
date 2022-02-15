# Exercise 9

修改kern/trap.c文件，使其能够实现：当在内核模式下发现页错，trap.c 文件会panic。

　　　　提示：

　　　　　为了能够判断这个page fault是出现在内核模式下还是用户模式下，我们应该检查 tf_cs 的低几位。

　　　　　阅读 user_mem_assert （在 kern/pmap.c），并且实现 user_mem_check;

　　　　　修改一下 kern/syscall.c 去检查输入参数。

　　　    启动内核后，运行 user/buggyhello 程序，用户环境可以被销毁，内核不可以panic，你应该看到：

　　　　　　[00001000] user_mem_check assertion failure for va 00000001
   　　　 [00001000] free env 00001000
         Destroyed the only environment - nothing more to do!

 　　 答：

　　　　　首先我们应该根据什么来判断当前运行的程序时处在内核态下还是用户态下？答案是根据 CS 段寄存器的低2位，这两位的名称叫做 CPL 位，表示当前运行的代码的访问权限级别，0代表是内核态，3代表是用户态。

　　　　　题目要求我们在检测到这个 page fault 是出现在内核态时，要把这个事件 panic 出来，所以我们把 page_fault_handler 文件修改如下：

```c
void
page_fault_handler(struct Trapframe *tf)
{
        uint32_t fault_va;

        // Read processor's CR2 register to find the faulting address
        fault_va = rcr2();

        // Handle kernel-mode page faults.

        // LAB 3: Your code here.
        if(tf->tf_cs && 0x01 == 0) {
                panic("page_fault in kernel mode, fault address %d\n", fault_va);
        }
        // We've already handled kernel-mode exceptions, so if we get here,
        // the page fault happened in user mode.

        // Destroy the environment that caused the fault.
        cprintf("[%08x] user fault va %08x ip %08x\n",
                curenv->env_id, fault_va, tf->tf_eip);
        print_trapframe(tf);
        env_destroy(curenv);
}
```

然后根据题目的要求，我们还要继续完善 kern/pmap.c 文件中的 user_mem_assert , user_mem_check 函数，通过观察 user_mem_assert 函数我们发现，它调用了 user_mem_check 函数。而 user_mem_check 函数的功能是检查一下当前用户态程序是否有对虚拟地址空间 [va, va+len] 的 perm| PTE_P 访问权限。

　　　 自然我们要做的事情应该是，先找到这个虚拟地址范围对应于当前用户态程序的页表中的页表项，然后再去看一下这个页表项中有关访问权限的字段，是否包含 perm | PTE_P，只要有一个页表项是不包含的，就代表程序对这个范围的虚拟地址没有 perm|PTE_P 的访问权限。以上就是这段代码的大致思想。

```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
        // LAB 3: Your code here.

        char * end = NULL;
        char * start = NULL;
        start = ROUNDDOWN((char *)va, PGSIZE);
        end = ROUNDUP((char *)(va + len), PGSIZE);
        pte_t *cur = NULL;

        for(; start < end; start += PGSIZE) {
                cur = pgdir_walk(env->env_pgdir, (void *)start, 0);
                if((int)start > ULIM || cur == NULL || ((uint32_t)(*cur) & perm) != perm) {
                        if(start == ROUNDDOWN((char *)va, PGSIZE)) {
                                user_mem_check_addr = (uintptr_t)va;
                        }
                        else {
                          user_mem_check_addr = (uintptr_t)start;
                        }
                return -E_FAULT;
                }
        }

        return 0;
}
```

最后按照题目要求我们还要补全 kern/syscall.c 文件中的一部分内容，即 sys_cputs 函数，这个函数要求检查用户程序对虚拟地指空间 [s, s+len] 是否有访问权限，所以我们恰好可以使用刚刚写好的函数 user_mem_assert() 来实现。

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

至此，我们就完成了这个练习，可以运行一下 make run-buggyhello，看一下它是否按照题目的要求输出了信息。