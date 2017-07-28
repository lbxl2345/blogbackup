---
title: RTL，指令和运算，识别特定指令
tags:
  - 编译安全
  - GCC
date: 2017-05-10 21:40:00
---
#### 运算
RTL中的运算，采用一种很直观、简单的方式来进行描述。通常，表达式都会借助于m mode。运算包含有算术、比较、向量操作、Bit域、类型转换等。  
#### Side Effect 表达式
之前所说的表达式代码，通常表示的是值，而非某种行为。但机器指令，只有在对机器的状态产生了改变时，才会有意义；因此对于这些对寄存器、执行状态产生影响的过程，是有对应的表达式的，被称为side effect表达式。  
一条指令的主体，一定是这些side effect之一；而之前表示值的表达式代码，只是作为它们的操作数出现的。  
set lval x：表示将x的止保存在lval当中。lval表示一个能够存放值的位置。它们包括（仅列出了比较常用的一部分）：  
return/simple_return：表述函数返回。  
call function nargs：表示函数调用。    
clobber x，表示不可预期的，可能的存储，将不可描述的值保存到x，必须为一个`reg`，`scratch`，`parallel`或者`mem`。说明x的值可能被修改（也就是说值会被改变？）  
use x：x的值被使用。x必须为寄存器。  
parallel：并行发生的side effects。  
cond_exec [cond expr]：条件执行的表达式。    
sequence [insns]：顺序执行的insn序列。  
除此之外，还有一些和内存地址有关的side effects，例如pre_dec:m x表示x减少一个标准值，其中m必须是指针的对应machine mode。例如(mem:DF (pre_dec:SI (reg:SI 39)))表示，减少伪寄存器39的值(减少DFMODE的大小)并且把结果作为一个DFMODE值。  
#### 指令
一个函数的代码的RTL形式，是保存在一个双向链表当中的，它被称为insns。insn是具有特殊代码的表达式，它们不作它用，其中一部分是用来生成真实的代码的，另一部分表示jump table，或者jump的labels。  
insn除了自身特定的数据之外，每一个insn还必须有一个独有的id－number，用来和当前函数中其他的insns进行区分，并且和其他的insns进行串联。在每个insns中，都有这样三个区域，分别用INSN_UID，PREV_INSN，NEXT_INSN这三个宏定义来取值。  
insn一定包含有如下的某一类表达式代码：  
insn：非jump/call的代码。  
jump_insn：可能会造成跳转的指令，包括ret。  
call_insn：可能进行函数调用的指令。call_insn也包含有一个域，CALL_INSN_FUNCTION_USAGE，它包含有一个链表，它记录了call调用所调用函数所需要利用的寄存器、内存地址等。    
code_label：表示jump_insn能够跳转到的label。它包含两个特殊的域，分别是CODE_LABEL_NUMBER和LABEL_NUSES，后者只在jump优化阶段结束之后，它包含目前这个函数中，该label被使用了多少次。    
jump_table_data：用来保存jumptable，位于tablejump_p insn之后。  
barrier：放在非条件jump指令之后。  
note：表示额外的调试信息。  
debug_insn：用来track的伪代码。  
#### insn，jump_insn,call_insn
这几类指令都含有额外的域：  
PATTERN(i)：这条指令的side effect表达式。  
INSN_CODE(i)：一个integer，表示了这个insn机器描述的pattern，但通常一般不会做匹配，因此一般都是－1，尤其是对于use，clobber，asm_inpu，addr_vec和addr_diff_vec表达式而言。  
LOG_LINK：insn_list链表，记录了instructions和basic block的依赖关系。只被schedulers和combine使用。  
REG_NOTES：一个链表，记录了和寄存器有关的信息。具体的，宏REG_NOTE_KIND返回了register note的类型。而PUT_REG_NOTE_KIND则用来修改类型。  
#### 一个例子
那么，不妨来看看在gcc里面，一个insn到底是如何表示的：

	(insn 21
	      20
	      22
	      6 
	      (set (reg:SI 59 [ D.1238 ])
	           (mem/c/i:SI (plus:SI (reg/f:SI 54 virtual-stack-vars)
	                                (const_int -8 [0xfffffffffffffff8])) [0 sum+0 S4 A32]))
	      /home/kito/sum.c:7
	      -1
	      (nil))
	      
