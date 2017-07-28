title: GCC，Overview
tags:
  - 编译安全
  - GCC
date: 2017-05-01 21:40:00
---
#### 前端：AST/GENERIC
AST:用`tree`来表示函数。其中，`tree`实际上是一个指针，它能够指向许多种类型的`tree_node`，这些`tree_node`定义在tree-core.h当中，就不一一赘述了。只需要明白，GCC当中最重要的数据就够就是tree，如果你想要添加一种`tree_node`，也可以在tree.def里面添加。  

tree-core.h中定义了两种数据结构，用来直接表示tree节点，分别是tree_list和tree_vec。它们都包含有一个tree_common结构。tree_common包含有一个chain，用来和其他的tree链接起来。而tree_list则包含有purpose和value，它们都是`tree_node`。**value包含了一个tree的主体**。  
可以说，在GENERIC中，`tree_node`才是主角。正如前面所说的，`tree_node`根据表示内容，也有不同的类型，而不同类型的`tree_node`的代码也不同了。比如指针类型使用POINTER_TYPE代码，而数组使用ARRAY_TYPE代码。  
函数的声明、属性、表达式等，都是通过tree_node来表示的。在AST中，函数可以分为几个部分，分别是名字、参数、结果和函数体。这几个部分，都是通过`tree_node`来表示的。当然函数的内部还包含有许多属性。值得一提的是，有时候tree会为后端保留一些slot，用来在树被转化为RTL时，给GCC后端使用。 

#### 中端：GIMPLE
**目前C和C++的前端直接从tree转换为GIMPLE**，不再先转换为GENERIC了。  
在GCC中，`gimplifier`过程将原始的GENERIC表达式，生成（不超过3个操作数）的GIMPLE元组。GIMPLE同样是基于tree结构的中间语言。其实GIMPLE还可以进行细分，它表现出了编译器在middle end的过程。没有完全下降的GIMPLE被称为“High GIMPLE”。“Low GIMPLE”完成了进一步下降，消除了隐式的跳转和异常表达式。  
在Low GIMPLE之后，是基于SSA的GIMPLE。SSA，“静态单赋值”保证程序中的变量，只在一个位置赋值（赋值多次就在每次赋值后产生新的变量），这样做的好处是利于数据流分析，大多数优化都是基于SSA来做的。基于GIMPLE的优化过程，都是与机器语言无关的，它包括变量的属性设置、数据流分析和优化、别名分析等。  
目前，大多数插桩的工作，也都是在GIMPLE上完成的，因为它属于与目标机器代码无关的中间语言；这也和LLVM IR类似。  

#### 后端：RTL
RTL则是从GIMPLE进一步拓展而来，它与机器语言直接相关。它也是大部分pass依赖的中间表示，和Lisp很类似。RTL会在优化过程中，逐渐去掉伪寄存器、合并指令，删除控制流图等。最后GCC根据RTL语言，根据insn-output当中的的模版，一一对应的输出汇编格式，再由汇编翻译成二进制代码。  
RTL语言的定义位于rtl.def当中，在make过程中，GCC会根据机器描述文件，从对应到机器代码中提取寄存器、指令的配置，并生成相应的insn-out.c等文件，它们会在make install的时候进一步生成GCC。  
值得一提的是，不论是RTL还是GIMPLE，都是和编译器所维护的控制流图相关的。控制流图由基本块和边来表示，并在编译过程中保持更新。控制流图在树拓展为RTL的过程中被丢弃。  

#### 小结
总体而言，GCC的编译过程可以归纳为：从源码到语法树、从语法树到GIMPLE、从GIMPLE到RTL，再从RTL到汇编。在每个阶段，而GCC的优化工作，主要在GIMPLE和RTL上面进行。    

