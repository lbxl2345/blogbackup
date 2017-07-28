---
title: 内存攻击与保护
tags:
  - 底层安全
  - 二进制
date: 2016-12-05 17:40:00
---

#### memory corruption
Step 1: 一个指针变成非法的指针   
第一种:越过它所引用的对象的边界(out-of-bounds pointer)，**spatial error**   
第二种:它所指向的对象已经被释放(dangling pointer)，**temporal error**  

----------

**Bugs lead to Out-of-bounds:**  
1. allocation failure --> null pointer  
2. 过多的增／减 array pointer --> buffer overflow/underflow  
3. indexing bugs --> pointer 指向任意位置

Bugs lead to dangling:  
objects释放了，但指向它的指针没有释放  
(大多数objects在堆上，但堆栈上多objects如果用了全局的pointer，也会出现dangling pointer)   

----------

Step 2: dereferences the pointer  
dereferences the pointer: 读/写  

----------

Step 3: corruption/leakage of data   
**Out-Of-Bounds**    
1 read from memory  
pointer指向了攻击者控制的位置，那么取的值就被攻击者操纵了

	- 指针指向控制相关的数据，现在攻击者将它指向了恶意的转移目标 --> divert control-flow
	- 指针指向输出数据，现在攻击者将它指向了一些隐私数据 --> leaks information  

2 write to memory  
pointer指向写的地址，那么攻击者就能够篡改内存中的任意数据  

	- 篡改一个控制相关的变量，例如vtable，返回地址 --> divert control-flow  
	- 篡改一个输出的地址(另一个指针) --> leaks information  
	
**Dangling  pointer**
由deallocated object释放的memory，会被另一个object重新使用，而两种object的type是不同的，因此新的object会被解释为旧的object。  
1 read from memory  
旧的object的vpointer -->  divert control-flow  
新的object中敏感数据可用旧的object输出 --> leaks information   
2 write to memory  
旧的object在栈上 -->  divert control-flow(return address)  
double-free leads to double-alloc --> arbitraty wirtes  

------------ 
#### attack type
**控制流劫持**:divert control-flow  
**data-only攻击**:gain more control,gain privileges 
**Information leak**：leaks information  

------------
#### Protections
Probabilistic: Randomization/encryption  
Deterministic: Memory Safety/Control-flow Integrity   
其中Deterministic protections又可以分为hardware/software两种  
其中，software的方法，可以通过静态/动态两种插桩方式来实现。  

##### Probabilitic Methods 
Probabilitic Method依赖于随机化，包括有: 
Address Space Randomization:代码和数据段的位置，在W⊕X后，主要应用于code    
Data Space Randomization:对所有的变量进行加密  

##### Deterministic Methods  
**Memory Safety**  
空间安全-指针边界:指针能指向的地址范围  
空间安全-对象边界:对象的地址范围，比指针边界的兼容性更好   
时序安全-特殊的allocators:申请内存时避免use-after-free，例如只使用相同类型的memory
时序安全-基于对象:在shadow memory中标记释放内存，但如果这段内存被重新申请则无效  
时序安全-基于指针:在内存释放/申请时，更新指针信息  

#### Generic Attatck Defenses  
**Data Integrity**  
不关注时序安全，只保护memory写，不保护memory读。  
safe objects integrity:分析出不安全的指针和对象，利用shadow memory记录对应关系  
points-to sets integrity:分析出不安全的指针和对象，限制其points-to set的对应关系  
**Data-Flow Integrity**  
通过检查read指令，检测任何数据的corruption(上一次write是否合法)，同样使用ID和set的方式。  

#### Control-Flow Hijack Defenses  
**Code Pointer Integrity**  
防止Code Pointer被篡改，例如对Code pointer进行加密等。  
**Control Flow Integrity**  
动态return integrity:对返回值进行保护，如shadow stacks。  
static CFI:求出控制流转移的集合，在运行时检查控制流转移是否合法。  