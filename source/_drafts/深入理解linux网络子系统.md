#### linux内核网络栈
![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%BD%91%E7%BB%9C/network-structure.png?raw=true)  
linux网络子系统，实际上也反映出了网络分层的结构。其中，应用层在用户空间当中实现，传输层、网络层、数据链路层都在内核区间内实现，而物理层在具体的网络中实现。  

#### 从应用层开始
网络在应用层的行为，是由操作系统和应用共同实现的。网络应用程序，都是通过linux socket编程接口，来和内核空间进行通信的。这里的关键结构就是`socket`了。

	struct socket {
		socket_state		state;
	
		kmemcheck_bitfield_begin(type);
		short			type;
		kmemcheck_bitfield_end(type);
	
		unsigned long		flags;
	
		struct socket_wq __rcu	*wq;
	
		struct file		*file;			//file实例，用于socket通信
		struct sock		*sk;
		const struct proto_ops	*ops;	//与具体协议相关联的操作
	};

可见socket也和VFS是紧密联系的，并且它同样定义了一系列的操作。`proto_ops`包含了一系列的操作，包括我们常见的`bind`，`connect`，`accept`，`poll`等等。然而应用程序并不需要关心这些细节，我们也容易意识到一点：操作系统为应用提供了统一的接口，开发者并不需要具体关心底层协议的差别；网络通信也被抽象成了对文件的读写，这就使得网络控制和对文件控制一样方便。对于一个socket文件，它的`file_operation`会被设置为`socket_file_ops`。  
那么`sock`又是干啥的呢？它指向一个和具体协议相关的结构，并且也包括一个`proto`成员，它同样也是一个ops的集合。`sock`主要负责内核的socket层和传输层的通信；而`socket`则用于系统调用的通信。这二者正是用户和内核之间关联的纽带。  
当我们在用户态使用`bing`、`connect`这些glibc中的库函数时，实际上最终使用的是`socketcall`系统调用，它定义在/net/socket.c当中，可以看到这些函数实际上在这里被统一处理了（通过一个switch），随后调用了具体的“子调用”。

	switch (call) {
		case SYS_SOCKET:
			err = sys_socket(a0, a1, a[2]);
			break;
		case SYS_BIND:
			err = sys_bind(a0, (struct sockaddr __user *)a1, a[2]);
			break;
		case SYS_CONNECT:
			err = sys_connect(a0, (struct sockaddr __user *)a1, a[2]);
			break;
		case SYS_LISTEN:
			err = sys_listen(a0, a1);
			break;
		case SYS_ACCEPT:
			err = sys_accept4(a0, (struct sockaddr __user *)a1,
					  (int __user *)a[2], 0);
			break;
			...
		}

这里不妨举几个例子。`sys_socket`正是创建一个新套接字的起点，它会调用`sock_create`和`sock_map_fd`两个函数。前者完成了sock的创建和初始化，而后者则为套接字创建一个伪文件，并为其分配一个文件描述符，**这正是系统调用所返回的结果**，也是用户用来识别、操作一个socket的“标识”。  
`sys_recvfrom`则用来接收数据，它会把目标套接字的文件描述符，传递到该系统调用。这个系统调用首先会根据`task_struct`和`fd`，来查找对应的`file`，随后确定它的`inode`并最终找到对应的socket。在一些准备工作之后，内核会调用特定协议的`sock->ops->recvmsg`来作为接收例程（例如TCP使用tcp_recvmsg，UDP使用udp_recvmsg）。  
`sys_sendto`是发送数据的一种实现。发送同样会根据文件描述符来查找相关的套接字。发送的数据通过`move_addr_to_kernel`从用户空间复制到kernel空间当中，并且调用特定于协议的发送例程`sock->ops->sendmsg`，产生一个所需协议格式的分组，转发到更低的协议层去。 
在对用户和应用统一的表面下，内核悄然做了工作，为统一的借口提供不同的服务。  

#### 传输层 
传输层的工作集中在af_inet、tcp.c和udp.c当中。af_inet是系统调用中下一层的函数实现，它主要作为具体传输层和函数和socket层之间的桥梁。在/net/ipv4/af_inet中，定义了一系列的`net_protocol`，例如：

	static const struct net_protocol tcp_protocol = {
		.early_demux	=	tcp_v4_early_demux,
		.handler	=	tcp_v4_rcv,
		.err_handler	=	tcp_v4_err,
		.no_policy	=	1,
		.netns_ok	=	1,
		.icmp_strict_tag_validation = 1,
	};
	
	static const struct net_protocol udp_protocol = {
		.early_demux =	udp_v4_early_demux,
		.handler =	udp_rcv,
		.err_handler =	udp_err,
		.no_policy =	1,
		.netns_ok =	1,
	};

inet层会根据不同的协议，调用对应的handler等。这里先看看UDP的情况，从前面可以看到其handler是`udp_rcv`。这里的handler，也就是对某个具体的协议的数据报进行处理的方法。该函数会进一步调用__udp4_lib_rcv，这里首先会进行一次checksum的检查，随后从udptable中，查找对应的sock。如果找到了套接字，那么就会调用`udp_queue_rcv_skb`进行进一步的处理，而后又立即到`sock_queue_rcv_skb`。当有数据到达时，会调用`sk_data_ready`指向的函数，去通知在sk_sleep等待队列睡眠的所有进程。
而相对来说，tcp的实现要复杂许多（这一点从TCP的11个状态就可以看出来）。与UDP类似的，TCP数据报的入口是`tcp_v4_err`。该函数同样会试图去找到一个已连接的套接字，不过在找到之后，TCP的工作才刚刚开始：它必须完成一次多路分解，按照对应的情况进行处理，包括被动连接建立、主动连接建立、分组传输、接收分组等。  
	
当用户发起网络调用时，会通过系统调用借口，进入内核中（glibc中的socket、bind、listen）  
inet层主要在af_inet.c当中，它是socket下层的函数调用，作为socket和传输层（TCP、UDP）的桥梁。socket层所对应的结构是`socket`，而INET层则使用`sock`。这两者的区别在于，`socket`是一个通用的套接字结构，而`sock`则是和具体的协议相关了。  

#### 网络层
传输层，最后会调用函数`ip_queue_xmit`，将发送数据的任务交给网络层。
#### socket缓冲区  
内核在接收/发送数据时，需要在各层协议之间交换数据，因此内核使用了socket缓冲区`sk_buff`，来避免来回复制分组数据，提升性能。它也是网络子系统的基石之一。  
这个数据结构包含一系列的指针，并且和一个内存区域相关联，其基本思想是通过指针来操作协议的首部。  
![](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E7%BD%91%E7%BB%9C/sk_buff.png?raw=true)  

#### 
网卡填满缓冲区，会产生硬件中断，系统会通过中断服务的子程序，将数据从硬件的缓冲区复制到内核空间的缓冲区，并且包装成为一个sk_buff。随后netif_rx将其发送给链路层，这里包含了软中断的策略。  
根据不同协议的类型，例如IP数据包，它会在完成本层的处理之后，根据IP首部中的传输层协议，调用相应协议的处理函数。
例如TCP会调用tcp_rcv，找到对应的socket，并把数据报加入对应的等待队列当中去。
应用层要取数据，根据inode获取socket，再从sock结构中的recieve_queue中读取数据包，并且复制到用户缓冲区。
