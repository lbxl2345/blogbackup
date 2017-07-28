title: 浅谈GCC编译优化  
tags:
  - 编译器
  - GCC
date: 2017-07-15 21:40:00
---
#### GCC Pass
在GCC完成词法、语法分析，并获得源代码对应的抽象语法树AST之后，会将其转换为对应的GIMPLE序列。随后，GCC会对GIMPLE中间表示进行一系列的处理，包括GIMPLE的低级化、优化、RTL生成等、RTL优化等。为了方便管理，GCC采用了一种称为Pass的组织形式，把它们分成一个个的处理过程，把每个输出结果作为下一个处理过程的输入。  
GCC中，Pass可以分为4类，GIMPLE_PASS，RTL_PASS，SIMPLE_IPA_PASS，IPA_PASS。其中除了RTL_PASS之外，处理对象都是GIMPLE中间表示。名字包含IPA的两类Pass的功能主要是过程分析，也即函数间调用和传递。  Pass的执行是以链表的形式组织的，每个Pass还可以包含子Pass，并且以函数为执行的基本单元。  
如果把GCC中的Pass全部打印出来，可以看到数量有数百个，因此这里只选择几个代表性的pass研究一下。值得说明的是，在GCC 4.6之后，增加了插件功能，它同样是通过Pass的形式来进行管理，这使得我们自定义处理过程变得很容易。    

#### 去除无用表达式
该Pass从GIMPLE序列中进行搜索，从中删除死代码。这些死代码主要包括：  
（1）空语句  
（2）无意义的语句块、条件表达式  
（3）目标地址就是下一条段GOTO表达式  
（4）无意义的malloc、free  
（5）...
在GCC 4.4版本中，这个Pass是`pass_remove_useless_stmts`，现在这个函数已经被移除了；其功能被拆分到了别的函数中。但gcc的文档中还是说明了这个Pass。  
类似这样的结构，在这个Pass中都会被优化掉：

	if(1)		//无意义的条件表达式
	
	goto next;	//无意义的goto
	next:
		//do sth
		
那么该Pass是如何进行判断的呢？我们来看看GCC 5.4版本中，类似函数的实现。在gcc/tree-ssa-dce.c当中，有这样一个函数`eliminate_unnecessary_stmts()`。这个函数会(逆序的)逐个遍历表达式，防止某些定义或名字被释放：

	for(gsi = gsi_last_bb (bb); !gsi_end_p (gsi); gsi = psi){
		stmt = gsi_stmt (gsi);
		...
		if (gimple_call_with_bounds_p (stmt)) {
			//如果有必要删除，那么会首先利用这个函数设置
			gimple_set_plt(...)
		}
		//对于需要删除的表达式进行删除
		if (!gimple_plf (stmt, STMT_NECESSARY))
		...
	}
	
如果进行了删除，那么这个函数还会对一部分CFG进行修改，并删除不可达的基本块。

#### 控制流图的构造
在GCC中，控制流图指的是**函数内的控制流图**，这和我平时在文章中看到的“CFG”是不同的。GCC中，CFG的节点为基本块，而边则是基本块之间的跳转关系。`pass_build_cfg`对函数对GIMPLE序列进行分析，完成基本块的划分，并且根据GIMPLE语义来构造基本块之间的跳转关系。  
控制流图的构造，主要是在`build_gimple_cfg()`当中完成的。

	static void
	build_gimgple_cfg (gimple_seq seq)
	{
		gimple_register_cfg_hooks();//注册这个函数，要构造cfg了
		init_empty_tree_cfg();//先构造一个空的cfg
		make_blocks(seq);//具体的基本块构造过程
		...
		make_edges();//创建基本块的边
	}
	
	static void
	make_blocks (gimple_seq seq)
	{
		while (!gsi_end_p (i))//遍历seq中的每一条
		{
			...	
			gimple_set_bb (stmt, bb);//把当前的语句添加到基本块
			if(stmt_ends_bb_p(stmt))
			{
			//如果stmt是基本块的终结，就进行处理，并且在下一次创建一个新的基本块
			...
			}
		}
	}

