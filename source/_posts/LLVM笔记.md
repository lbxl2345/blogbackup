---
title: LLVM框架与Pass
tags: 
	- llvm 
	- 编译器 
	- 编译安全
date: 2017-03-23 15:27:32
---

### LLVM简介
传统的编译器，采用的是针对单语言源代码，生成对应平台机器码的方式，类似于：  
![Simple](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/llvm/SimpleCompiler.png?raw=true)
而LLVM则采用了一种多前端，多后端的方式，如下：  
![Retargetable](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/llvm/RetargetableCompiler.png?raw=true)  
在LLVM当中，LLVM IR是一种low-level的，虚拟指令集。所有的前端语言都能够生成LLVM，从而能够被统一的进行处理。在LLVM当中，还提供了对LLVM IR Optimization进行优化的方式，能够对现有的源码进行搜索，匹配对应的pattern，从而进行指令的替换、修改。  
LLVM中，每个过程都是从Pass继承来的，LLVM优化器提供了许多passes，每个都写的很简洁，它们被编译成了库文件，并且在编译的时候被调用。这些库提供了分析和变换的能力，并且既能独立运作，又能合作。  
### 代码生成
那么LLVM这种“多对多”的编译方式，是如何将LLVM IR转化为机器码的呢？LLVM将代码的生成划分成了多个独立的过程：指令选择、寄存器生成，调度，代码布局优化，生成汇编代码。这样不同的平台，也能够利用自己的优势，对相同的LLVM IR进行优化。
LLVM采用了一种“mix and match”的方式，允许target作者，对于架构做出明确的指示，例如对寄存器的使用、限制做出明确的指示。LLVM利用Target-8description文件来指定对应架构的特性、使用的指令、寄存器。
![X86Target](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/llvm/X86Target.png?raw=true)  
而LLVM利用**tblgen**工具从这些.md文件当中，能够读取出足够的信息，并且在instruction selection、register allocator等过程中，选择处理的过程。而.cpp文件，则是实现一些特定的过程，例如浮点指针栈。  
### LLVM pass
LLVM pass完成编译器的变换、优化工作；它们是Pass的子类，根据不同的需要，可以选择去继承ModulePass，CallGraphSCCPass，FunctionPass，或者LoopPass，RegionPass和BasicBlockPass等。
llvm是需要编写、编译、加载和执行的，它相当于一个可以加载的模块。  
如果想编写一个模块，可以再llvm/lib/Transform当中创建对应的目录，并且在Transform以及目标文件夹下同时修改两个CMakeLists。在编译时，llvm的编译链会自动生成对应的pass。  
pass是基于中间语言llvm IR来进行的。因此它用来对.bc文件进行优化，例如：

	opt -load /lib/LLVMLbpass.so -lbpass <foo.bc> /dev/null

