---
title: 详解页缓存page cache
date: 2016-06-08 17:40:00
tags:
- linux
- I/O

---
### buffer cache/page cache
linux中存在有两个缓存，buffer cache是针对设备的缓存，而page cache是针对文件的缓存。对于一个ext4文件系统来说，每个文件都有一棵radix树管理文件的缓存页，这些缓存页就是page cache；而对于每个块设备来说，都有一棵radix树来管理数据的缓存块，这些缓存块被称为buffer cache。在常见的linux系统中，page cache通常以4kb为单位，而buffer cache的大小由块设备来决定，通常是512B。总的来说，page cache是对文件数据的缓存，而buffer cache是对设备数据的缓存。  
在linux 2.4之前，这两个cache是有区别的，但这明显会产生一些浪费。因此在2.4之后的内核版本中，这两个cache就被统一化了：使用page cache。如果一个缓存数据既代表文件又代表块，那么buffer cache就直接指向page cache。  
但是buffer cache依然是保留的。因为内核依然需要进行block的I/O。由于大部分block表示的是文件数据，因此它们都通过page cache的形式来缓存。但是剩下的小部分数据不是文件：它们是metadata活着原始的block I/O，这一部分依然由buffer cache来保存。  
linux当中，所有的文件I/O操作，都是通过page cache来实现的。写操作是通过将page cache中对应的页标记为脏页来实现的；读操作是通过从page cache中返回数据来实现的。如果数据还不在cache中，就先把它读到cache里面。  
如果只是研究一般文件的读写，那么就只需要在意page cache，不用去关心buffer cache。

### 关系
现在我们知道，在linux中，大部分文件都采用了page cache的形式来进行缓存。但是块设备的读写，却是以块的形式来进行的。前面有提到，page cache通常以4kb为单位，而buffer cache则通常是512B的。实际上，一个或多个buffer cache组成了一个page cache。  
![page&buffer](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/cache.jpg)
linux支持的文件系统，大多以块的形式组织文件。在文件以块的形式调入内存后，就以buffer cache的形式，对它们进行管理。buffer cache由两个部分组成，分别是缓冲区的首部buffer_head，和实际的缓冲区内容。buffer_head中，有一个指向数据的指针，和一个缓冲区长度的字段，这两个部分并不相邻。每当以块的形式，将数据读入内存时，它就要被存储在一个缓冲区当中，而buffer_head则起到一个描述符的作用。  
![bufferhead](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/bufferhead.jpg)
在从块设备中读写文件页的时候，会根据不同情况，来构造bio。bio中，io_vec中，bv_page字段，会指向page。在2.6版本后，buffer_head只给上层提供有关其描述的块的当前状态，描述磁盘块到物理内存的映射关系，而bio则负责所有块I/O操作。  
在linux中，mpage_readpage试图读取文件中一个page大小的数据。最理想的情况下，这个page大小的数据都在连续的物理磁盘上吗，函数只需要提交一个bio就可以获取所有的数据。这里使用get block函数，检查物理块是否连续。如果连续，则直接调用mpage_bio_submit函数请求整个page的数据，不连续则调用block_read_full_page逐个block读取，建立bh和bio之间的关系。mpage从来不回把不完整的页放进bio中，除非是文件的结尾。  

### 页高速缓存到用户空间
所谓的页高速缓存到用户空间，实际上分为两种：一种是read到用户空间，也就是复制到用户空间中的堆中去；第二种是映射，mmap是在堆外的空间。  
读取，要经过两次复制：第一次是从磁盘中读取来填充页缓存中的页；第二次是将是从内存中的页缓存，读取到进程堆空间的内存中。
![read](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/read.png)  
映射，只有一次复制：从磁盘中复制到缓存中。mmap会创建一个虚拟内存区域vm_area_struct，进程的task_struct包含了进程页表项，让这些页表项指向页缓存所在的物理页page。  
![mmap](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/mmap.png)  
由于程序的代码段必然是通过mmap来实现的，因此它们在使用时，其实是保存在页缓存中的。