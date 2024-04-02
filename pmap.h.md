# pmap.h

```c
LIST_HEAD(Page_list, Page);
typedef LIST_ENTRY(Page) Page_LIST_entry_t;
```

这里是定义了两个结构体，具体的定义在 queue.h 里面：

```c
#define LIST_HEAD(name, type)                  \\
	struct name                                  \\
	{                                            \\
		struct type *lh_first; /* first element */ \\
	}
#define LIST_ENTRY(type)                                          \\
	struct                                                          \\
	{                                                               \\
		struct type *le_next;	 /* next element */                     \\
		struct type **le_prev; /* address of previous next element */ \\
	}
```

定义了以 Page 为头的 Page_list 链表和 list 里面干连接关系的 Page_LIST_entry_t

```c
struct Page {
	Page_LIST_entry_t pp_link; /* free list link */
	u_short pp_ref;
};
```

定义了 Page 结构体，里面有两部分，一部分是连接，一部分是计数，计的是有多少个虚拟地址映射到了这个页面

```c
extern struct Page *pages;
extern struct Page_list page_free_list;
```

这里是从 pmap.c 里面 extern 的，pages 是指向 Page 的指针

```c
static inline u_long page2ppn(struct Page *pp) {
	return pp - pages;
}
```

page2ppn: 这里算的其实是 pp 在 pages 里面的索引号，即是第几个页面

```c
static inline u_long page2pa(struct Page *pp) {
	return page2ppn(pp) << PGSHIFT;
}
```

上一步算出来了是第几个页面，乘上页面的大小就是这个物理页的基地址

```c
static inline struct Page *pa2page(u_long pa) {
	if (PPN(pa) >= npage) {
		panic("pa2page called with invalid pa: %x", pa);
	}
	return &pages[PPN(pa)];
}
```

这里面的 PPN 定义如下：

```c
// include/mmu.h
#define PPN(pa) (((u_long)(pa)) >> PGSHIFT)
```

这个 pa2page 是通过物理内存地址返回对应的页面，具体是通过 PPN 找到页号（除以页面大小），然后判断，返回的是一个指向对应页面的指针。这里感觉就是拆解一个物理内存的地址，PPN+PPO，PPO 的位数和 PHSHIFT 的是一样的，这里取出来的是 PPN 也就是物理页号

```c
static inline u_long page2kva(struct Page *pp) {
	return KADDR(page2pa(pp));
}
```

这里是把物理页转成核内的虚拟地址，其中用到的 KADDR 如下：

```c
// include/mmu.h
// translates from physical address to kernel virtual address
#define KADDR(pa)                                                                                  \\
	({                                                                                         \\
		u_long _ppn = PPN(pa);                                                             \\
		if (_ppn >= npage) {                                                               \\
			panic("KADDR called with invalid pa %08lx", (u_long)pa);                   \\
		}                                                                                  \\
		(pa) + ULIM;                                                                       \\
	})
// 是kseg0的起始地址
#define ULIM 0x80000000
```

传进来的是一个物理地址，算的\_ppn 是为了算地址是否合法，如果合法的话地址+ULIM 就是内核对应的虚拟地址

```c
static inline u_long va2pa(Pde *pgdir, u_long va) {
	Pte *p;

	pgdir = &pgdir[PDX(va)];
	if (!(*pgdir & PTE_V)) {
		return ~0;
	}
	p = (Pte *)KADDR(PTE_ADDR(*pgdir));
	if (!(p[PTX(va)] & PTE_V)) {
		return ~0;
	}
	return PTE_ADDR(p[PTX(va)]);
}
// include/mmu.h
#define PTE_ADDR(pte) (((u_long)(pte)) & ~0xFFF)
```

这里是把虚拟地址转成物理地址，传入的是一个一级页目录和虚拟地址，虚拟地址分成页目录，页表和页内偏移，PDX 是找页目录的，PTX 是找页表的，一开始先找到页目录项，如果没找到或者没有效报错。这里的 p 是一个指向页表项的指针，PTE_ADDR(\*pgdir)是把低 12 位清零，低 12 位是页内偏移，剩下的是页表的地址。pgdir 是在页目录表里面找到需要的那一项，这是个指针指着这里，把后 12 位抹了就是物理页号了，也就是二级页表的物理基地址，然后再通过 KADDR 转成核内的地址，p 实质上变成了一个指着一个二级页表基地址的指针。然后就是从 p 开始找页表项，如果没找到报错，最后返回的是找到的这一项的物理页号。

![屏幕截图 2024-03-25 003849](./img_lab2//屏幕截图%202024-03-25%20003849.png)
