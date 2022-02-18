# Exercise 5

`lock_kernel()`通过调用和`unlock_kernel()`在适当的位置 应用如上所述的大内核锁 。



答：

i386_init–>BSP获得锁–>boot_ap–>(BSP建立为每个cpu建立idle任务、建立用户任务，mp_main)—>BSP的sched_yield–>其中的env_run释放锁–>AP1获得锁–>执行sched_yield–>释放锁–>AP2获得锁–>执行sched_yield–>释放锁…..



kern/init.c

```c
void
i386_init(void)
{
	...
    lock_kernel();
    boot_aps();
    ...
}

...
    
void
mp_main(void)
{
    ...
    lock_kernel();
    sched_yield();
    ...
}
```

kern/trap.c

```c
void
trap(struct Trapframe *tf)
{
	...
    if ((tf->tf_cs & 3) == 3) {
	lock_kernel();
	assert(curenv);
	...
}
```

kern/env.c

```c
void
env_run(struct Env *e)
{
	...
    lcr3(PADDR(curenv->env_pgdir));
    unlock_kernel();
    env_pop_tf(&curenv->env_tf);
    ...
}
```

