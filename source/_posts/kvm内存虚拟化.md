title: EPT缺页异常源码分析
tags:
  - kvm
  - 虚拟化
date: 2016-06-27 11:40:00
---
#### EPT Violation  
与操作系统当中的缺页中断类似，EPT中的缺页被称为“EPT violation”。当GPA->HPA的映射不存在时，就会触发VM-EXIT，由KVM来捕获异常，并进行异常处理。  

1 vmx.c当中，`handle_ept_violation`是EPT缺页中断的入口，其参数只有一个，`struct kvm_vcpu *vcpu`。它从VMCS中读取数据，判断OS能否解决这个异常；这里，提取了几个关键的值：

	//引发异常的客户机地址&页框号
	gpa = vmcs_read64(GUEST_PHYSICAL_ADDRESS);
	gfn = gpa >> PAGE_SHIFT;
	
	//发生异常的状态信息，被转化为error_code
	exit_qualification = vmcs_read1(EXIT_QUALIFICATION);  
	
2 随后，该函数进一步调用了`kvm_mmu_page_fault(struct kvm_vcpu *vcpu, gva_t cr2, u32 error_code, void *insn, int insn_len)`  

其中，cr2寄存器存放的为实际引起缺页异常的线性地址，不论是shadow page还是EPT模式，都是通过这个函数来进行处理，但在下一步中，实际调用的函数就不同了。在EPT模式开启的情况下，它会调用`tdp_page_fault`。
   
3 `static int tdp_page_fault(struct kvm_vpu *vcpu, gva_t gpa, u32 error_code, bool prefualt)`
	
这个函数位于mmu.c当中，它完成了具体的EPT建立工作。这个函数又有两个关键部分，一个是`try_async_pf`，它完成了从gfn获得pfn的过程；另一个是__direct_map，也就是根据gfn和pfn的对应关系，建立EPT表项的过程。  
	
4 `try_async_pf`首先获取gfn对应的memslot结构，随后调用`__gfn_to_pfn_memslot`，利用memslot来获取pfn。 

memslot的数据结构如下：
	
	struct kvm_memory_slot {
	gfn_t base_gfn;					//对应虚拟机页框的地址
	unsigned long npages;
	unsigned long *dirty_bitmap;
	struct kvm_arch_memory_slot arch;
	unsigned long userspace_addr;	//对应HVA地址
	u32 flags;
	short id;
	};
	
可见memslot表示的是虚拟机中物理地址和宿主机中虚拟地址之间的映射关系。这里获取memslot，随后再通过`__gfn_to_pfn_memslot`，首先获得HVA，然后通过`hva_to_pfn`，获取对应的PFN。  

5 `__direct_map`的主要目标是完成针对给定的gfb地址，填充EPT各级表项，最后将gfn与pfn的关系映射到EPT的过程。  

这里的“direct”，和shadow相对应，表达的正是在EPT模式下的映射过程，其差别主要反映在`kvm_mmu_get_page`当中。在`__direct_map`中，首先通过宏定义`for_each_shadow_entry`，遍历了每一级EPT页表。它利用了一个迭代器结构，定义在mmu.c当中：

	struct kvm_shadow_walk_iterator {
		u64 addr;				//目标帧 gfn<<PAGE_SHIFT
		hpa_t shadow_addr;		//当前EPT页表项的物理地址
		u64 *sptep;				//下一级EPT页表的指针
		int level;				//当前页表级别
		unsigned index;			//当前页表索引
	}

`shadow_walk_okay`查询当前页表，获取下一级EPT页表的基址，或者最终的物理内存单元地址。`mmu_set_spte`设置当前请求level的EPT页表项：

	if (iterator.level == level) {//进入指定的level，为最后一级页表
        mmu_set_spte(vcpu, iterator.sptep, ACC_ALL,
                 write, &emulate, level, gfn, pfn,
                 prefault, map_writable);//设置页表项
        direct_pte_prefetch(vcpu, iterator.sptep);
        ++vcpu->stat.pf_fixed;
        break;
    }
    
 这里“==”主要是判断当前的页表是否属于最后一级页表，也即内容是否直接指向一个宿主机的物理页面。如果指针对应的页表不存在，则会创建新的页表：首先通过`kvm_mmu_get_page`分配一个EPT页表页，随后通过`link_shadow_page`->`mmu_spte_set`设置页表项，并且刷新TLB。
 
 		if (!is_shadow_present_pte(*iterator.sptep))            
		{
		//如果在4---level之间该指针对应的页表不存在，则需要进行创建新的页表，并填充上去
        u64 base_addr = iterator.addr;

        base_addr &= PT64_LVL_ADDR_MASK(iterator.level);
        pseudo_gfn = base_addr >> PAGE_SHIFT;
        sp = kvm_mmu_get_page(vcpu, pseudo_gfn, iterator.addr,
                      iterator.level - 1,
                      1, ACC_ALL, iterator.sptep);

        link_shadow_page(iterator.sptep, sp, true);
		}

6 `mmu_spte_update`函数主要用来更新状态位，它表示映射的pfn并没有改变，而是更新状态位的情况（主要与读写执行权限、设置脏页有关系）。通常在更新后要刷新TLB。  

		
