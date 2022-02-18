# Exercise 4

trap_init_percpu() (kern/trap.c) 中的代码为 BSP 初始化 TSS 和 TSS 描述符。 它在 Lab 3 中工作，但在其他 CPU 上运行时不正确。 更改代码，使其可以在所有 CPU 上运行。 （注意：您的新代码不应再使用全局 ts 变量。）



答：

由于有多个CPU，所以在这里不能使用原先的全局变量ts，应该利用thiscpu指向的CpuInfo结构体和cpunum函数来为每个核的TSS进行初始化。

```c
void
trap_init_percpu(void)
{
    ...
    // Setup a TSS so that we get the right stack
    // when we trap to the kernel.
    thiscpu->cpu_ts.ts_esp0 = (uintptr_t)percpu_kstacks[cpunum()];
    thiscpu->cpu_ts.ts_ss0 = GD_KD;
    thiscpu->cpu_ts.ts_iomb = sizeof(struct Taskstate);

    // Initialize the TSS slot of the gdt.
    gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id] = SEG16(STS_T32A, (uint32_t) (&thiscpu->cpu_ts),
                                                  sizeof(struct Taskstate) - 1, 0);
    gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id].sd_s = 0;

    // Load the TSS selector (like other segment selectors, the
    // bottom three bits are special; we leave them 0)
    ltr(GD_TSS0 + (thiscpu->cpu_id << 3));

    // Load the IDT
    lidt(&idt_pd);
    /**
    // Setup a TSS so that we get the right stack
    // when we trap to the kernel.
    ts.ts_esp0 = KSTACKTOP;
    ts.ts_ss0 = GD_KD;
    ts.ts_iomb = sizeof(struct Taskstate);

    // Initialize the TSS slot of the gdt.
    gdt[GD_TSS0 >> 3] = SEG16(STS_T32A, (uint32_t) (&ts),
                                    sizeof(struct Taskstate) - 1, 0);
    gdt[GD_TSS0 >> 3].sd_s = 0;

    // Load the TSS selector (like other segment selectors, the
    // bottom three bits are special; we leave them 0)
    ltr(GD_TSS0);

    // Load the IDT
    lidt(&idt_pd);
    **/

}
```



完成上述练习后，使用 make qemu CPUS=4(或make qemu-nox CPUS=4) 在具有 4 个 CPU 的 QEMU 中运行 JOS，您应该会看到如下输出：

```
...
Physical memory: 66556K available, base = 640K, extended = 65532K
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_installed_pgdir() succeeded!
SMP: CPU 0 found 4 CPU(s)
enabled interrupts: 1 2
SMP: CPU 1 starting
SMP: CPU 2 starting
SMP: CPU 3 starting
```
