# kern/printk.c

```c
void outputk(void *data, const char *buf, size_t len)
{
	for (int i = 0; i < len; i++)
	{
		printcharc(buf[i]);
	}
}
```

这个data暂时不知道是干什么的，这个函数是投入一个字符串起始指针，打印长度为len的字符串

补：data是传参的功能，printk调用vprintfmt，处理并输出格式串，如果最后的结果需要改动printk里面的东西，比如说实现sprintf，把结果存储在str里面，这个时候就可以通过data把str传给vprintfmt，然后在里面对data（指针）做出修改，也方便里面输出的时候与outputk的传参。

```c
// print主函数
void printk(const char *fmt, ...)
{
	va_list ap;												 // 定义变长参数列表
	va_start(ap, fmt);								 // 初始化为fmt
	vprintfmt(outputk, NULL, fmt, ap); // 格式化输出字符串处理（后面printk的主要部分）
	va_end(ap);
}
```

上面的三个点是表示变长参数，va_list定义一个变长参数列表，va_start是初始化这个变长参数列表，里面的最后一个固定参数是fmt，指向fmt。vprintfmt是主函数，进行格式化处理输出函数。

```c
void print_tf(struct Trapframe *tf)
{
	for (int i = 0; i < sizeof(tf->regs) / sizeof(tf->regs[0]); i++)
	{
		printk("$%2d = %08x\\n", i, tf->regs[i]);
	}
	printk("HI  = %08x\\n", tf->hi);
	printk("LO  = %08x\\n\\n", tf->lo);
	printk("CP0.SR    = %08x\\n", tf->cp0_status);
	printk("CP0.BadV  = %08x\\n", tf->cp0_badvaddr);
	printk("CP0.Cause = %08x\\n", tf->cp0_cause);
	printk("CP0.EPC   = %08x\\n", tf->cp0_epc);
}
```

这里是陷入trap之后打印寄存器里面的信息