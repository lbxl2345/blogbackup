---
title: IDAPython:进阶（一）
tags:
  - 操作系统
  - 系统安全
  - linux
date: 2017-04-05 21:40:00
---
#### IDAPython:进阶（一）
#### Instruction feature
在IDAPython中，时常可以看到使用`insn.get_canon_feature()`来获取指令“特性”。这里，`instruc_t`类中包含有两个变量，`name`和`feature`，其中`feature`是一系列的指令特征位。 

#### IDA SDK
IDA SDK分为很多hpp，不过它的文档不是特别友好，这里我把比较重要的header的作用给列出来，方便日后进行分析和使用：  

+ area:程序中地址范围的集合，它表示一段连续的地址范围，由起始地址和终止地址来表示，例如程序中的segments就是用area来表示的，它在IDA的数据库当中，是采用B树的形式来保存的。  
+ bitrange:用来管理一段连续bits的容器，类似一个数组。  
+ bytes:处理byte特性的函数，在程序中，每个byte都被识别成一个32-bit的值，他被IDA称为`flags`。对于bits和flags，都只能通过特定的函数去修改、访问。flags被保存在*.id1这样的文件当中。  
+ diskio:IDA的文件IO函数，通常不使用标准的C来进行I/O，而使用这个hpp当中的函数。  
+ entry:处理entry入口的函数，每个entry包含有地址、名称、序号。  
+ fixup:处理地址、偏移量的修正等。  
+ frame:处理栈桢，包括参数、返回值、保存的寄存器和局部变量等。  
+ funcs:处理反汇编程序的函数，函数由多个chunk组成。  
+ idp:包含IDP模块的接口，包括有目标的汇编器，以及当前处理器的信息。例如判断一条指令是否为jmp、ret等，都可以使用这种方式。
+ lines:处理反汇编text line的生成。
+ name:处理命名，给一个特定地址命名等，但是指令和数据的中间是不能命名的。  
+ netnode:database的底层接口，程序被以B树的形式保存，而B树的信息则是存放在netnode当中的。  
+ offset:用来处理offset的函数，一个操作数可能自身，或者一部分表示了程序中的偏移量。  
+ search:中间层的搜索函数，包括寻找数据、代码等。  
+ segments:用来和程序中段进行交互的函数，IDA需要程序中的所有地址，属于一个segment(每个地址都必须属于一个segment)，如果地址不属于一个segment，那么这些bytes不能被转换为指令、不能用有名字、不能拥有注释。每个segment都有一个selector。  
+ ua:处理程序指令的反汇编。它包含两种函数，第一类是通过kernel来反汇编，第二类是通过IDP模块来反汇编，它们是“helper”。反汇编可以分为三步：分析、仿真、转化为文字。  
+ xref:处理交叉引用的情况。包括有CODE和DATA的引用。  

#### 获取文件名
SDK提供了两个api，idaapi.get_root_filename能够用来获取当前的文件名，而idaapi.get_input_file_path能够用来获取包含路径的文件名。这两个函数定义在nalt当中。  
