# pmap.c

### 开头定义

```c
/* These variables are set by mips_detect_memory(ram_low_size); */
static u_long memsize; /* Maximum physical address */
u_long npage;	       /* Amount of memory(in pages) */

Pde *cur_pgdir;

struct Page *pages;
static u_long freemem;

struct Page_list page_free_list; /* Free list of physical pages */
```

一开始定义了一些会用到的变量：

- memsize 最大可用物理内存
- npage 物理内存可以有多少页
- Pde *cur_pgdir 当前页目录项
- *pages 好多个页面的集合
- freemem 空闲内存地址
- page_free_list 空闲页面列表

### mips_detect_memory

```c
void mips_detect_memory(u_int _memsize) {
	/* Step 1: Initialize memsize. */
	memsize = _memsize;

	/* Step 2: Calculate the corresponding 'npage' value. */
	/* Exercise 2.1: Your code here. */
	npage = memsize / PAGE_SIZE;
	printk("Memory size: %lu KiB, number of pages: %lu\\n", memsize / 1024, npage);
}
```

这是要实现的函数，作用是探测硬件可用内存，并完成npage的初始化

memsize，表示总物理内存对应的字节数，npage，表示总物理页数。

### alloc

```c
/* Lab 2 Key Code "alloc" */
/* Overview:
    Allocate `n` bytes physical memory with alignment `align`, if `clear` is set, clear the
    allocated memory.
    This allocator is used only while setting up virtual memory system.
   Post-Condition:
    If we're out of memory, should panic, else return this address of memory we have allocated.*/
void *alloc(u_int n, u_int align, int clear) {
	extern char end[];
	u_long alloced_mem;

	/* Initialize `freemem` if this is the first time. The first virtual address that the
	 * linker did *not* assign to any kernel code or global variables. */
	if (freemem == 0) {
		freemem = (u_long)end; // end
	}

	/* Step 1: Round up `freemem` up to be aligned properly */
	freemem = ROUND(freemem, align);

	/* Step 2: Save current value of `freemem` as allocated chunk. */
	alloced_mem = freemem;

	/* Step 3: Increase `freemem` to record allocation. */
	freemem = freemem + n;

	// Panic if we're out of memory.
	panic_on(PADDR(freemem) >= memsize);

	/* Step 4: Clear allocated chunk if parameter `clear` is set. */
	if (clear) {
		memset((void *)alloced_mem, 0, n);
	}

	/* Step 5: return allocated chunk. */
	return (void *)alloced_mem;
}
```

这里的`end` 是Lab1里面start.S里面填过的那个0x80400000，在mmu.h里面kseg0是分成了两个部分，低地址给内核，高地址给物理内存，分界线就是kernel的end，所以这是物理内存的起点，当没有分配过内存的时候，把`freemem=end`

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/ad6eec52-b46d-4fe5-b480-a79835278400/38861d33-9e8f-4b4b-b31a-39b2c267dae0/Untitled.png)

然后把freemem按照align的要求对齐，就是往上找到最近的align的整倍数

alloced_mem存的是申请内存的起点地址

freemem申请n个内存

这里的freemem是内核地址，通过转化成物理地址来砍断是否溢出

如果标明了clear要给新申请的这一部分内存清零

返回的是指向申请的起始地址的指针

### mips_vm_init

```c
void mips_vm_init() {
	/* Allocate proper size of physical memory for global array `pages`,
	 * for physical memory management. Then, map virtual address `UPAGES` to
	 * physical address `pages` allocated before. For consideration of alignment,
	 * you should round up the memory size before map. */
	pages = (struct Page *)alloc(npage * sizeof(struct Page), PAGE_SIZE, 1);
	printk("to memory %x for struct Pages.\\n", freemem);
	printk("pmap.c:\\t mips vm init success\\n");
}
```

这里是完成内存管理数据结构（page）的空间分配

申请npage个页面，按页大小对齐，需要clear清零

alloc返回的是一个指针，这里把它规定成Page类型的指针

