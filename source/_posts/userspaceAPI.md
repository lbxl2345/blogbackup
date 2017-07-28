---
title: namespace API
date: 2016-10-10 20:00:00
tags:
- linux内核
- namespace
- container

---
### namespace API
namespace的API包含3个系统调用——clone，unshare，setns，以及一系列的/proc文件。为了指定操作的namespace类型，这3个系统调用都使用了一个CLONE_NEW常量(CLONE_NEWIPC,CLONE_NEWNS,etc)。  
#### clone
通过clone，可以创建一个namespace，它是一个创建新的process的系统调用。其函数原型为：

	int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
	
clone可以看作fork()的通用版本，其功能能够通过flags参数CLONE\_*来控制，这些参数包含了parent和child是否共享虚拟内存、打开文件描述符等。而如果参数中CLONE\_NEW位被指定了，那么就会创建一个新的，对应类型的namespace，而新的进程则成为这个namespace中的一个成员。  
和大多数其他的namespaces一样，创建一个UTS namespace是需要特权的，例如CAP_SYS_ADMIN，这对于避免需要设置user ID的应用来说是有必要的：如果能够使用任意的hostname，那么一个非特权用户就能够破坏lock file的作用，或者能够改变应用的行为。  
#### /proc文件
对于每一个进程来说，都有一个/proc/PID/ns目录，这其中每一种类型的namespace，都对应了一个文件。从linux 3.9开始，这些文件都被符号链接，作为处理这个进程相关namespace的handler。  

	$ ls -l /proc/$$/ns         # $$ is replaced by shell's PID
    total 0
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user -> user:[4026531837]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
    
这些符号链接的作用之一，就是用来检查两个进程是否处于同一个命名空间当中。kernel保证如果两个进程在同一个namespace当中，那么/proc/PID/ns中的inode number就会是一致的。inode numbers能够通过stat()系统调用来得到。  
但是，kernel还是会构造/proc/PID/ns的符号链接，并使得它指向字符串，这个字符串包含了namespace的类型和inode number。	
如果这个符号链接被打开，那么即使namespace中的进程全部终止了，namespace也不会消被清除。
#### setns
setns可以被用来加入一个已存在的namespace。保持一个没有任何进程的namespace，是因为随时可以加入新的进程到这个namespace当中去，这也是setns系统调用的作用。其函数原型为：  

	int setns(int fd, int nstype);
	
更准确的说，setns解除一个进程和之前对应nstype的namespace的联系，并且将其关联到新的，对应类型的namespace中去。这里，fd指定了对应的namespace，它是/proc/PID/ns目录下的一个文件描述符。而nstype则会用来检查fd指向的namespace的类型。  
利用setns和execve，能够构造一个很有效的工具：一个加入指定namespace然后再namespace中执行一条命令的程序。  
从linux 3.8开始，setns能够加入任何类型的namespace。  
#### unshare
unshare用来离开namespace。  
unshare的功能类似于clone，它创建一个新的namespaces，并且让调用者称为这个命名空间的一部分。它的主要目的，是在不创建新的进程或线程的前提下，完成namespace的分离工作。  

	clone()
	和
	if(fork() == 0)
		unshare()
	是等价的
	

