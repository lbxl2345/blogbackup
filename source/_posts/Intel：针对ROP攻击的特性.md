---
title: Intel:针对ROP攻击的新硬件特性
date: 2016-06-13 17:40:00
tags:
- linux
- intel
- 硬件

---
### 1 针对ROP/JOP/COP的intel特性
ROP/JOP/COP攻击通常有两个特性：第一是包含执行特权命令的代码片段，至少包含一条控制流转移指令，其转移目的由返回stack或者寄存器中的目标地址决定；第二是它们将控制流指令(RET，CALL，JMP)的地址修改到新的目标地址。  
intel提供Shadow Stack和Indirect branch tracking两种功能，来防御这一类攻击。Shadow Stack提供返回地址的保护，使程序免遭ROP攻击的影响；Indirect branch tracking提供分支保护，来防御JOP/COP攻击。  
##### 1.1 Shadow Stack
shadow stack是仅仅用来实现控制流转移的“第二个”stack，它和data stack分离。当shadow stack被激活时，CALL指令会把返回地址同时push进data stack和shadow stack中。在RET指令执行时，两个stack中的值都会pop出来，并进行一次比较，如果这两个值不相同，那么处理器就会发出一个control protection exception。
##### 1.2 Indirect branch tracking
ENDBRANCH是一个新的指令，它被用来标识合法的indirect call/jmp指令。在不支持CET的机器上，这个指令被翻译为一个NOP指令；在支持CET的处理器上，它依然是一个NOP指令，它主要被用做一个标记指令，用来标记处理器管道中有序的部分，来检测控制流的错误。CPU会实现一个状态及，来追踪jmp和call指令，一旦某个这类指令被发现了，状态机就从IDLE跳转到WAIT_FOR_ENDBRACH状态。在这个状态下，下一条指令必须是ENDBRANCH。如果下一条指令不是ENDBRANCH，那么处理器就制造一个control protection fault。  

### 2 Shadow stacks环境
shadow stack的读写是被严格限制的，它只能由控制流转移指令和shadow stack管理指令来完成。它能在user mode(CPL == 3)和supervisor mode(CPL < 3)中分别打开。shadow stacks必须在开启分页功能的保护模式下使用。    
使用shadow stack时，处理器会支持一个新的寄存器，shadow stack pointer(SSP)，它不能被直接编码为指令中的源地址、目的地址、随机内存地址等。它指向shadow stack的顶部。它保存的是一个线性地址，并通过FAR RET，IRET和RSTORSSP指令来装载入寄存器中。它的大小根据当前的mode确定为32bit或64bit。  
##### 2.1 Near CALL/RET
激活shadow stack时，Near CALL会将返回地址同时push到data stack和shadow stack中去，Near RET会将返回地址同时从data stack和shadow stack中pop出来。如果两者的返回地址不同，则触发near-ret异常。  
##### 2.2 Far CALL/RET
激活shadow stack时，Far CALL会将CS，LIP(linear address of return address)以及SSP三者push到shadow stack上，并在far RET的时候按SSP，LIP和CS的顺序pop。如果CS和LIP的值和CS以及EIP中的返回地址值不符合，就会触发FAR-RET/IRET异常。  
far CALL到更高权限的过程如下：  
当far CALL处于user mode(CPL3)时，返回地址不被push到supervisor shadow stack。相似的，一个从更高权限(CPL<3)回到CPL3的far RET也不会对返回地址进行验证。CPL3到CPL<3的过程中，user space SSP会被保存在一个MSR寄存器中，在CPL<3到CPL3的过程中，利用这个寄存器可以恢复user space SSP。  
对于不同权限之间的call，call指令会进行一次stack的切换。supervisor程序的data stack位于当前的TSS段。相似的，shadow stack也进行这种切换。根据权限的不同(ring 0, ring 1, ring 2)，supervisor程序会选择不同的MSR来获取SSP。  
从ring 2到ring 1，从ring 2 到ring 0或者从ring 1到ring 0被认为是“同样权限级别”之间的切换。也就是说，对于这样的calls，会将调用者程序短的CS，LIP和SSP push到被调用程序段的shadow stack上，而在far RET时，会验证其中的CS和LIP是否和data stack中的CS和EIP是一致的。  
对于一个不同权限级别之间的far CALL，CET会验证一个“supervisor shadow stack token”。这个token在shadow stacks创建的时候，由supervisor设置，是一个64bit的值。这个token中，有效部分有两个，其一是63:3bit，它们是这个token的地址；其二是第一个bit(busy bit)，它表明了对应的shadow stack是否在使用。  
在far CALL时，首先根据IA32_Plx_SSP中的地址，获取对应的token；随后判断它的busy bit是否为0，并且检查MSR中的地址和token中的是否吻合。吻合的话要将busy bit设置为1，并且切换SSP。  
在far RET时，首先通过SSP加载token，随后检查token的busy bit是否为1，并且检查token是否和SSP一致。如果通过，则要清除busy bit位。  
##### 2.3 Interrupt/Exception
64bit模式提供了一种stack切换机制，称为Interrupt stack table(IST)，其中的64bit IDT能够被用来在无权限变化发生时，指定64bit TSS中7个data stack指针中的一个。如果IST索引是0，说明没有权限变化，stack会切换到同一个stack。  
为了支持这个stack切换机制，shadow stack提供了一个MSR，IA32_INTERRUPT_SSP_TABLE，用来存放一个表的线性地址，它包含7个shadow stack指针。如果是一个非0的IST值，并且在call的过程中没有发生权限变化，那么MSR就指向内存中的一个64byte表，这个表利用IST来作索引。
##### 2.4 Shadow Stack&Task Switch
task切换能够通过三种方法完成：  

