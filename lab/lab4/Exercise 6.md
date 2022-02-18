# Exercise 6

练习 6. 如上所述在 sched_yield() 中实现循环调度。 不要忘记修改 syscall() 以调度 sys_yield()。

确保在 mp_main 中调用 sched_yield()。

修改 kern/init.c 以创建三个（或更多！）环境，它们都运行程序 user/yield.c。

运行 make qemu。 在终止之前，您应该看到环境在彼此之间来回切换五次，如下所示。

也用几个 CPU 进行测试：make qemu CPUS=2。

```
...
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
```



答：

kern/init.c

```c
void
mp_main(void)
{
	...
	// Remove this after you finish Exercise 6
	//for (;;);
}
```



kern/sched.c

```c
sched_yield(void)
{
	struct Env *idle;

	struct Env *cur_env = curenv;
	if (cur_env) {
		for (int i = ENVX(cur_env->env_id) + 1; i < NENV; i++) {
			if (envs[i].env_status == ENV_RUNNABLE) {
				env_run(&envs[i]);
			}
		}
		for (int i = 0; i < ENVX(cur_env->env_id); i++) {
			if (envs[i].env_status == ENV_RUNNABLE) {
				env_run(&envs[i]);
			}
		}
		if (cur_env->env_status == ENV_RUNNING) {
			env_run(cur_env);
		}
	} else {
		for (int i = 0; i < NENV; i++) {
			if (envs[i].env_status == ENV_RUNNABLE) {
				env_run(&envs[i]);
			}
		}
	}
	// sched_halt never returns
	sched_halt();
}
```

kern/syscall.c

```c
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
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
        case SYS_yield:
			sys_yield();
            return 0;
        default:
            return -E_INVAL;
    }
}

```



```c
void
i386_init(void)
{
	...
    #if defined(TEST)
            // Don't touch -- used by grading script!
            ENV_CREATE(TEST, ENV_TYPE_USER);
    #else
            // Touch all you want.
            //ENV_CREATE(user_primes, ENV_TYPE_USER);
            ENV_CREATE(user_yield, ENV_TYPE_USER);
            ENV_CREATE(user_yield, ENV_TYPE_USER);
            ENV_CREATE(user_yield, ENV_TYPE_USER);
    #endif // TEST*
    ...
}
```

完成 exercise 5，6后，应该可以看到下面的输出：

```
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001001, iteration 1.
Back in environment 00001000, iteration 1.
Back in environment 00001002, iteration 1.
Back in environment 00001001, iteration 2.
Back in environment 00001000, iteration 2.
Back in environment 00001002, iteration 2.
Back in environment 00001001, iteration 3.
Back in environment 00001000, iteration 3.
Back in environment 00001002, iteration 3.
Back in environment 00001001, iteration 4.
```

