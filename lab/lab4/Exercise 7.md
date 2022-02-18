# Exercise 7

在 kern/syscall.c 中实现上述系统调用，并确保 syscall() 调用它们。 您将需要使用 kern/pmap.c 和 kern/env.c 中的各种函数，尤其是 envid2env()。 现在，每当您调用 envid2env() 时，请在 checkperm 参数中传递 1。 确保检查任何无效的系统调用参数，在这种情况下返回 -E_INVAL。 使用 user/dumbfork 测试你的 JOS 内核，并确保它在继续之前工作。



答：

首先是sys_exofork函数，这个系统调用将创建1个新的空白进程，没有映射的用户空间且无法运行。在调用函数时新进程的寄存器状态与父进程相同，但是在父进程会返回子进程的ID，而子进程会返回0。通过设置子进程的eax为0，来让系统调用的返回值为0。



接着是sys_env_set_status函数，设置进程的状态为ENV_RUNNABLE或者ENV_NOT_RUNNABLE。



然后是env_page_alloc函数，分配1个物理页并映射到给定进程的进程空间的虚拟地址。



接着是sys_page_map函数，从1个进程的页表中拷贝1个页映射到另1个进程的页表中。将进程id为srcenvid的进程的srcva处的物理页的内容，映射到进程id为dstenvid的进程的dstva处。



最后是sys_page_unmap函数，解除指定进程中的1个页映射。

kern/syscall.c

```c
static envid_t
sys_exofork(void)
{
	//panic("sys_exofork not implemented");
	struct Env* new_env;
	int result;
	static envid_t last_id;
	struct Env* cur_env = curenv;

	if ((result = env_alloc(&new_env, curenv->env_id)) < 0) {
		return result;
	}
	new_env->env_status = ENV_NOT_RUNNABLE;
	new_env->env_tf = cur_env->env_tf;
	new_env->env_tf.tf_regs.reg_eax = 0;
	// cprintf("new_env: %d\n", new_env->env_id);
	return new_env->env_id;
}


static int
sys_env_set_status(envid_t envid, int status)
{
    // LAB 4: Your code here.
	//panic("sys_env_set_status not implemented");
	struct Env* new_env;
	int result;

	if ((result = envid2env(envid, &new_env, 1)) < 0){
		return result;
	}
	if (status != ENV_RUNNABLE && status != ENV_NOT_RUNNABLE) {
		return -E_INVAL;
	}
	new_env->env_status = status;

	return 0;
}


static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
    //panic("sys_page_alloc not implemented");
	struct Env* new_env;
	int result;
	struct PageInfo *p = NULL;

	if ((uintptr_t)va >= UTOP || (uintptr_t)va % PGSIZE ) {
		return -E_INVAL;
	}
	if (perm & ~PTE_SYSCALL) {
		return -E_INVAL;
	}
	if ((result = envid2env(envid, &new_env, 1)) < 0) {
		return result;
	}
	if (!(p = page_alloc(ALLOC_ZERO))) {
		return -E_NO_MEM;
	}
	if ((result = page_insert(new_env->env_pgdir, p, va, perm | PTE_U)) < 0){
		page_free(p);
		return result;
	}
	return 0;
}


static int
sys_page_map(envid_t srcenvid, void *srcva,
	     envid_t dstenvid, void *dstva, int perm)
{
    //panic("sys_page_map not implemented");
	struct Env* src_env;
	struct Env* dst_env;
	int result;
	struct PageInfo *p = NULL;
	pte_t *pte;

	if ((uintptr_t)srcva >= UTOP || (uintptr_t)srcva % PGSIZE || (uintptr_t)srcva >= UTOP || (uintptr_t)srcva % PGSIZE) {
		return -E_INVAL;
	}
	if (perm & ~PTE_SYSCALL) {
		return -E_INVAL;
	}
	if ((result = envid2env(srcenvid, &src_env, 1)) < 0){
		return result;
	}
	if ((result = envid2env(dstenvid, &dst_env, 1)) < 0){
		return result;
	}
	if (!(p = page_lookup(src_env->env_pgdir, srcva, &pte))) {
		return -E_INVAL;
	}
	if (!(*pte & PTE_W)  && (perm & PTE_W)) {
		return -E_INVAL;
	}
	if ((result = page_insert(dst_env->env_pgdir, p, dstva, perm | PTE_U)) < 0){
		return result;
	}
	return 0;
}

static int
sys_page_unmap(envid_t envid, void *va)
{
	//panic("sys_page_unmap not implemented");
	struct Env* new_env;
	int result;

	if ((uintptr_t)va >= UTOP || (uintptr_t)va % PGSIZE ) {
		return -E_INVAL;
	}
	if ((result = envid2env(envid, &new_env, 1)) < 0){
		return result;
	}
	page_remove(new_env->env_pgdir, va);
	return 0;
}


int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    ...
		case (SYS_env_destroy):
			return sys_env_destroy(a1);
    	case SYS_yield:
    		sys_yield();
    		return 0;
		case SYS_exofork:
			return sys_exofork();
		case SYS_env_set_status:
			return sys_env_set_status((envid_t)a1, (int)a2);
		case SYS_page_alloc:
			return sys_page_alloc((envid_t)a1, (void*)a2, (int)a3);
		case SYS_page_map:
			return sys_page_map((envid_t)a1, (void*)a2, (envid_t)a3, (void*)a4, (int)a5);
		case SYS_page_unmap:
			return sys_page_unmap((envid_t)a1, (void*)a2);
		default:
			return -E_INVAL;
	}
}
```




