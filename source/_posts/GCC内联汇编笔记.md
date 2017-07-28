---
title: GCC内联汇编笔记
date: 2016-03-22 17:40:00
tags:
- GCC 
- x64 assembly

---
### GCC汇编语法
GCC使用的是AT&T语法。一些基本的语法如下:  
1. 源地址/目的地址语序：Opcode src dst  
2. 寄存器命名：寄存器添加前缀"%"  
3. 立即操作数：立即操作数和静态C变量要有前缀"$"  
4. 操作数大小：内存操作数大小取决于opcode对最后一个字母，例如"b","q"等  
5. 内存操作数：用"()"来进行内存引用  
6. 偏移量：写在"()"之前  

tips:经过实验，发现寄存器的使用必须使用%%register，而变量可以使用%val，也就是说%的数量说明了操作数的类型


### 拓展内联汇编
拓展汇编基本格式：

	asm ( assembler template
     : output operands                  /* optional */
     : input operands                   /* optional */
     : list of clobbered registers      /* optional */
     );

其中，assembler template(汇编模板)即汇编指令。output operands为输出操作数，input为输入操作数。它们之间用冒号间隔开来。最后一部分是用来保护内联汇编中可能被污染的寄存器的。如果没有输出但是有输入操作数，则必须用两个冒号进行说明。多个操作数由逗号分开，而多个指令应由/n/t分隔开来。

### 操作数约束
在汇编程序模板中，操作数通过编号来引用，从0开始，一直到n-1。常见的寄存器操作数约束如下:  

r|Resigter
---|---
a|%rax
b|%rbx
c|%rcx
d|%rdx
S|%rsi
D|%rdi

其中，输入的格式为"r"(val),输出的格式为"=r"()。




