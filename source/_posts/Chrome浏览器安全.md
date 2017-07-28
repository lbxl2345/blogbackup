---
title: Chrome隔离/SOP策略
tags:
  - 浏览器安全
  - chrome
date: 2016-12-27 21:00:00
---

##### The "Web/Local" Boundary is Fuzzy
Process-based隔离，是浏览器安全的基础。Chrome的设计，并不隔离每个web源，而是主要保证“本地系统”和“web”，但是由于Dropbox，Google Drive这样的web端云服务(这些服务和本地系统一体化了)，浏览器不再能保证web和本地系统的隔离了。这篇文章提出了chrome中存在的一个问题，如果process-based隔离，忽略了同源策略，那么就很难保证web/local隔离的有效性。  
这篇文章提出，利用renderer中存在的漏洞，能够将可执行文件、脚本存放到本地文件系统中，肆意安装应用，滥用传感器等。这个攻击也是一个完全data-oriented的攻击，它不需要去劫持程序的控制流，或者引入外部代码。  

#### 浏览器隔离保护技术
web浏览器，在对象、资源的划分上，一直存在漏洞，这也导致一个binary-level的漏洞，能够让攻击者劫持整个browser。为了对抗这些恶意的网站，现代的浏览器架构，普遍应用了sandbox技术来加强web域和local域之间的隔离。sandbox保证，即使浏览器中出现了memory errors，也不会影响到本地的系统。在此之上，现代浏览器还采用了process隔离的设计。这个设计的目标，是进一步将exploit的影响限制在边界之内，这样其它进程就不会受到影响。Google chrome和IE都应用了这种设计。  
Goolge chrome从两方面进行了隔离：其一是将浏览器的kernel process和renderer process隔离；其二是将website或tabs，划分到不同的renderer process当中去。为了对性能进行提升，chrome降低了sandbox的粒度，放弃了SOP(同源策略)。SOP的任务完全交给了renderer来承担。  

#### chrome web/local隔离
chrome中，进程分为两类： 
 
1. kernel，用来和本地系统交互；  
2. renderer，负责显示网页。  

kernel process，负责网络请求，访问cookies，显示bitmaps；而renderer process则负责解释web文件(html，ccs，js)。每个renderer process都被限制在一个sandbox中，也只能通过kernel process来访问有限的资源。  
chrome的sandbox技术，是借助于操作系统的。例如在Linux当中，Chrome有两层sandbox，第一层为不同的renderer process创建不同的PID namespaces和网络资源；第二层保护browser kernel不受用户空间中恶意代码的影响。因此renderers只能通过IPC call，通过kernel来访问有限的资源。  

#### chrome SOP策略
在chrome中，SOP是由renderer process来全权负责的。它负责脚本只能在同源时，跨站进行访问。  
例如途中，当A的脚本访问B中对象时，首先要通过security monitor进行检查。  
![chrome-SOP](https://github.com/lbxl2345/blogbackup/blob/master/source/pics/%E6%B5%8F%E8%A7%88%E5%99%A8/chrome-SOP.png?raw=true)  
对于process内部，chrome将无关的数据放置在不同的区域中。例如，对每个renderer，其heap都是分隔开的，chrome随机化每个部分的起始地址，并用guardpages进行保护。

#### 绕过chrome SOP
Google chrome采用了共享一个renderer process的架构，因此SOP检查必须放在同一个security monitor，它位于renderer进程中。如果能够篡改monitor中的关键数据，就能够绕过SOP，在A中获得B的权限来执行任意代码。  
在chrome中，security monitor是由一系列的函数调用来实现的。在内存中，保存了大量的flags、以及SOP检查的记过。而从A，或者B中，均能够对这些数据区域进行访问。在chrome当中，甚至还有能够对全局进行访问的标志位。

	bool SecurityOrigin::canAccess(const SecurityOrigin*     other) const	{
		￼if (m_universalAccess) return true;		if (this == other) return true;		if (isUnique() || other->isUnique()) return false;		return canAccess; 	}
而利用系统中的漏洞，对这些critical data进行修改，就能够实现肆意的访问、执行。  
攻击的第一步，是绕过ASLR。在攻击者的脚本中，是能够创建一个layout形式可以预测的object的。攻击者通过线性的扫描，是能够确认object的位置的。而这个object位置，就揭露了随机化的地址。  
第二步，是绕过浏览器内部的隔离。这里，chrome采用的是一种in-memory的隔离方式，也就是将无关的数据放在分隔的部分中去。这里，不同的隔离区被guard pages保护了，并且其位置也是随机化的。然而，通过利用指针，进行跨隔离区的引用，也能够识别出隔离区内的地址。  
