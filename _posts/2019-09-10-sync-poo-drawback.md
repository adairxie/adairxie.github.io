---
layout:     post
title:      "请问sync.Pool有什么缺点？"
subtitle:   "golang标准库"
date:       2019-09-06
author:     "sf2012"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - 源码分析
    - go标准库
typora-root-url: ..
---


![](/img/in-post/sync-pool-drawback.png)

1.12及之前版本的sync.Pool有三个问题：
 1. 每次GC都回收所有对象，如果缓存对象数量太大，会导致STW1阶段的耗时增加。
 2. 每次GC都回收所有对象，导致缓存对象命中率下降，New方法的执行造成额外的内存分配消耗。   
 3. Pool.Get方法底层有锁，极端情况下，要尝试最多P次抢锁，也获取不到缓存对象，最后得执行New方法返回对象。

这些问题就对sync.Pool的使用提出了要求，不满足时，性能并不会有大幅提升：
 1. 最好是高并发场景。（对应问题3）
 2. 最好两次GC之间的间隔足够长。（对应问题1，2）

### 先看下sync.Pool的原理，看那块代码造成这三个问题。

sync.Pool对象内部为每个P都分配了一个**private区**和**shared区**。
private区只能存放一个可复用对象，因为每个P在任意时刻只运行一个G，所以在private区上写入和取出对象是不用加锁的。
shared区可以放多个可复用对象，它本身是slice。进shared区就append，出shared区就slice[:last-1]。但shared区上写入和取出对象要加锁，因为别的G可能过来偷对象。
```go
type poolLocalInternal struct {
	// 私有对象，每个P都有，用于不同G执行get和put可以无锁操作
	private interface{}
	// 共享对象数组，每个P都有一个，同一个P上不同G可以多次执行put方法。并且别的P上的G可能过来偷，所以要加锁
	shared []interface{}
	// 对shared进行加锁，private不用加锁
	Mutex
}
```

问题3就是由于shared区是一个带锁的后进先出队列造成的。每次Pool.Get方法在调用时，执行顺序是：
 1. 先看当前P的private区是否为空。
 2. 加锁，看当前P的shared区是否为空。
 3. 加锁，循环遍历看其他P的shared区是否为空。
 4. 只要上面三步任意一步不为空，就可以把缓存对象返回了。但若都为空，最后就得调用。  New方法返回对象。

```go
// 遍历一次其他P的共享区，偷一个，每次尝试偷都得上锁
for i := 0; i < int(size); i++ {
	// 定位到某个P上的shared区
	l := indexLocal(local, (pid+i+1)%int(size))
	l.Lock()
	last := len(l.shared) - 1
	if last >= 0 {
		// 如果有缓存对象，就返回，并解锁
		x = l.shared[last]
		l.shared = l.shared[:last]
		l.Unlock()
		break
	}

	// 没有缓存对象，解锁，继续遍历下一个P
	l.Unlock()
}
```

这一系列的加锁操作和Mutex锁自带的阻塞唤醒开销，Get方法在极端情况下就会有性能问题。

问题1和2都是由于每次GC时，遍历清空所有缓存对象造成的。
sync.Pool在init()中向runtime注册了一个cleanup方法，它在STW1阶段被调用的。如果它执行过久，就会硬生生延长STW1阶段耗时。
```go
func init() {
	runtime_registerPoolCleanup(poolCleanup)
}
```
这个cleanup方法干的事情是遍历所有的sync.Pool对象，再遍历每个sync.Pool对象中的每个P的shared区，把shared区每个缓存对象设置为nil。代码中就是三层for循环，简单粗暴时间复杂度高。
```go
func poolCleanup() {
	// ...
	for i, p := range allPools {
		// 有多少个sync.Pool对象，遍历多少次
		allPools[i] = nil
		for i := 0; i < int(p.localSize); i++ {
			// 有多少个P，遍历多少次
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				// 清空shared区中每个缓存对象
				l.shared[j] = nil
			}
			l.shared = nil
		}
		// ...
	}
	// ...
}
```

好消息是1.13beta1已经解决了这三个问题。注意是beta版本，而不是stable版本。
接下来主要看1.13通过什么思路解决这些问题的。

### 取消每个GC默认对全部对象进行回收
解决问题1和2的思路就是不能全部回收。但该回收多少呢？
@aclements提出了一种思路。**这轮在sync.Pool中的对象，最快也在下轮GC才被回收**。
>https://github.com/golang/go/issues/22950#issuecomment-352935997 

还记得上面说过每个P都有private区和shared区吗？现在每个P里两个区合在一起构成数组，名字叫**local**。1.13版本的实现中再引入一个**victim**，它结构与local一致。
```go
// 1.13版源码
type Pool struct {
	local 		unsafe.Pointer  // 实际指向[P]poolLocal
	localSize 	uintptr 		// P的个数

	victim 		unsafe.Pointer 	// 指向上轮的local
	victimSize 	uintptr			// 指向上轮的localSize

	New func() interface{}
}

type poolLocal struct {
	poolLocalInternal
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

type poolLocalInternal struct {
	private interface{}
	shared 	poolChain
}
```

有了victime，Get和Put方法的步骤就有所变化：
 1. Get时， 先从local里尝试取出缓存对象（包括所有的P）。如果失败，就尝试从victim里取。
 2. victim里也取对象失败，就调用New方法。
 3. Put时，只放local里。
