# lib/elfloader.c

```c
// 判断给定的二进制数据binary是否符合Elfheader的格式
const Elf32_Ehdr *elf_from(const void *binary, size_t size)
{
	const Elf32_Ehdr *ehdr = (const Elf32_Ehdr *)binary;
	// 判size和魔数
	if (size >= sizeof(Elf32_Ehdr) && ehdr->e_ident[EI_MAG0] == ELFMAG0 &&
			ehdr->e_ident[EI_MAG1] == ELFMAG1 && ehdr->e_ident[EI_MAG2] == ELFMAG2 &&
			ehdr->e_ident[EI_MAG3] == ELFMAG3 && ehdr->e_type == 2)
	{
		return ehdr;
	}
	return NULL;
}
```

是一个判定函数，判断是不是一个Elfheader

```c
// 把elf文件中的一个段加载进内存里面，包括对跨页的处理
int elf_load_seg(Elf32_Phdr *ph, const void *bin, elf_mapper_t map_page, void *data)
{
	u_long va = ph->p_vaddr;				// va存的是在进程空间的虚拟地址
	size_t bin_size = ph->p_filesz; // 在文件中大小
	size_t sgsize = ph->p_memsz;		// 内存镜像里面的大小
	u_int perm = PTE_V;							// 设置valid位，具体在mmu.h里面，为了TLB
	if (ph->p_flags & PF_W)					// 如果可写设置脏位标记是否已经被写了，也在mmu.h里面
	{
		perm |= PTE_D;
	}

	int r;
	size_t i;
	u_long offset = va - ROUNDDOWN(va, PAGE_SIZE); // 页内偏移
	if (offset != 0)															 // 处理不是页面对齐的映射
	{
		if ((r = map_page(data, va, offset, perm, bin,
											MIN(bin_size, PAGE_SIZE - offset))) != 0)
		{
			return r;
		}
	}

	/* Step 1: load all content of bin into memory. 逐页映射将整个二进制数据bin加载到内存中，同时处理了因起始地址可能不是页对齐而导致的特殊情况*/
	for (i = offset ? MIN(bin_size, PAGE_SIZE - offset) : 0; i < bin_size; i += PAGE_SIZE)
	{
		if ((r = map_page(data, va + i, 0, perm, bin + i, MIN(bin_size - i, PAGE_SIZE))) !=
				0)
		{
			return r;
		}
	}

	/* Step 2: alloc pages to reach `sgsize` when `bin_size` < `sgsize`. <=的时候继续申请空的内存*/
	while (i < sgsize)
	{
		if ((r = map_page(data, va + i, 0, perm, NULL, MIN(bin_size - i, PAGE_SIZE))) !=
				0)
		{
			return r;
		}
		i += PAGE_SIZE;
	}
	return 0;
}
```