在基本块创建之后，就可以根据基本块，构造基本块中的边，对于函数中的不同情况，生成了不同的边，并且创建了对应的label。

	static void
	make_edges (basic_block min, basic_block max, int updata_p)
	{
		...
		//处理min和max之间的基本块
		FOR_BB_BETWEEN (bb, min, max->next_bb, next_bb)
		{
			//在5.4版本中，是直接对RTL进行处理
			rtx_insn *insn;
			insn = BB_END(bb);
			//如果为JUMP指令，还包括了条件跳转
			if(code == JUMP_INSN)
			...
			//如果为CALL指令，还包括了异常处理
			else if(code == CALL_INSN || cfun->can_throw_non_call_exceptions)
			...
		}
	}
	
#### 指令调度
前面介绍了GCC当中，GIMPLE表达式的部分优化过程，现在我们来说说RTL中的优化过程。指令调度是对当前函数中的insn序列进行重新排列，从而更充分地利用目标机器的硬件资源，提高指令执行的效率。指令调度主要考虑有数据的依赖关系、控制相关性、结构相关、指令延迟和指令代价等，通常指令调度与目标架构上的流水线有关。在GCC中，指令调度分为两个部分：寄存器分配之前的pass_sched和寄存器分配之后的pass_sched2。  
不过，在这里应该先阐述一下：为什么要进行指令调度？这是由硬件的特性决定的：  
（1）指令拥有不同的执行时间。对于指令来说，其实latency（指令开始执行后，多少时钟数能够为其他指令提供数据）、throughtput（指令完成计算需要的时钟数）、μops（指令包含的微操作数）都是不同的。  
（2）指令流水线能够并行执行指令的微操作，从而在等待时完成一部分工作。  
（3）超标量处理器，比如现在的i7处理器，能够同时运行多条指令，虽然硬件完成了一部分调度工作，但存在一个窗口的问题，依然需要编译器进行调度。  
正因如此，编译器需要对指令进行调度，来提高运行时的效率。指令在调度时，存在着以下限制：  
（1）数据依赖关系。包括flow（一条指令定义的值和寄存器在后面的指令用到）、anti（一条指令用的值或者寄存器会被后面的指令修改）、output（一条指令定义的值或寄存器在后面会被改；  
<!--（2）控制依赖关系。也即涉及到控制流转移的指令，例如if语句等； --> 
（2）循环限制，多次迭代之间的关系，循环内部的依赖关系等；  
（3）硬件资源的限制，例如能够同时并行的指令数；
这些限制必须作为指令调度的输入被考虑到，通过这些依赖关系，可以把所有的指令，构成一个依赖关系图。  
指令调度是一个典型的NP-hard问题，而**表调度算法**就是一种典型的启发式算法。通常其调度的单元在**基本块的内部**，并把数据依赖关系图作为输入，并计算出指令的优先级。优先级的计算涉及：依赖图的根到节点的路径长度、某个节点的后继数量、节点的latency等。  
随后，保持两个表：**ready**表中保存了能够不延时执行的指令，它们根据优先级从高到低排列；**active**表包括了所有正在执行的指令。在算法的每一步：  
（1）从**ready**表中取出优先级最高的指令进行调度，将它移到**active**队列当中，它会在队列中呆上一个时延的时间；
（2）更新指令的依赖关系，把新的就绪指令插入**ready**表中。这里我们用一个例子来说明：  
在每一轮，都取了**ready**队列中，优先级最高的指令，如果**active**中的指令没有执行完，导致依赖关系无法满足，那么**ready**可能是空的，此时就继续下一轮调度。  
![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%BC%96%E8%AF%91%E5%99%A8/%E8%A1%A8%E8%B0%83%E5%BA%A6.png?raw=true)  

cycle | ready | active  
---|---|---
1 | a13, c12, e10, g8 | a
2 | c12, e10, g8 | a, c
3 | e10, g8 | a, c, e
4 | b10, g8 | b, c, e
5 | d9, g8 | d, e
6 | g8 | d, g
7 | f7 | f, g
8 |  | f, g
9 | h5 | h
10 |  | h
11 |  i3 | i  