- JMP/CALL GDT中的TSS描述符  
- JMP/CALL GDT中的task-gate描述符或当前的LDT  
- 一个interrupt或exception向量，指向IDT中中的task-gate描述符  

在激活shadow stack的情况下，一个新的task必须与一个32bit TSS关联，并且不能工作在8086虚模式。32bit的新task的SSP，位于32bit TSS的偏移量104bytes处。因此新task的TSS至少要在108bytes的位置。SSP必须是8bytes对齐的，并且指向shadow stack token。  
一个用CALL指令实现的嵌套task切换，旧task的SSP不会保存到旧进程的TSS，而是和CS与LIP一起，被push到新task的shadow stack。类似的，一个以IRET实现的非(解)嵌套task切换，新task的SSP被从旧task的shadow stack中恢复。如果旧shadow stack中CS和LIP与新taskCS和EIP所决定的返回地址一致，说明控制流正确，如果不吻合，就会出现异常。    
### 3 Indirect branch环境
当indirect branch traking特性被激活时，indirect jmp/call指令的过程会做出相应的改变。  
jmp:如果indirect jmp的下一条指令不是ENDBR32指令(在遗留以及兼容模式)，或者ENDBR64指令，就产生#CP异常。只有以下形式的jmp指令会被跟踪：

- jmp r/m16, r/m32, r/m64  
- jmp m16:16, m16:32, m16:64  
 
call:如果indirect call的下一条指令不是ENDBR32指令(在遗留以及兼容模式)，或者ENDBR64指令，就产生#CP异常。只有以下形式的call指令会被跟踪：

- call r/m16, r/m32, r/m64  
- call m16:16, m16:32, m16:64

ENDBRANCH在支持/不支持CET的机器上，都不会造成任何执行上的影响。唯一的区别是，支持CET的处理器上，实现了一个2个状态的状态机，用来跟踪indirect call/jmp。在user mode和supervisor mode，各有一个这样的状态机。在除indirect call和jmp指令之外的指令时，状态机状态保持在IDLE；在indirect call和jmp指令时，状态机切换到WAIT_FOR_ENDBRANCH。在WAIT_FOR_ENDBRANCH状态下，只允许下一条指令是ENDBRANCH，或者兼容模式下的某些指令。  
当#CP(ENDBRANCH)异常发生时，高优先级的异常会先于#CP异常发生。在将控制流转交给异常handler时，高权限的状态机会保持原状态，指令指针压入堆栈中的是引发异常的indirect call/jmp的指令地址。  
##### 3.1 不追踪前缀：3EH
使用寄存器的Near indirect call/jmp指令，如果有3EH前缀，就说明这个控制流不需要被追踪。但Far call/jmp，以及使用内存地址的Near indirect call/jmp，即使有3EH前缀，也会忽略掉这个前缀，总是被追踪。  
##### 3.2 CPL 3和CPL<3之间的控制流转移
硬件实现了两个CET状态机：user mode和supervisor mode各有一个。当前使用哪个状态机，是根据此时CPL的值来决定的。当从CPL3切换到CPL<3时，会停用user mode的状态机，转而使用supervisor mode的状态机，反之亦然。  
在任何情况下，源状态机变成不激活状态，并且保存它的原有状态，如果没有异常情况，目标状态机激活。具体情况有以下几类：  

- Far call/jmp，SYSCALL/SYSENTER:tracker被激活，解除禁止，并转移到WAIT_FOR_ENDBRANCH，强迫far call/jump之后必须跟随着ENDBRACH。  
- Hardware interrupt/trap/exception/NMI/Software interrupt/Machine Checks:traker被激活，解除禁止并转移到WAIT_FOR_ENDBRANCH。  
- iret:tracker被激活，并且保持原有状态。  

##### 3.3 CPL 3和CPL<3内部的控制流转移
在同一个权限内发生控制流转移时，在控制流转移前后，不切换状态机。具体情况可分为以下几类：  

- FAR CALL/JMP:tracker解除禁止，并转移到WAIT_FOR_ENDBRANCH
- Near indirect call/jmp:如果tracker没有被禁止，转移到WAIT_FOR_ENDBRANCH
- Hardware interrupt/trap/exception/NMI/Software interrupt/Machine Checks:tracker unsuppressed，转移到WAIT_FOR_ENDBRACH
- iret:激活的tracker保持其状态

##### 3.4 INT3 处理
INT3在WAIT_FOR_ENDBRANCH中被特别处理。INT3的出现不会将tracker移动到IDLE状态，而INT3引发的#BP trap被作为一个更高优先级的事件处理，从而忽略了ENDBRANCH。  

##### 3.5 与遗留系统兼容
启用了CET的程序在满足条件的情况下，能够对遗留系统保持兼容性。首先，通过设置LEG_IW_EN位，可以开启遗留系统的兼容选项；其次，控制流的转移通过indirect call/jmp到non-endbranch的形式实现；第三，legacy code page bitmap被设置成指明目标控制流是遗留代码页。  
其中，legacy code page bitmap是一个程序内存中的数据结构，用来给硬件决定一个控制流转移是否是到遗留代码页的。这个bitmap的地址由EB_LEG_BITMAP_BASE，bitmap中的每个bit代表了线性内存中的一个4k页。如果bit是1，说明对应的代码页是遗留的代码页，否则它是一个启用了CET的代码页。线性地址的bits 31:12被用作这个bitmap的索引。

