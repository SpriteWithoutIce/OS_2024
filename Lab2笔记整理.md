# Lab2 笔记整理

![屏幕截图 2024-04-01 095632](./img/屏幕截图%202024-04-01%20095632.png)

Lab2 整体过程分两个部分，内核的初始化和用户进程访问 tlb

## 1 kernel init

在`init/init.c` 里面多了 lab2 的内容：

```c
// lab2:
	mips_detect_memory(ram_low_size);
	mips_vm_init();
	page_init();
```

### mips_detect_memory(ram_low_size)

目的：初始化 memsize，计算有多少个页面 npage

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

作用是探测硬件可用内存，并完成 npage 的初始化

memsize，表示总物理内存对应的字节数，npage，表示总物理页数。

### mips_vm_init( )

目的：建立一个二级页表

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

申请 npage 个页面，按页大小对齐，需要 clear 清零

alloc 返回的是一个指针，这里把它规定成 Page 类型的指针

这里其实是申请了 npage\*PAGE_SIZE 大小的地方但是没有都用，只申请了每一块开头的一部分，也就是 Page 的大小，里面存的是一个 pp_link 指针块和一个 ref 计数器

这一段只是去核实一段内存，内存里面这一段本身也是没有用过的

**这里调用了函数 alloc**

```c
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

这里的`end` 是 Lab1 里面 start.S 里面填过的那个 0x80400000，在 mmu.h 里面 kseg0 是分成了两个部分，低地址给内核，高地址给物理内存，分界线就是 kernel 的 end，所以这是物理内存的起点，当没有分配过内存的时候，把`freemem=end`

![屏幕截图 2024-03-28 103111](./img/屏幕截图%202024-03-28%20103111.png)

然后把 freemem 按照 align 的要求对齐，就是往上找到最近的 align 的整倍数

alloced_mem 存的是申请内存的起点地址

freemem 申请 n 个内存

这里的 freemem 是内核地址，通过转化成物理地址来砍断是否溢出

如果标明了 clear 要给新申请的这一部分内存清零

返回的是指向申请的起始地址的指针

### page_init( )

目的：初始化页结构和内存空链表，页数组里面每个物理有一个 entry，空的页面存在一个链表里面

```c
// 前面的一些定义
LIST_HEAD(Page_list, Page);
extern struct Page *pages;
extern struct Page_list page_free_list;

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

在这里讲一下里面的 page_list 的结构：

![屏幕截图 2024-04-01 174015](./img/屏幕截图%202024-04-01%20174015.png)

这是指导书里面给的示意图

对于每个 Page，有三个部分组成：data 块，le_next 指向下一个 page，le_prev 指向上一个 page 的 le_next

page_list 的结构是这样的：

```c
struct Page_list{
	struct {
		struct {
			struct Page *le_next;
			struct Page **le_prev;
		} pp_link;
		u_short pp_ref;
	}* lh_first;
}
```

可以看出 Page_list 里面其实就只有一个指针，一个 Page 类型的指针，指向第一个黑色框

---

这里是初始化物理页面管理：

1. 利用 LIST_INIT 对链表进行初始化

   这里用到了一个新的函数`LIST_INIT`

   ```c
   // include/queue.h
   #define LIST_INIT(head)        \\
   	do                           \\
   	{                            \\
   		LIST_FIRST((head)) = NULL; \\
   	} while (0)

   #define LIST_FIRST(head) ((head)->lh_first)
   ```

   传入一个链表 head，对第一个元素清空

   这里传入的是 page_free_list，是 Page_list 类型的，Page_list 是通过`LIST_HEAD(Page_list, Page);` 定义的：

   ```c
   #define LIST_HEAD(name, type)                  \\
   	struct name                                  \\
   	{                                            \\
   		struct type *lh_first; /* first element */ \\
   	}
   ```

