title: Hack GCC，改变函数的顺序
tags:
  - 编译安全
  - GCC
date: 2017-05-12 21:40:00
---
#### GCC的函数输出
直观的来看，在`cgraphunit.c`当中，`expand_all_functions()`被用来输出所有需要输出的函数。这里，它首先进行了一次拓扑排序，在`ipa_reverse_postorder()`中完成。这样做的好处是，当一个函数被输出时，它所调用的所有函数都已经被输出了。这个函数还会先把内联的情况给处理掉。  
在`expand_all_functions()`中，随后在允许函数重排列的情况下，利用`qsort`，对其顺序进行了重新排列。其比较函数，是通过比较两个`cgraph_node`代表的函数第一次执行的时间来进行的。 
奇怪的是，在我修改了`expand_all_functions()`中的order之后，输出文件的格式并没有变。这里我出了一点小差错，结果在编译GCC的时候出现了问题，不过这也恰好说明GCC是通过自举来完成编译的。   
于是我继续在上一层函数进行查找，在`symbol_table::compile`当中，实际上通过了了一个`flag_toplevel_reorder`来进行判断。在该值为false时，函数的调用会直接调用：`output_in_order(false)`。在实验之后，我发现gcc默认使用的是`-fno-toplevel-reorder`的方式，也即使用的是`output_in_order(false)`；但它自举的时候使用的是`-ftoplevel-reorder`的方式，也即`output_in_order(ture)`。这很自然：gcc优化自己的代码，这是它自身的需要。

	if(!flag_toplevel_reorder)
		output_in_order(false);
	else{
		output_in_order(true);
		expand_all_functions();
		output_variables();
	}
	
那么不妨看看`output_in_order()`中的代码。

	FOR_EACH_DEFINED_FUNCTION(pf){
		...
		if(no_reorder && !pf->no_reorder)
		continue;
		i = pf->order;
		nodes[i].kind = ORDER_FUNCTION;
		nodes[i].u.f = pf;
	}
	
很明显，如果采用reorder的方式，也即output_in_order(ture)，那么这里会跳过nodes的排序部分，并且进一步执行`expand_all_functions()`。
而如果采用的是no_reorder，那么nodes[i]所对应的也就是通过FOR_EACH_DEFINED_FUNCTION遍历到的第i个函数了，也就是严格按照次序来决定的。
#### 改变函数的输出次序
那么如果想要更改汇编输出的函数次序，对于优化的情况，在`expand_all_functions()`中，在`qsort`对函数进行重排列之后，对order进行修改就可以了。那么GCC自身为什么要改变函数的顺序呢？因为把具有调用关系的函数，放在同一个页当中，是可以有效的减少缺页的情况的。不过现在内存的大小都很大，其实代码页并不会占据很多内存，所以从这个角度上来看，其实差异没有想象中那么大。另一种方式是采用和没有优化的方式相同的处理方法，但是在编译时加上`-fno-toplevel-reorder`选项。    
而假如没有进行优化，那么直接在`output_in_order()`当中，对nodes中的次序进行修改就行了。那么函数的随机化就可以在这些知识的基础之上实现了。  