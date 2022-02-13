# Exercise 12

 修改堆栈回溯函数以显示每个`eip`对应的函数名、源文件名和行`号`。

在`debuginfo_eip`中，`__STAB_*`来自哪里？这个问题有一个很长的答案；为了帮助您找到答案，以下是您可能想做的一些事情：

- 在文件`kern/kernel.ld`中查找`__STAB_*`
- 跑objdump -h obj/kern/kernel
- 跑objdump -G obj/kern/kernel
- 运行gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c，然后查看 init.s。
- 查看引导加载程序是否将符号表加载到内存中作为加载内核二进制文件的一部分

`debuginfo_eip`通过插入调用来`stab_binsearch`查找地址的行号来完成 的实现。

向内核监视器添加一个`回溯`命令，并扩展您的实现`mon_backtrace`以调用`debuginfo_eip`并打印表单的每个堆栈帧的一行：

```
K>回溯
堆栈回溯：
  ebp f010ff78 eip f01008ae args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: 监视器+106
  ebp f010ffd8 eip f0100193 参数 00000000 00001aac 00000660 00000000 00000000
         内核/init.c:49: i386_init+59
  ebp f010fff8 eip f010003d 参数 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <未知>+0
ķ>
```

`每行给出堆栈帧的eip`的文件名和该文件中的行，然后是函数的名称和`eip`从函数的第一条指令开始的偏移量（例如， `monitor+106`表示返回的`eip`为 106 个字节过去 `监视器`的开头）。

请务必在单独的行上打印文件和函数名称，以避免混淆评分脚本。

提示： printf 格式的字符串提供了一种简单但晦涩难懂的方式来打印非空终止字符串，例如 STABS 表中的字符串。 最多`printf("%.*s", length, string)`打印`length`. `string`查看 printf 手册页以了解其工作原理。

您可能会发现回溯中缺少某些功能。例如，您可能会看到对 的调用， `monitor()`但不会看到对 的调用`runcmd()`。这是因为编译器内联了一些函数调用。其他优化可能会导致您看到意外的行号。如果您从 `GNUMakefile中删除``-O2`，则回溯可能更有意义（但您的内核将运行得更慢）。



这个最后一题需要我们先去读代码，然后修改代码实现一些功能。

首先是需要我们去读懂/kern/kdebug.c里的几个函数。可以看到其中一个函数是利用地址去符号表中进行二分搜索，另一个函数是调用了这个函数，去填充一个代表信息的数据结构。

我们需要补全debuginfo_eip函数，具体就是加这么几句话：

```c
	stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
	if(lline <= rline)
		info -> eip_line = stabs[lline].n_desc;
	else
		return -1;
```

然后需要在之前完成的backtrace加入调用这个函数的部分，然后把backtrace 加入runcmd的列表里。具体搜索一下"help", 就能知道是怎么加的了。

backtrace 函数改成这样：

```
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	// The data structure to save information about EIP
	struct Eipdebuginfo info;
	uint32_t arg[4];
	uint32_t* ebp = (uint32_t*)read_ebp();
	while(ebp != NULL)
	{
		uint32_t eip = *(ebp + 1);
		for(int i = 0; i < 4; i++)
			arg[i] = *(ebp + i + 2);
		cprintf("ebp %08x eip %08x args ", ebp, eip);
		for(int i = 0; i < 4; i++)
			cprintf("%08x ", arg[i]);
		cprintf("\n");
		if(debuginfo_eip(eip, &info) == 0)
		{
			cprintf("%s:%d: ", info.eip_file, info.eip_line);
			cprintf("%.*s", info.eip_fn_namelen, info.eip_fn_name);
			cprintf("+%d\n", eip - (uint32_t)info.eip_fn_addr);
		}
		else
			cprintf("Error happened when reading symbol table\n");
		ebp = (uint32_t*) (*ebp);
	}
	return 0;
}
```