2. 已经使用过的地址进行地址对齐

   ```c
   /* Rounding; only works for n = power of two */
   #define ROUND(a, n) (((((u_long)(a)) + (n)-1)) & ~((n)-1))
   #define ROUNDDOWN(a, n) (((u_long)(a)) & ~((n)-1))
   ```

   ROUND 是向上找最近的对齐的地址

3. 找到用了的地址对应的页号：具体是先利用 PADDR 把内核地址转化成物理地址（具体的转换其实就是减去了 ULIM，内核地址和物理地址是差了一个偏移量），然后再用 PPN 取物理页号（就是祭祖里面的物理地址分 PPN 和 PPO，PPO 是 offset）

   ```c
   #define PADDR(kva)                                                                                 \\
   	({                                                                                         \\
   		u_long _a = (u_long)(kva);                                                         \\
   		if (_a < ULIM)                                                                     \\
   			panic("PADDR called with invalid kva %08lx", _a);                          \\
   		_a - ULIM;                                                                         \\
   	})
   #define PPN(pa) (((u_long)(pa)) >> PGSHIFT)
   ```

   PADDR 把 va 转成物理地址，PPN 把物理地址右移 12 位，得到的是物理页号

4. 把 use 以前的页面标上 ref=1，把没用过的标上 ref=0 并且加到空页面 list 里面

   这里用到了`LIST_INSERT_HEAD`

   ```c
   #define LIST_INSERT_HEAD(head, elm, field)                          \\
   	do                                                                \\
   	{                                                                 \\
   		if ((LIST_NEXT((elm), field) = LIST_FIRST((head))) != NULL)     \\
   		/*前面拿到的是elm->field.le_next，后面拿到的是lh_first，事实上是一个指向第一个块的指针，
   		现在是让这两个指针相等，也就是le_next指向现有的lh_first指向的那个块*/
   			LIST_FIRST((head))->field.le_prev = &LIST_NEXT((elm), field); \\
   		/*LIST_FIRST((head))取出未变的lh_first指针，它的le_prev应该变成插入的新块elm的le_next*/
   		LIST_FIRST((head)) = (elm);                                     \\
   		/*变更lh_first，仍然是指针，指向插入的这个块的指针*/
   		(elm)->field.le_prev = &LIST_FIRST((head));                     \\
   		/*第一个块的le_prev是要指向自己的*/
   	} while (0)
   ```

## 2 TLB miss

遇到 tlb miss 之后，进入 do_tlb_refill 重填

### do_tlb_refill

这一段其实代码里面的英文注释很完备了

```nasm
NESTED(do_tlb_refill, 24, zero)
	mfc0    a1, CP0_BADVADDR
	mfc0    a2, CP0_ENTRYHI
	andi    a2, a2, 0xff /* ASID is stored in the lower 8 bits of CP0_ENTRYHI */
.globl do_tlb_refill_call;
do_tlb_refill_call:
	addi    sp, sp, -24 /* Allocate stack for arguments(3), return value(2), and return address(1) */
	sw      ra, 20(sp) /* [sp + 20] - [sp + 23] store the return address */
	addi    a0, sp, 12 /* [sp + 12] - [sp + 19] store the return value */
	jal     _do_tlb_refill /* (Pte *, u_int, u_int) [sp + 0] - [sp + 11] reserved for 3 args */
	lw      a0, 12(sp) /* Return value 0 - Even page table entry */
	lw      a1, 16(sp) /* Return value 1 - Odd page table entry */
	lw      ra, 20(sp) /* Return address */
	addi    sp, sp, 24 /* Deallocate stack */
	mtc0    a0, CP0_ENTRYLO0 /* Even page table entry */
	mtc0    a1, CP0_ENTRYLO1 /* Odd page table entry */
	nop
	/* Hint: use 'tlbwr' to write CP0.EntryHi/Lo into a random tlb entry. */
	/* Exercise 2.10: Your code here. */
	tlbwr
	jr      ra
END(do_tlb_refill)
```

