# Exercise 11

完成 lib/pgfault.c 中的 set_pgfault_handler()。



答：

进程在运行前注册自己的页错误处理程序，重点是申请用户异常栈空间，最后添加上系统调用号。

```c
void
set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
        int r;

        if (_pgfault_handler == 0) {
                // First time through!
                // LAB 4: Your code here.
                //panic("set_pgfault_handler not implemented");
                if (sys_page_alloc(0, (void *)(UXSTACKTOP - PGSIZE), PTE_W|PTE_U|PTE_P)) {
                        panic("set_pgfault_handler page_alloc failed");
                }
                if (sys_env_set_pgfault_upcall(0, _pgfault_upcall)) {
                        panic("set_pgfault_handler set_pgfault_upcall failed");
                }
        }

        // Save handler pointer for assembly to call.
        _pgfault_handler = handler;
}

```

完成作业11后，可以看到`user/faultread`，`user/faultalloc`,`user/faultallocbad`等可以正常通过测试了。这里要注意 faultalloc和faultallocbad为什么会表现不同？看代码可以知道 faultallocbad 是直接通过系统调用 sys_puts 输出字符串的，而在内核的sys_cputs中有对访问地址进行检查 `user_mem_assert(curenv, s, len, 0);`，因为访问了非法地址，所以直接报错了。而faultalloc是通过cprintf访问的，在调用 sys_cputs之前，会将要输出的字符串存储到 printbuf中，此时会访问地址 0xdeadbeef，从而导致页错误。

这里有意思的是访问 0xcafebffe 这个地址的内容会报两次页错误，因为第一次是 0xcafebffe 处没有映射，于是会分配 0xcafeb000到0xcafebfff的一页。而后面输出 `this string...` 时，因为snprintf访问到了 0xcafec000 后面的地址，导致再次发生页错误，所以会输出两行fault。

```
# make run-faultalloc
fault deadbeef
this string was faulted in at deadbeef
fault cafebffe
fault cafec000
this string was faulted in at cafebffe

# make run-faultallocbad
[00001000] user_mem_check assertion failure for va deadbeef
```

