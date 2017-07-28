### 重要数据结构
在与enclave操作相关的主要数据结构有两个，分别是SGX Enclave Control Structure（SECS)和Thread Control Structure(TCS)。每个enclave都有一个SECS，SECS包含enclave的元数据，不能被软件直接访问，secs还保存有measurement值MRENCLAVE。  
而每个enclave都包含有一个或多个TCS。它包含有进出enclave是，用来储存、恢复特定信息的元数据。其中的FLAGS能被软件访问(debug)。SECS是在ECREATE时产生的，而TCS在EADD时产生(SGX2中还有别的指令。)
#### Version Array(VA)
VA是一类特殊的EPC页(它还是属于EPC页)，称为Version Array，每个VA页包含512个slots，它们包含一个8bytes的version number(被换出的页)，当EPC页被换出时，软件需要选择一个空的VA page中的slot，它接受这个页的version number。在这个页被重新加载时，必须有一个VA slot保存了它的version number。  
#### SECS
SECS需要4K对齐。它占据一个页，主要包含这些部分：

field | description 
------------ | -------------
SIZE | enclave的大小
BASEADDR | enclave的线性基地址
SSAFRMESIZE |页中一个SSA frame的大小
MISCSELECT |指定储存到MISC域的extended features
ATTRIBUTES | Enclave属性
MRENCLAVE | measurement:build
MRSIGNER | measurement:public key
ISVPRIODID | product ID
ISVSVN | SVN
RESERVED | 包含EID，PAD
#### TCS
enclave中每一个运行的thread都和一个TCS相关联。它同样占据一个页，主要包含这些部分：

field | description 
------------ | -------------
STAGE | thread enclave的执行状态。0表示TCS允许enclave entry，1表示处理器正在运行这个TCS点上下文。
FLAGS | thread的运行flags
OSSA |
CSSA |
OENTRY|
AEP |
OFSBASGX |
OGSBASGX |
FSLIMIT |
GSLIMIT |
RESERVED |
#### SSA
STATE SAVE AREA(SSA)FRAME。当一个AEX出现时(Asynchronous Enclave Exit)。当enclave中运行时，出现AEX时，architectural state就被保存在当前线程的SSA frame当中，而TCS保存了它的指针。SSA frame也是页对齐的。  
field | description 
------------ | -------------
XSAVE | 
Pad |
MISC | EXINFO & future extension
GPRSGX |
### EPC
EPC是用来保存enclave页的结构。硬件在检查页的访问权限之后，会检查当前EPC页是否能够被访问。EPC是由4KB大小的EPC页来构成的。  
EPC当中的页可以分为两种：valid(属于一个enclave实例)，每个enclave实例都有一个页用来保存SECS。每个EPC页的元数据都被保存在EPCM中。EPC是有privileged software来管理的，也可以在BIOS中被配置。在EPC内存是系统DRAM一部分的情况下，它的内容被加密引擎保护。
EPCM是处理器用来追踪EPC内容的数据结构，EPC中的每个页，都对应EPCM中的一个entry。  
### enclave指令
enlave的指令包含两种：ENCLS(ring0)和ENCLU(ring3)。每个子函数利用EAX来作为索引号，他们不使用ModR/M的编码形式。
在支持SGX的处理器上，IA32_FEATURE_CONTROL提供了SGX_ENABLE(bit 18)。这个寄存器要由BIOS来设置。
### enclave执行环境
enclave被创建时，有一段线性地址ELRANGE。在ELRANGE之内的线性地址必须映射到EPC页中，否则当enclave尝试访问时，就会产生错误。
SGX采用基于页对访问控制。对于EPC的访问，也可以分为明确的访问和潜在的访问两种。潜在访问指的是物理地址被处理器缓存的潜在访问，它们不会出发访问控制的异常。这个地址在一系列检查之后（包括EPT）才会被使用。
### 驱动
linux驱动的入口函数：module_init(func)，也即把func作为这个驱动的入口。  
在isgx_main中，module_init(isgx_init)，那么sgx驱动的入口也即isgx_init。  
在isgx_init中，进一步调用isgx_init_platform()，它决定了EPC的范围，随后module_init也打印了EPC的范围。  
isgx_init_platform，根据硬件的情况，对EPC进行了初始化。这里EPC指的是一块物理的内存空间，它专门给sgx使用。这个EPC应该在操作系统初始化的时候就完成了，这里CPUID只是去查询它，来获取这个EPC的范围。  

	isgx_epc_mem = ioremap_cache(isgx_epc_base, isgx_epc_size)

