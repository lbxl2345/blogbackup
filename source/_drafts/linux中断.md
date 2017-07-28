### linux中断
#### IDT初始化
IDT在BIOS启动，处于实模式下时，就被初始化和使用了；但在linux接管之后，IDT就被移到RAM中的另一块位置，并且第二次进行了初始化。在linux当中，变量idt指向了IDT，它包含256个entries，而6byte大小的变量idt_descr保存了IDT的地址，但它只在内核初始化idtr寄存器时使用。  
在内核初始化时，汇编函数setup_idt首先将idt_table的256个entries全部用相同的中断门填充（这些中断门指向ignore_int）。

#### asynchronous page fault
异步的page fault是KVM虚拟化技术中的一种优化手段。kvm利用shadow pages或者EPT这样的方式，来虚拟化guest当中的内存。由于guest中的内存并不总是映射到guest的地址空间的，所以当vcpu试图去访问一个没有映射到guest guest地址空间的页时，KVM就会收到通知，并且由KVM来把这个页映射到guest的地址空间当中，并且让vcpu返回执行。如果这个page是一个换出的页，那么直到这个页被换入为止，vcpu的执行都会保持挂起的状态，这无疑是低效的，因为vcpu在等待页换出之前，是可以去做其他工作的。  
为了克服这种低效的方式，kvm的某个patch中，增加了asynchronous page fault这一机制。在vcpu尝试去访问一个被换出的页时，KVM会向vcpu发送async PF，让vcpu的执行。当vcpu收到asynchronous PF时，它会将产生fault的任务休眠，直到它收到一个“唤醒”中断，转而去执行别的任务。当这个页被换入到host内存当中时，KVM会向vcpu发送唤醒中断，随后guest任务就恢复执行了。  

#### 
lidtl指令是将idt的地址载入ldtr寄存器当中。

early_trap_pf_init  

	x86_init_ops{
		...
		struct x86_init_irqs irqs;
		...
	}
	
	x86_init->irqs->
	
	//平台相关的interrupt setup
	struct x86_init_irqs{
		void(*pre_vector_init)(void);
		void(*intr_init)(void);
		void(*trap_init)(void);
	};

arch/x86/kernel/irqinit.c:x86_init.irqs.intr_init();  
arch/x86/kernel/traps.c :x86_init.irqs.trap_init();  
arch/x86/kernel/kvm.c:x86_init.irqs.trap_init=kvm_apf_trap_init;  
arch/x86/lguest/boot.c:x86_init.irqs.intr_init=lguest_init_IRQ;  
arch/x86/xen/enlighten.c:x86_init.irqs.intr_init=xen_init_IRQ;  
arch/x86/xen/irq.c:x86_init.irqs.intr_intr=xen_init_IRQ;  

arch/x86/kernel/traps.c当中，有函数trap_init，它调用了x86_init_irqs.trap_init()，这函数是和平台相关的interrupt set，也就是说它只包含有一部分的中断，而其余的中断则是在该函数当中逐个设置的。
init/main.c中trap_init这个函数，则是在start_kernel当中调用的。在调用trap_init之后，才会依次调用early_irq_init()、init_IRQ()

early_trap_init当中包含的是x86_32的set_intr_gate(X86_TRAP_PF)，而early_trap_pf_init当中则是x86_64的set_intr_gate(X86_TRAP_PF)，这两个函数都是在__init setup_arch当中被调用的。这一步是正式设置了IDT当中的page_fault_handler。  最后会调用_set_gate，它包含了gate，type，addr，dpl，ist，seg

page_fault只是一个函数指针，它定义在kvm_host.h当中。set_intr_gate是一个宏定义，它会把(void*)trace_##addr的地址赋值给对应的GATE_INTERRUPT。  

宏定义里面#会把参数转化为字符数组，而##表示分隔连接方式，它用来代替空格。所以trace\_\#\#page_fault也就转化成了trace_page_fault。 

