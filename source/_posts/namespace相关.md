---
title: namespace相关
date: 2016-10-10 11:40:00
tags:
- linux内核
- namespace

---
### namespace
当前，linux实现了6种不同类型的namespaces。每种namespace，都用来**包含一类特定的系统资源**，这样从命名空间内部的进程来看，它们就拥有了隔离的全局资源。namespaces的一个目标就是**容器**，一种轻量级的虚拟化工具，让一组进程认为它们是系统上仅有的一组进程。  

#### mount namespaces
mount namespace(CLONE_NEWNS)隔离一组进程所能看到文件系统mount点。在不同mount namespaces中的进程，对于文件系统有不同的视图。在使用了mount namespaces之后，mount和umount系统调用不再对所有进程可见的，全局的mount points进行操作，而是只会影响和发起调用的进程相关的mount namespace。  
利用主从关系，还可以让一个mount namespace自动拥有另一个mount namespace的内容，例如一个硬盘设备挂在到某个namespace中后会自动显示在另一个namespace中。  
mount namespace是linux上实现的第一种namespace。  
#### UTS namespaces
UTS namespace(CLONE_NEWUTS)隔离两种系统标识符：nodename和domainname。在容器的上下文环境中，UTS namespaces特性允许每个容器拥有自身的hostname和NIS domain name。这允许了根据容器的name来定义它们的行为，uts指的是UNIX Time-sharing System，它是传递给uname系统调用的参数。  
#### IPC namespaces
IPC namespaces(CLONE_NEWIPC)隔离inter-process communication resources，也即跨进程的通讯资源，System V IPC，以及POSIX message queues。这些IPC机制的共性时，IPC objects是由特殊机制来进行识别的，而不是文件系统的路径。在每个namespaces当中，又有其自身所拥有的System V IPC标识符和POSIX message queue filesystem。  
#### PID namespaces
PID namespaces(CLONE_NEWPID)隔离进程ID空间，也就是说，不同PID命名空间的进程，可以拥有相同的PID。这样做的一个好处是，容器能够在不同的hosts之间转移，但是又能够保持其中的进程ID不变。而且PID namespace能够允许每个容器拥有自己的init(pid 1)，对初始化、孤儿进程等事件进行处理。  
从一个PID namespace的角度来看，一个进程拥有两个PID：namespace内部的PID，以及namespace外部的，host上的PID。PID namespaces也是可以层叠的，从进程所归属的PID命名空间开始，一直到根PID namespace，它都有一个PID；一个进程只能看到处于它所在PID namespace当中的，以及更下层的其他进程。  
#### network namespaces
network namespaces(CLONE_NEWNET)将系统中与网络相关的资源隔离。也就是说，每个namespace当中拥有自身的网络设备、IP地址、IP路由表，端口号等。  
network namespaces让containers能够被应用到网络的层面上。每个container能够拥有自身的网络设备、并且其应用能够被绑定到namespace中特有的端口号上，对于特定的container，还可以设置特殊的路由规则。例如，可以在同一个host系统上，运行多个用container包含的servers，并且它们都绑定了80端口。  
#### user namespaces
user namespaces(CLONE_NEWUSER)将用户和group ID空间隔离。这也就是说，在一个user namespace内外，同一个进程点user和group id可以是不同的。例如，一个进程可以在一个user namespace外部，拥有一个普通的、无特权的user ID；而在在namespace中拥有UID 0。也就是说在namespace当中拥有root权限，但在namespace外部则不行。  
从Linux 3.8开始，无特权的进程能够创建它们自身的user namespaces，这为应用提供了新的可能：由于一个进程能够在其user namespaces中拥有root权限，那么它们就能够去使用那些本身只能由root用户使用的功能。但这确实会带来一些安全问题。  
#### c++中的namespace
编程语言中的namespace，虽然拥有相同的名称，其含义是完全不同的。但主要的思想是一致的，这里的命名空间也就是将空间内定义的内容放在一个盒子里，而命名空间也就是这个区域，using namespace 空间名，就将区域引入到了操作范围之内。  
这里，namespace是一种描述逻辑分组的机制，比如可以将某些属于同一个任务的类声明在同一个命名空间当中。标准C++库当中的所有内容，都被定义在命名空间std当中了。