这里其实是申请了npage*PAGE_SIZE大小的地方但是没有都用，只申请了每一块开头的一部分，也就是Page的大小，里面存的是一个pp_link指针块和一个ref计数器

### page_init

```c
/* Overview:
 *   Initialize page structure and memory free list. The 'pages' array has one 'struct Page' entry
 * per physical page. Pages are reference counted, and free pages are kept on a linked list.
 *
 * Hint: Use 'LIST_INSERT_HEAD' to insert free pages to 'page_free_list'.
 */
void page_init(void) {
	/* Exercise 2.3: Your code here. (4/4) */
	/* Step 1: Initialize page_free_list. */
	/* Hint: Use macro `LIST_INIT` defined in include/queue.h. */
	/* Exercise 2.3: Your code here. (1/4) */
	LIST_INIT(&page_free_list);
	/* Step 2: Align `freemem` up to multiple of PAGE_SIZE. */
	/* Exercise 2.3: Your code here. (2/4) */
	freemem = ROUND(freemem, PAGE_SIZE);
	/* Step 3: Mark all memory below `freemem` as used (set `pp_ref` to 1) */
	/* Exercise 2.3: Your code here. (3/4) */
	u_long use = PPN(PADDR(freemem));
	for (u_long i = 0; i < use; i++)
	{
		pages[i].pp_ref = 1;
	}
	/* Step 4: Mark the other memory as free. */
	/* Exercise 2.3: Your code here. (4/4) */
	for (u_long i = use; i < npage; i++)
	{
		pages[i].pp_ref = 0;
		LIST_INSERT_HEAD(&page_free_list, &pages[i], pp_link);
	}
}
```

这里是初始化物理页面管理：

1. 利用LIST_INIT对链表进行初始化
2. 已经使用过的地址进行地址对齐
3. 找到用了的地址对应的页号：具体是先利用PADDR把内核地址转化成物理地址（具体的转换其实就是减去了ULIM，内核地址和物理地址是差了一个偏移量），然后再用PPN取物理页号（就是祭祖里面的物理地址分PPN和PPO，PPO是offset）
4. 把use以前的页面标上ref=1，把没用过的标上ref=0并且加到空页面list里面

### page_alloc

```c
int page_alloc(struct Page **new)
{
	/* Step 1: Get a page from free memory. If fails, return the error code.*/
	struct Page *pp;
	/* Exercise 2.4: Your code here. (1/2) */
	if (LIST_EMPTY(&page_free_list))
		return -E_NO_MEM;
	pp = LIST_FIRST(&page_free_list);
	LIST_REMOVE(pp, pp_link);

	/* Step 2: Initialize this page with zero.
	 * Hint: use `memset`. */
	/* Exercise 2.4: Your code here. (2/2) */
	memset((void *)page2kva(pp), 0, PAGE_SIZE);
	*new = pp;
	return 0;
}
```

这个函数实现了申请空页面，主要步骤是：

1. 如果空页面链表非空，拿走这个链表的头，删掉这个头；如果为空返回报错
2. 利用memset，把对应的内存地址都清零

### page_free

```c
void page_free(struct Page *pp)
{
	assert(pp->pp_ref == 0);
	/* Just insert it into 'page_free_list'. */
	/* Exercise 2.5: Your code here. */
	LIST_INSERT_HEAD(&page_free_list, pp, pp_link);
}
```

assert是一个确认函数：

```c
// mmu.h
#define assert(x)                                                                                  \\
	do {                                                                                       \\
		if (!(x)) {                                                                        \\
			panic("assertion failed: %s", #x);                                         \\
		}                                                                                  \\
	} while (0)
```

确认ref=0，然后把这个页面加入`page_free_list` 里面

指导书里面提到，page_free和page_decref关系密切：

```c
void page_decref(struct Page *pp)
{
	assert(pp->pp_ref > 0);

	/* If 'pp_ref' reaches to 0, free this page. */
	if (--pp->pp_ref == 0)
	{
		page_free(pp);
	}
}
```

