title: 深入理解I/O体系结构（一）
tags:
  - linux
  - 操作系统
date: 2017-04-10 11:40:00
---
#### I/O体系结构
虚拟文件系统利用底层函数，调用每个设备的操作，那么这些操作是如何在设备上执行的，操作系统又是如何知道设备的操作是什么的呢？这些是由操作系统决定的。  
我们知道，操作系统的工作，是依赖于数据通路的，它们让信息得以在CPU、RAM、I/O设备之间传递。这些数据通路称为**总线**。这就包括数据总线（PCI、ISA、EISA、SCSI等）、地址总线、控制总线。I/O总线，指的就是用于CPU和I/O设备之间通信的**数据总线**。I/O体系的通用结构如图所示：  

![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%A3%81%E7%9B%98IO/IO%E7%BB%93%E6%9E%84.png?raw=true =600x400)  

那么CPU是如何通过I/O总线和I/O设备交互呢？这首先得从内存和外设的编址方式说起。第一种是“独立编址”，也就是内存和外设分开编址，I/O端口有独立的地址空间，这也被称为**I/O映射方式**。每个连接到I/O总线上的设备，都分配了自己的I/O地址集（在I/O地址空间中），它被称为I/O端口。`in`、`out`等指令用语CPU对I/O端口进行读写。在执行其中一条指令时，CPU使用地址总线选择所请求的I/O端口，使用数据总线在CPU寄存器和端口之间传送数据。这种方式编码逻辑清晰，速度快，但空间有限。    
第二种是“统一编址”，也被称为**内存映射方式**，I/O端口还可以被映射到内存地址空间（这也正是现代硬件设备倾向于使用的方式），这样CPU就可以通过对内存操作的指令，来访问I/O设备，并且和DMA结合起来。这种方式更加统一，易于使用。它实际上使用了`ioremap()`。**自从PCI总线出现后，不论采用I/O映射还是内存映射方式，都需要将I/O端口映射到内存地址空间**。  

每个I/O设备的I/O端口都是一组寄存器：控制寄存器、状态寄存器、输入寄存器和输出寄存器。内核会纪录分配给每个硬件设备的I/O端口。    

#### 设备驱动程序模型
在内核中，设备不仅仅需要完成相应的操作，还要对其电源管理、资源分配、生命周期等等行为进行统一的管理。因此，内核建立了一个统一的设备模型，提取设备操作的共同属性，进行抽象，并且为添加设备、安装驱动提供统一的接口。它们本身并不代表具体的对象，只是用来维持对象间的层次关系。   
这里首先要提的是**sysfs**文件系统。和/proc类似，安装于/sys目录，其目的是表现出设备驱动程序模型间的层次关系。在驱动程序模型当中，有三种重要的数据结构（旧版本），自上到下分别是`subsystem`、`kset`、`kobject`。如果要理解这个模型中，每个数据结构的作用，就必须理解它们和操作系统中的什么东西相对应。它们均对应着**/sys中的目录**。`kobject`是这个对象模型中，所有对象的基类。`kset`本身首先是一个`kobject`，而它又承担着一个`kobject`容器的作用，它把`kobject`组织成有序的目录；subsys则是更高的一层抽象，它本身首先是一个`kset`。驱动、总线、设备都能够用设备驱动程序模型中的对象表示。   

#### 设备驱动程序模型中的组件
设备驱动程序模型建立在几个基本数据结构之上，它们描述了总线、设备、设备驱动器等等。这里，我们来看看它们的数据结构。首先，`device`用来描述设备驱动程序模型中的设备。  

	struct device {
	struct device		*parent;//父设备
	struct kobject kobj;		//对应的kobject
	const char		*init_name; //初始化名
	
	const struct device_type *type;//设备的类型

	struct mutex		mutex;	//驱动的互斥量

	struct bus_type	*bus;		//设备在什么类型的总线
	struct device_driver *driver;	//设备的驱动
	
	void		*driver_data;	//驱动私有数据指针
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;
	//dma相关变量
	u64		*dma_mask;			
	u64		coherent_dma_mask;	
	unsigned long	dma_pfn_offset;
	struct device_dma_parameters *dma_parms;
	struct list_head	dma_pools;	
	struct dma_coherent_mem	*dma_mem; 

	dev_t			devt;	//dev目录下的描述符
	u32			id;	

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct klist_node	knode_class;
	struct class		*class;	//类
	
	void	(*release)(struct device *dev);//释放设备描述符时候的回调函数
	};
	        
