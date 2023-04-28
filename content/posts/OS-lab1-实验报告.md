---
title: "OS Lab1 实验报告"
date: 2023-04-28T13:27:06+08:00
draft: true
---

# OS-lab1-实验报告

## 一、思考题

1. Thinking 1.1

   题面：

   `请阅读附录中的编译链接详解，尝试分别使用实验环境中的原生 x86 工具
   链（gcc、ld、readelf、objdump 等）和 MIPS 交叉编译工具链（带有 mips-linux-gnu前缀），重复其中的编译和解析过程，观察相应的结果，并解释其中向 objdump 传入的参
   数的含义。 `

   解答：

   `objdump`中参数：

   ```
   --disassemble 
   -d 
   从objfile中反汇编那些特定指令机器码的section。 
   
   -D 
   --disassemble-all 
   与 -d 类似，但反汇编所有section. 
   
   -s 
   --full-contents 
   显示指定section的完整内容。默认所有的非空section都会被显示。 
   
   -S 
   --source 
   尽可能反汇编出源代码，尤其当编译的时候指定了-g这种调试参数时，效果比较明显。隐含了-d参数。 
   ```

   对于一个文件test.c：

   ```c
   int a = 3;
   char b;
   int main() {
     int c = 12;
     char* s = "hello,world";
     return 0;
   }
   ```

   先对其进行编译但不链接，并进行反汇编：

   ```
   mips-linux-gnu-gcc -c test.c
   mips-linux-gnu-objdump -DS test.o > test.txt
   ```

   test.txt内容如下：

   ```txt
   Disassembly of section .text:
   
   00000000 <main>:
      0:   3c1c0000        lui     gp,0x0
      4:   279c0000        addiu   gp,gp,0
      8:   0399e021        addu    gp,gp,t9
      c:   27bdffe8        addiu   sp,sp,-24
     10:   afbe0010        sw      s8,16(sp)
     14:   03a0f021        move    s8,sp
     18:   2402000c        li      v0,12
     1c:   afc2000c        sw      v0,12(s8)
     20:   8f820000        lw      v0,0(gp)
     24:   24420000        addiu   v0,v0,0
     28:   afc20008        sw      v0,8(s8)
     2c:   00001021        move    v0,zero
     30:   03c0e821        move    sp,s8
     34:   8fbe0010        lw      s8,16(sp)
     38:   27bd0018        addiu   sp,sp,24
     3c:   03e00008        jr      ra
     40:   00000000        nop
           ...
   ```

   然后进行链接，并反汇编test

   ```shell
   mips-linux-gnu-ld hello.o -o test
   mips-linux-gnu-objdump -DS test > test2.txt
   ```

   test2.txt的内容如下:

   ```txt
   004000b0 <main>:
     4000b0:       3c1c0fc0        lui     gp,0xfc0
     4000b4:       279c7f50        addiu   gp,gp,32592
     4000b8:       0399e021        addu    gp,gp,t9
     4000bc:       27bdffe8        addiu   sp,sp,-24
     4000c0:       afbe0010        sw      s8,16(sp)
     4000c4:       03a0f021        move    s8,sp
     4000c8:       2402000c        li      v0,12
     4000cc:       afc2000c        sw      v0,12(s8)
     4000d0:       8f828018        lw      v0,-32744(gp)
     4000d4:       24420100        addiu   v0,v0,256
     4000d8:       afc20008        sw      v0,8(s8)
     4000dc:       00001021        move    v0,zero
     4000e0:       03c0e821        move    sp,s8
     4000e4:       8fbe0010        lw      s8,16(sp)
     4000e8:       27bd0018        addiu   sp,sp,24
     4000ec:       03e00008        jr      ra
     4000f0:       00000000      
   ```

2. Thinking 1.2

   题面：

   `思考下述问题：
   • 尝试使用我们编写的 readelf 程序，解析之前在 target 目录下生成的内核 ELF 文
   件。
   • 也许你会发现我们编写的 readelf 程序是不能解析 readelf 文件本身的，而我们刚
   才介绍的系统工具 readelf 则可以解析，这是为什么呢？（提示：尝试使用 readelf
   -h，并阅读 tools/readelf 目录下的 Makefile，观察 readelf 与 hello 的不同）`

   解答：

   原因：内核文件是大端存储，而我们的readelf程序解析的文件是小端存储的文件。

