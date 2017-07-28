#### define_insn
在具体机器的描述文件当中，对每条指令进行了描述。例如对i386平台，其指令就描述在i386.md中。指令可以分成几个操作数：名字、模版、条件、输出模版、匹配用的vector。  
在翻译时，rtx会找到这样一个对象，在process_rtx中，
"｜"表示多种输出的可能

match_operand: type n
这里的数字n指的是操作数的编号
在后面的指令输出里面用%n表示这个操作数  
register_operand(rtx op, machine_mode mode)：如果OP是一个mode类型的寄存器引用，返回1。如果MODE是VOIDmode那么任何类型的寄存器都可以。这个函数主要用来处理match_operand。

expand_normal：产生计算expression EXP的code，会返回所计算exp的rtx。

define_mode_attr bnd_prt [(BND32 "SI")(BND64 "DI")]

定义在insn-attrtab.c
int insn_default_length
调用switch(recog_memoized)。如果出现了case -1，就会调用fatal_insn_not_found。这里，recog_memoized是这样一个函数：

	recog_memoized(rtx_insn *insn){
		if(INSN_CODE(insn) < 0)
			INSN_CODE(insn) = recog(PATTERN(insn), insn, 0);
		return INSN_CODE;
	}
	
这里INSN_CODE还没有确定下来，就进一步调用recog来识别它的CODE。

1. 前端读取源代码->建立解析树。  
2. 基于命名的指令模式，解析树被用来生成rtl insn列表
3. rtl insn->匹配RTL模版，产生汇编代码。  

操作数约束：
可以告诉一个操作数是否可以在寄存器中，以及哪种寄存器；操作数是否可以为内存引用，以及哪种寻址方式；操作数能否为立即数，以及可以具有什么值。  
"Ts"是一种特殊的约束，它表示Address operand without segment register，也即不包含段寄存器的地址操作数。

	define_mode_attr bnd_ptr [(BND32 "SI") (BND64 "DI")] 
	
	(define_insn) "*<mode>_<bndcheck>"  
	  [(parallel) [unspec [(match_operand:BND 0 "register_operand" "w")  
	  						(match_operand<bnd_ptr> 1 "address_no_seg_operand" "Ts")] BNDCHECK)  
	  		   		set (match_operand:BLK 2 "bnd_mem_operator")  
	  		   			(unspec:BLK 2 "bnd_mem_operator")  
	  		   			(unspec:BLK [(match_dup 2)] UNSPEC_MPX_FENCE))])]  
	"TARGET_MPX"
	"bnd<bndcheck>\t{%a1,%0|%0, %a1}"
	[(set_attr "type" "mpxchk")]
	
来分析一下这个约束吧。首先，这个pattern是一个parallel，它包含两个指令，第一部分是一个unspec，第二部分是一个set。`match_operand<bnd_ptr> 1 "address_no_seg_operand" "Ts"`，表示操作数1它是一个`<bnd_ptr>`（在64bit模式下也就是`DI`)。其后，后面的address_no_seg_operand是predicate，也就是对match_operand进行的一次判断，看它是否满足条件，只有满足条件才会匹配，否则指令模式根本不匹配。那么在这里，这个指令必须满足"address_no_seg_operand"。这里，address_no_seg_operand定义在predicates.m当中，生成在insn-preds.c当中。其定义如下：

	int address_no_seg_operand_1(rtx op, machine_mode mode ATTRIBUTE_UNUSED)
	{
		struct ix86_address parts;
		int ok;
		if(!CONST_INT_P(op)
			&& mode != VOIDmode
			&& GET_MODE(op)	!= mode)
			return false;
			
			ok = ix86_decompose_address(op, &parts);
			gcc_assert(ok);
			return parts.seg == SEG_DEFAULT;
	}
	
对这个模版来说，address_no_seg_operand_1把`DI`作为mode，然后判断op是否和DI一致；然后用ix86_decompose_address来分析它是否表示一个正确的地址，并且把(parts.seg == SEG_DEFAULT)返回。  


我把gcc5-build/gcc/insn-preds.c修改了。我发现错误发生在address_no_seg_operand_1之后。说明检验通过了。

我对insn_default_length进行了修改。我发现指令通过recog_memoized返回的值是-1，它没有被识别出来。这个函数调用了:

	recog(PATTERN(insn), insn, 0); 
	
recog定义在gcc-build/gcc/insn-recog.c当中。

	把pattern，insn作为输入。首先是对PATTERN(insn)进行判断，GET_MODE(x0)。把BLK、PARALLEL、都识别出来。这实际上相当于一个自动机。
	
	假如是一个PARALLEL，那么会首先判断其长度。
	
	如果是一个SET。这时会先XEXP(x0,0)取它的mode。
	
	
RTX>pattern>code>class

我在gcc-build/gcc/insn-recog.c当中，把限制给删除了。结果匹配时没能匹配对应的constrain。我觉得还是生成的方式不对。要满足对应的约束条件才行。这里看看address_no_seg_operand它的判定，因为实际上出错的就是这里。 

于是我又回到address_no_seg_operand_1。再次处理它。所以这里最先出错的是：address_operand(op, VOIDmode)。address_operand定义在gcc/recog.c当中。

我用XEXP取了src里面的第一项，也就是一个exp，然后把它作为gen_bud64_cu的输入，然后成功的插桩了。

那么现在我要对插桩进行一次优化，试着把多余的插桩给去掉。这时候就需要对寄存器进行统计了……寄存器的个数和类别，定义在i386.h/i386.md当中。可以看到一共有80个寄存器，包括bounds寄存器在内。  
下一步是提取出rtx中的寄存器。可以看到rtl.h当中，REGNO()是能够提取一个REG rtx的寄存器的。
三目 运算  
RTL规定，对于可交换的二进制操作，常量应该放到第二个操作数的位置。

我先获取RTX的格式，保证它的format是ee，两个表达式，这样一定不会出错。  

我决定把现阶段的成果在SPEC上先进行尝试一下。我发现现在的状况是，我需要把共享库全部编译一遍，然后再对SPEC进行编译。总之工作量很大，编译共享库的时候或许还会有bug。雷打不动 晚上复习，现在总算编译成功了gcc。 

gcc在编译时可以指定dynamic linker。其实AG里面都告诉我了，只不过我自己看的不够仔细，所以没有发现……

在今天最后的时候，我发现自己的插桩其实有许多漏洞的，比如，对于mem来说，自增自减就需要特殊处理。我想了一下，今天的任务暂时定为，把插桩的问题给解决了吧～

特殊的情况：
1. mem是常量
2. mem是push/pop
3. cmp的情况，两个其中只要有一个是addr，就对那个做检查
4. 算术表达式没有被插桩


算术运算的情况，set的第二个数，不一定是mem，它是算术表达式中，包含mem的情况。我个人认为最保险的写法，是对着rtl.def里面，一个一个的进行处理…

所有的RTX_COMM_ARITH都是“ee”。奇怪的是，似乎，gcc并没有控制rip的能力…

现在希望对函数的trampoline进行处理。cfghooks中，提供了创建、删除基本块等功能；这里首先需要去了解基本块，而且要去了解怎么去创建symbol。

FOR_EACH_FUNCTION对symtable里面的每个函数都进行了处理。首先要创建trampoline，再把那些取函数入口的指令修改为trampoline…在i386.c中，有一个函数，`construct_plt_address`，它生成的是move,add，并且放到一个寄存器当中去，这是在调用的时候生成的指令。pl是由linker由.rel来添加的。  

dir ./gcc5-build/gcc

这样看来，是函数的问题，而不是insn的问题。   