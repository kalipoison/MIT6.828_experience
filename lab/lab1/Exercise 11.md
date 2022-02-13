# Exercise 11

上面的练习应该为您提供实现堆栈回溯功能所需的信息，您应该调用`mon_backtrace()`. `这个函数的原型已经在kern/monitor.c`等着你了。您可以完全在 C 中完成，但您可能会发现`inc/x86.h``read_ebp()`中的函数很有用。您还必须将这个新功能挂接到内核监视器的命令列表中，以便用户可以交互地调用它。``

回溯函数应显示以下格式的函数调用帧列表：

```
堆栈回溯：
  ebp f0109e58 eip f0100a62 参数 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8 eip f01000d6 参数 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

每行包含一个`ebp`、`eip`和`args`。`ebp`值表示指向该函数使用的堆栈的基指针：即，刚进入函数后堆栈指针的位置，并且函数序言代码设置了基指针。列出的`eip`值是函数的*返回指令指针*：当函数返回时控制将返回的指令地址。返回指令指针通常指向`调用`指令之后的指令（为什么？）。最后，`args后面列出的五个十六进制值` 是所讨论函数的前五个参数，在函数被调用之前它们会被压入堆栈。当然，如果调用函数时使用的参数少于五个，那么并非所有五个值都有用。（为什么回溯代码不能检测到实际有多少参数？如何解决这个限制？）

打印的第一行反映了*当前正在执行*的函数，即`mon_backtrace`它自己，第二行反映了调用的函数，`mon_backtrace`第三行反映了调用那个函数的函数，以此类推。您应该打印*所有*未完成的堆栈帧。通过研究`kern/entry.S` ，您会发现有一种简单的方法可以判断何时停止。



monitor.c中`mon_backtrace`函数：

```
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
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
		ebp = (uint32_t*) (*ebp);
	}
	return 0;
}
```

