# tools/readelf/elf.h

```c
/* Type for a 16-bit quantity.  */
typedef uint16_t Elf32_Half;

/* Types for signed and unsigned 32-bit quantities.  */
typedef uint32_t Elf32_Word;
typedef int32_t Elf32_Sword;

/* Types for signed and unsigned 64-bit quantities.  */
typedef uint64_t Elf32_Xword;
typedef int64_t Elf32_Sxword;

/* Type of addresses.  */
typedef uint32_t Elf32_Addr;

/* Type of file offsets.  */
typedef uint32_t Elf32_Off;

/* Type for section indices, which are 16-bit quantities.  */
typedef uint16_t Elf32_Section;

/* Type of symbol indices.  */
typedef uint32_t Elf32_Symndx;
```

这里定义了一堆类型，比如说32位的有无符号，64位的有无符号，32位存地址和offset，16位的节

```c
typedef struct
{
	unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
	Elf32_Half e_type;								/* 类型 */
	Elf32_Half e_machine;							/* 目标文件的结构 */
	Elf32_Word e_version;							/* 目标文件版本 */
	Elf32_Addr e_entry;								/* 程序入口点虚拟地址 */
	Elf32_Off e_phoff;								/* 程序头表的offset */
	Elf32_Off e_shoff;								/* 节头表的offset */
	Elf32_Word e_flags;								/* Processor-specific flags */
	Elf32_Half e_ehsize;							/* ELF头大小 */
	Elf32_Half e_phentsize;						/* 程序头表每个条目大小 */
	Elf32_Half e_phnum;								/* 程序头表条目数量 */
	Elf32_Half e_shentsize;						/* 同上，节头表的 */
	Elf32_Half e_shnum;								/* Section header table entry count */
	Elf32_Half e_shstrndx;						/* Section header string table index */
} Elf32_Ehdr;
```

后面定义了魔数

```c
#define EI_MAG0 0		 /* File identification byte 0 index */
#define ELFMAG0 0x7f /* Magic number byte 0 */

#define EI_MAG1 1		/* File identification byte 1 index */
#define ELFMAG1 'E' /* Magic number byte 1 */

#define EI_MAG2 2		/* File identification byte 2 index */
#define ELFMAG2 'L' /* Magic number byte 2 */

#define EI_MAG3 3		/* File identification byte 3 index */
#define ELFMAG3 'F' /* Magic number byte 3 */
```

节头表定义：

```c
typedef struct
{
	Elf32_Word sh_name;			 /* Section name */
	Elf32_Word sh_type;			 /* Section type */
	Elf32_Word sh_flags;		 /* Section flags */
	Elf32_Addr sh_addr;			 /* Section addr */
	Elf32_Off sh_offset;		 /* Section offset */
	Elf32_Word sh_size;			 /* Section size */
	Elf32_Word sh_link;			 /* Section link */
	Elf32_Word sh_info;			 /* Section extra info */
	Elf32_Word sh_addralign; /* Section alignment */
	Elf32_Word sh_entsize;	 /* Section entry size */
} Elf32_Shdr;
```

看名字就能看出来是干什么用的

段头表：

```c
typedef struct
{
	Elf32_Word p_type;	 /* Segment type ，类型*/
	Elf32_Off p_offset;	 /* Segment file offset ，相对文件偏移量*/
	Elf32_Addr p_vaddr;	 /* Segment virtual address ，本段内容的开始位置在进程空间中的虚拟地址*/
	Elf32_Addr p_paddr;	 /* Segment physical address ，本段开始内容在进程空间的物理地址，大多数不用*/
	Elf32_Word p_filesz; /* Segment size in file 本段内容在文件中的大小*/
	Elf32_Word p_memsz;	 /* Segment size in memory 本段内容在内容镜像里面的大小*/
	Elf32_Word p_flags;	 /* Segment flags ，段权限，对应下面的Legal values for p_flags (segment flags)*/
	Elf32_Word p_align;	 /* Segment alignment，如何对齐，0/1没有对其要求，否则是2的幂次数*/
} Elf32_Phdr;
```

然后定义了一堆用于段头表的常量：

```c
#define PT_NULL 0						 /* 程序头表条目未使用 */
#define PT_LOAD 1						 /* 可加载的程序 */
#define PT_DYNAMIC 2				 /* 动态链接信息 */
#define PT_INTERP 3					 /* 程序解释器 */
#define PT_NOTE 4						 /* 辅助信息 */
#define PT_SHLIB 5					 /* 保留类型 */
#define PT_PHDR 6						 /* 指向程序头表本身的条目 */
#define PT_NUM 7						 /* 定义的类型数量  */
#define PT_LOOS 0x60000000	 /* OS特定类型起始值 */
#define PT_HIOS 0x6fffffff	 /* OS特定类型结束值 */
#define PT_LOPROC 0x70000000 /*处理器特定类型起始值 */
#define PT_HIPROC 0x7fffffff /* 处理器特定类型结束值 */
```

还有p_flags的取值：

```c
#define PF_X (1 << 0)					 /* Segment is executable */
#define PF_W (1 << 1)					 /* Segment is writable */
#define PF_R (1 << 2)					 /* Segment is readable */
#define PF_MASKPROC 0xf0000000 /* Processor-specific */
```