当且仅当用户程序主动放弃某个页面、或用户程序执行完毕退出回收所有页面时，会调用 page_decref 来减少页面的引用，当页面引用被减为 0 时会回收该页面。

### pgdir_walk

```c
static int pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte)
{
	Pde *pgdir_entryp;
	struct Page *pp;
	/* Step 1: Get the corresponding page directory entry. */
	/* Exercise 2.6: Your code here. (1/3) */
	pgdir_entryp = pgdir + PDX(va);
	/* Step 2: If the corresponding page table is not existent (valid) then:
	 *   * If parameter `create` is set, create one. Set the permission bits 'PTE_C_CACHEABLE |
	 *     PTE_V' for this new page in the page directory. If failed to allocate a new page (out
	 *     of memory), return the error.
	 *   * Otherwise, assign NULL to '*ppte' and return 0.
	 */
	/* Exercise 2.6: Your code here. (2/3) */
	if (!(*pgdir_entryp & PTE_V))
	{
		if (create)
		{
			if (page_alloc(&pp) != 0)
				return -E_NO_MEM;
			pp->pp_ref++;
			*pgdir_entryp = page2pa(pp) | PTE_C_CACHEABLE | PTE_V;
		}
		else
		{
			*ppte = NULL;
			return 0;
		}
	}
	/* Step 3: Assign the kernel virtual address of the page table entry to '*ppte'. */
	/* Exercise 2.6: Your code here. (3/3) */
	Pte *pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));
	*ppte = pgtable + PTX(va);
	return 0;
}
```

实现的功能：将一级页表基地址 pgdir 对应的两级页表结构中 va 虚拟地址所在的二级页表项的指针存储在 ppte 指向的空间上，returns a pointer to the page table entry for virtual address 'va'，返回的是**指向va所在page虚拟地址入口的指针**

注：Pde表示一级页表项类型，Pte表示二级页表项类型

传进来的是一级页目录的基地址，要查的虚拟地址项，如果为空是否创建页面，一个指向二级页表项指针的指针（等待赋值）

1. 找到一级页目录项，通过PDX去到对应页目录项的地址位，加到基地址上

2. 判断：如果没有取到页目录项或者非valid，如果create，就page_alloc一个页面，标注引用，赋给页目录项，具体是把申请来的物理地址或上一些标志位

   如果不create，就赋值NULL，返回

3. 已经找到（或者创建）页目录项，32位的页表项由20位物理页号和12位标志位组成，PTE_ADDR是抹掉低12位，只留下物理页号，因为要实现的是虚拟地址入口，所以这里KADDR把二级页表的基地址转化成了虚拟地址（具体为什么在这一步就转了而不是查出来之后再转在下面解释），然后再这个的基础上加上二级页表偏移量，就能拿到最后要的物理页号

解释一下**为什么在二级页表基地址的时候就用了KADDR:**

```c
#define KADDR(pa)                                                                                  \\
	({                                                                                         \\
		u_long _ppn = PPN(pa);                                                             \\
		if (_ppn >= npage) {                                                               \\
			panic("KADDR called with invalid pa %08lx", (u_long)pa);                   \\
		}                                                                                  \\
		(pa) + ULIM;                                                                       \\
	})
// 这里是KADDR的代码，带入这里就是(PTE_ADDR(*pgdir_entryp))+ULIM
// 如果我想先取出来正确的物理地址，就是(Pte *)PTE_ADDR(*pgdir_entryp)+PTX(va)
// 把它放到KADDR里面，就是((Pte *)PTE_ADDR(*pgdir_entryp)+PTX(va))+ULIM
// 这是以指针的方式进行的加法！
// 所以实际上加了4*(0x80000000)大小，溢出了已经
// 所以如果printk的话，KADDR是没有用的
```

### page_insert

