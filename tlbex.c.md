# tlbex.c

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

![Untitled](./img_lab2/屏幕截图%202024-03-31%20010612.png)

所以来看是怎么传成这个格式的。

首先，GENMASK 是一个生成掩码的，GENMASK(high, low)是可以把 high 到 low 的位都置 1，放到代码里面看，PGSHIFT 是页内偏移 12 位，所以 GENMASK 把 0-12 位都置 1，反一下就是 0-12 位都是 0，去除了虚拟地址 va 的 0-12 位，和图里面的是对应上的。

之后是 ASID 的置位，NASID-1 是 255，也就是 0-7 位都是 1，asid 只保留 0-7 位的数，其它的都是 0

所以它们合起来就是符合 ETRYHI 要求的

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
