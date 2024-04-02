# tools/readelf/readelf.c

```c
// 看是不是个ELF文件，这些在elf.h里面有宏定义
int is_elf_format(const void *binary, size_t size)
{
	Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
	return size >= sizeof(Elf32_Ehdr) && ehdr->e_ident[EI_MAG0] == ELFMAG0 &&
				 ehdr->e_ident[EI_MAG1] == ELFMAG1 && ehdr->e_ident[EI_MAG2] == ELFMAG2 &&
				 ehdr->e_ident[EI_MAG3] == ELFMAG3;
}
```

这个和之前的`const Elf32_Ehdr *elf_from(const void *binary, size_t size)` 作用是一样的，这个返回的是判断结果，那个返回的是符合格式的Ehdr指针

```c
int readelf(const void *binary, size_t size)
{
	Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;

	// Check whether `binary` is a ELF file.
	if (!is_elf_format(binary, size))
	{
		fputs("not an elf file\\n", stderr);
		return -1;
	}
	// 节头
	// Get the address of the section table, the number of section headers and the size of a
	// section header.
	const void *sh_table;
	Elf32_Half sh_entry_count;
	Elf32_Half sh_entry_size;
	/* Exercise 1.1: Your code here. (1/2) */
	sh_table = binary + ehdr->e_shoff;
	sh_entry_count = ehdr->e_shnum;
	sh_entry_size = ehdr->e_shentsize;
	// For each section header, output its index and the section address.
	// The index should start from 0.
	// 打印节头和编号
	for (int i = 0; i < sh_entry_count; i++)
	{
		const Elf32_Shdr *shdr;
		unsigned int addr;
		/* Exercise 1.1: Your code here. (2/2) */
		shdr = sh_table + i * sh_entry_size;
		addr = (unsigned int)shdr->sh_addr;
		printf("%d:0x%x\\n", i, addr);
	}
	return 0;
}
```

这个是根据elf相关的结构体，读里面存的信息，包括节头大小编号等等