# kern/machine.c

```c
void printcharc(char ch)
{
	if (ch == '\\n')
	{
		printcharc('\\r');
	}
	while (!(*((volatile uint8_t *)(KSEG1 + MALTA_SERIAL_LSR)) & MALTA_SERIAL_THR_EMPTY))
	{
	}
	*((volatile uint8_t *)(KSEG1 + MALTA_SERIAL_DATA)) = ch;
}
```

解释是：Send a character to the console. If Transmitter Holding Register is currently occupied by some data, wait until the register becomes available.

这里是在 MIPS 架构的 Malta 开发板上通过串行端口发送字符，先处理换行符，然后循环检查串行端口的线路状态寄存器（LSR），等待发送保持寄存器为空（THR_EMPTY），表示串行端口准备好接收新的字符发送，利用KSEG1访问I/O端口。直接操作硬件寄存器实现字符的串行输入。

```c
int scancharc(void)
{
	if (*((volatile uint8_t *)(KSEG1 + MALTA_SERIAL_LSR)) & MALTA_SERIAL_DATA_READY)
	{
		return *((volatile uint8_t *)(KSEG1 + MALTA_SERIAL_DATA));
	}
	return 0;
}
```

和上面一样，从寄存器里面读取值

```c
void halt(void)
{
	*(volatile uint8_t *)(KSEG1 + MALTA_FPGA_HALT) = 0x42;
	printk("machine.c:\\thalt is not supported in this machine!\\n");
	while (1)
	{
	}
}
```

重置/停止系统

Halt/Reset the whole system. Write the magic value GORESET(0x42) to SOFTRES register of theFPGA on the Malta board, initiating a board reset. In QEMU emulator, emulation will stopinstead of rebooting with the parameter '-no-reboot'