# Exercise 1

在文件 kern/pmap.c 中，你必须要完成以下几个子函数的代码boot_alloc();  mem_init();  page_init();   page_alloc();   page_free(); check_page_free_list()和check_page_alloc()两个函数将会检测你写的页分配器代码是否正确。



## 答：

我们观察一下pmap.c中的代码，其中最重要的函数就是```mem_init()``` 了，在内核刚开始运行时就会调用这个子函数，对整个操作系统的内存管理系统进行一些初始化的设置，比如设定页表等等操作。

```
// Set up a two-level page table:
//    kern_pgdir is its linear (virtual) address of the root
//
// This function only sets up the kernel part of the address space
// (ie. addresses >= UTOP).  The user part of the address space
// will be set up later.
//
// From UTOP to ULIM, the user is allowed to read but not write.
// Above ULIM the user cannot read or write
```

下面进入这个函数，首先这个函数调用 ```i386_detect_memory```  子函数，这个子函数的功能就是检测现在系统中有多少可用的内存空间。

```c
// Find out how much memory the machine has (npages & npages_basemem).
static void
i386_detect_memory(void)
{
        size_t basemem, extmem, ext16mem, totalmem;

        // Use CMOS calls to measure available base & extended memory.
        // (CMOS calls return results in kilobytes.)
        basemem = nvram_read(NVRAM_BASELO);
        extmem = nvram_read(NVRAM_EXTLO);
        ext16mem = nvram_read(NVRAM_EXT16LO) * 64;

        // Calculate the number of physical pages available in both base
        // and extended memory.
        if (ext16mem)
                totalmem = 16 * 1024 + ext16mem;
        else if (extmem)
                totalmem = 1 * 1024 + extmem;
        else
                totalmem = basemem;

        npages = totalmem / (PGSIZE / 1024);
        npages_basemem = basemem / (PGSIZE / 1024);

        cprintf("Physical memory: %uK available, base = %uK, extended = %uK\n",
                totalmem, basemem, totalmem - basemem);
}
```

jos把整个物理内存空间划分成三个部分：

1. 一个是从0x00000~0xA0000，这部分也叫basemem，是可用的。

2. 紧接着是0xA0000~0x100000，这部分叫做IO hole，是不可用的，主要被用来分配给外部设备了。

3. 再紧接着就是0x100000~0x，这部分叫做extmem，是可用的，这是最重要的内存区域。



这个子函数中包括三个变量，其中 npages 记录整个内存的页数， npages_basemem 记录basemem的页数，npages_extmem 记录extmem 的页数

执行完这个函数，下一条指令为：

```c
        // create initial page directory.
        kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
        memset(kern_pgdir, 0, PGSIZE);
```

其中kern_pgdir是一个指针，pde_t *kern_pgdir，它是指向操作系统的页目录表的指针，操作系统之后工作在虚拟内存模式下时，就需要这个页目录表进行地址转换。我们为这个页目录表分配的内存大小空间为PGSIZE，即一个页的大小。并且首先把这部分内存清0。

这里调用了```boot_alloc```函数，这个函数使我们要首先实现的函数：

这个函数就像在注释中说的那样，它只是被用来暂时当做页分配器，之后我们使用的真实页分配器是page_alloc()函数。而这个函数的核心思想就是维护一个静态变量nextfree，里面存放着下一个可以使用的空闲内存空间的虚拟地址，所以每次当我们想要分配n个字节的内存时，我们都需要修改这个变量的值。

所以添加的代码为：

```c
　　　　result = nextfree；
　　　　nextfree = ROUNDUP(nextfree+n, PGSIZE);
　　　　if((uint32_t)nextfree - KERNBASE > (npages*PGSIZE))
   　　　　 panic("Out of memory!\n");
　　　　return result;
```

所以这条```kern_pgdir = (pde_t *) boot_alloc(PGSIZE);```指令就会分配一个页的内存，并且这个页就是紧跟着操作系统内核之后。

