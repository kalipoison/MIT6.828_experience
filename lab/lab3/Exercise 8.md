# Exercise 8

把我们刚刚提到的应该补全的代码补全，然后重新启动内核，此时你应该看到 user/hello 程序会打印 "hello, world", 然后在打印出来 "i am environment 00001000"。user/hello 然后就会尝试退出，通过调用 sys_env_destroy()。由于内核目前仅仅支持一个用户运行环境，所以它应该汇报 “已经销毁用户环境”的消息，然后退回内核监控器(kernel monitor)。

　　　　

答：

　　其实这个练习就是让你通过程序获得当前正在运行的用户环境的 env_id , 以及这个用户环境所对应的 Env 结构体的指针。 env_id 我们可以通过调用 sys_getenvid() 这个函数来获得。那么如何获得它对应的 Env结构体指针呢？

　　通过阅读 lib/env.h 文件我们知道，env_id的值包含三部分，第31位被固定为0；第10~30这21位是标识符，标示这个用户环境；第0~9位代表这个用户环境所采用的 Env 结构体，在envs数组中的索引。所以我们只需知道 env_id 的低 0~9 位，我们就可以获得这个用户环境对应的 Env 结构体了。

lib/libmain.c

```c
void
libmain(int argc, char **argv)
{
        // set thisenv to point at our Env structure in envs[].
        // LAB 3: Your code here.
        thisenv = &envs[ENVX(sys_getenvid())];

        // save the name of the program so that panic() can use it
        if (argc > 0)
                binaryname = argv[0];

        // call user main routine
        umain(argc, argv);

        // exit gracefully
        exit();
}
```

