---
title: RTL，类型和操作数
tags:
  - 编译安全
  - GCC
date: 2017-05-07 21:40:00
---
#### RTX
rtx(RTL expression)，也即一个RTL的表达式。在`coretypes.h`中，其被定义为`struct rtx_def *rtx`，也即一个指向rtx_def的指针。那么rtx_def是一个什么结构呢？它同样定义在rtl.h当中：

	struct GTY((chain_next ("RTX_NEXT (&%h)"),
	    chain_prev ("RTX_PREV (&%h)"), variable_size)) rtx_def {
	  /* 表达式类型 */
	  ENUM_BITFIELD(rtx_code) code: 16;
	
	  /* 表达式包含值的类型 */
	  ENUM_BITFIELD(machine_mode) mode : 8;
	
	  /* 一系列位段，在不同情况下表示不同含义*/
	  unsigned int jump : 1;
	  
	  unsigned int call : 1;
	 
	  unsigned int unchanging : 1;
	  
	  unsigned int volatil : 1;
	 
	  unsigned int in_struct : 1;
	 
	  unsigned int used : 1;
	  
	  unsigned frame_related : 1;
	  
	  unsigned return_val : 1;
	  
	  /* rtx的第一个操作数，操作数的个数和类型是在code域中定义的*/
	  union u {
	    rtunion fld[1];
	    HOST_WIDE_INT hwint[1];
	    struct block_symbol block_sym;
	    struct real_value rv;
	    struct fixed_value fv;
	  } GTY ((special ("rtx_def"), desc ("GET_CODE (&%0)"))) u;
	};
	
[https://dmalcolm.fedorapeople.org/gcc/2013-10-31/doxygen/html/structrtx__def.html](https://dmalcolm.fedorapeople.org/gcc/2013-10-31/doxygen/html/structrtx__def.html)中对每个位段进行了更详细的描述。

其中，rtx所包含的所有可能情况，都包含在了rtl.def当中；注意这里rtx和insn其实有一些微妙的关系：rtx表示一条中间指令的时候，它是一个insn；rtx不一定是insn，但insn一定是rtx。

#### RTL：对象
如果想在RTL阶段上，进行编译的优化，那么首先就必须好好了解RTL。RTL使用五种类型的对象： expressions、integers、wide integers、strings和vectors。其中，RTX是最为重要的的一个，它是一个C数据结构，通常用指针的形式来调用，也就是前面所提到的rtx。  
interger和wide integer分别对应着机器上的整型和长整型。string则是一串非空的字符；vector则包含指向任意数量expression的指针，用[...]的形式来表示，并用空格分割。expressions被称为RTX code，它定义在rtl.def当中，其含义是与机器无关的。一个rtx的code部分，可以用`GET_CODE(x)`来获取，并且可以用`PUTCODE(x, newcode)`来替换。  
expression code决定了：表达式所包含的操作数个数、它们是什么类型的objdect。RTL不像Lisp，不能通过operand自身来确定它是什么类型的object，而必须知道它的上下文（例如一个subreg code，它的第一个操作数被当成表达式，而第二个被认为是一个整型；又例如plus code，两个operands都被当成了表达式）。expression写作：()，包含表达式类型、flags、机器码等，以及用空格分割的操作数。  
#### RTL：类和格式
expression code可被归为许多类，它们用**一个字符**来表示，`GET_RTX_CLASS(code)`能够用来获取RTX代码的类型，这些类定义在`rtl.def`当中。  
RTX_OBJ：表示一个具体的object，例如REG/MEM/SYMBOL等  
RTX_CONST_OBJ：表示一个常量object  
RTX_COMPARE：表示不对称的比较，例如GEU/LT。  
RTX_COMM_COMPARE：表示对称的比较。(表示满足交换律的)  
RTX_UNARY：一元的算数操作，例如NEG、NOT、ABS、类型转换等  
RTX_COMM_ARITH：满足交换律的二进制操作，例如PLUS、AND。  
RTX_BIN_ARITH：不满足交换律的二进制操作，例如MINUX、DIV。  
RTX_BITFIELD_OPS：位操作，包含ZERO_EXTRACT，SIGH_EXTRACT。  
RTX_TERBARY：其他有三个输入的操作，例如IF_THEN_ELSE。  
RTX_INSN：表示整条INSN，包括INSN、JUMP_INSN、CALL_INSN。  
RTX_MATCH：表示insns当中的matches，例如MATCH_DUP。  
RTX_AUTOINC：表示寻址时的自增，例如POST_INC。  
RTX_EXTRA：所有其它的codes。  
这些class，能够让expression code采用很方便的方式来表示，比如subreg直接可以表示为'ei'。  
`GET_RTX_LENGTH(code)`和`GET_RTX_FORMAT(code)`分别会返回操作数的数量，以及格式(简写，例如一个比较的操作写成ee)  
#### 访问操作数
通过宏定义，可以访问一个表达式的操作数，包含`XEXP`、`XINT`、`XWINT`、`XSTR`。这些宏都包含了两个参数：其一是一个RTX，其二是从0开始数的操作数编号。例如`XEXP(x,2)`访问了x的第二个操作数(作为expression来访问)。`XINT(x,2)`访问了x的第二个操作数(作为integer来访问)。  
使用者必须自己根据expression code来判断每个操作数是什么类型的。而对vector的访问则更复杂，`XVEC`能够获取vector指针，而`XVECEXP`和`XVECLEN`可以用来访问一个vector的元素和长度。    
对于某些的RTL节点，还有一些特殊的操作数访问方式。例如MEM、REG、SYMBOL_REF都能够用来获取不同的相关信息。
#### Machine Modes
Machine Modes描述了数据对象的大小，以及它的表现形式。Machine mode以枚举的形式来表示，machine_mode，定义在machmode.def当中。每个RTL表达式，都包含有一个machine mode的区域。在dump文件、机器描述中，RTL的machine mode被写在表达式的后面，用一个冒号分隔。  
在GCC中，insn相当于rtx的组合。例如，
  
	insn: mov ax, 8
	rtx: ax
	rtx: 8
	rtx: mov ax, 8