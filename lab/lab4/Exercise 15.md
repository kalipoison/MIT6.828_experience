# Exercise 15

在 kern/syscall.c 中实现 sys_ipc_recv 和 sys_ipc_try_send。 在实施它们之前阅读两者的评论，因为它们必须一起工作。 当您在这些例程中调用 envid2env 时，您应该将 checkperm 标志设置为 0，这意味着任何环境都可以向任何其他环境发送 IPC 消息，并且内核除了验证目标 envid 是否有效之外，不会进行特殊的权限检查。

然后在 lib/ipc.c 中实现 ipc_recv 和 ipc_send 函数。

使用 user/pingpong 和 user/primes 函数来测试你的 IPC 机制。 user/primes 将为每个素数生成一个新环境，直到 JOS 用完环境。 您可能会发现阅读 user/primes.c 以了解幕后进行的所有分叉和 IPC 会很有趣。



答：

 lib/ipc.c

```c
int32_t
ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
{
        // LAB 4: Your code here.
        //panic("ipc_recv not implemented");
        int result;

        if((result = sys_ipc_recv(pg? pg: (void*)UTOP) < 0)) {
            if(from_env_store)
                *from_env_store = 0;
            if(perm_store)
                *perm_store = 0;
            return result;
        }
        if(from_env_store)
            *from_env_store = thisenv->env_ipc_from;
        if(perm_store)
            *perm_store = thisenv->env_ipc_perm;
        //cprintf("ipc_recv from %08x to %08x value %d\n",thisenv->env_ipc_from, thisenv->env_id, thisenv->env_ipc_value);
        return thisenv->env_ipc_value;
}

void
ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
{
        // LAB 4: Your code here.
        //panic("ipc_send not implemented");
        int result;

        do {
            result = sys_ipc_try_send(to_env, val, pg? pg: (void*)UTOP, perm);
            if (result != 0) {
                sys_yield();
            }
        } while(result == -E_IPC_NOT_RECV);
        if (result != 0) {
            panic("ipc_send failed: %d", result);
        }
}

```



kern/syscall.c

```c
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
	// LAB 4: Your code here.
	//panic("sys_ipc_try_send not implemented");
	struct Env* new_env;
	int result;
	struct Env* cur_env = curenv;

	if ((result = envid2env(envid, &new_env, 0)) < 0){
		return result;
	}
	if (!new_env->env_ipc_recving) {
		return -E_IPC_NOT_RECV;
	}
	if (srcva < (void*)UTOP && new_env->env_ipc_dstva) {
		//cprintf("map pages %08x", srcva);
		if ((size_t)srcva % PGSIZE) {
			cprintf("(size_t)srcva not PGSIZE");
			return -E_INVAL;
		}
		if (perm & ~PTE_SYSCALL) {
			cprintf("perm & ~PTE_SYSCALL");
			return -E_INVAL;
		}
		struct PageInfo *p = NULL;
		pte_t *pte;

		if (!(p = page_lookup(cur_env->env_pgdir, srcva, &pte))) {
			return -E_INVAL;
		}
		if (!(*pte & PTE_W)  && (perm & PTE_W)) {
			return -E_INVAL;
		}
		if ((result = page_insert(new_env->env_pgdir, p, new_env->env_ipc_dstva, perm | PTE_U)) < 0){
			return result;
		}
		new_env->env_ipc_perm = perm;
	} else {
		new_env->env_ipc_perm = 0;
	}
	new_env->env_ipc_recving = false;
	new_env->env_ipc_from = cur_env->env_id;
	new_env->env_ipc_value = value;
	new_env->env_status = ENV_RUNNABLE;
	new_env->env_tf.tf_regs.reg_eax = 0; 
	new_env->env_ipc_dstva = 0;
	//cprintf("sys_ipc_try_send to %08x %d %08x %d curenv id %08x\n", new_env->env_id, value, srcva, perm, cur_env->env_id);
	return 0;
}


static int
sys_ipc_recv(void *dstva)
{
        // LAB 4: Your code here.
        //panic("sys_ipc_recv not implemented");
        int result;
        struct Env* cur_env = curenv;

        cur_env->env_ipc_recving = true;
        if (dstva < (void*)UTOP) {
            if ((size_t)dstva % PGSIZE) {
                cprintf("(size_t)dstva not PGSIZE");
                return -E_INVAL;
            }
            cur_env->env_ipc_dstva = dstva;
        }
        //cprintf("sys_ipc_recv %08x wait\n", cur_env->env_id);
        cur_env->env_tf.tf_regs.reg_eax = 0; 
        cur_env->env_ipc_from = 0;
        cur_env->env_ipc_value = 0;
        cur_env->env_status = ENV_NOT_RUNNABLE;
        sched_yield();
        return 0;
}

int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
			...
                case SYS_page_unmap:
                        return sys_page_unmap((envid_t)a1, (void*)a2);
                case SYS_env_set_pgfault_upcall:
                        return sys_env_set_pgfault_upcall(a1, (void *)a2);
                case SYS_ipc_try_send:
                    return sys_ipc_try_send((envid_t)a1, a2, (void*)a3, (unsigned int)a4);
                case SYS_ipc_recv:
                    return sys_ipc_recv((void*)a1);
                default:
                        return -E_INVAL;
			...
}
```

