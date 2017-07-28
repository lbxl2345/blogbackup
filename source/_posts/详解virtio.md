---
title: vring前端
date: 2016-05-26 17:40:00
tags:
- linux
- 虚拟化

---
### bio
bi_next其实只是用在request queue当中的，作为一个指向下一个bio结构体的指针。  
bi_bdev指向了设备。  
bi_iter的作用是用来对这个bio中的bio_vec进行迭代，这个bio_vec以数组的形式进行存放。  
每个bio都包含一个磁盘存储区标识符(存储区中的起始扇区号和扇区数目)和一个或多个描述与I/O操作相关的内存区的段。其中bvec_iter bi_iter包含了扇区信息。

	struct bvec_iter {
		sector_t		bi_sector;	/* device address in 512 byte
							   sectors */
		unsigned int		bi_size;	/* residual I/O count */
	
		unsigned int		bi_idx;		/* current index into bvl_vec */	
		unsigned int            bi_bvec_done;	/* number of bytes completed in current bvec */
	};

而bio_vec的数据结构如下:
	
	struct bio_vec {
		struct page	*bv_page;
		unsigned int	bv_len;
		unsigned int	bv_offset;
	};
	
每个bio包含了一个由bio_vec表示的片段链表，每一个片段都是一段连续的内存空间。

### 从bio构造request:blk_make_request
	struct request *blk_make_request(struct request_queue *q, struct bio *bio, gfp_t gfp_mask)
blk_make_request:对于一个bio(或bio链)，生成一个对应的request，并加入到队列中去。其中，q代表的是目标request队列。  
bio_data_dir返回的是bio的rw情况。blk_get_request返回队列q中的一个free request。
for_each_bio(bio)，这里处理的是一条bio队列中每一个bio。
request_queue:定义在blkdev.h中。
virtblk_req的数据结构如下:  

	struct virtblk_req {
		struct request *req;
		struct virtio_blk_outhdr out_hdr;
		//out_hdr用来向后端描述这次请求
		struct virtio_scsi_inhdr in_hdr;
		u8 status;
		struct scatterlist sg[];
	};

这个函数对一个bio，生成了一个request，然后把它(以及它所在bio链中的所有bio)放在同一个request中，并且将它加入队列。
	
### virtblk_probe
virtblk_probe里面，完成了:  
vblk的申请(并把它和vdev关联 vdev->priv = blk)  
disk的申请(vblk->disk = alloc_disk)  
queue的申请(blk_mq_init_queue)  
指明这个硬件队列所对应的virtio_blk对象(q->queuedata = vblk)  

### 从request_queue到vring:virtio_queue_rq
每个virtblk_req对应一个request，对应一个scatterlist链表。
virtio_queue_rq以request为单位，将请求中的bio_vec映射成scatter-gather list，并将其添加到vring中，然后通过kick通知host。它包含这样两个个部分：  
1. blk_rq_map_sg，它根据request对象来设置scatterlist，并且返回scatterlist中存有多少项。这个函数有三个参数，分别是request_queue，request和scatterlist。首先，我们知道**request->bio是一个链表**，这里**\_\_blk_bios_map_sg**完成了通过这个对象中的bio链表中的bio，来设置sglist的工作。这个函数判断是否读写同一个块，如果是同一个块，直接调用sg_set_page，bvec的内容交给sg。否则则对bio中的每一个bio_vec，调用__blk_segment_map_sg。这个函数将scatterlist根据bvec尽可能的合并，对于不能合并的，使用一个新的scatterlist项。也就是说，**每一个request都包含一个bios链表，将这个bios链表，整理称为一个scatterlist**。  
2. \_\_virtblk_add_req，根据request构建一部分scatterlist，然后和之前构造好的scatterlist，一起放进vring中去。这个函数中，首先按顺序，构造一个scatterlist，包括out_hdr，request->cmd，data_sg，request->sense，in_hdr，status。这相当于一个储存信息的header。随后，这个函数调用了**virtqueue_add_sgs**，来完成进一步的工作：首先函数计算出总共的scatterlist数量(使用一个双重循环的方式)，然后调用**virtqueue_add**。virtqueue_add首先判断是否支持indirect descriptor这种方式。这种方式是一种节约vring空间的方式，它只使用一个单独的vring.desc，然后由这一项指向一个内存中的desc数组，然后这个数组再指向每个scatterlist的描述符。如果不使用这个方式，那么每个scatterlisg都占用一个virng中的desc。随后按照先out_sgs，后in_sgs的方式，依次给vring中的描述符赋值，并且修改vring.avail和free list的head。递增vring_virtqueue的num_added，假设这个值等于(1<<16)-1，那么就调用virtqueue_kick函数通知后端来进行处理。  
vring可以分为三个部分:  
descriptor tables 用来描述vring中的scatterlist  
available ring 描述的是guest提供给host的所有buffers  
used ring 描述的是host已经使用过的buffers  

### scatterlist中的固定信息
scatterlist的结构是这样的：

	struct scatterlist {
		unsigned long	page_link;
		unsigned int	offset;
		unsigned int	length;
		dma_addr_t	dma_address;
	};
那么ring中是如何传递和磁盘相关的信息的呢？在virtqueue_add_sgs中，首先申请了scatter几个scatterlist。这里对它们进行逐个的分析。  
首先是hdr。hdr中的内容是virtblk_req中的out_hdr。它是一个virtio_blk_outhdr。  

	struct virtio_blk_outhdr {
	/* VIRTIO_BLK_T* */
	__virtio32 type;
	/* io priority. */
	__virtio32 ioprio;
	/* Sector (ie. 512 byte offset) */
	__virtio64 sector;
	};
	
随后是virtblk_req->request->cmd，request的结构定义在blkdev.h中。cmd是一个字符串指针。  
data_sg指向的是数据的scatterlist。再者是virtblk_req->request->sense。它是sense data的指针。  
之后是virtblk_req->in_hdr，它是一个virtio_scsi_inhder。

	struct virtio_scsi_inhdr {
	__virtio32 errors;
	__virtio32 data_len;
	__virtio32 sense_len;
	__virtio32 residual;
	};

最后是virtblk_req->status，它是状态位。  