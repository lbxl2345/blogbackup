title: 深入理解I/O体系结构（二）
tags:
  - linux
  - 操作系统
date: 2017-04-11 11:40:00
---
#### 块设备的驱动
和字符设备类似，操作系统中的块设备，也是以文件的形式来访问。这里有一个很拗口的问题：磁盘是一个块设备，块设备有一个块设备文件。那么访问块设备文件和访问普通的磁盘上的文件有什么关系呢？  
不论是块设备文件还是普通的文件，它们都是通过VFS来统一访问的。只不过对于一个普通文件，它可能已经在RAM中了（高速缓存机制），因此它的访问可能会直接在RAM中进行；但如果说要修改磁盘上的内容，或者文件内容不在RAM中，则也会间接地，通过块设备文件进行访问。这个驱动模型可以用这样一个图表示：  
![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%A3%81%E7%9B%98IO/%E5%9D%97%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8.png?raw=true)  
这里我们只考虑最底层的情况：内核从块设备读取数据。为了从块设备中读取数据，内核必须知道数据的物理位置，而这正是**映射层**的工作。映射层的工作包括两步：（1）根据文件所在文件系统的块，将文件拆分成块，然后内核能够确定请求数据所在的块号；（2）映射层调用文件系统具体的函数，找到数据在磁盘上的位置，也就是完成文件块号，到逻辑块号的映射关系。  
随后的工作在**通用块层**进行，内核在这一层，启动I/O操作。通常一个I/O操作对应一组连续的块，我们把它称为`bio`，它用来搜集底层需要的信息。  
**I/O调度层**负责根据内核中的各种策略，把待处理的I/O数据传送请求，进行归类。它的作用是把物理介质上相邻的数据请求，进行合并，一并处理。  
最后一层也就是通过块设备的驱动来完成了，它向I/O接口发送适当的命令，从而进行实际的数据传送。

#### 通用块层
通用块层负责处理所有块设备的请求，其核心数据结构就是`bio`。它代表**一次块设备I/O请求**。

	struct bio {
	struct bio		*bi_next;		//请求队列中的下一个bio
	struct block_device	*bi_bdev;	//块设备描述符指针
	unsigned long		bi_flags;	/* status, command, etc */
	unsigned long		bi_rw;		//rw位

	struct bvec_iter	bi_iter;	

	unsigned int		bi_phys_segments;//合并后有多少个段

	unsigned int		bi_seg_front_size;
	unsigned int		bi_seg_back_size;

	atomic_t		bi_remaining;//剩余的bio_vec

	bio_end_io_t		*bi_end_io;//bio结束的回调函数

	void			*bi_private;

	unsigned short		bi_vcnt;	//bio中biovec的数量

	unsigned short		bi_max_vecs;//最多能有多少个

	atomic_t		bi_cnt;		//结构体的使用计数

	struct bio_vec		*bi_io_vec;	//bio_vec数组
	};  
	
在这个数据结构中，还包含了一个`bio_vec`。这是什么意思呢？在linux中，相邻数据块被称为一个段，每个`bio_vec`对应一个内存页中的段。在io操作期间，bio是会一直更新的，其中的`bi_iter`用来在数组中遍历，按每个段来执行下一步的操作。    
![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%A3%81%E7%9B%98IO/biovec.gif?raw=true)  

那么当通用块层收到一个I/O请求操作时，会发生什么呢？首先内核会为这次操作分配`bio`描述符，并对它进行填充。随后通用块层会调用`generic_make_request`，这个函数的作用很明确：它会进行一系列检查和设置，保证bio中的信息是**针对整个磁盘，而不是磁盘分区的**；随后获取这个块设备相关的请求队列q，调用`q->make_request_fn`，把bio插入请求队列中去。  

