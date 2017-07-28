---
title: IDAPython：入门
tags:
  - IDA pro
  - python
  - 静态分析
date: 2017-04-05 21:40:00
---
#### 取地址  
`ScreenEA()`和`here()`是最常用的取地址函数，它们会返回一个整数值。  
`MaxEA()`和`MinxEA()`可以用来取最大和最小地址。  
`GetDisasm()` `GetMnem()` `GetOpand()` 可以用来取某个地址中的指令、操作符、操作数等。  
#### 遍历：Segments，Functions,and Instructions
IDAPython的强大之处在于迭代。从`Segments()`开始是一个好的选择。这段代码能够遍历所有的segments。

	for seg in idautils.Segments():
		print idc.SegName(seg),	idc.SegStart(seg),	idc.SegEnd(seg)
		
同理的，对于Functions的遍历，可以有：
	
	for func in idautils.Functions():
		print hex(func), idc.GetFunctionName(func)

这里，`Functions()`是可以添加范围的，例如`Functions(start_addr, end_addr)`，而`get_func()`返回的函数，则还具有`startEA`和`endEA`属性，也即起起始和结束地址。这里，`get_func()`实际上是返回了一个类。`dir(class)`能够返回这个类中的成员。  
`idc.NextFunction(ea)`和`idc.PrevFunction(ea)`能够用来访问毗邻的两个函数。这里，`ea`只要是某个函数边界中的任何一个地址就可以了。利用这种方式来遍历也存在问题，如果不是函数的话，那么代码块会被IDA跳过。那么如果想在函数中遍历指令怎么办呢？`idc.NextHead`会帮你找到下一条指令。但这依赖于函数的边界，而且可能被jump影响。更好的方式是使用`idautils.FuncItems(ea)`，来遍历function当中的地址。  
一旦知道了一个function，就可以遍历其中的指令。idautils.FuncItems(ea)返回的是一个指向`list`的迭代器。这个`list`包含了每一条指令的起始地址。以下是一个综合的应用，它输出一段代码中的非直接跳转。    

	for func in idautils.Functions():		flags = idc.GetFunctionFlags(func)		if flags & FUNC_LIB or flags & FUNC_THUNK:			continue		dism_addr = list(idautils.FuncItems(func)) 
		for line in dism_addr:			m = idc.GetMnem(line)			if m == 'call' or m == 'jmp':				op = idc.GetOpType(line, 0) 
				if op == o_reg:					print "0x%x %s" % (line, idc.GetDisasm(line))
这里，`GetOpType`会返回一个整型值，它能被用来查看一个操作数是否为寄存器，内存引用。  
#### Operands
前面也有提到，`GetOpType`能够用来获取操作数的类型。一共有这些类型：  
`o_void`表示不包含操作数   
`o_reg`表示一个通用寄存器   
`o_mem`表示直接内存引用   
`o_phrase`操作数包含基址寄存器+偏移寄存器   
`o_displ`操作数是由一个寄存器和一个偏移量组成的   
`o_imm`操作数直接是一个整型数   
`o_far`和`o_near`在x86和x86_64下均不常见。指用立即数表示的far/near地址   
`idaapi.decode_insn()`可用来对某个地址的指令进行解码。而`idaapi.cmd`则能用来提取指令中的各部分结构。  

#### Searching
前文已经提到了一些搜索的方式，但有些时候可能会需要搜索一些特定的bytes序列，比如`0x55 0x8b 0xec`，这时就需要利用到`FindBinary`，对对应的格式进行搜索。其具体形式为`FindBinary(ea, flag, searchstr, radix)`。这其中，searchstr也即所搜索的pattern。radix是和CPU相关的，可以不写。  

	addr = MinEA()
	for x in range(0,5):
		addr = idc.FindBinary(addr, SEARCH_DOWN|SEARCH_NEXT, pattern);
		if addr != idc.BADADDR:
			print hex(addr), idc.Getdisasm(addr)

与之类似的，还有`FindText(ea, flag, y, x, searchstr)`，这个函数可以用来搜索一些字符串，例如一些变量中的内容可能是一些特定的字符串，就可以利用这个函数进行搜索。此外，还有`isCode()`和`isData()`，`isTail()`，`isUnknown()`等函数，来判断一个地址的属性。`FindCode()`能够找到下一个被标记为代码的地址；而`FindData()`则能够返回下一个被标记为数据的地址。与之类似的`FindUnexplored()`，`FindExplored()`则是搜索下一个未识别／识别的地址。`FindImmediate()`则是找到特定的立即数。  
不过，并不是每一次都需要对data/code进行搜索的。有时候，我们已经知道了code或者data的位置了，只是需要选择对应的内容进行分析。这时就可以用`SelStart()``SelEnd()`等一系列的函数。  
对于数据、代码，还可以对其原始格式进行访问，也即它们的二进制形式。这些数据可以用`Byte()``Word()``Dword()``Qword()``GetFloat()``GetDouble()`来获取。


