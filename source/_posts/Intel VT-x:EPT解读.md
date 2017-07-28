---
title: VT-x/EPT解读
date: 2016-06-15 17:40:00
tags:
- intel
- 虚拟化

---
### 1 VT-x
VT-x是intel针对硬件辅助虚拟化的技术，它解决x86指令集不能被虚拟化的问题，并且简化了VMM软件，减少了软件虚拟化的需求。Virtual Machine Extensions定义了一系列新的操作，称为VMX操作，来提供处理器级别的支持。同时它提供了一个新的特权等级VMX root给VMM，从而避免了ring deprivileging方法(让操作系统运行于ring 1，VMM使用ring 0)带来的虚拟化漏洞。VMX操作可以分为两类：  
root:VMM执行的VMX root 操作  
non-root:Guest执行的VMX non-root操作  
对于这两种模式之间的转换，VMX提供了确切的说法：  
VM Entry:转换到VMX non-root操作  
VM Exit:从VMX non-root操作转换到VMX root操作  
实际上，这个操作过程也就是VMM和虚拟机之间的转换过程。VMCS是一个用来管理VMX non-root操作和VMX转换的数据结构。它由VMM配置，指定guest OS状态，并在VM exits发生时进行控制。  
![VMCS](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/VT-x/VMCS.png)
### 2 MMU虚拟化
第一代VT-x在每次VMX转换都会进行TLB冲洗，这会造成在所有VM exits和大部分VM entries时的性能流失，因此对TLB清洗必须有更好的VMM软件控制。VPID(virtual Processor Identifier)是VMCS中的一个16bit域，它用来缓存线性翻译的结果。VPID启动时，就不需要冲洗TLB，不同虚拟机的TLB项能在TLB中共存。  
既往的VMM维持一个shadow page table，将guest的virtual pages，直接映射到machine pages。同时，guest中的V->P表，与VMM中V->M shadow page table同步。为了维持guest page table和shadow page table之间的关系，会因为VMM traps造成额外的代价，每次切换都会丢失性能，并且为了复制guest page table，在内存上也会有额外花费。  
### 3 EPT(Extended Page Table)
EPT这样的硬件支持(在AMD架构中类似的技术是NPT，nested page table)能够有效解决传统shadow page table的花销问题。在KVM最新的内存虚拟化技术中，采用了两级页表映射。第一级页表，客户虚拟机采用的是传统操作系统的页表，也即guest page table，记录着客户机虚拟地址(GVA)到客户机物理地址(GPA)的映射。而在KVM中，维护的第二级页表是extended page table(EPT)，记录的是虚拟机物理地址(GPA)到宿主机物理地址(HPA)的映射。  
![TDP](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/VT-x/EPT.png)  
在图中可以看到，包括guest CR3在内，一共有5个GPA，它们都要通过硬件走一次EPT，得到下一个HPA。那么如何通过EPT计算出对应的HPA呢，KVM是如何操作的呢？EPT和传统的页表一样，也分为4层(PML4、PDPT、PD、PT)，一个gpa通过四级页表的寻址，再加上gpa最后12位的offset，得到了hpa。  
在这个架构中，页可以分为两种：物理页(physical page)和页表页(MMU page)。物理页就是真正存放数据的页，页表页是存放EPT页表的页。这两种页创建的方式也不同，物理页可以通过内核提供的__get_free_page来创建，而页表页则是通过mmu_page_cache获得。这个page cache是在KVM初始化vcpu时通过linux内核中的slab机制分配的，它作为之后的MMU pages的cache来使用。在KVM中，每个MMU page对应一个数据结构kvm_mmu_page，在EPT处理过程中，它是极为重要的一个数据结构。  
一条地址如何翻译？首先non-root状态下的CPU加载guest CR3，由于guest CR3是一条GPA，CPU需要通过EPT来实现GPA->HPA的转换。但首先，MMU会先查询硬件的TLB，来判断有没有GPA到HPA的映射。如果没有GPA到HPA的映射，那么在cache中查询EPT/NPT。如果cache里面没有缓存，则逐层向下层存储查询，最终获得guest CR3所映射的物理地址单元内容，作为下一级guest页表的索引基址。当CPU访问EPT页表，查找HPA，发现相应的页表项不存在时，就会抛出EPT Violation，由VMM截获处理它。随后通过GVA的偏移量，计算出下一条GPA，依次循环下去，直到最终获得客户机请求的页。整个过程如下图所示。  
![violate](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/VT-x/violate.png)  

