---
title: virtio原理研究
date: 2016-05-06 17:40:00
tags:
- linux
- 虚拟化

---
### virtio简介
virtio是KVM虚拟环境下，针对I/O虚拟化的通用框架。virtio是一个半虚拟化驱动。首先说明一下全虚拟化和半虚拟化的区别。全虚拟化是指guest操作系统运行在物理机器上的hypervisor上，它不知道自己已被虚拟化，不需要任何更改就可以工作。半虚拟化指的是guest操作系统不仅知道它运行在hypervisor上，还包括让guest操作系统更高效与hyperviosr交互的代码(驱动程序)。  
### QEMU模拟I/O
如果使用QEMU模拟I/O，当guest中的设备驱动程序发起I/O操作请求时，KVM中的I/O操作捕获代码会将这次I/O请求拦截，在经过处理后将这次I/O请求的信息放在I/O共享页，然后通知QEMU程序。QEMU获得I/O操作的具体信息后，交由硬件模拟代码模拟出本次I/O操作，完成后，将结果放回I/O共享页，并通知KVM模块中的I/O操作捕获代码。最后，KVM模块中的捕获代码读取I/O共享页中的操作结果，把结果返回到客户机中。倘若guest通过DMA访问大块I/O时，QEMU不会把操作结果放在I/O共享页中，而是通过内存映射的方式将结果直接写到guest的内存中。  

<img src="https://github.com/lbxl2345/blogbackup/blob/master/source/pics/virtio/qemu.jpg?raw=true" height="50%" width="50%"/>
### virtio模拟I/O
下图中，最上面一排(virtio_blk等)是前端驱动，它们是在客户机中存在的驱动程序模块，而后端处理程序是在QEMU中实现的。在前端和后端之间，定义了两层来支持guest和QEMU之间的通信。virtio层是虚拟队列借口，一个前端驱动程序可以使用多个队列。虚拟队列实际上是guest操作系统和hyperviosr的衔接点。而virtio-ring实现了环形缓冲区，它用来保存前端驱动和后端处理程序执行的信息，并且它可以一次性保存前端驱动的多次I/O请求，并且交由后端驱动去批量处理，最后实际调用host中设备驱动实现物理上的I/O操作，这样做就可以实现批量处理，而不是客户机中的每次I/O请求都需要处理一次，从而提高了guest和hypervisor信息交换的效率。 

<img src="https://github.com/lbxl2345/blogbackup/blob/master/source/pics/virtio/virtio.jpg?raw=true" height="50%" width="50%"/> 
### virtio_blk
在linux中，对于块设备的访问，通常是用一个I/O队列，来维护一系列的bio数据结构，通常一个请求可能包含多个bio结构。bio是上层内核vfs与下层驱动连接的纽带。  
virtio_blk结构体中的gendisk结构多request_queue队列接收block层的bio请求，按照request_queue队列默认处理过程，bio请求会在io调度层转化为request，然后进入request_queue队列，最后调用virtblk_request将request转化为vbr结构，最后由QEMU接管处理。  
QEMU处理过vdr之后，会将它加入到virtio_ring的request队列，并发一个中断给队列，队列的中断响应函数vring_interrupt调用队列的回调函数virtblk_done。  
最后由request_queue注册的complete函数virtblk_request_done处理，通过blk_mq_end_io通告块设备层IO结束。 

<img src="https://github.com/lbxl2345/blogbackup/blob/master/source/pics/virtio/lifecycle.png?raw=true" height="50%" width="50%"/>  