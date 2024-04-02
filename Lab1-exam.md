# Lab1-exam

题目：新加一个case，%P，实现对两个数x, y，计算z=|(x+y)*(x-y)|，输出(x,y,z)

```c
// 在函数开头
int num1,num2;
case 'P':
			if(long_flag){
				num1=va_arg(ap,long int);
				num2=va_arg(ap,long int);
			}
			else{
				num1=va_arg(ap,int);
				num2=va_arg(ap,int);
			}
			long int tmp1=num1+num2;
			long int tmp2=num1-num2;
			long int tmp=tmp1*tmp2;
			if(tmp<0)
				tmp=-tmp;
			out(data,"(",1);
			if(num1<0){
				neg_flag=1;
				num1=-num1;
			}
			print_num(out,data,num1,10,neg_flag,width,ladjust,padc,0);
			out(data,",",1);
			neg_flag=0;
			if(num2<0){
                                  neg_flag=1;
                                  num2=-num2;
                        }
                        print_num(out,data,num2,10,neg_flag,width,ladjust,padc,0);
			out(data,",",1);
			neg_flag=0;
			print_num(out,data,tmp,10,neg_flag,width,ladjust,padc,0);
			out(data,")",1);
			break;
```