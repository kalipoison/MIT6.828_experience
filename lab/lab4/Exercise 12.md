# Exercise 12

在 lib/fork.c 中实现 fork、duppage 和 pgfault。

使用 forktree 程序测试您的代码。 它应该产生以下消息，穿插“新环境”、“自由环境”和“优雅退出”消息。 消息可能不会按此顺序出现，并且环境 ID 可能不同。



答：



```c
static void
pgfault(struct UTrapframe *utf)
{
        void *addr = (void *) utf->utf_fault_va;
        uint32_t err = utf->utf_err;
        int r;

        // Check that the faulting access was (1) a write, and (2) to a
        // copy-on-write page.  If not, panic.
        // Hint:
        //   Use the read-only page table mappings at uvpt
        //   (see <inc/memlayout.h>).

        // LAB 4: Your code here.
        if (!((err & FEC_WR) && (uvpd[PDX(addr)] & PTE_P)
                        && (uvpt[PGNUM(addr)] & PTE_COW) && (uvpt[PGNUM(addr)] & PTE_P)))
                panic("page cow check failed");
        // Allocate a new page, map it at a temporary location (PFTEMP),
        // copy the data from the old page to the new page, then move the new
		// page to the old page's address.
        // Hint:
        //   You should make three system calls.
        addr = ROUNDDOWN(addr, PGSIZE);
        // LAB 4: Your code here.
        if ((r = sys_page_alloc(0, PFTEMP, PTE_P|PTE_U|PTE_W)))
                panic("sys_page_alloc: %e", r);

        memmove(PFTEMP, addr, PGSIZE);

        if ((r = sys_page_map(0, PFTEMP, 0, addr, PTE_P|PTE_U|PTE_W)))
                panic("sys_page_map: %e", r);

        if ((r = sys_page_unmap(0, PFTEMP)))
                panic("sys_page_unmap: %e", r);

        //panic("pgfault not implemented");
}


static int
duppage(envid_t envid, unsigned pn)
{
        int r;

        // LAB 4: Your code here.
        //panic("duppage not implemented");
        void *addr = (void *)(pn * PGSIZE);
        if (uvpt[pn] & (PTE_W|PTE_COW)) {
                if (sys_page_map(0, addr, envid, addr, PTE_COW|PTE_U|PTE_P) < 0)
                        panic("sys_page_map COW:%e", envid);

                if (sys_page_map(0, addr, 0, addr, PTE_COW|PTE_U|PTE_P) < 0)
                        panic("sys_page_map COW:%e", 0);
        } else {
                if (sys_page_map(0, addr, envid, addr, PTE_U|PTE_P) < 0)
                        panic("sys_page_map UP:%e", envid);
        }
        return 0;
}


envid_t
fork(void)
{
        // LAB 4: Your code here.
        //panic("fork not implemented");
        set_pgfault_handler(pgfault);

        envid_t envid = sys_exofork();
        uint8_t *addr;
        if (envid < 0)
                panic("sys_exofork:%e", envid);
        if (envid == 0) {
                thisenv = &envs[ENVX(sys_getenvid())];
                return 0;
        }

        extern unsigned char end[];
        for (addr = (uint8_t *)UTEXT; addr < end; addr += PGSIZE) {
                if ((uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P)
                                && (uvpt[PGNUM(addr)] & PTE_U)) {
                        duppage(envid, PGNUM(addr));
                }
        }

        duppage(envid, PGNUM(ROUNDDOWN(&addr, PGSIZE)));

        int r;
        if ((r = sys_page_alloc(envid, (void *)(UXSTACKTOP - PGSIZE), PTE_P|PTE_U|PTE_W)))
                panic("sys_page_alloc:%e", r);

        extern void _pgfault_upcall();
        sys_env_set_pgfault_upcall(envid, _pgfault_upcall);

        if ((r = sys_env_set_status(envid, ENV_RUNNABLE)))
                panic("sys_env_set_status:%e", r);

        return envid;
}

```

