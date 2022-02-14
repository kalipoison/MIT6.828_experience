# Exercise 4

在文件`kern/pmap.c`中，您必须实现以下函数的代码。

```
        pgdir_walk() 
        boot_map_region() 
        page_lookup() 
        page_remove() 
        page_insert()
```

`check_page()`，调用 from `mem_init()`，测试您的页表管理例程。在继续之前，您应该确保它报告成功。



答：



首先完成pgdir_walk函数，函数原型 pgdir_walk(pde_t *pgdir, const void *va, int create)，该函数的功能在注释中解释道：

　　　　**给定一个页目录表指针 pgdir ，该函数应该返回线性地址va所对应的页表项指针。**

　　　　所以在这里我们应该完成以下几个步骤：

　　　　1. 通过页目录表求得这个虚拟地址所在的页表页对于与页目录中的页目录项地址 dic_entry_ptr。(7-8)

　　　　2. 判断这个页目录项对应的页表页是否已经在内存中。 (10)

　　　　3. 如果在，计算这个页表页的基地址page_base，然后返回va所对应页表项的地址 &page_base[page_off] (23-25)

　　　　4. 如果不在则，且create为true则分配新的页，并且把这个页的信息添加到页目录项dic_entry_ptr中。(11-18)

　　　　5. 如果create为false，则返回NULL。(19-20)

```c
 1 pte_t * pgdir_walk(pde_t *pgdir, const void * va, int create)
 2 {
 3       unsigned int page_off;
 4       pte_t * page_base = NULL;
 5       struct PageInfo* new_page = NULL;
 6       
 7       unsigned int dic_off = PDX(va);
 8       pde_t * dic_entry_ptr = pgdir + dic_off;
 9 
10       if(!(*dic_entry_ptr & PTE_P))
11       {
12             if(create)
13             {
14                    new_page = page_alloc(1);
15                    if(new_page == NULL) return NULL;
16                    new_page->pp_ref++;
17                    *dic_entry_ptr = (page2pa(new_page) | PTE_P | PTE_W | PTE_U);
18             }
19            else
20                return NULL;      
21       }  
22    
23       page_off = PTX(va);
24       page_base = KADDR(PTE_ADDR(*dic_entry_ptr));
25       return &page_base[page_off];
26 } 
```

接下来完成boot_map_region函数，函数原型 static void boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)，这个函数的功能在注释中被这样解释：

　　　　把虚拟地址空间范围[va, va+size)映射到物理空间[pa, pa+size)的映射关系加入到页表pgdir中。这个函数主要的目的是为了设置虚拟地址UTOP之上的地址范围，这一部分的地址映射是静态的，在操作系统的运行过程中不会改变，所以这个页的PageInfo结构体中的pp_ref域的值不会发生改变。

　　　　这个函数要完成的步骤如下：

   1. 需要完成一个循环，在每一轮中，把一个虚拟页和物理页的映射关系存放到响应的页表项中。直到把size个字节的内存都分配完。

      ```c
      static void
      boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
      {
          int nadd;
          pte_t *entry = NULL;
          for(nadd = 0; nadd < size; nadd += PGSIZE)
          {
              entry = pgdir_walk(pgdir,(void *)va, 1);    //Get the table entry of this page.
              *entry = (pa | perm | PTE_P);
      
      
              pa += PGSIZE;
              va += PGSIZE;
      
          }
      }
      ```

      

接下来再继续查看page_insert()，函数原型如下 page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)，功能上是完成：把一个物理内存中页pp与虚拟地址va建立映射关系。

　　　　这个函数的主要步骤如下：

　　　　1. 首先通过pgdir_walk函数求出虚拟地址va所对应的页表项。(4)

　　　　2. 修改pp_ref的值。(8)

　　　　3. 查看这个页表项，确定va是否已经被映射，如果被映射，则删除这个映射。(9-13)

　　　　4. 把va和pp之间的映射关系加入到页表项中。(14-15)

```c
 1 int
 2 page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
 3 {
 4     pte_t *entry = NULL;
 5     entry =  pgdir_walk(pgdir, va, 1);    //Get the mapping page of this address va.
 6     if(entry == NULL) return -E_NO_MEM;
 7 
 8     pp->pp_ref++;
 9     if((*entry) & PTE_P)             //If this virtual address is already mapped.
10     {
11         tlb_invalidate(pgdir, va);
12         page_remove(pgdir, va);
13     }
14     *entry = (page2pa(pp) | perm | PTE_P);
15     pgdir[PDX(va)] |= perm;                  //Remember this step!
16         
17     return 0;
18 }
```

　　这里要注意，pp->pp_ref++这条语句，一定要放在page_remove之前，这是为了处理一种特殊情况：pp已经映射到va上了。至于为什么要这么做，大家可以思考一下。

　　　

　　　　接下来继续完成page_lookup()函数，函数原型：struct PageInfo * page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)， 函数的功能为：

　　　　返回虚拟地址va所映射的物理页的PageInfo结构体的指针，如果pte_store参数不为0，则把这个物理页的页表项地址存放在pte_store中。

　　　　这个函数的功能就很容易实现了，我们只需要调用pgdir_walk函数获取这个va对应的页表项，然后判断这个页是否已经在内存中，如果在则返回这个页的PageInfo结构体指针。并且把这个页表项的内容存放到pte_store中。

```c
 1 struct PageInfo *
 2 page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
 3 {
 4     pte_t *entry = NULL;
 5     struct PageInfo *ret = NULL;
 6 
 7     entry = pgdir_walk(pgdir, va, 0);
 8     if(entry == NULL)
 9         return NULL;
10     if(!(*entry & PTE_P))
11         return NULL;
12     
13     ret = pa2page(PTE_ADDR(*entry));
14     if(pte_store != NULL)
15     {
16         *pte_store = entry;
17     }
18     return ret;
19 }
```

 最后一个就是page_remove函数，它的原型是：void page_remove(pde_t *pgdir, void *va)，功能就是把虚拟地址va和物理页的映射关系删除。

　　　　注释里面还提示了要注意的几个细节：

　　　　1. pp_ref值要减一

　　　　2. 如果pp_ref减为0，要把这个页回收

　　　　3. 这个页对应的页表项应该被置0

　　　　代码：

```c
 1 void
 2 page_remove(pde_t *pgdir, void *va)
 3 {
 4     pte_t *pte = NULL;
 5     struct PageInfo *page = page_lookup(pgdir, va, &pte);
 6     if(page == NULL) return ;    
 7     
 8     page_decref(page);
 9     tlb_invalidate(pgdir, va);
10     *pte = 0;
11 }
```