前面几行也挺重要，因为写到后面发现找不到 va 的传参了）前面是先设定好\_do_tlb_refill 的传参，从 BADVADDR 里面读出来引发 TLB miss 的虚拟地址，存在 a1 里面，a2 存的是 ASID（先存再与取低 8 位）

这段代码整体上是利用堆栈 sp 先存起来跳转地址，这里的 addi 是为传参做准备，因为调用的 c 语言函数默认 a0 对应的是传入的第一个参数也就是 Pte\*类型的那个，这个一会是要存结果的，所以先给它分配好地方，也就是 sp+12 的这个位置（或者说是地址）

然后跳转.c 文件，得到奇偶页地址，存在 sp 里面，然后把他们加载进 a 参数里面，再传给 ENTRYLO0, 1 寄存器，通过 tlbwr 写进 tlb 里面

这里面用到了 c 函数 \_do_tlb_refill

### \_do_tlb_refill

```c
void _do_tlb_refill(u_long *pentrylo, u_int va, u_int asid) {
	tlb_invalidate(asid, va);
	Pte *ppte;
	/* Hints:
	 *  Invoke 'page_lookup' repeatedly in a loop to find the page table entry '*ppte'
	 * associated with the virtual address 'va' in the current address space 'cur_pgdir'.
	 *
	 *  **While** 'page_lookup' returns 'NULL', indicating that the '*ppte' could not be found,
	 *  allocate a new page using 'passive_alloc' until 'page_lookup' succeeds.
	 */

	/* Exercise 2.9: Your code here. */
	while (page_lookup(cur_pgdir, va, &ppte) == NULL)
	{
		passive_alloc(va, cur_pgdir, asid);
	}
	ppte = (Pte *)((u_long)ppte & ~0x7);
	pentrylo[0] = ppte[0] >> 6;
	pentrylo[1] = ppte[1] >> 6;
}
```

先给传进来的虚拟地址对应的表项进行一个 invalidate

后面的过程说的很详细，循环查，如果找不到页面，申请一个，把查到（或者申请到）的结果存在 pentrylo 里面，ppte 以(Pte\*)的大小挪动，取奇偶页，pentrylo 对应的是存返回值的 sp+12 地址

里面用到了一个新的 tlb_invalidate 函数，page_lookup 函数和 passive_alloc 函数，后面依次写

### tlb_invalidate

```c
// mmu.h
#define NASID 256
#define PGSHIFT 12

void tlb_invalidate(u_int asid, u_long va) {
	tlb_out((va & ~GENMASK(PGSHIFT, 0)) | (asid & (NASID - 1)));
}
```

具体的 tlb_out 在.S 文件里面解释了，这里传的参数和后面的`mtc0    a0, CP0_ENTRYHI` 这一条指令有关，可以看出来是直接把传过去的参数写进 ENTRYHI 里面了，ENTRYHI 的结构图如下：

![屏幕截图 2024-03-31 010612](./img/屏幕截图%202024-03-31%20010612.png)

所以来看是怎么传成这个格式的。

首先，GENMASK 是一个生成掩码的，GENMASK(high, low)是可以把 high 到 low 的位都置 1，放到代码里面看，PGSHIFT 是页内偏移 12 位，所以 GENMASK 把 0-12 位都置 1，反一下就是 0-12 位都是 0，去除了虚拟地址 va 的 0-12 位，和图里面的是对应上的。

之后是 ASID 的置位，NASID-1 是 255，也就是 0-7 位都是 1，asid 只保留 0-7 位的数，其它的都是 0

所以它们合起来就是符合 ETRYHI 要求的

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

实现的目的：查找虚拟地址 va 对应的 page（物理的）

1. 通过 pgdir_walk(后面说)找 va 对应的内核物理地址，放到 pte 里面，如果没有或者对应的位上面不有效，返回 NULL