#### I/O调度层
在块设备上，每个I/O请求操作都是异步处理的，通用块层的请求会被加入块设备的请求队列中，每个块设备都会单独地进行I/O的调度，这样能够有效提高磁盘的性能。  
前面提到，通用块层会调用一个`q->make_request_fn`，向I/O调度程序发送一个请求，该函数会进一步调用`__make_request()`。这个函数的目的，就是把`bio`放进请求队列当中：（1）如果请求队列是空的，就构造一个新的请求插入；（2）如果请求队列不是空的，但是`bio`不能合并（不能合并到某个请求的头和尾），也构造一个新的请求插入；（3）请求队列不是空的，并且`bio`可以合并，就合并到对应的请求中去。注意，bio，请求和请求队列的关系如下：  

	-- request_queue
			|-- request1
					|-- bio0
			|-- request2
					|-- bio1
					|-- bio2
					
而I/O的调度，就是对请求队列进行排序，针对磁盘的特点，降低寻道的次数。这里说说几个常见的算法：
（1）CFQ完全公平队列：默认的调度算法，完全公平排队。每个进程/线程都单独创建一个队列，并且用上面提到的策略进行管理。队列间采用时间片的方式来分配I/O。
（2）Deadline最后期限算法：在电梯调度的基础上，根据读写请求的“最后期限”进行排序，并通过读期限短于写期限来保证写操作不被饿死。  
（3）预期I/O算法：与最后期限类似，但是在读操作时，会预先判断当前的进程是否马上会有读操作，并且优先地进行处理。      
（4）NOOP：适用于固态硬盘，不进行任何优化。  
 
 总而言之，I/O调度层的作用，就是把请求的队列重新排序，并逐个交给块设备驱动程序进行处理。

#### 块设备驱动程序
I/O调度层排序好的请求，会由块设备的驱动程序来处理。同样，块设备也遵循着我们前面提到的驱动程序模型：块设备对应一个`device`，而驱动程序对应了一个`device_driver`。对于块设备来说，驱动程序也要通过`register_blkdev()`注册一个设备号。随后，驱动程序要初始化`gendisk`描述符，以及它所包含的设备操作表`fops`。在此之后，是“请求队列”的初始化，以及中断程序的设置：要为设备注册IRQ线。最后要把磁盘注册到内核（`add_disk`）,并把它激活。  
当一个块设备文件被`open()`时，内核同样也要为它初始化操作。对于块设备来说，其默认的文件操作如下：

	const struct file_operations def_blk_fops = {
	.open		= blkdev_open,
	.release	= blkdev_close,
	.llseek		= block_llseek,
	.read		= new_sync_read,
	.write		= new_sync_write,
	.read_iter	= blkdev_read_iter,
	.write_iter	= blkdev_write_iter,
	.mmap		= generic_file_mmap,
	.fsync		= blkdev_fsync,
	.unlocked_ioctl	= block_ioctl,
	#ifdef CONFIG_COMPAT
	.compat_ioctl	= compat_blkdev_ioctl,
	#endif
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
	};

`dentry_open()`方法会调用`blkdev_open()`。它（1）首先会获取块设备的描述符：如果块设备已经打开，则可以通过inode->i_bdev直接获取，否则则需要根据设备号去查找块设备描述符。（2）获取块设备相关的`gendisk`地址，`get_gendisk`是通过设备号来找到gendisk的。（3）如果是第一次打开块设备，则要根据它是整盘还是分区，进行相应的设置和初始化。（4）如果不是第一次打开，只需要按需要执行自定义的`open()`函数就行了。  

#### 补充：I/O的监控方式
（1）轮询：CPU重复检查设备的状态寄存器，直到寄存器的值表明I/O操作已经完成了。  
（2）中断：设备发出中断信号，告知I/O操作已经完成了，数据放在对应的端口，当数据缓冲满了时，由CPU去取，CPU需要控制数据传输的过程。  
（3）DMA：由CPU的DMA电路来辅助数据的传输，CPU不需要参与内存和IO之间的传输过程，只需要通过DMA的中断来获取信息。DMA能够在所有数据处理完时才通知CPU处理。  