### 最简单的例子
一个内核模块，需要一个**入口**和一个**出口**。通常，这个入口函数是init_module，它在模块被insmoded到内核时调用；而出口函数是cleanup_module，它在模块被rmmod时被调用。不过现在这两个函数的名字可以自己决定了。通常init_module在内核中注册一个handler，或者替换kernel中的某些代码；而cleanup_module会undo init_module的工作，将模块卸载掉。  

	#include <linux/module.h>	/* Needed by all modules */
	#include <linux/kernel.h>	/* Needed for KERN_INFO */

	int init_module(void)
	{
		printk(KERN_INFO "Hello world 1.\n");
		return 0;
	}

	void cleanup_module(void)
	{
		printk(KERN_INFO "Goodbye world 1.\n");  
	}
	
如果想用自定义的函数名称，那么就必须使用module_init(init函数名)和module_exit(exit函数名)两个宏定义，莱
	
### 内核模块编译
这是单个文件内核模块的Makefile。这里，KERNELDIR是现在运行内核的源文件地址，M之后则是模块程序所在的地址。

	ifneq ($(KERNELRELEASE),)
	obj-m:=hellomodule.o
	else
	KERNELDIR:=/lib/modules/$(shell uname -r)/build
	PWD:=$(shell pwd)
	default:
		$(MAKE) -C $(KERNELDIR)  M=$(PWD) modules
	clean:
		rm -rf *.o *.mod.c *.mod.o *.ko
	endif