3. Thinking 1.3

   题面：

   `在理论课上我们了解到，MIPS 体系结构上电时，启动入口地址为 0xBFC00000
   （其实启动入口地址是根据具体型号而定的，由硬件逻辑确定，也有可能不是这个地址，但
   一定是一个确定的地址），但实验操作系统的内核入口并没有放在上电启动地址，而是按照
   内存布局图放置。思考为什么这样放置内核还能保证内核入口被正确跳转到？
   （提示：思考实验中启动过程的两阶段分别由谁执行。） `

     解答：

   内核入口是`_start`函数在boot/start.S，`main`函数在init/main.c，内核启动先执行`_start`入口函数，然后从这个函数跳转到`main`函数，从start.S文件中`jal mips_init`命令可以看出。这样内核启动的入口地址就可以固定下来，只需要传递`main`函数的地址就可以实现不同位置的`main`函数的调用。

## 二、难点分析

本次实验遇到的难点在Exercise1.4中`vprintfmt`函数的设计：

​	1.fmt每次完成匹配后需要自增使其指向下一个待检查的字符。

​	2.对于flag的匹配注意`-`和`0`可以同时出现。

​		当flag为`-`时：在给定的宽度(width)上左对齐输出，默认为右对齐。
​		当flag为`0`时：当输出宽度和指定宽度不同的时候，在空白位置填充0。

​	3.每一次循环时，需要对所有标志位进行初始化。

​	4.`print_num`传入的数字默认为无符号数，对于%d和%D数据处理我们需要预先判断正负。

实现代码如下：