首先，可以看到`device`中包含有一个`kobject`，还包含有它相关驱动对象。所有的device对象，全部收集在devices_kset中，它对应着/sys/devices中。设备的引用计数则是由kobject来完成的。device还会被嵌入到一个更大的描述符中，例如`pci_dev`，它除了包含`dev`之外，还有PCI所特有的一些数据结构。`device_add`完成了新的device的添加工作。我注意到，`error = bus_add_device(dev); `，也就是说device的添加会把它和bus关联起来。  

-------

再来看看驱动程序的结构。其数据结构为`device_driver`。相对于设备的数据结构来说，它相对较为简单：对于每个设备驱动，都有几个通用的方法，分别用语处理热插拔、即插即用、电源管理、探查设备等。同样，驱动也会被嵌入到一个更大的描述符中，例如`pci_driver`。  

	struct device_driver {
	const char		*name;		//驱动名
	struct bus_type		*bus;	//总线描述符

	struct module		*owner;
	const char		*mod_name;	//模块名

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);		//探测设备
	int (*remove) (struct device *dev);	//移除设备
	void (*shutdown) (struct device *dev);	//断电方法
	int (*suspend) (struct device *dev, pm_message_t state);//低功率
	int (*resume) (struct device *dev);	//恢复方法
	const struct attribute_group **groups;

	const struct dev_pm_ops *pm;	//电源管理的操作

	struct driver_private *p;
	};

为什么这里没有`kobject`呢？它实际上保存在了`driver_private`当中，这个结构和device_driver是双向链接的。  

	struct driver_private {
	struct kobject kobj;
	struct klist klist_devices;
	struct klist_node knode_bus;
	struct module_kobject *mkobj;
	struct device_driver *driver;
	};  
	
driver的添加，通过调用`driver_register()`来完成，它同样包含一个函数：`bus_add_driver()`，也就是将driver添加到某个bus。  
	
------
	
再来看看总线的结构。bus是连接device和driver的桥梁，bus中的很多代码，都是为了让device找到driver来设计的。总线的数据结构如下：  

	struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	struct device_attribute	*dev_attrs;	/* use dev_groups instead */
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;
	//检查驱动是否支持特定设备
	int (*match)(struct device *dev, struct device_driver *drv);	//回调事件，在kobject状态改变时调用
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	//探测设备
	int (*probe)(struct device *dev);
	//从总线移除设备
	int (*remove)(struct device *dev);
	//掉电
	void (*shutdown)(struct device *dev);	

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	//改变电源状态和恢复
	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;
	};

同样，总线也有一个`subsys_private`，它保存了kobject。`but_type`中定义了一系列的方法。例如，当内核检查一个给定的设备是否可以由给定的驱动程序处理时，就会执行`match`方法。可以用`bus_for_each_drv()`和`bus_for_each_dev()`函数分别循环扫描drivers和device两个链表中的所有元素，来进行match。   

#### 设备文件
设备驱动程序使得硬件设备，能以特定方式，响应控制设备的编程接口（一组规范的VFS函数，open，read，lseek，ioctl等），这些函数都是由驱动程序来具体实现的。在设备文件上发出的系统调用，都会由内核转化为对应的设备驱动程序函数，因此设备驱动必须被注册，也即构造一个`device_driver`，并且加入到设备驱动程序模型中。在注册时，内核会试图进行一次match。注意，这个注册的过程基本`driver_register`通常不会在驱动中直接调用，但我们但驱动通常都会间接的调用它来完成注册。
遵循linux“一切皆文件”的原则，I/O设备同样可以当作设备文件来处理，它和磁盘上的普通文件的交互方式一样，例如都可以通过`write()`系统调用写入数据。设备文件可以通过`mknod()`节点来创建，它们保存在/dev/目录下。  
linux当中，硬件设备可以花费为两种：字符设备和块设备。其中，块设备指的是可以随机访问的设备，例如硬盘、软盘等；而字符设备则指的是声卡、键盘这样的设备。设备文件同样在VFS当中，但它的索引节点没有指向磁盘数据的指针，相反地它对应一个标识符（包含一个主设备号和一个次设备号）。VFS会在设备文件打开时，改变一个设备文件的缺省文件操作，让它去调用和设备相关的操作。

#### 字符设备驱动程序
这里我们以字符设备驱动程序为例。首先，字符设备的驱动，在linux系统中，是以`cdev`结构来表示的：

	struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;	//包括的inode的devices
	dev_t dev;
	unsigned int count;
	};

现在让我们回顾一下inode的数据结构：

	struct inode {
		...
		union {
		struct pipe_inode_info	*i_pipe;
		struct block_device	*i_bdev;
		struct cdev		*i_cdev;
	};
		...
	}

