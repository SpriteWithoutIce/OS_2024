# Lab1-extra

题目：模仿printk实现scanf

- 在include/print.h里面加入（题目给的）

```c
typedef void (*scan_callback_t)(void *data, char *buf, size_t len);
int vscanfmt(scan_callback_t in, void *data, const char *fmt, va_list ap);
```

- 在include/printk.h里面加入（题目给的）

```c
int scanf(const char *fmt, ...);
```

- 在kern/printk.c里面加入

```c
/* 这个是题目给的 */
void inputk(void *data, char *buf, size_t len) {
	for (int i = 0; i < len; i++) {
		while ((buf[i] = scancharc()) == '\\0') {
		}
		if (buf[i] == '\\r') {
			buf[i] = '\\n';
		}
	}
}
/* 这里是要自己填的 */
int scanf(const char *fmt, ...) {
	// Lab 1-Extra: Your code here. (1/5)
		va_list ap;
    va_start(ap, fmt);
    vscanfmt(inputk, NULL, fmt, ap);
    va_end(ap);
}
```

- 主函数：在lib/print.c里面实现

scanf要实现四种类型的输入：%d, %c, %x, %s

%d：读入十进制的整数，其中不包含空白符\t, \n, 空格，要注意实现负号的读入和前导零

%c：读入一个字符

%x：读入一个十六进制的数，把它本来的大小（十进制的）存在对应的变量里面，字母都为小写，要实现负号的读入和前导零

%s：读入一个字符串，不包含空白符，在字符串结尾加上’\0’

举个例子：这是样例对应的test文件夹

```c
void scanf_1_check() {
	printk("Running scanf_1_check\\n");
	int num = 0;
	char ch = '0';
	scanf("%d%c", &num, &ch);
	printk("Finished 1st scanf\\n");
	printk("%c %d\\n", ch, num);
}
// 输入：400 +
// 输出：+ 400
void mips_init(u_int argc, char **argv, char **penv, u_int ram_low_size) {
	scanf_1_check();
	halt();
}
```

```c
// lib/print.c
int vscanfmt(scan_callback_t in, void *data, const char *fmt, va_list ap) {
	int *ip;
	char *cp;
	char ch;
	int base, num, neg, ret = 0;

	while (*fmt) {
		if (*fmt == '%') {
			ret++;
			fmt++; // 跳过 '%'
			do {
				in(data, &ch, 1);
			} while (ch == ' ' || ch == '\\t' || ch == '\\n'); // 跳过空白符
			// 注意，此时 ch 为第一个有效输入字符
			switch (*fmt) {
			case 'd': // 十进制
				// Lab 1-Extra: Your code here. (2/5)		
				ip=va_arg(ap,int*);
				num=0;
				neg=0;
				if(ch=='-'){
					neg=1;
					in(data,&ch,1);
				}
				do{
					num=num*10;
					num+=ch-'0';
					in(data,&ch,1);
				}while(ch>='0' && ch<='9');
				if(neg==1)
					num=-num;
				*ip=num;
				break;
			case 'x': // 十六进制
				// Lab 1-Extra: Your code here. (3/5)
				num=0;
				ip=va_arg(ap,int*);
				neg=0;
				if(ch=='-'){
					neg=1;
					in(data,&ch,1);
				}
				do{
					num=num*16;
					if(ch>='a' && ch<='f')
						num+=ch-'a'+10;
					else
						num+=ch-'0';
					in(data,&ch,1);
				}while((ch>='0' && ch<='9') || (ch>='a' && ch<='f'));
				if(neg==1)
					num=-num;
				*ip=num;
				break;
			case 'c':
				// Lab 1-Extra: Your code here. (4/5)
				cp=va_arg(ap,char*);
				*cp=ch;
				break;
			case 's':
				// Lab 1-Extra: Your code here. (5/5)
				cp=(char*)va_arg(ap,char*);
				do{
					*cp=ch;
					in(data,&ch,1);
					cp++;
				}while(ch!='\\t' && ch!='\\n' && ch!=' ');
				*cp='\\0';
				break;
			}
			fmt++;
		}
	}
	return ret;
}
```