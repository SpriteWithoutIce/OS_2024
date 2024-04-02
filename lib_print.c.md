# lib/print.c

```c
void vprintfmt(fmt_callback_t out, void *data, const char *fmt, va_list ap)
{
	char c;
	const char *s;
	long num;

	int width = 0;
	int long_flag;	 // output is long (rather than int)
	int neg_flag;		 // output is negative，负数
	int ladjust = 0; // output is left-aligned,左对齐
	char padc = ' '; // padding char，占位符

	for (;;)
	{
		/* scan for the next '%' */
		/* Exercise 1.4: Your code here. (1/8) */
		const char *p = fmt;
		while (*p != '%' && *p != '\\0')
			p++;
		/* Exercise 1.4: Your code here. (2/8) */
		out(data, fmt, p - fmt);
		/* check "are we hitting the end?" */
		/* Exercise 1.4: Your code here. (3/8) */
		if (*p == '\\0')
			break;
		/* we found a '%' */
		/* Exercise 1.4: Your code here. (4/8) */
		p++;
		/* check format flag */
		/* Exercise 1.4: Your code here. (5/8) */
		if (*p == '-')
		{
			ladjust = 1;
			p++;
		}
		else if (*p == '0')
		{
			padc = '0';
			p++;
		}
		/* get width */
		/* Exercise 1.4: Your code here. (6/8) */
		while (*p >= '0' && *p <= '9' && *p != '\\0')
		{
			width *= 10;
			width += (*p - '0');
			p++;
		}
		/* check for long */
		/* Exercise 1.4: Your code here. (7/8) */
		if (*p == 'l')
		{
			p++;
			long_flag = 1;
		}
		else
			long_flag = 0;

		fmt = p;
		neg_flag = 0;

		// 前面是对格式串的处理，是否是longlong，是否是负数，是否需要左对齐，是否需要占位字符
		switch (*fmt)
		{
		case 'b': // 二进制无符号
			if (long_flag)
			{
				num = va_arg(ap, long int);
			}
			else
			{
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 2, 0, width, ladjust, padc, 0);
			break;

		case 'd':
		case 'D': // 十进制有符号
			if (long_flag)
			{
				num = va_arg(ap, long int);
			}
			else
			{
				num = va_arg(ap, int);
			}

			/*
			 * Refer to other parts (case 'b', case 'o', etc.) and func 'print_num' to
			 * complete this part. Think the differences between case 'd' and the
			 * others. (hint: 'neg_flag').
			 */
			/* Exercise 1.4: Your code here. (8/8) */
			if (num < 0)
				neg_flag = 1;
			if (neg_flag)
				print_num(out, data, -num, 10, 1, width, ladjust, padc, 0);
			else
				print_num(out, data, num, 10, 0, width, ladjust, padc, 0);
			break;

		case 'o':
		case 'O': // 八进制无符号
			if (long_flag)
			{
				num = va_arg(ap, long int); // 从可变长的ap list里面读出来一个long int的参数
			}
			else
			{
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 8, 0, width, ladjust, padc, 0);
			break;

		case 'u':
		case 'U': // 十进制无符号
			if (long_flag)
			{
				num = va_arg(ap, long int);
			}
			else
			{
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 10, 0, width, ladjust, padc, 0);
			break;

		case 'x': // 十六进制，小写abcde
			if (long_flag)
			{
				num = va_arg(ap, long int);
			}
			else
			{
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 16, 0, width, ladjust, padc, 0);
			break;

		case 'X': // 十六进制，大写ABCDE
			if (long_flag)
			{
				num = va_arg(ap, long int);
			}
			else
			{
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 16, 0, width, ladjust, padc, 1);
			break;

		case 'c': // 打印字符
			c = (char)va_arg(ap, int);
			print_char(out, data, c, width, ladjust);
			break;

		case 's': // 打印字符串
			s = (char *)va_arg(ap, char *);
			print_str(out, data, s, width, ladjust);
			break;

		case '\\0':
			fmt--;
			break;

		default:
			/* output this char as it is */
			out(data, fmt, 1);
		}
		fmt++;
	}
}
```

格式化解析字符串，并且利用一些print_num,print_str等等，把解析完了的打印出来

```c
void print_char(fmt_callback_t out, void *data, char c, int length, int ladjust)
{
	int i;

	if (length < 1)
	{
		length = 1;
	}
	const char space = ' ';
	if (ladjust)// 需要左对齐的
	{
		out(data, &c, 1);
		for (i = 1; i < length; i++)
		{
			out(data, &space, 1);
		}
	}
	else// 右对齐
	{
		for (i = 0; i < length - 1; i++)
		{
			out(data, &space, 1);
		}
		out(data, &c, 1);
	}
}
void print_str(fmt_callback_t out, void *data, const char *s, int length, int ladjust)
{
	int i;
	int len = 0;
	const char *s1 = s;
	while (*s1++)
	{
		len++;
	}
	if (length < len) // 防止length给错了，再算一遍字符串多长
	{
		length = len;
	}

	if (ladjust) // 左对齐
	{
		out(data, s, len);
		for (i = len; i < length; i++)
		{
			out(data, " ", 1);
		}
	}
	else
	{
		for (i = 0; i < length - len; i++)
		{
			out(data, " ", 1);
		}
		out(data, s, len);
	}
}
void print_num(fmt_callback_t out, void *data, unsigned long u, int base, int neg_flag, int length,
							 int ladjust, char padc, int upcase)
{
	/* algorithm :包括一些要求
	 *  1. prints the number from left to right in reverse form.
	 *  2. fill the remaining spaces with padc if length is longer than
	 *     the actual length
	 *     TRICKY : if left adjusted, no "0" padding.
	 *		    if negtive, insert  "0" padding between "0" and number.
	 *  3. if (!ladjust) we reverse the whole string including paddings
	 *  4. otherwise we only reverse the actual string representing the num.
	 */
	int actualLength = 0;
	char buf[length + 70];
	char *p = buf;
	int i;
	// buf先存reverse的
	do
	{
		int tmp = u % base;
		if (tmp <= 9)
		{
			*p++ = '0' + tmp;
		}
		else if (upcase)
		{
			*p++ = 'A' + tmp - 10;
		}
		else
		{
			*p++ = 'a' + tmp - 10;
		}
		u /= base;
	} while (u != 0);

	if (neg_flag)
	{
		*p++ = '-';
	}
	// 存好反着的字符串了

	/* figure out actual length and adjust the maximum length */
	actualLength = p - buf;
	if (length < actualLength)
	{
		length = actualLength;
	}

	/* add padding */
	if (ladjust) // 如果左对齐，空的地方用空填
	{
		padc = ' ';
	}
	if (neg_flag && !ladjust && (padc == '0')) // 如果是负的，右对齐，空白用0填，那就把负号放在最右边然后中间填0
	{
		for (i = actualLength - 1; i < length - 1; i++)
		{
			buf[i] = padc;
		}
		buf[length - 1] = '-';
	}
	else
	{
		for (i = actualLength; i < length; i++)
		{
			buf[i] = padc;
		}
	}

	/* prepare to reverse the string */
	int begin = 0;
	int end;
	if (ladjust)
	{
		end = actualLength - 1;
	}
	else
	{
		end = length - 1;
	}

	/* adjust the string pointer */
	while (end > begin)
	{
		char tmp = buf[begin];
		buf[begin] = buf[end];
		buf[end] = tmp;
		begin++;
		end--;
	}

	out(data, buf, length);
}
```