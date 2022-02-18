# Exercise 1

　　mmio_map_region在kern/pmap.c中 实现。要了解它是如何使用的，请查看 kern/lapic.c的开头lapic_init。在运行测试之前，您还必须进行下一个练习 。 mmio_map_region

　　回答：
　　在kern/lapic.c 中 lapic_init函数的开头就会调用mmio_map.

```c
        // lapicaddr is the physical address of the LAPIC's 4K MMIO
        // region.  Map it in to virtual memory so we can access it.
        lapic = mmio_map_region(lapicaddr, 4096);
```

​		在kern/pmap.c中，有具体的提示，设置个静态变量记录每次变化后的虚拟基地址，使用boot_map_region函数将[pa,pa+size)的物理地址映射到[base,base+size)，记得把size roundup到PGSIZE。由于这是设备内存并不是正常的DRAM，所以使用cache缓存访问是不安全的，我们可以用页的标志位来实现。

kern/pmap.c

```c
void *
mmio_map_region(physaddr_t pa, size_t size)
{
    static uintptr_t base = MMIOBASE;
	uintptr_t start = base;
	base += ROUNDUP(size, PGSIZE);
	boot_map_region(kern_pgdir, start, ROUNDUP(size, PGSIZE), pa, PTE_PCD|PTE_PWT|PTE_W);
	if (base > MMIOLIM) {
		panic("mmio_map_region overflows MMIOLIM");
	}
	return (void*)start;
}
```

完成mmio_map_region函数，用到boot_map_region，注意对begin和end分别向下向上取整。另外设置的page的权限是 `PTE_PCD|PTE_PWT|PTE_W`，权限不要搞错了。
