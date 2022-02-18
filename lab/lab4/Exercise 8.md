# Exercise 8

实现 sys_env_set_pgfault_upcall 系统调用。 在查找目标环境的环境 ID 时，请务必启用权限检查，因为这是一个“危险”的系统调用。



答：

kern/syscall.c

```c
// Set the page fault upcall for 'envid' by modifying the corresponding struct
// Env's 'env_pgfault_upcall' field.  When 'envid' causes a page fault, the
// kernel will push a fault record onto the exception stack, then branch to
// 'func'.
//
// Returns 0 on success, < 0 on error.  Errors are:
//      -E_BAD_ENV if environment envid doesn't currently exist,
//              or the caller doesn't have permission to change envid.
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
        // LAB 4: Your code here.
        //panic("sys_env_set_pgfault_upcall not implemented");
        struct Env *e;
        if (envid2env(envid, &e, 1)) return -E_BAD_ENV;

        e->env_pgfault_upcall = func;
        return 0;
}


int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    ...
		case SYS_page_map:
			return sys_page_map((envid_t)a1, (void*)a2, (envid_t)a3, (void*)a4, (int)a5);
		case SYS_page_unmap:
			return sys_page_unmap((envid_t)a1, (void*)a2);
    	case SYS_env_set_pgfault_upcall:
    		return sys_env_set_pgfault_upcall(a1, (void *)a2);
		default:
			return -E_INVAL;
	}
}

```

