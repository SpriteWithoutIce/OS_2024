# tlb_asm.S

首先解释几个函数

- tlbr(tlb read)：以index寄存器值为索引，读出tlb对应表项到EntryHi，EntryLo0，EntryLo1
- tlbwi(tlb write index)：以index寄存器值为索引，把三个寄存器的值写进去
- tlbwr(tlb write random)：把三个寄存器的值随机写道一个表项里面
- tlbp(不知道是怎么翻译的)：根据EntryHi中的Key(VPN & ASID)，查找，把索引存进index寄存器

### tlb_out 注释解释

```nasm
LEAF(tlb_out)
.set noreorder /* 规定下面指令必须按顺序执行 */
	mfc0    t0, CP0_ENTRYHI /*把ENTRYHI原来存着的值存起来，因为一会要用这个寄存器置0，所以还需要恢复现场*/
	mtc0    a0, CP0_ENTRYHI /*a0传入的是ENTRYHI规定的格式，可以看.c文件里面传的东西*/
	nop /*保证mtc0执行，因为是给寄存器赋值*/
	/* Step 1: Use 'tlbp' to probe TLB entry */
	/* Exercise 2.8: Your code here. (1/2) */
	tlbp /*查，根据ENTRYHI里面的东西拼成KEY，查结果存在index里面*/
	nop
	/* Step 2: Fetch the probe result from CP0.Index */
	mfc0    t1, CP0_INDEX /*取出来index寄存器的值*/
.set reorder
	bltz    t1, NO_SUCH_ENTRY /*如果是负的（index最高位置1）就是没查到，跳转*/
.set noreorder
	mtc0    zero, CP0_ENTRYHI /*开始置0*/
	mtc0    zero, CP0_ENTRYLO0
	mtc0    zero, CP0_ENTRYLO1
	nop
	/* Step 3: Use 'tlbwi' to write CP0.EntryHi/Lo into TLB at CP0.Index  */
	/* Exercise 2.8: Your code here. (2/2) */
	tlbwi /*把0写回表项*/
.set reorder

NO_SUCH_ENTRY:
	mtc0    t0, CP0_ENTRYHI /*恢复现场，把ENTRYHI的值还回去*/
	j       ra
END(tlb_out)
```

### do_tlb_refill tlb重填

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

前面几行也挺重要，因为写到后面发现找不到va的传参了）前面是先设定好_do_tlb_refill的传参，从BADVADDR里面读出来引发TLB miss的虚拟地址，存在a1里面，a2存的是ASID（先存再与取低8位）

这段代码整体上是利用堆栈sp先存起来跳转地址，这里的addi是为传参做准备，因为调用的c语言函数默认a0对应的是传入的第一个参数也就是Pte*类型的那个，这个一会是要存结果的，所以先给它分配好地方，也就是sp+12的这个位置（或者说是地址）

然后跳转.c文件，得到奇偶页地址，存在sp里面，然后把他们加载进a参数里面，再传给ENTRYLO0, 1寄存器，通过tlbwr写进tlb里面