---
title: GCC插件：注册、编译和使用
tags:
  - 编译安全
  - GCC
date: 2017-05-05 21:40:00
---
#### GCC插件:注册、编译、和使用
GCC 4.5版本后，为用户提供了接口，允许用户去编写额外的*代码优化过程*、*改造代码形式*、*对代码进行分析*。在GCC中，插件是以共享库的形式工作的。在安装好GCC之后，如果想编写插件，首先可以确定API的位置：

	gcc -print-file-name=plugin
	
在gcc-plugin.h中，提供了以下的结构：  
`struct plugin_name_args`包含了GCC调用插件所需要的信息，`plugin_info`是插件的帮助信息，`plugin_gcc_version`包含了插件支持的GCC版本。  

对于一个插件，首先要决定它是在编译的那个阶段执行的，并且通过注册回调的方式，在编译的特殊时间段调用这个插件。GCC中的插件可以在GIMPLE、IPA、RTL等阶段工作。插件必须定义一个对应的数据结构，并且将它的信息，传递给插件框架，由它来回调插件的功能。插件所能够完成多功能，被称为"events"，它们定义在plugin.def文件当中。

每一个gcc插件，都需要有一个初始化模块。对于一个gimple-pass插件，其初始化数据结构为：

	static struct gimple_opt_pass myplugin_pass = 
    {
        .pass.type = GIMPLE_PASS,
        .pass.name = "myplugin", /* For use in the dump file */
    
        /* Predicate (boolean) function that gets executed before your pass.  If the
         * return value is 'true' your pass gets executed, otherwise, the pass is
         * skipped.
         */I
        .pass.gate = myplugin_gate,  /* always returns true, see full code */
        .pass.execute = myplugin_exec, /* Your pass handler/callback */
    };
    
在tree-pass.h中，定义了一系列的数据结构，用来设置编写pass的执行顺序。在gcc/passes.def当中，定义了所有的pass。
[http://gcc-python-plugin.readthedocs.io/en/latest/tables-of-passes.html](http://gcc-python-plugin.readthedocs.io/en/latest/tables-of-passes.html)上标注了GCC所有的passes，以及它们的所属的阶段。我注意到，在pass编写中，是这样标注pass的位置的：

	pass.reference_pass_name = "ssa";
	pass.ref_pass_instance_number = 1;
	pass.pos_op = PASS_POS_INSERT_AFTER;
	
注释中写道，这个plugin将在GCC完成SSA表现形式之后，调用这个plugin，并且在第一个SSA之后调用。从tree-pass.h中，可以看到这样的定义：
	
	struct register_pass_info
	{
		opt_pass *pass;
		const char* reference_pass_name;/*引用新pass的原有pass的名字*/
		int ref_pass_instance_number;/*在原有pass后的特定位置插入新pass*/
		enum pass_positioning_ops pos_ops;/*具体的插入方式，替换，之前，还是之后*/
	}
在pass的init过程中，还需要进行回调的注册，把这个info传递给插件框架。

	register_callback("myplugin", PLUGIN_PASS_MANAGER_SETUP, NULL, &pass);
	register_callback("myplugin", PLUGIN_INFO, NULL, &myplugin_info);
	
当然，插件的运行也可以不用通过pass manager来进行hook，而是在某个特定的时间段运行。在`gcc/plugins.text`中，可以看到plugin_event的定义，它包含了一系列段时间点，于是乎在某些特定的时间点调用插件变得很方便，例如before gimplification, after compilation等。不过，如果想要更精确的设置调用的时机，那便还是要利用pass_manager来进行hook了。  

插件的编译与一般的共享库编译并无不同。在老版本的GCC中，插件都是用C语言编写的，然而在我看到的版本(4.9.4)中，却一律变成了C++，看来用面向对象的语言来处理还是更为方便。  

那么在编译好插件之后，就可以在gcc编译时调用它:
	
	gcc -fplugin=./myplugin.so -c -o test test1.c