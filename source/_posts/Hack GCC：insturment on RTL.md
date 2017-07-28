title: Hack GCC，在RTL上插桩
tags:
  - 编译安全
  - GCC
date: 2017-05-16 21:40:00
---
#### insn的创建
在`emit-rtl.c`当中，列举了一系列的`emit_insn`函数：  
	
	/* 在before前插入x */
	emit_insn_before_noloc(rtx x, rtx before)
	/* 在after后插入x */
	emit_insn_after_noloc(rtx x, rtx after)
	/* 插入x的同时插入jump */
	emit_jump_insn_before(rtx x, rtx before)
	emit_jump_insn_after(rtx x, rtx after)
	/* 插入x的同时插入call */
	emit_call_insn_before(rtx x, rtx before)
	emit_call_insn_after(rtx x, rtx after)
	
emit，也即发射。在指令生成之后，可以利用这一系列的API，把它“发射”到指令序列当中去。首先是要生成相应的指令，包括。其实insn-emit当中，已经提供了许多模版，比如`gen_nop`，就可以生成一条nop指令。    

#### instrument bound check 
在gcc/config/i386/i386.c当中，`ix86_expand_builtin`函数包含了一系列“处理器内置函数”的expand过程。这个函数中，首先用`DECL_FUNCTION_CODE()`对对应的内置功能进行判断，随后根据内置功能的类型进行进一步判断。其中，`IX86_BUILTIN_BNDCU/IX86_BUILTIN_BNDCL`就是bound check指令对应的处理。不妨来看看这两类指令是如何生成的吧：

	case IX86_BUILTIN_BNDCU:
	arg0 = CALL_EXPR_ARG(exp, 0);
	arg1 = CALL_EXPR_ARG(exp, 1);
	op0 = expand_normal(arg0);
	op1 = expand_normal(arg1);
	if(!register_operand(op0, Pmode))
		op0 = ix86_zero_expand_to_Pmode(op0);
	if(!register_operand(op1, BNDmode))
		op1 = copy_to_mode_reg(BNDmode, op1);
	emit_insn(BNDmode == BND64mode ? gen_bnd64_cu(op1, op0):gen_bnd32_cu(op1, op0));
	
可见,`arg0`和`arg1`是由`exp`得到的，随后被转化为`op0`和`op1`。这里对于`op0`和`op1`的模式进行了判断。这里的判断，是对操作数的machine mode进行了判断。对于`op0`来说，是判断它是否满足`Pmode`，也即指针使用的模式（比如x86_64用了48bit，这里的P表示partial）；而BNDmode则表示的是指针"bounds"的模式。后面的`gen_bnd64_cu(op1, op0)`也表明，`op1`是指针指向的地址，`op0`是bound寄存器的值。  

#### 寄存器的申请
对于`op1`来说，其值也就是相应的指令进行访问的地址。而`op0`则是具体的寄存器了，那么这个寄存器如何申请呢？在RTL当中，寄存器的选择是由`gen_rtx_REG()`来决定的。它的第一个参数，选择了这个寄存器的mode，第二个参数则是通过一个编号，来选择寄存器。这些寄存器的编号，定义在gcc/config/i386/i386.h当中。  