而trace_idtentry则是一个宏定义，
	
	.macro trace_idtentry sym do_sym has_error_code:req
	trace_idtentry 
	trace_idtentry page_fault do_page_fault has_erroe_code=1
	
	在entry_64.S当中，定义了：
	idtentry async_page_fault;
	trace_idtentry page_fault;

	而do_page_fault的定义如下：

在kvm当中，有set_inrt_gate(14,async_page_fault)，这也就是说，它将中断从page_fault，修改成了async_page_fault。其对应的处理函数为do_async_page_fault。  
这个函数会首先判断pagefault的原因，默认情况下，它会做trace_do_page_fault，也就是通常情况下的page_fault。而如果是KVM_PV_REASON_PAGE_NOT_PRESENT，说明这个页被host换出了，这也就需要等待；而KVM_PV_REASON_PAGE_READY，则说明这个页准备好了，此时唤醒相应的进程。但目前来说，因为拥有了EPT技术，所以实际上只会使用通常的page_fault处理。

unlikely是指明这个值为false的可能性更大

	do_page_fault(struct pt_regs *regs, unsigned long error_code)
	//首先读取CR2的值，也即faulting address
	unsigned long address = read_cr2();
	//保存当前的上下文信息
	prev_state = exception_enter();
	__do_page_fault(regs, erroe_code, address);
	//恢复当前上下文信息
	exception_exit(prev_state);
	
	__do_page_fault(regs, error_code, address)处理了具体的事务。
	
也就是说__do_page_fault中的所有相关代码/数据都需要移植到隔离区内部。这包含有当前的task_struct \*current，mm_struct\* mm  

缺页异常的详细说明：

缺页地址是否处于kernel空间？这里还需要进行判断，地址位于内核空间，并不代表异常发生于内核空间，可能是用户态访问了内核空间的地址，这里还要判断是否是TLB未更新等情况造成的，或者内核自身存在缺陷。如果处于内核空间，并且处于内核态，就用vmalloc_fault()来解决这个异常。  

而用户空间中的缺页异常可以分为，两种情况；第一种是线性地址处于vma中，但还没有分配物理页；第二种是线性地址不在vma当中，此时需要判断是否因为栈空间消耗完而触发了缺页异常，如果是则在用户空间的栈区域进行拓展，并且分配物理页。否则则按非法访问来处理。

如果是user_mode，首先local_irq_enable()，因为cr2被保存，并且不是vmalloc fault的前提下，本地中断是安全的。如果缺页中断发生于中断或其它atomic上下文中时，则产生异常。这里还要判断缺页是否发生在内核上下文，此时需要查找是否有exception table。如果在用户态或者有exception_table，那么就不是内核异常。 这里首先要根据地址去查找vma。如果找不到vma，那么释放信号量之后，进入__bad_area_nosemaphore处理。如果能找到，并且地址处于vma的有效范围内，那么是正常的缺页异常，那么请求掉页，分配物理内存。（我们要做的东西就处于这种情况，因为肯定是有vma的）这里还有一种情况，就是栈的大小不够用了，这种情况是需要动态拓展栈的大小的；

而对于缺页异常的正常处理，可能的情况有1、请求调页/按需分配；2、COW；3、缺页位于交换分区，需要换入。

__handle_mm_fault进行具体的处理，涉及页表…



一个进程，其进程描述符task_struct，它包含有进程的mm_struct。mm_struct包含有进程地址空间的所有信息。
这其中mm_struct用一个红黑树来保存线性区。那么从mm_struct是如何来查线性区的？mm_struct中包含一个mmap字段，它指向链表中的第一个线性区描述符。由于linux进程可能会有成百上千的线性区，因此，链表的实现就会很低效，所以线性区是存放在红黑树里面的。linux同时使用了链表和红黑树两种手段。红黑树一般用来确定指定地址的线性区，而链表通常在扫描整个线性区集合时使用。
其中  mm_struct定义在linux/mm_types.h当中，其中，mmap是线性区对象的链表头，rb_root是线性区对象的红黑树根。