```shell
#include <print.h>

/* forward declaration */
static void print_char(fmt_callback_t, void *, char, int, int);
static void print_str(fmt_callback_t, void *, const char *, int, int);
static void print_num(fmt_callback_t, void *, unsigned long, int, int, int, int, char, int);

void vprintfmt(fmt_callback_t out, void *data, const char *fmt, va_list ap) {
	char c;
	const char *s;
	long num;

	int width;
	int long_flag; // output is long (rather than int)
	int neg_flag;  // output is negative
	int ladjust;   // output is left-aligned
	char padc;     // padding char

	for (;;) {
		/* scan for the next '%' */
		/* Exercise 1.4: Your code here. (1/8) */
		const char *curfmt = fmt;
		while(1) {
			if(*curfmt == '\0') break;
			if(*curfmt == '%') break;
			curfmt++;
		}
		out(data, fmt, curfmt-fmt);
		fmt = curfmt;
		/* flush the string found so far */
		/* Exercise 1.4: Your code here. (2/8) */
		if(*fmt == '\0') break;
		/* check "are we hitting the end?" */
		/* Exercise 1.4: Your code here. (3/8) */
		fmt++;
		long_flag = 0;
		neg_flag = 0;
		width = 0;
		ladjust = 0;
		padc = ' ';
		/* we found a '%' */
		/* Exercise 1.4: Your code here. (4/8) */
		/* check format flag */
		/* Exercise 1.4: Your code here. (5/8) */
		if(*fmt == '-') ladjust = 1, fmt++;
		if(*fmt == '0') padc = '0', fmt++;
		/* get width */
		/* Exercise 1.4: Your code here. (6/8) */
		while (*fmt<='9' && *fmt>='0') {
			width = width*10 + (*fmt-'0');
			fmt++;
		}
		/* check for long */
		/* Exercise 1.4: Your code here. (7/8) */
		if(*fmt == 'l') {
			long_flag = 1;
			fmt++;
		}
		neg_flag = 0;
		switch (*fmt) {
		case 'b':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 2, 0, width, ladjust, padc, 0);
			break;

		case 'd':
		case 'D':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}

			/*
			 * Refer to other parts (case 'b', case 'o', etc.) and func 'print_num' to
			 * complete this part. Think the differences between case 'd' and the
			 * others. (hint: 'neg_flag').
			 */
			/* Exercise 1.4: Your code here. (8/8) */
			if(num < 0) {
				neg_flag = 1;
				num = -num;
			}
			print_num(out, data, num, 10, neg_flag, width, ladjust, padc, 0);
			break;
		case 'o':
		case 'O':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 8, 0, width, ladjust, padc, 0);
			break;

		case 'u':
		case 'U':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 10, 0, width, ladjust, padc, 0);
			break;

		case 'x':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 16, 0, width, ladjust, padc, 0);
			break;

		case 'X':
			if (long_flag) {
				num = va_arg(ap, long int);
			} else {
				num = va_arg(ap, int);
			}
			print_num(out, data, num, 16, 0, width, ladjust, padc, 1);
			break;

		case 'c':
			c = (char)va_arg(ap, int);
			print_char(out, data, c, width, ladjust);
			break;

		case 's':
			s = (char *)va_arg(ap, char *);
			print_str(out, data, s, width, ladjust);
			break;

		case '\0':
			fmt--;
			break;

		default:
			/* output this char as it is */
			out(data, fmt, 1);
		}
		fmt++;
	}
}

/* --------------- local help functions --------------------- */
void print_char(fmt_callback_t out, void *data, char c, int length, int ladjust) {
	int i;

	if (length < 1) {
		length = 1;
	}
	const char space = ' ';
	if (ladjust) {
		out(data, &c, 1);
		for (i = 1; i < length; i++) {
			out(data, &space, 1);
		}
	} else {
		for (i = 0; i < length - 1; i++) {
			out(data, &space, 1);
		}
		out(data, &c, 1);
	}
}

void print_str(fmt_callback_t out, void *data, const char *s, int length, int ladjust) {
	int i;
	int len = 0;
	const char *s1 = s;
	while (*s1++) {
		len++;
	}
	if (length < len) {
		length = len;
	}

	if (ladjust) {
		out(data, s, len);
		for (i = len; i < length; i++) {
			out(data, " ", 1);
		}
	} else {
		for (i = 0; i < length - len; i++) {
			out(data, " ", 1);
		}
		out(data, s, len);
	}
}

void print_num(fmt_callback_t out, void *data, unsigned long u, int base, int neg_flag, int length,
	       int ladjust, char padc, int upcase) {
	/* algorithm :
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

	do {
		int tmp = u % base;
		if (tmp <= 9) {
			*p++ = '0' + tmp;
		} else if (upcase) {
			*p++ = 'A' + tmp - 10;
		} else {
			*p++ = 'a' + tmp - 10;
		}
		u /= base;
	} while (u != 0);

	if (neg_flag) {
		*p++ = '-';
	}

	/* figure out actual length and adjust the maximum length */
	actualLength = p - buf;
	if (length < actualLength) {
		length = actualLength;
	}

	/* add padding */
	if (ladjust) {
		padc = ' ';
	}
	if (neg_flag && !ladjust && (padc == '0')) {
		for (i = actualLength - 1; i < length - 1; i++) {
			buf[i] = padc;
		}
		buf[length - 1] = '-';
	} else {
		for (i = actualLength; i < length; i++) {
			buf[i] = padc;
		}
	}

	/* prepare to reverse the string */
	int begin = 0;
	int end;
	if (ladjust) {
		end = actualLength - 1;
	} else {
		end = length - 1;
	}

	/* adjust the string pointer */
	while (end > begin) {
		char tmp = buf[begin];
		buf[begin] = buf[end];
		buf[end] = tmp;
		begin++;
		end--;
	}

	out(data, buf, length);
}
```

## 三、体会与感想

本次lab1花费了很长时间，主要有以下问题：

​	1.在实现print.c时需要阅读三个文件的代码，切换起来有些不方便，导致浪费了较多时间；

​	2.文件的每一个代码段，都大量使用宏定义和一些全局变量，需要手动grep去查找，虽然说这样是为了代码的可移植性，容错性更高，但是在实际阅读时仍然对于很多宏理解不清楚甚至混淆；

​	3.给出的代码一般都预先定义好了变量，但由于不能理解变量名称（比如不知道带ptr的变量是指针），我在面对一块待补充的部分时有些茫然，在学习了一些变量命名规范后发现给出的变量名已经暗中提示了相关的操作，甚至提示了变量的数据类型。

往后的lab中，我一定要学习了解较多的预置知识，并且弄清楚其他的文件、宏定义、函数，了解各个文件、函数之间的层次关系，争取做到明晰多文件是如何划分的，它们又是怎么协同完成操作系统的启动的。