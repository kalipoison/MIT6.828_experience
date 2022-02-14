# Lab2 : Memory Management

lab2中多出来的几个文件：

　　inc/memlayout.h

　　kern/pmap.c

　　kern/pmap.h

　　kern/kclock.h

　　kern/kclock.c

memlayout.h描述了虚拟地址空间的结构，我们需要通过修改pmap.c文件来实现这个结构。memlayout.h和pmap.h文件定义了一个PageInfo结构，利用这个结构可以记录有哪些物理页是空闲的。kclock.c和kclock.h文件中操作的是用电池充电的时钟，以及CMOS RAM设备。在这个设备中记录着PC机拥有的物理内存的数量。在pmap.c中的代码必须读取这个设备中的信息才能弄清楚到底有多少内存。



# Part 1 : Physical Page Management

操作系统必须跟踪物理 RAM 的哪些部分是空闲的，哪些部分当前正在使用。JOS 以*页粒度*管理 PC 的物理内存， 以便它可以使用 MMU 来映射和保护每块分配的内存。

您现在将编写物理页分配器。它通过对象的链接列表跟踪哪些页面是空闲的`struct PageInfo`（与 xv6 不同，这些对象*不*嵌入空闲页面本身），每个对象对应一个物理页面。您需要先编写物理页面分配器，然后才能编写其余的虚拟内存实现，因为您的页表管理代码将需要分配物理内存来存储页表。



**Exercise 1** 

