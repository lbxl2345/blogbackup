---
title: SGX虚拟化相关
tags:
  - SGX
  - KVM
  - QEMU
date: 2017-01-17 12:00:00
---

### KVM-SGX
虚拟机中的epc，使用sgx_vm_epc_buffer结构来管理的。其数据结构如下：  
  
  	struct sgx_vm_epc_buffer {
		struct list_head buf_list;
		struct list_head page_list;
		unsigned int nr_pages;
		unsigned long userspace_addr;
		__u32 handle;
	};

Kai Huang添加了sgx_vm.c。

	__alloc_epc_buf:申请一个新的epc buffer，在这里对每一个页，都申请了一个iso_page。它调用sgx_alloc_vm_epc_page来完成具体的申请。这个函数定义在sgx_alloc_epc_page当中，实际上调用的就是sgx_alloc_epc_page。
	
	__get_next_epc_buf_handle():每一个epc buffer都有一个handle，这个handle相当于一个标识，利用handle来找到对应的epc buffer，这个函数用来在申请一个epc_buf的时候，获取它的handle。  
	
	__free_epc_buf():释放epc buffer，这里同样也要完成对应的iso_pages的释放。  
	
	__sgx_map_vm_epc_buffer():按页实际完成按页的映射，它调用了vm_insert_pfn函数。它进而调用vm_insert_pfn_prot，这个函数位于mm/memory.c中，也即完成一个页的映射(pfn to pfn)。

在kvm_main.c中，hva_to_pfn函数被进行了修改，这是为了让pfn的映射，支持到EPC这一段内存，而epc并不是连续的，所以需要从页表中搜索到pfn。hva_to_pfn()，其参数address为host virtual address，通过一个guest页的hva，来找到对应的pfn。它调用了follow_pfn(使用user virtual address来查找一个页框)。

	只有在hva_to_pfn_fast/hva_to_pfn_slow都无法找到pfn时，才会使用这种方法。因为EPC不是连续的内存段，所以会用这种方法特殊处理。
	
arch/x86/kvm/vmx.h/vmx.c中，做了VMX和SGX交互的修改。

	首先对于一部分enclave中不允许使用的指令，例如CPUID，INVD，做了#GP处理。
	
	vmx_exit_from_enclave():enclave中发生了VMEXIT的情况。这里使用了VM_EXIT_REASON的bit 27和GUEST_INTERRUPTIBILITY_INFO中的bit 4来表示VMEXIT的原因。
	
	vmx_handle_exit是对exit的具体处理。利用VM_EXIT_REASON的bit 27，可以判断这个exit是在enclave当中发生的。这个函数确定exit的类型，再交由对应的handler进行处理。

### QEMU-SGX
在target-i386/kvm.c当中，qemu首先为isgx定义了一个设备node:/dev/sgx，并且定义了VM中SGX的状态SGXState。在kvm.c中，定义了epc的alloc、free

	#define SGX_IOC_ALLOC_VM_EPC  _IOWR(SGX_MAGIC, 0x03, struct sgx_alloc_vm_epc)
	#define SGX_IOC_FREE_VM_EPC  _IOW(SGX_MAGIC, 0x04, struct sgx_free_vm_epc)
	
这两个宏定义，最后交给了struct中对应的handle来处理，也即ISGX驱动中对应的函数。而qemu内部的接口则是kvm_alloc_epc/kvm_free_vm_epc。kvm.c中，还定义了epc的初始化和销毁。epc的计算是在vcpus创建之前完成的，这里epc的大小被限制在256M之内，它被放置在below_4g_memory_size的位置，位于PCI的基址之下。在target-i386/cpu.c中，添加了对cpuid对应功能的支持。  在/hw/i386/acpi-build.c当中，添加了EPC的ACPI table项。

	
	