```c
int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm)
{
	Pte *pte;

	/* Step 1: Get corresponding page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);

	if (pte && (*pte & PTE_V))
	{
		if (pa2page(*pte) != pp)
		{
			page_remove(pgdir, asid, va);
		}
		else
		{
			tlb_invalidate(asid, va);
			*pte = page2pa(pp) | perm | PTE_C_CACHEABLE | PTE_V;
			return 0;
		}
	}

	/* Step 2: Flush TLB with 'tlb_invalidate'. */
	/* Exercise 2.7: Your code here. (1/3) */
	tlb_invalidate(asid, va);
	/* Step 3: Re-get or create the page table entry. */
	/* If failed to create, return the error. */
	/* Exercise 2.7: Your code here. (2/3) */
	if (pgdir_walk(pgdir, va, 1, &pte))
		return -E_NO_MEM;
	/* Step 4: Insert the page to the page table entry with 'perm | PTE_C_CACHEABLE | PTE_V'
	 * and increase its 'pp_ref'. */
	/* Exercise 2.7: Your code here. (3/3) */
	*pte = page2pa(pp) | perm | PTE_C_CACHEABLE | PTE_V;
	pp->pp_ref++;
	return 0;
}
```

要实现的目标：将物理页面“pp”映射到虚拟地址“va”。如果在“va”处已经映射了一个页面，请调用page_remove（）来释放此映射。

1. 通过pgdir_walk拿到虚拟地址va对应的物理页号
   - 如果有对应的物理页号，且映射的不是函数传进来的这个页号，那就取消va原来的映射
   - 如果就是原来的页号，说明这个函数要实现的目的其实是已经存在的，那就flush TLB（这个现在还不知道是干什么的，等TLB再解释吧），然后对具体的映射物理地址进行一些改变，page2pp是把物理页号左移，留出来右边标志位的位置，然后置上标志位，结束
2. Flush TLB
3. 找va对应的物理页号，如果没有可以申请，申请不对返回报错
4. 和1.2一样，给映射的物理地址置标志位

### page_lookup

```c
struct Page *page_lookup(Pde *pgdir, u_long va, Pte **ppte) {
	struct Page *pp;
	Pte *pte;

	/* Step 1: Get the page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);

	/* Hint: Check if the page table entry doesn't exist or is not valid. */
	if (pte == NULL || (*pte & PTE_V) == 0) {
		return NULL;
	}

	/* Step 2: Get the corresponding Page struct. */
	/* Hint: Use function `pa2page`, defined in include/pmap.h . */
	pp = pa2page(*pte);
	if (ppte) {
		*ppte = pte;
	}
	return pp;
}
```

实现的目的：查找虚拟地址va对应的page（物理的）

1. 通过pgdir_walk找va对应的内核物理地址，放到pte里面，如果没有或者对应的位上面不有效，返回NULL

2. 拿到的是内核物理地址，所以是有12位的标志位，这里使用的pa2page是这样的：

   ```c
   static inline struct Page *pa2page(u_long pa) {
   	if (PPN(pa) >= npage) {
   		panic("pa2page called with invalid pa: %x", pa);
   	}
   	return &pages[PPN(pa)];
   }
   ```

   可以看出来，传入的是一个内核物理地址，用PPN提取物理页号，然后从pages数组里面取这个序号的页面

   这里传入了一个参数ppte，如果有的话把内核虚拟地址赋给它

   返回指向找到了的物理页的指针pp

### page_remove

```c
void page_remove(Pde *pgdir, u_int asid, u_long va) {
	Pte *pte;

	/* Step 1: Get the page table entry, and check if the page table entry is valid. */
	struct Page *pp = page_lookup(pgdir, va, &pte);
	if (pp == NULL) {
		return;
	}

	/* Step 2: Decrease reference count on 'pp'. */
	page_decref(pp);

	/* Step 3: Flush TLB. */
	*pte = 0;
	tlb_invalidate(asid, va);
	return;
}
```

### physical_memory_manage_check

如果所有断言都通过，则该测试表明物理内存管理器正在正常工作。

### page_check

如果所有断言都通过，则该测试表明页表和页面目录正在正常工作。