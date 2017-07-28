title: Hack GCC，在编译时构造新的函数
tags:
  - 编译安全
  - GCC
date: 2017-06-09 11:40:00
---
#### 怎么做和为什么？  
首先对于GCC来说，**添加一个新的函数**是完全可以实现的，但我并没有看到任何相关的资料。我给gcc的mailing list发了封邮件，但没人给出回答。倒是有人私下给我发了封邮件，表示他对这个做法很好奇，希望我做出来之后能和大家分享一下方法。  
看来求人终究不如求自己，我看了好几天代码，终于能够在编译的过程中添加函数了。  
至于为什么要这么做，我认为如果编译时**插桩的代码是大量重复的**，那么去构造一个函数，并且插桩对这个函数的调用，是能够减少最后编译的可执行文件的体积的。  


#### 从函数的结构出发  
要构造一个新的函数，不妨先看看函数包含有哪些部分：[这里](https://gcc.gnu.org/onlinedocs/gccint/Function-Basics.html#Function-Basics)说明了函数的四个核心部分：函数名、参数、返回值、函数体。那么生成一个函数，也就要构造这些部分；并且对于函数的某些property，也应该进行相应的设置。  

有兴趣的朋友可以参考这几个函数：tree-parloops.c中的`create_loop_fn()`，omp-low.c当中的`create_omp_child_function()`，以及cgraphunit中的`init_lowered_empty_function`。  

#### 函数的构造  
总的来说，如果要构造一个函数，需要完成这些步骤：

1. 构造函数的type
2. 构造函数的name
3. 构造函数的declaration
4. 构造函数体
4. 创建函数的cgraph_node
5. 设置函数的attributes
6. 构造函数的result
7. 构造函数的paramater
8. 添加新的函数

##### type&name  
build_function_type(tree return_type, ...)`用来构造一个function的type。return_type是函数返回的类型。可变参数是用来设置额外的参数类型的，参数类型必须由NULL_TREE来终结。  

那么这个这些type是从哪儿来的呢？`build_complex_type`用来生成一系列组合的type，比如`unsigned long`，而其他的基本类型实际上已经定义好了，例如`void_type_node`，`ptr_type_node`等。可见，数据的类型，就是通过这种方式来定义的，对于用户自己定义的数据类型，同样会通过build_complex_type来处理。  
	
`get_identifier`会返回一个标识符id，如果name能够找到，它会返回原有值，如果找不到，就会创建一个新的表项并返回。随后`decl`会实际去构造一个声明。`build_decl`的内部如下： 

	tree
	build_decl_stat(location_t loc, enum tree_code code, tree name, tree type MEM_STAT_DECL){
	tree t;
	t = make_node_stat(code PASS_MEM_STAT);

##### declaration

`build_fn_decl(const char *name, tree type)`函数用来构造并且返回一个函数的声明。这里详细看看这个函数的内部：

	tree build_fn_decl(const char *name, tree type){
		tree id = get_identifier(name);
		tree decl = build_decl(input_location, FUNCTION_DECL, id, type);
		DECL_EXTERNAL(decl) = 1;
		TREE_PUBLIC(decl) = 1;
		DECL_ARTIFICIAL(decl) = 1;
		TREE_NOTHROW(decl) = 1;
	}
	
##### 函数体  
然而，这里仅仅是构造了一个声明，对于一个函数来说，不仅应该有它的声明，还应该有其函数体。`allocate_struct_function(tree fndecl, bool abstract_p)`会为一个声明生成一个function structure，并且把它设置为默认内容。`abstract_p`表示这个函数是一个抽象的(类似于函数模版)。  
在创建函数体之后，还要创建函数体的基本块：赋予函数最基本的结构，并且正确的连接ENTRY_BLOCK和EXIT_BLOCK。这个步骤可以通过`create_basic_block`来完成。  

##### cgraph_node  
`cgraph_node`定义在cgraph.h中，它是一个`symtab_node`的子类，也就是符号表的子类。它包含有函数声明的调用关系（这也是它的名字call graph node的由来），也就是`callee`和`caller`。  
`cgraph_node::get_create()`能够创建一个函数的call graph。这个函数会为某个函数声明，找到对应的的call graph。如果不存在这样的call graph，就为它构造一个call graph。  
这个过程还将函数加入符号表`symtab`，这是极其重要的。函数间调用的转换，也依赖于这个符号表。  

##### attributes  
`tree_cons`用来创建TREE_LIST结点。它根据purpose、value（实际上也就是tree_list的两个元素），创建一个结点，并设置这个node的tree_chain。对于函数来说，它的attributes通常是保存在一个TREE_LIST当中的。  

##### paramaters&result  
对一个函数来说，它的参数和返回值，同样都是利用tree结构来表示的。对于一个函数，参数是可以不设置的，但返回值不行。这两者可以通过这样的形式来构造声明：  

	t = build_decl(UNKNOWN_LOCATION, RESULT_DECL, NULL_TREE, void_type_node);
	DECL_CONTEXT(t) = fndecl;
	DECL_RESULT(fndecl) = t;
	t = build_decl(UNKNOWN_LOCATION, PARAM_DECL, NULL_TREE, void_type_node);
	DECL_CONTEXT(t) = fndecl;
	DECL_ARGUMENTS(fndecl) = t;
	
当然，还要设置这它们和函数之间的上下文关系。  

##### 添加新的函数  
cgraphunit.c当中，提供了一个函数`cgraph_node::add_new_function`。在编译过程中，编译器会插入一些新的函数，这个函数将新的函数添加到一个数组`cgraph_new_nodes`当中去。这个函数还会对所添加的函数，执行之前所有“错过”的pass。因此不论在任何时间点插入函数，其所经历的优化过程其实并没有减少；值得一提的是，这个函数只允许插入GIMPLE形式的函数（high，low，ssa），所以我想[这个问题](https://gcc.gnu.org/ml/gcc/2003-10/msg00330.html)目前应该是不能实现的；因为如果直接写RTL，可能会无法恢复成GIMPLE，也就没法去进行某些优化了。  
`cgraph_new_nodes`中所保存的函数，会被`process_new_functions`处理，将它们添加到call graph当中去。现在这些函数就像原有的函数一样了。  