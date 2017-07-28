---
title: SGX接口
date: 2016-09-18 17:40:00
tags:
- SGX

---
### 1 SGX
SGX:Software Guard Extensions  
在运行时，程序可以分为几个部分：  
1. Untrusted Run-Time System:在SGX enclave外部执行的部分，负责加载和管理一个enclave，并且Ecall enclave，接受enclave中的Ocall。  
2. Trusted Run-Time System:在SGX enclave内部执行的部分，接受Ecall，进行Ocall，并对enclave自身进行管理。
3. Edge routines：指的是函数边的情况。  
4. 第三方库：为SGX定制的库。

---
ECall：Enclave call，调用enclave当中的函数。  
OCall：Out call，从enclave内部到外部的调用。  
在SGX中，enclave是用来减少信任基的，在运行时不可信域决定了可信域函数的调用顺序，也决定了起内部的上下文；并且ECall和OCall的返回值和参数也是不可信的。  
![sgx](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/SGX/SGX.png?raw=true)

---

文件格式：除了代码段、数据段之外，enclave文件还包含**metadata**,一个untrusted loader需要使用这个metadata来决定这个enclave如何被装载。  
### 2 Enclave接口  
将一个应用划分为trusted和untrusted两部分之后，需要定义二者之间的接口。不可信的应用通过ISV接口函数，对共享库进行调用(ECall)；而从Enclave对外部进行调用时(OCall)，在函数执行完后会返回可信的区域继续执行；中断也不会破坏这个过程。  
Enclave需要暴露一部分接口给外部应用(ECalls)，同时又要声明哪些外部提供的服务(OCalls)是必须的。  
Enclave的输入和输出都是不可信的代码可见的，因此enclave不能信任任何不可信域中的信息，检测ECall的输入参数和OCall的返回值。  
Enclave中的参数等，都保存在可信的环境中，并且其读写不会对ISV代码和数据的完整性造成影响；而参数的长度、返回值等都由ISV来指定。对于引用的输入，enclave会进行更特殊的处理，确定指针所指的内存区域是否在enclave的线性范围之内。  
对于操作系统的服务，Enclave是不能直接使用的，必须通过OCall作为接口，其return值作为输入传回给Encalve，这个值也是不可信的。  
如果在OCall中使用了ECall，这就是一个nested ECall，使用者应该避免这种情况的发生，如果必须使用，则要对接口进行限制。  

### 3 Signature
在Enclave中，可信环境的建立有三个主要的部分，分别是  
Measurement：当前环境下的，enclave的身份证明  
Attestation：向其他部分证明自身可信  
Sealing：能够在可信环境恢复时，恢复其相关的数据  

---

#### Measurement
Enclave包含一个由author提供的证书，也即Enclave Signature，它能够让SGX来检测enclave文件是否被篡改了，从而证明这个enclave是可信的。但硬件只在装载的时候进行检验，因此enclave signature还会对author进行验证，它包含这些部分：  
Enclave Measurement、Enclave Author的公钥、Security Verision Number、Product ID  

#### Attestation
Attestation指的是第三方能够证实软件是在SGX平台上运行的。Intel SGX架构支持两种验证：本地验证和远程验证。  
本地验证：一个enclave和另一个enclave协作，那么这两者之间就需要进行验证。enclave能够使用硬件来生成credential(report)，用来发给另一个enclave进行验证  
远程验证：一个拥有enclave的应用需要使用一个平台外的服务时，能够使用enclave来制造一个report，并且将它给平台服务，来产生一个credential(quote)，使用EPID技术来进行验证它来进行检测  

#### Sealing
在enclave被销毁时，需要识别出其中需要保护的data和state，以便在之后仍然能够在enclave中使用这些数据。这些数据只有被保存在enclave的外部。有两种情形：  
Seal到当前Enclave：在enclave创建时会有一个MRENCLAVE，只有拥有相同的MRENCLAVE才能unseal  
Seal到Enclave作者：在enclave创建时会有一个MRSIGNER，只有拥有相同MRSIGNER才能unseal  

### 4 处理器特性
enclave writer需要依赖编译器和库，他无法知道生成的enclave是否使用了任何特殊的CPU拓展特性。不可信的loader可能会允许所有的特性，但是通过设置Enclave Signature Structure，是能够指定重载这部分的设置的。  
在Enclave中，有一些指令是非法的，包括可能VMEXIT的，无法被软件处理的中断的，以及需要改变权限级别的，以及CPUID。  

### 5 Power Management
现代操作系统提供的一种机制，允许应用被能耗事件通知。当平台进入S3和S4状态时，密钥会被擦除，所有的enclave会被销毁。Intel SGX并不直接提供Power down事件到enclave当中。应用可以为这些事件注册相应的回调函数，在其被调用时，将secret state保存到磁盘上。但OS不保证enclave有足够的时间去做这件事情，因此enclave最好通过power transition events对enclave state data进行周期性的保护。    

### 6 线程相关
当多线程的程序运行时，thread Binding，TLS的使用都可能带来问题。对于enclave来说，开发者可以选择Non-Binding和Binding两种模式。Non-Binding模式会在不可信运行时，使用任意的TTC，并且使用一个root call进入enclave中，每次root call时，TLS都会被初始化；Binding模式下，不可信线程会和一个enclave中的可信线程绑定。  