GCC中默认的指令调度算法是**表调度算法**。其处理过程在/gcc/sched-rgn.c中定义。`schedule_insns()`是调度的主要函数，它会执行两次，分别在：  
（1）寄存器分配之前，在`pass_sched`中调用，实现以区域为调度范围的指令调度；  
（2）寄存器分配之后，在`pass_sched2`中调用，通常只在每个基本块内部进行指令调度；  

	schedule_insns()
	{
		int rgn;
		//如果没有包含代码的基本块，直接退出
		if (n_basic_blocks_for_fn (cfun) == NUM_FIXED_BLOCKS)
    		return;
    	//初始化指令调度信息
		rgn_setup_common_sched_info ();
		//初始表调度的基本信息
  		rgn_setup_sched_infos ();
		//haifa表调度的数据结构初始化，进行数据流分析
  		haifa_sched_init ();
  		//初始化区域信息，将CFG的区域提取出来
  		sched_rgn_init (reload_completed);

  		bitmap_initialize (&not_in_df, 0);
  		bitmap_clear (&not_in_df);	
  		
  		//对每个区域内的基本块进行调度
  		for (rgn = 0; rgn < nr_regions; rgn++)
    	if (dbg_cnt (sched_region))
      		schedule_region (rgn);

  		//清除
  		sched_rgn_finish ();
  		bitmap_clear (&not_in_df);
  		haifa_sched_finish ();
	}  

#### 统一寄存器分配
我们知道，RTL在生成和处理的过程中，大量地使用了虚拟寄存器，那么这些虚拟寄存器在转换为目标机器的汇编码之前，需要映射到目标机器中的物理寄存器上，该过程即为寄存器分配。那么合理地分配、使用物理寄存器，是GCC后端及其重要的一个任务。  
GCC中采用的是一种把寄存器合并、寄存器生存范围划分、寄存器优选、代码生成和染色整合在一起的算法，所以被称为统一寄存器分配。整个统一寄存器分配是这样一个过程：  

![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%BC%96%E8%AF%91%E5%99%A8/ira.png?raw=true)  

第一步是建立IRA的中间表示，其代码在ira-build.c中，入口函数为`ira-build()`；  
**第二步**是寄存器的着色。这一步按照自顶向下的顺序，遍历每个区域，对于区域中的分配元进行着色，由`ira_color()`函数完成；   
第三步是代码移动，解决父子区域中，寄存器被spill/store的问题，如果子区域需要spill到内存，那么父区域也可以直接spill到内存，这样就不用多次进行移入内存/存内存取值到寄存器的过程；  
第四步是代码修改，在着色处理后，父子区域中，相同虚拟寄存器可能会分配到不同的物理寄存器活着存储位置，而某个区域内部也可能会会对同一个虚拟寄存器分配不同的物理寄存器。此时IRA就分配新的虚拟寄存器进行取代，并且在区域边界实现新的分配元并进行交换；   
第五步：将所有区域的分配元合并到一个区域中；  
第六步：尝试对spill操作的分配元分配物理寄存器；  
第七步：Reload Pass。前面的过程可能会引入新的代码、虚拟寄存器，产生不能满足模版约束的情况，reload处理这些情况，因此要重新进行第二步开始的操作，直到不再产生新的代码。  

这里只说明一下寄存器着色的过程。在第一步中，我们能够根据每个虚拟寄存器的生存范围，能够确定它们的冲突关系：生存范围有冲突的虚拟寄存器，不能被分配到同一个硬件寄存器当中去（`ira_build_conflicts()`的工作）。而寄存器着色，就是根据虚拟寄存器之间的冲突关系图，进行图的着色处理。在这个图中，每个节点代表需要着色的虚拟寄存器，而每条边都定义了冲突关系，也就是这两个虚拟寄存器不能着相同的颜色。那么如果有N个寄存器可以分配，就相当于用N个颜色来着色；如果N不够大，那么就需要物理内存来保存虚拟寄存器的值。  

#### 小结
编译器的优化是一个及其复杂的过程，这里我只是从几个点出发，了解它的工作过程，对编译优化形成一个浅薄的认识。日后如果真的要从事编译优化方面的研究，再好好看看编译器的代码吧！ 



