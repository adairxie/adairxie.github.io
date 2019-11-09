---
layout:     post
title:      "Goroutine调度分析"
subtitle:   "golang标准库"
date:       2019-11-09
author:     "sf2012"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - 源码分析
    - goroutine 调度
typora-root-url: ..
---

# golang的调度模型

调度的机制用一句话描述：
runtime准备好G,P,M，然后M绑定P，M从各种队列中获取G，切换到G的执行栈上并执行G上的任务函数，调用goexit做清理工作并回到M, 如此反复。

## 基本概念

### M (machine)

	* M代表着真正的执行计算资源，可以认为它就是os thread（系统线程）。
	* M是真正调度系统的执行者，每个M就像一个勤劳的工作者，总是从各种队列中找到可运行的G，而且这样的M可以同时存在多个。
	* M在绑定有效的P后，进入调度循环，而且M并不保留G状态，这是G可以跨M调度的基础。

### P (processor)

	* P表示逻辑processor，是线程M的执行G的上下文。
	* P的最大作用是其拥有的各种G对象队列、链表、cache和状态。

### G (goroutine)

	* 调度系统的最基本单位goroutine，存储了goroutine的执行栈的信息、goroutine状态以及goroutine的任务函数等。
	* 在G的眼中只有P，P就是运行G的"CPU"。
	* 相当于两级线程

线程实现模型

				+-----+        +-----+
				| KSE |        | KSE |
				+-----+  	   +-----+
				   |              |              内核空间
---------------------------------------------------------
				   |              | 			 用户空间
				+-----+	       +-----+
				|  M  |        |  M  |
				+-----+		   +-----+
			   |      |       |       |       |
		    +----+  +----+  +----+  +----+  +----+
			| P  |  | P  |  | P  |  | P  |  | P  |
			+----+  +----+  +----+  +----+  +----+
         |     |     |       |       |      |     |
	   +---+ +---+ +---+   +---+   +---+  +---+ +---+
	   | G | | G | | G |   | G |   | G |  | G | | G |
	   +---+ +---+ +---+   +---+   +---+  +---+ +---+

* KSE (Kernel Scheduling Entity)是内核调度实体
* M与P，P与G之间的关联都是动态的，可以变的

 
