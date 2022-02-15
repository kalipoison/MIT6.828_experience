# Exercise 2

在文件 env.c中，完成下列函数：

1. env_init(): 初始化所有的在envs数组中的 Env结构体，并把它们加入到 env_free_list中。 还要调用env_init_percpu，这个函数要配置段式内存管理系统，让它所管理的段，可能具有两种访问优先级其中的一种，一个是内核运行时的0优先级，以及用户运行时的3优先级。

2. env_setup_vm(): 为一个新的用户环境分配一个页目录表，并且初始化这个用户环境的地址空间中的和内核相关的部分。

3. region_alloc(): 为用户环境分配物理地址空间

4. load_icode(): 分析一个ELF文件，类似于boot loader做的那样，我们可以把它的内容加载到用户环境下。

5. env_create(): 利用env_alloc函数和load_icode函数，加载一个ELF文件到用户环境中

6. env_run(): 在用户模式下，开始运行一个用户环境。





答：

​	env_init函数很简单，就是遍历 envs 数组中的所有 Env 结构体，把每一个结构体的 env_id 字段置0，因为要求所有的 Env 在 env_free_list 中的顺序，要和它在 envs 中的顺序一致，所以需要采用头插法。

```c
void
env_init(void)
{
        // Set up envs array
        // LAB 3: Your code here.
        int i;
        env_free_list = NULL;
        for(i = NENV - 1; i >= 0 ; i--){
                envs[i].env_id = 0;
                envs[i].env_status = ENV_FREE;
                envs[i].env_link = env_free_list;
                env_free_list = &envs[i];
        }
        // Per-CPU part of the initialization
        env_init_percpu();
}
```

​	env_setup_vm 函数主要是初始化新的用户环境的页目录表，不过只设置页目录表中和操作系统内核跟内核相关的页目录项，用户环境的页目录项不要设置，因为所有用户环境的页目录表中和操作系统相关的页目录项都是一样的（除了虚拟地址UVPT，这个也会单独进行设置），所以我们可以参照 kern_pgdir 中的内容来设置 env_pgdir 中的内容。

```c
static int
env_setup_vm(struct Env *e)
{
        int i;
        struct PageInfo *p = NULL;

        // Allocate a page for the page directory
        if (!(p = page_alloc(ALLOC_ZERO)))
                return -E_NO_MEM;

        // LAB 3: Your code here.
        e->env_pgdir = (pde_t *)page2kva(p);
        p->pp_ref++;

        // Map the directory below UTOP
        for(i = 0 ; i < PDX(UTOP) ; i++){
                e->env_pgdir[i] = 0;
        }

        // Map the directory above UTOP
        for(i = PDX(UTOP) ; i < NPDENTRIES ; i++){
                e->env_pgdir[i] = kern_pgdir[i];
        }

        // UVPT maps the env's own page table read-only.
        // Permissions: kernel R, user R
        e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

        return 0;
}


```



​	region_alloc 为用户环境分配物理空间，这里注意我们要先把起始地址和终止地址进行页对齐，对其之后我们就可以以页为单位，为其一个页一个页的分配内存，并且修改页目录表和页表。代码：

```c
static void
region_alloc(struct Env *e, void *va, size_t len)
{
        // LAB 3: Your code here.
        // (But only if you need it for load_icode.)
        //
        // Hint: It is easier to use region_alloc if the caller can pass
        //   'va' and 'len' values that are not page-aligned.
        //   You should round va down, and round (va + len) up.
        //   (Watch out for corner-cases!)
        void* start = (void *)ROUNDDOWN((uint32_t)va, PGSIZE);
        void* end = (void *)ROUNDUP((uint32_t)va+len, PGSIZE);
        struct PageInfo *p = NULL;
        void* i;
        int r;
        for(i=start; i<end; i+=PGSIZE){
                p = page_alloc(0);
                if(p == NULL)
                panic(" region alloc, allocation failed.");

                r = page_insert(e->env_pgdir, p, i, PTE_W | PTE_U);
                if(r != 0) {
                        panic("region alloc error");
                }
        }
}

```

​	

​	load_icode 功能是为每一个用户进程设置它的初始代码区，堆栈以及处理器标识位。每个用户程序都是ELF文件，所以我们要解析该ELF文件。 代码：