这里，ioremap_cache将这一段物理内存，映射到内核的虚拟地址空间中去，它调用的是ioremap_caller这个内核函数。

	isgx_page_cache_init(isgx_epc_base, isgx_epc_size)
	
随后isgx_page_cache_init，它在这一段虚拟地址空间中，按页来进行申请，并且把它们加入到isgx_free_list中去。

	//生成一个workqueue，用来add page
	isgx_add_page_wq = alloc_workqueue()
	//注册isgx设备
	misc_register()  
	//注册通知项
	register_pm_notifier()  

最后，isgx_enable()启用sgx，设置CPU中的每个核的CR4。

### 设备：isgx_dev

	struct miscdevice isgx_dev = {
	.name = "isgx",
	.fops = &isgx_fops,
	.mode = S_IRUGO | S_IWUGO,
	}

这里的isgx_fops，定义了isgx设备相关的一系列操作，其定义如下：

	struct_operations isgx_fops = {
	.owner = THIS_MODULE,
	.unlocked_ioctl = isgx_ioctl,
	.compat_ioctl = isgx_compat_ioctl,
	.mmap = isgx_mmap,
	.get_unmapped_area = isgx_get_unmapped_area,
	}
	
这样操作就和对应的isgx函数关联上。  

### isgx_ioctl
isgx_ioctl中，对于enclave的创建、页的添加等，定义了这些操作：  
isgx_ioctl_enclave_create  
isgx_ioctl_enclave_add_page  
isgx_ioctl_enclave_init  
isgx_ioctl_enclave_destory  
isgx_ioctl把handler设置为这几个函数中的某一个，来进行对应的处理。  
这些操作都是针对enclave来进行的。isgx_ioctl_enclave_create会获取参数，确定需要映射的secure数据范围，使用vm_mmap来进行映射。这里除了secs->base，还要进行一次backing的vm_mmap映射。enclave的大小似乎是由用户来申请的(存放在arg中)。这个函数中
	
	__ecreate((void *)&pginfo, secs_vaddr);

调用了ECREATE来创建这个环境。

isgx_ioctl_enclave_add_page中，EADD和EEXTEND是一个迭代的过程。在SECS被创建之后，enclave页就被通过EADD加入到enclave中去，将一个free到EPC页变成PT_REG或PT_TCS页。处理器会自动根据页的类型来进行设置对应的EPCM项。EADD在SECS中记录EPCM的信息，并且复制4KB的数据到生成的EPC页中去。**操作系统来负责free EPC page的选择**，并且还要提供一系列相关的信息，在添加之后，它还需要完成一个EEXTEND的验证工作，对一整个4KB的页，必须执行16次EEXTEND。  

isgx_ioctl_enclave_init在完成了add和measure之后，就要利用EINIT来初始化。在初始化之后，EADD和EEXTEND就被禁止了。这个步骤主要构建**enclave identity**和**sealing identity**。它会调用__isgx_enclave_init再进一步调用__einit。

isgx_ioctl_enclave_destory

### enclave的创建
1. 应用递交内容，利用API创建服务  
2. ECREATE创建环境，分配ELRANGE，初始化SECS  
3. EADD将EPC页交给enclave，EEXTEND来进行检查  
4. EINIT，finalize measurement，在EINIT完成后，enclave才能开始执行。  

### enclave的进出
EENTER：在程序控制下，进入enclave，软件要提供要进入enclave的TCS的地址，TCS保存转移控制的位置，以及SSA的指针。每个TCS只能被一个线程使用，多个TCS就能支持多线程。软件必须提供给EENTER相关的AEP参数，它是enclave的地址，是一个异常处理。通常它会包含ERESUME来返回到enclave当中去。  
EEXIT：在程序控制下离开enclave，接收一个enclave之外的地址，要清除寄存器，EEXIT会返回一个AEP，供回到enclave的时候用。  