把它和"iuuBeiie"对应起来，那么其实也就是一对一的关系，这里解释的很到位：  

	(insn 21 # 1. i
	      20 # 2. u
	      22 # 3. u
	      6  # 4. B
	      (set (reg:SI 59 [ D.1238 ])
	           (mem/c/i:SI (plus:SI (reg/f:SI 54 virtual-stack-vars)
	                                (const_int -8 [0xfffffffffffffff8])) [0 sum+0 S4 A32])) # 5. e
	      /home/kito/sum.c:7 # i
	      -1 # 6. i
	      (nil)) # 7. e
                        
i 代表该 INSN 的 uid   
u 上一道 INSN 的 uid  
u 下一道 INSN 的 uid  
B Basic Block的 编号 
e 该 INSN 的主要內容, 例如上面那道指令所描述的是從内存读取一個值到寄存器中  
i 该 INSN 对应源码的位置  
i 放 RTL pattern Name  
e 放 REG_NOTES 主要和寄存器有关   

其中，这个`“iuuBeiie”`是打印的格式，每个字符的含义定义在rtl.c当中。这里，我比较关心的'e'指的是一个rtl表达式。再来看一个例子。call_insn对应的输出是`“iuuBeiiee”`，其输出为：

	(call_insn	16 #1. i
				15 #2. u
				17 #3. u
				2  #4. B
				(call (mem:QI (symbol_ref:DI ("printf")[flags 0x41] <function_decl 0x7f1fd042e100 printf>)[0 __builtin_printf S1 A8]) const_int 0 [0])))) #5. e
				hello.c:12 #6. i
				649 	#7. i
				(nil)	#8. i
				(expr_list (use (reg:QI 0 ax))
					(expr_list:DI (use (reg:DI 5 di))
						(expr_list:SI (use (reg:SI 4 si))
							(nil)))))	#9. e
	)

和普通的insn相比，多出来的一项expr_list，是一个链表，它包含了用来传递参数的寄存器、内存的信息。  
#### RTL中的寄存器  
从GCC官方的说明来看，RTL和llvm一样，首先是假设“有无限个寄存器”的，这也就是说，在RTL代码中，必然会有伪寄存器存在。在编译过程中，这些伪寄存器，要么被替换为真实的、硬件寄存器，要么被替换为内存引用。我也为此困扰过：那么在翻译时，是如何确定RTL具体对应的机器码的呢？RTL既然是一个和机器相关的语言，那么寄存器的选择，应该与最后的指令生成无关了。  
其实这个问题是在gcc内置的rtl pass当中解决了。具体的，`register alloction`这个（准确的说是一系列）pass，它保证了所有的伪寄存器被清除了，其方式正是将它们替换为硬件寄存器、常量或者栈上的值。这也就是说，假设想要做一些和寄存器的选择相关的优化，最好是在这个过程之后去做。  
在这个过程之后，rtx中的`reg`就表示的是实实在在的寄存器了。而相对的，`subregs`也不存在了，而`mem`毫无疑问也就表示了对表达式addr所示地址的主内存进行引用。其中`mem:m addr alias`中，m描述的是被访问内存单元的大小，alias代表这个引用点别名集合，只有引用相同内存地址的项，才会放在一个别名集合里面。  
#### RTL，读取内存
既然知道`mem:m addr alias`用来引用内存，那么`addr`这个东西到底是怎么表示的？它是用一个寄存器来保存指针吗？还是用一个变量名称呢？首先我们来看看有哪些表达式会读取内存吧：    
**副作用表达式**  
`set lval x`，表示将x对值放到由lval表示的地方。那么如果x是一个`mem`，无疑是会对内存进行引用的。  
`parallel [x0 x1 ...]`，多个并行执行的副作用，那么其中的每一个`set`都会发生内存的引用。  
**算术表达式**  
在RTL中，算术表达式并不会改变一个值，它只是一个单独的运算，代表一个结果。举个例子，如果是一个`ADD`指令，那么它实际上在RTL中是通过一个parallel来表示，这个parallel包括了`plus`，还包含了`set`，只不过在汇编阶段生成的只是一条指令而已。所以说其实RTL里面的parallel，并不是说的概念上的并行，而是说某些表达式，会同时发生、生效。  
在`mem:m addr alias`当中，mem既可以是寄存器中存储的地址，也可以是常量指针中的地址。  

