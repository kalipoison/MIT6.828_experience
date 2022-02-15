　　Exercise 10

　　　　重新启动你的内核，并且运行 user/evilhello。内核应该不能 panic，并且输出如下信息：

　　　　  [00000000] new env 00001000
   　   [00001000] user_mem_check assertion failure for va f010000c
        [00001000] free env 00001000　　  

  答：

　　　　其实 Exercise 9 完成后，Exercise 10 其实也完成了，你可以直接运行 make run-evilhello，看一下是否输出要求的结果。