再看下一条命令：

```c
　　　　　　kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

这一条指令就是再为页目录表添加第一个页目录表项。通过查看memlayout.h文件，我们可以看到，UVPT的定义是一段虚拟地址的起始地址，0xef400000，从这个虚拟地址开始，存放的就是这个操作系统的页表kern_pgdir，所以我们必须把它和页表kern_pgdir的物理地址映射起来，PADDR(kern_pgdir)就是在计算kern_pgdir所对应的真实物理地址。

下一条命令需要我们去补充，这条命令要完成的功能是分配一块内存，用来存放一个struct PageInfo的数组，数组中的每一个PageInfo代表内存当中的一页。操作系统内核就是通过这个数组来追踪所有内存页的使用情况的。我写的代码如下：

```c
　　　　pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
　　　　memset(pages, 0, npages * sizeof(struct PageInfo));
```

　下一条指令我们将运行一个子函数，page_init()，这个子函数的功能包括：

　　　　1. 初始化pages数组 2.初始化pages_free_list链表，这个数组中存放着所有空闲页的信息

　　　　我们可以到这个函数的定义处具体查看，整个函数是由一个for循环构成，它会遍历所有内存页所对应的在npages数组中的PageInfo结构体，并且根据这个页当前的状态来修改这个结构体的状态，如果页已被占用，那么要把PageInfo结构体中的pp_ref属性置一；如果是空闲页，则要把这个页送入pages_free_list链表中。根据注释中的提示，第0页已被占用，io hole部分已被占用，还有在extmem区域还有一部分已经被占用，所以我们的代码如下：

```c
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

```

初始化关于所有物理内存页的相关数据结构后，进入check_page_free_list(1)子函数，这个函数的功能就是检查page_free_list链表的所谓空闲页，是否真的都是合法的，空闲的。当输入参数为1时，这个函数要在检查前先进行一步额外的操作，对空闲页链表free_page_list进行修改，经过page_init，free_page_list中已经存放了所有的空闲页表，但是他们的顺序是按照页表的编号从大到小排列的。当前操作系统所采用的页目录表entry_pgdir（不是kern_pgdir）中，并没有对大编号的页表进行映射，所以这部分页表我们还不能操作。但是小编号的页表，即从0号页表开始到1023号页表，已经映射过了，所以可以对这部分页表进行操作。那么check_page_free_list(1)要完成的就是把这部分页表对应的PageInfo结构体移动到free_page_list的前端，供操作系统现在使用。

　　剩下的操作就是对你的free_page_list进行检查了。

　　```check_page_free_list(1)```执行完成，我们将进入下一个检查函数```check_page_alloc()```，这个函数的功能是检查```page_alloc()，page_free()```两个子函数是否能够正确运行。所以我们首先要实现这两个子函数。

　　先实现```page_alloc()```函数，通过注释我们可以知道这个函数的功能就是分配一个物理页。而函数的返回值就是这个物理页所对应的PageInfo结构体。

　　所以这个函数的大致步骤应该是：

　　1. 从free_page_list中取出一个空闲页的PageInfo结构体

　　2. 修改free_page_list相关信息，比如修改链表表头

　　3. 修改取出的空闲页的PageInfo结构体信息，初始化该页的内存

　　代码如下：

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
        // Fill this function in
        struct PageInfo *result;
        if (page_free_list == NULL)
                return NULL;

        result= page_free_list;
        page_free_list = result->pp_link;
        result->pp_link = NULL;

        if (alloc_flags & ALLOC_ZERO)
                memset(page2kva(result), 0, PGSIZE);

        return result;
}
```

　然后实现page_free()方法，根据注释可知，这个方法的功能就是把一个页的PageInfo结构体再返回给page_free_list空闲页链表，代表回收了这个页。

　　主要完成以下几个操作：

　　1. 修改被回收的页的PageInfo结构体的相应信息。

　　2. 把该结构体插入回page_free_list空闲页链表。

　　代码如下：