```c
static void
load_icode(struct Env *e, uint8_t *binary)
{
        // LAB 3: Your code here.
        struct Elf* header = (struct Elf*)binary;

        if(header->e_magic != ELF_MAGIC) {
                panic("load_icode failed: The binary we load is not elf.\n");
        }

        if(header->e_entry == 0){
                panic("load_icode failed: The elf file can't be excuterd.\n");
        }

        e->env_tf.tf_eip = header->e_entry;

        lcr3(PADDR(e->env_pgdir));   //?????

        struct Proghdr *ph, *eph;
        ph = (struct Proghdr* )((uint8_t *)header + header->e_phoff);
        eph = ph + header->e_phnum;
        for(; ph < eph; ph++) {
                if(ph->p_type == ELF_PROG_LOAD) {
                        if(ph->p_memsz - ph->p_filesz < 0) {
                                panic("load icode failed : p_memsz < p_filesz.\n");
                        }

                        region_alloc(e, (void *)ph->p_va, ph->p_memsz);
                        memmove((void *)ph->p_va, binary + ph->p_offset, ph->p_filesz);
                        memset((void *)(ph->p_va + ph->p_filesz), 0, ph->p_memsz - ph->p_filesz);
                }
        }
        // Now map one page for the program's initial stack
        // at virtual address USTACKTOP - PGSIZE.
        region_alloc(e,(void *)(USTACKTOP-PGSIZE), PGSIZE);
        // LAB 3: Your code here.
}

```

　　　　

​	env_create 是利用env_alloc函数和load_icode函数，加载一个ELF文件到用户环境中。代码：

```c
void
env_create(uint8_t *binary, enum EnvType type)
{
        // LAB 3: Your code here.
        struct Env *e;
        int rc;
        if((rc = env_alloc(&e, 0)) != 0) {
                panic("env_create failed: env_alloc failed.\n");
        }

        load_icode(e, binary);
        e->env_type = type;
}
```

　

​	env_run 是真正开始运行一个用户环境。代码：

```c
void
env_run(struct Env *e)
{
		// LAB 3: Your code here.
        if(curenv != NULL && curenv->env_status == ENV_RUNNING) {
                curenv->env_status = ENV_RUNNABLE;
        }

        curenv = e;
        curenv->env_status = ENV_RUNNING;
        curenv->env_runs++;
        lcr3(PADDR(curenv->env_pgdir));

        env_pop_tf(&curenv->env_tf);

        panic("env_run not yet implemented");
}

```



 用户环境的代码被调用前，操作系统一共按顺序执行了以下几个函数：

```c
　　　　* start (kern/entry.S)

　　　　* i386_init (kern/init.c)

　　　　　　　cons_init

　　　　　　　mem_init

　　　　　　　env_init

　　　　　　　trap_init （目前还未实现）

　　　　  　　env_create

　　　　  　　env_run

　　　　　　　　env_pop_tf
```

​		一旦你完成上述子函数的代码，并且在QEMU下编译运行，系统会进入用户空间，并且开始执行hello程序，直到它做出一个系统调用指令int。但是这个系统调用指令不能成功运行，因为到目前为止，JOS还没有设置相关硬件来实现从用户态向内核态的转换功能。当CPU发现，它没有被设置成能够处理这种系统调用中断时，它会触发一个保护异常，然后发现这个保护异常也无法处理，从而又产生一个错误异常，然后又发现仍旧无法解决问题，所以最后放弃，我们把这个叫做"triple fault"。通常来说，接下来CPU会复位，系统会重启。

　　所以我们马上要来解决这个问题，不过解决之前我们可以使用调试器来检查一下程序要进入用户模式时做了什么。使用make qemu-gdb 并且在 env_pop_tf 处设置断点，这条指令应该是即将进入用户模式之前的最后一条指令。然后进行单步调试，处理会在执行完 iret 指令后进入用户模式。然后依旧可以看到进入用户态后执行的第一条指令了，该指令是一个cmp指令，开始于文件 lib/entry.S 中。 现在使用 b *0x... 设置一个断点在hello文件（obj/user/hello.asm）中的sys_cputs函数中的 int $0x30 指令处。这个int指令是一个系统调用，用来展示一个字符到控制台。如果你的程序运行不到这个int指令，说明有错误。