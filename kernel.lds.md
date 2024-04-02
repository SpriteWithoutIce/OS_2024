# kernel.lds

```verilog
/*
 * Set the architecture to mips.
 */
OUTPUT_ARCH(mips)
```

指定mips架构

```verilog
/*
 * Set the ENTRY point of the program to _start.
 */
ENTRY(_start)
```

定义入口start

```verilog
SECTIONS {
	/* Exercise 3.10: Your code here. */

	/* fill in the correct address of the key sections: text, data, bss. */
	/* Hint: The loading address can be found in the memory layout. And the data section
	 *       and bss section are right after the text section, so you can just define
	 *       them after the text section.
	 */
	/* Step 1: Set the loading address of the text section to the location counter ".". */
	/* Exercise 1.2: Your code here. (1/4) */
	. = 0x80020000;
	.text : { *(.text) }
	.data : { *(.data) }
	.bss : { *(.bss) }
	/* Step 2: Define the text section. */

	/* Step 3: Define the data section. */
	bss_start = .;
	/* Step 4: Define the bss section. */
	bss_end = .;
	. = 0x80400000;
	end = . ;
}
```

这里是根据mmu.h里面找到对应的地址