#### 指令的遍历
在GCC，所有的pass，都是以函数为粒度来进行调用的。它是一种“run on function”的机制。在gcc内部，当前处理的函数用`cfun`来进行访问。  
而指令，是由特殊的rtx表达式：`insn`来表示的。那么如何访问每一条指令呢？我个人认为，首先遍历基本块，随后再在基本块的内部进行遍历，是更为有效的。也就是这样的方式：

	FOR_EACH_BB_FN(bb, cfun){
		FOR_BB_INSNS(bb, insn){
			...	//handle insn
		}
	}

但是，INSN中其实还包含有`note`，如果只想留下`insn`，那么用`INSN_P(insn)`进行过滤就可以。这样一来，剩下的就只有`insn`、`jump_insn`、`call_insn`三种情况了（还有`debug_insn`，这里不考虑这种情况）。这三种情况，实际上还可以用宏继续细分判断，分别是`JUMP_P`，`CALL_P`，`NONJUMP_INSN_P`。  

#### 需要关注哪些指令？  
这里，我想定义的内存读取，指的是“通过一个指针，从内存中取值，并且把它放在寄存器当中”，当然这个取值可能会在计算之后放在寄存器中，例如`ADD`，也可能是通过`MOV`直接放到寄存器里面。无论如何，只要涉及到访问内容的数据（通过指针实现），并且结果被放到了寄存器中，这个行为就被认为是内存读取了。  
当然，在RTL中，指令还没有那么的直观。那么如何识别出这些指令呢？首先，有一点是可以肯定的，那就是实际上只有`nonjump_insn`会产生这些指令。我在分析之后，发现：`jump_insn`、`call_insn`，即使是间接跳转的（比如c++里面，常常可以见到的vpointer），它们其实也只是代表控制流转移的过程而已，其指针的提取、计算，其实都是放在这条指令之前的`insn`中去执行的。  


#### 指令中的内存读取  
下一步，就是从这些指令中，把真正的指令读取给提取出来。对应一个insn，它的输出格式是`"iuuBeiie"`，其中，e也即rtx表达式。在输出的时候，这部分表示的是这个insn的实际代码。`GET_CODE`能够用来提取出insn的代码部分，其实它只是一个`enum`：`#define GET_CODE(RTX) ((enum rtx_code) (RTX)->code)`，而`PATTERN`则是获取这个insn的副作用。这里，不应该被其名字所迷惑：因为`GET_CODE`不过是获取rtx code的类型，它和这个insn的内容，反而没有联系；相反，`PATTERN`则说明了这个insn会带来什么副作用，也就是对值的影响。  
在看了`var-tracking.c`和`gcse.c`当中的源码之后，我注意到一条`set`指令中的`src`和`dest`，实际上都可以用宏提取，也即`rtx src = SET_SRC(PATTERN(insn))`和`rtx dest = SET_DEST(PATTERN(insn))`。  
而PATTERN的代码类型其实也很有限。在`var-tracking`之后，实际上`sequence`也已经被去除了。那么实际上我们需要关注的PATTERN也就只有两种：分别是`set`和`parallel`。（实际上还有一种，那就是`asm_input`，这里先不予考虑）  
那么对于`insn`，也就只有两种情况，一是一个单独的`set`，二是一个`parallel`，它可能包含了很多条`insn`，那么其中的指令就要逐条分析了。  
注意：在gcc 5.0之后点版本，专门的数据结构rtx_insn表示一条指令，而不是用rtx来一概而论了。  

#### SET的分析  
一个`set val x`，其lval可以是`reg`，`mem`，`pc`，`parallel`或者`cc0`。其中，`parallel`用来表示一个函数通过多个寄存器来返回一个结构体的情况，这里不用关注这个情况，只需要关注lval是`reg`，而x是`mem`的情况。（目前只考虑了这个最基本的情况）   
对于`x`来说，set的源操作数可能是一个内存地址，还可能是一个表达式的，比如加减乘除、位操作等。  