### AEX
同步、异步的事件，例如异常、中断等，有可能在enclave的执行过程中出现，这些成为Enclave exiting events。在EEE发生时，处理器的状态会保存在enclave的SSA中。大多数情况下，AEP都会入栈，在这里 可以执行ERESUME来重新进入enclave并且恢复执行。  
AEX处理完后，处理器不处于enclave中，接下来发生的中断也会按照正常在enclave外部的方式去处理。只有在ERESUME之后才会返回enclave。在ERESUME时，如果造成异常的事件没有处理，那么这个事件又会被触发，SGX为开发者提供了在内部处理异常的方法，软件能够在调用enclave内部的异常处理函数，而内部的handler能够通过SSA来读取异常的信息并进行处理。  

### Measurement
在enclave创建的过程中，有两种measurement：MRENCLAVE和MRSIGNER。其中，MRENCLAVE表示enclave的内容和创建过程；而MRSIGNER表示对enclave进行签名的实体。MRENCLAVE用来识别装载入enclave的code和数据，在init前，每一次对enclave内容的操作都会修改MRENCLAVE的值。每个enclave都用一个RSA密钥来签名，储存在SIGSTRUCT当中，MRSIGNER是signer的公钥。  

### EPC管理
EPC是特殊实现的，能够通过CPUID指令来获取它的值，通常是在系统启动的时候，由BIOS来配置的。EPC的大小、数量都是由BIOS的设置来决定的。EPC是有限的资源，SGX1提供了**leaf function**来实现EPC页的合理SWAP。  
需要被换出的页可以用EBLOCK指令变成BLOCKED状态。在换出前，EPCM还要先进行ETRACK来保证没有错误的映射关系留下。EWB完成加密，写到enclave外部的操作。对于EPC中的每一个需要驱逐的页，首先在VA页中选一块空的slot，再移除enlave上下文映射表中的映射信息(页表和EPT表)；最后对每个目标页执行EBLOCK。对于这些页，执行ETRACK，保证映射信息被清除；随后对于所有运行所选择的enclaves的处理器，进行一次内部中断，清除缓存的映射信息；而每一个EWB驱逐的页，都需要一个指向EPC的指针。VA slot，以及一个4kb的buffer来存储页内容，和一个128byte的buffer来保存metadata。  
EWB指令，需要几个参数，分别是pageinfo的地址，EPC页的地址，以及VA slot的地址。它将EPC中的一个页复制到普通的内存，这个页会被加密。
在linux的驱动中，这个工作在evict_cluster中完成。这个函数被kisgxswapd和isgx_alloc_epc_page调用。kisgxswapd是一个独立的内核线程，它使用一个等待队列，这个线程在isgx_page_cache_init的时候就初始化了，而另一次则是在isgx_resume中，它发生在power_event相关的地方(power的恢复？重新激活了isgx)isgx_alloc_epc_page_fast中，会唤醒这个队列(如果页的数量不够？)。来进行一个swap的过程，提供足够的页。
kisgxwapd会调用isolate_cluster，随后调用isolate_cluster会调用isolate_enclave，再进一步向下调用isolate_ctx。  
再kisgxwapd中，首先是isolate_cluster,然后再evict_cluster。可见这里isolate_cluster起到一个筛选的作用。在isolate_cluster中，首先有：

	//nr_to_scan一直是一个常数  
	enclave = isolate_enclave(nr_to_scan)  
	
在isolate_enclave中，又有

	ctx = isolate_ctx(nr_to_scan);
	
而isolate_ctx中，是一个循环：

	for(i = 0, ctx = NULL;i < nr_to_scan; i++, ctx = NULL)
	//取isgx_tgid_ctx_list中的第一个entry
	ctx = list_first_entry(&isgx_tgid_ctx_list, struct isgx_tgid_ctx, list)


