# init/start.S

```verilog
.text
EXPORT(_start)
```

导出程序入口点

> 本段代码中的 EXPORT 是一个宏，它将 _start 函数导出为一个符号，使得链接器可以找到它。可以简单的理解为，它实现了一种在汇编语言中的函数定义。

```verilog
.set at
.set reorder
```

设置汇编器的一些属性，at是记录汇编的当前位置，reorder是开启指令重排，提高效率

```verilog
/* Lab 1 Key Code "enter-kernel" */
	/* clear .bss segment */
	la      v0, bss_start
	la      v1, bss_end
/*这里是给bss段清零*/	
clear_bss_loop:
	beq     v0, v1, clear_bss_done
	sb      zero, 0(v0)
	addiu   v0, v0, 1
	j       clear_bss_loop
/* End of Key Code "enter-kernel" */

clear_bss_done:
	/* disable interrupts */
	mtc0    zero, CP0_STATUS

	/* hint: you can refer to the memory layout in include/mmu.h */
	/* set up the kernel stack */
	/* Exercise 1.3: Your code here. (1/2) */
	la	sp, KSTACKTOP
	/* jump to mips_init */
	/* Exercise 1.3: Your code here. (2/2) */
	j	mips_init
```

清零bss段、禁用中断、设置内核栈指针，并跳转到内核初始化函数mips_init，这是内核启动时的一些关键操作