我们看到了`cdev`指针的影子，可见cdev和inode确实是直接相关的。要实现驱动，首先就要对cdev进行初始化，注册字符设备。驱动的安装，首先要分配cdev结构体、申请设备号并初始化cdev。注意，驱动程序是如何和刚才我们所说的设备驱动模型建立联系的呢？实际上在初始化cdev的时候，就调用了`kobject_init()`，在模型中添加了一个`kobject`。  
随后，驱动要注册cdev，也即调用`cdev_add()`函数。这个工作主要是由`kobj_map()`来实现的，它是一个数组。对于每一类设备，都有一个全局变量，例如字符设备的`cdev_map`，块设备的`bdev_map`。最后要进行硬件资源的初始化。  

	int cdev_add(struct cdev *p, dev_t dev, unsigned count)
	{
		int error;
	
		p->dev = dev;
		p->count = count;
	
		error = kobj_map(cdev_map, dev, count, NULL,
				 exact_match, exact_lock, p);
		if (error)
			return error;
	
		kobject_get(p->kobj.parent);
	
		return 0;
	}  

kobj_map的结构如下，它用来保存设备号和kobject的对应关系
	
	struct kobj_map {
		struct probe {
			struct probe *next;
			dev_t dev;
			unsigned long range;
			struct module *owner;
			kobj_probe_t *get;
			int (*lock)(dev_t, void *);
			void *data;
		} *probes[255];
		struct mutex *lock;
	};

![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%A3%81%E7%9B%98IO/kobjmap.jpg?raw=true)  

不过到现在为止，我们都还没有说明，程序在访问字符设备时，是如何去调用正确的方法的。我们曾提到过，`open()`系统调用会改变字符文件对象的f_op字段，将默认文件操作替换为驱动的操作。在字符设备文件创建时，会调用`init_special_inode`来进行索引节点对象的初始化。其inode的操作(def_chr_fops)只包含一个默认的文件打开操作，也即`chrdev_open`。它会根据inode，首先利用`cdev_map`，找到对应的kobject，随后再进一步找到cdev，然后从中提取出文件操作的函数`fops`，并把它填充到file当中去。    

	static int chrdev_open(struct inode *inode, struct file *filp)
	{
		const struct file_operations *fops;
		struct cdev *p;
		struct cdev *new = NULL;
		int ret = 0;
	
		spin_lock(&cdev_lock);
		p = inode->i_cdev;
		if (!p) {
			struct kobject *kobj;
			int idx;
			spin_unlock(&cdev_lock);
			kobj = kobj_lookup(cdev_map, inode->i_rdev, &idx);//获取对应的kobject
			if (!kobj)
				return -ENXIO;
			new = container_of(kobj, struct cdev, kobj);
			spin_lock(&cdev_lock);
			/* Check i_cdev again in case somebody beat us to it while
			   we dropped the lock. */
			p = inode->i_cdev;
			if (!p) {
				inode->i_cdev = p = new;
				list_add(&inode->i_devices, &p->list);//将device加入到cdev的list中去
				new = NULL;
			} else if (!cdev_get(p))
				ret = -ENXIO;
		} else if (!cdev_get(p))
			ret = -ENXIO;
		spin_unlock(&cdev_lock);
		cdev_put(new);
		if (ret)
			return ret;
	
		ret = -ENXIO;
		fops = fops_get(p->ops)
		if (!fops)
			goto out_cdev_put;
	
		replace_fops(filp, fops);//替换file当中的fops  	
		return ret;
	}

这里很奇怪的是，我们并没有看到类似前面提到的`driver_register()`、`device_register()`这样的函数。实际上这里并没有真正创建一个设备，而只是说创建了一个接口，所以有这样一个这个问题：[为什么cdev_add没有产生设备节点？](http://blog.csdn.net/luckywang1103/article/details/47860805)对于这个问题，我们应该理解为`cdev`和`driver/device`二者是配套工作的，cdev用来和用户交互，而device则是内核中的结构。  
另一个问题是，在上面的过程中，似乎没有提及设备文件的创建。实际上，作为一个rookie，那么设备文件常常是用`mknod`命令手动创建的。当然，linux自然也提供了自动创建的借口，那就是利用udev来实现，调用`device_create()`函数。  
当然，这个例子只是为了说明，操作系统的驱动程序是如何工作的，为什么对I/O设备的操作可以抽象成对设备文件的操作，程序在操作I/O文件时，是如何使用正确的操作的。  