这里需要了解几个变量：  
isgx_nr_free_epc_pages表示的是free页的数量。  
isgx_nr_low_epc_pages表示的是  
isgx_nr_high_epc_pages表示的是  
SGX1同样提供两个指令，用来将被换出的页，重新加载到enclave当中：ELDU和ELDB。随后要在enclave的映射表(页表和EPT)中创建映射关系，使应用能够访问这个页。  
特殊的，换出一个SECS页需要保证enclave中的所有页已经被换出；换出一个VA页，需要在另一个VA页中，指定一个slot来提供相关的versioning信息。  
### ctx相关
isgx_tgid_ctx_list是一个全局变量，它是一个list_head。它存放的是什么？

	list_add(&ctx->list, &isgx_tgid_ctx_list)；
	list_add(struct list_head *entry, struct list_head *head):在指定的list_head后面，添加一个entry。
	那么这里就是，把申请的/查询到的ctx->list，加入到isgx_tgid_ctx_list当中去。所以isgx_tgid_ctx_list保存的就是这样一个，所有ctx->list构成的list。

tgid也即thread group id，线程组id，它是主线程的pid，也即进程的PID。  

	struct isgx_tgid_ctx{
	struct pid *tgid;
	atomic_t epc_cnt;
	struct kref refcount;
	struct list_head enclave_list;
	struct list_head list;
	}:
	kref是linux内核的计数器，它由原子操作来进行处理。这里它能够表示一个ctx被多少个enclave使用了。
	
add_tgid_ctx:在create enclave的时候被调用。这个函数首先获取当前的tgid，随后调用find_tgid_ctx_mutex。它会搜索isgx_tgid_ctx_list中的每一项，如果，并且试图找到ctx->tgid与当前tgid相同的ctx；如果这个tgid已经存在了，那么就把enclave->tgid_ctx直接设置为ctx。如果没有找到，则申请一个新的ctx，并且将enclave的ctx设置为这个创建的ctx。**也就是说，一个tgid，对应一个ctx，而一个ctx可能会对应多个enclave。并且要把这个ctx，设置为当前enclave的tgid_ctx。**
在isolate_ctx中，schedule()起到什么作用？  
	
	list_first_entry:从list中取出第一个元素  
	
	list_move_tail:将链表项移动到链表的尾部  
	
### Enclave EXiting Events
EEE是那些会造成控制流转移到enclave外部的事件，而AEX则是相对的，用来完成这个过程的步骤，它保存寄存器的状态在enclave中，并且用synthetic state来加载新的值。  
AEX会根据已经决定好的synthetic状态来加载寄存器值，这些寄存器会入栈到一个由EEE定义的合适的栈。(这时已经是enclave外部了，称为Exit Stack)  
SSA保存了AEX发生时处理器的状态。SSA是以一个栈的形式来表示的，而其栈帧是由TCS和SECS中的某些值来控制的，包括栈帧的大小、SSA slots的数量、当前的SSA slot、SSA slot相对于enclave基地址的偏移量。当AEX发生时，硬件会通过检查TCS.CSSA来选择SSA的栈帧。处理器状态被保存到SSA frame当中，并且加载合成的值，来放置隐私的泄露。  
### isgx_mmap
isgx_mmap主要是完成映射的操作。它主要对vm_area_struct进行操作，将vma的vm_ops设置为isgx_vm_ops。  
其中isgx_vm_ops定义如下： 

	struct isgx_vm_ops = {
	.close = isgx_vma_close,
	.open = isgx_vma_open,
	.fault = isgx_vma_fault,
	.access = isgx_vma_access,
	};  
isgx_vma这一系列操作，都是针对。  

### isgx_unmapped_area
isgx_unmapped_area用来获取一段没有被映射的内存。   

	addr = current->mm->get_unmapped_area(file, addr, 2*len, pgoff, flags); 
	
这里current指的是一个全局变量，它指向当前的task_struct。而current->mm则指向了当前task_struct的mm_struct，也即内存描述符。  
这里get_unmapped_area的函数实际上是isgx_get_unmapped_area，它又进一步调用了linux的get_unmapped_area，确认当前虚拟地址空间有足够的空闲空间。    