2. 拿到的是内核物理地址，所以是有 12 位的标志位，这里使用的 pa2page 是这样的：

   ```c
   static inline struct Page *pa2page(u_long pa) {
   	if (PPN(pa) >= npage) {
   		panic("pa2page called with invalid pa: %x", pa);
   	}
   	return &pages[PPN(pa)];
   }
   ```

   可以看出来，传入的是一个内核物理地址，用 PPN 提取物理页号，然后从 pages 数组里面取这个序号的页面

   这里传入了一个参数 ppte，如果有的话把内核虚拟地址赋给它

   返回指向找到了的物理页的指针 pp

里面用到了 pgdir_walk 函数

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

实现的功能：将一级页表基地址 pgdir 对应的两级页表结构中 va 虚拟地址所在的二级页表项的指针存储在 ppte 指向的空间上，returns a pointer to the page table entry for virtual address 'va'，返回的是**指向 va 所在 page 虚拟地址入口的指针(也就是内核物理地址)**

注：Pde 表示一级页表项类型，Pte 表示二级页表项类型

传进来的是一级页目录的基地址，要查的虚拟地址项，如果为空是否创建页面，一个指向二级页表项指针的指针（等待赋值）

1. 找到一级页目录项，通过 PDX 去到对应页目录项的地址位，加到基地址上

2. 判断：如果没有取到页目录项或者非 valid，如果 create，就 page_alloc 一个页面，标注引用，赋给页目录项，具体是把申请来的物理地址或上一些标志位

   如果不 create，就赋值 NULL，返回

3. 已经找到（或者创建）页目录项，32 位的页表项由 20 位物理页号和 12 位标志位组成，PTE_ADDR 是抹掉低 12 位，只留下物理页号，因为要实现的是虚拟地址入口，所以这里 KADDR 把二级页表的基地址转化成了虚拟地址（具体为什么在这一步就转了而不是查出来之后再转在下面解释），然后再这个的基础上加上二级页表偏移量，就能拿到最后要的物理页号

解释一下**为什么在二级页表基地址的时候就用了 KADDR:**

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

这里面又用到了一个 page_alloc 申请页面

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
2. 利用 memset，把对应的内存地址都清零

### passive_alloc

```c
static void passive_alloc(u_int va, Pde *pgdir, u_int asid) {
	struct Page *p = NULL;

	if (va < UTEMP) {
		panic("address too low");
	}

	if (va >= USTACKTOP && va < USTACKTOP + PAGE_SIZE) {
		panic("invalid memory");
	}

	if (va >= UENVS && va < UPAGES) {
		panic("envs zone");
	}

	if (va >= UPAGES && va < UVPT) {
		panic("pages zone");
	}

	if (va >= ULIM) {
		panic("kernel address");
	}

	panic_on(page_alloc(&p));
	panic_on(page_insert(pgdir, asid, p, PTE_ADDR(va), (va >= UVPT && va < ULIM) ? 0 : PTE_D));
}
```

这里面前面都是验证，重要的是最后面两个，一个 page_alloc 申请页面，在前面写了，后面 page_insert 是映射物理页面 p 到指定位置 PTE_ADDR(va)

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

要实现的目标：将物理页面“pp”映射到虚拟地址“va”。如果在“va”处已经映射了一个页面，请调用 page_remove（）来释放此映射。

1. 通过 pgdir_walk 拿到虚拟地址 va 对应的物理页号
   - 如果有对应的物理页号，且映射的不是函数传进来的这个页号，那就取消 va 原来的映射
   - 如果就是原来的页号，说明这个函数要实现的目的其实是已经存在的，那就 flush TLB（这个现在还不知道是干什么的，等 TLB 再解释吧），然后对具体的映射物理地址进行一些改变，page2pp 是把物理页号左移，留出来右边标志位的位置，然后置上标志位，结束
2. Flush TLB
3. 找 va 对应的物理页号，如果没有可以申请，申请不对返回报错
4. 和 1.2 一样，给映射的物理地址置标志位

这里又有一个新的函数，page_remove(pgdir, asid, va)

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

把本身映射过去的物理块映射数量 ref- -(page_decref 干的)，pte 拿到的是内核物理地址，置 0，然后 flush TLB
