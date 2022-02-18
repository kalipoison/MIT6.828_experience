# Exercise 2

阅读 kern/init.c 中的 boot_aps() 和 mp_main() 以及 kern/mpentry.S 中的汇编代码。 确保您了解 AP 引导期间的控制流传输。 

然后修改您在 kern/pmap.c 中的 page_init() 实现，以避免将 MPENTRY_PADDR 处的页面添加到空闲列表中，这样我们就可以安全地在该物理地址复制和运行 AP 引导代码。 

您的代码应该通过更新的 check_page_free_list() 测试（但可能无法通过更新的 check_kern_pgdir() 测试，我们将很快修复）。



答：
　　 在boot_aps函数中将启动代码放到了MPENTRY_PADDR处，而代码的来源则是在kern/mpentry.S中，功能与boot.S中的非常类似，主要就是开启分页，转到内核栈上去，当然这个时候实际上内核栈还没建好。在执行完mpentry.S中的代码之后，将会跳转到mp_main函数中去。而这里需要提前做的，修改 `kern/pmap.c`中的`page_init()`将 MPENTRY_PADDR(0x7000)这一页不要加入到page_free_list。



kern/pmap.c

```c
void page_init(void)
{
    cprintf("Init pages alloc...\n");

    size_t i;
    for (i = 1; i < npages_basemem; i++) {
        if (i * PGSIZE != MPENTRY_PADDR){
            pages[i].pp_ref = 0;
            pages[i].pp_link = page_free_list;
            page_free_list = &pages[i];
        }
    }

    for (i = PGNUM(PADDR(boot_alloc(0))); i < npages; i++) {
        pages[i].pp_ref = 0;
        pages[i].pp_link = page_free_list;
        page_free_list = &pages[i];
    }
    /**
    size_t i;
    page_free_list = NULL;
    // the page number of the extmem area has been used
    int num_alloc = ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE;
    // the page number of the io hole area 
    int num_iohole = 96;
    for (i = 0; i < npages; i++) {
        if (i == 0)
        {
            pages[i].pp_ref = 1;
        }
        else if(i == MPENTRY_PADDR / PGSIZE)
            continue;
        else if(i >= npages_basemem && i < npages_basemem + num_iohole + num_alloc)
        {
            pages[i].pp_ref = 1;
        }
        else
        {
            pages[i].pp_ref = 0;
            pages[i].pp_link = page_free_list;
            page_free_list = &pages[i];
        }
    }
	**/
}
```

