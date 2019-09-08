---
layout:     post
title:      "sync.Pool原理及源码分析"
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

### pool关键作用 ###

  1. 减轻GC的压力。   
  2. 复用对象内存。又是不一定希望复用内存，单纯是想减轻GC压力也可主动给pool塞对象。   
  > Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector. That is, it makes it easy to build efficient, thread-safe free lists. However, it is not suitable for all free lists.   

### 原理简述 ###

sync.Pool就是围绕New字段，Get和Put方法来使用。用过都懂，比较简单就不介绍了。

Go是提供goroutine进行并发编程，在并发环境下，sync.Pool的使用不会造成严重性能问题是它的的设计考虑点。

容易想到的方法就是Pool对象为每个P都分配一个空间，这样在P上运行的G进行Get和Put操作时，就可以在P本地的空间上进行读写。这样比Pool对象维护一个全局空间有明显好处。全局空间的读写肯定要加锁。
   
即使每个P都有了自己的本地空间，也不是说就可以完全避免使用锁。不要忘了Pool提供了内存复用功能，每个P上的G都使用的是P本地的空间的话，那内存复用就有局限性，即只能局限在一个P上。 
而sync.Pool提供的内存复用是覆盖所有P。如果一个G在执行Get方法时，所在的P上没有可复用的对象，这时就到别的P那儿去偷。偷这个动作就要枷锁了。因为偷别人可复用对象时，别人也坑同时在读写。

前面开始说每个P有自己的空间，作用是避免锁，后面又说到别的P上偷对象，又要加锁，是不是矛盾了？不矛盾，让我们来看看sync.Pool的实现原理。

sync.Pool对象底层有两个关键字段，**local**和**localSize**，local指向一个数组，localSize表示数组的大小。localSize的大小跟P的个数保持一致。数组每个元素就是代表每个P自己的本地空间，类型是**poolLocal**。

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
```
**poolLocal**类型有两个关键字段， **private**和**shared**：

```go
// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{}   // Can be used only by the respective P.
	shared  []interface{} // Can be used by any P.
	Mutex                 // Protects shared.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```
- shared是一个数组，读写要加锁。
- private只能存一个对象，读写不需要加锁。

来理一下在Pool对象上读写的逻辑：
  1. Get操作时，先返回本地P上的private上的对象。
  2. 如果private为空，继续从本地P上的shared找，这里要加锁。
  3. 如果shared也没有，就到别的P那儿从shared里偷。
  4. 所有其他P都遍历过了，没有任何对象可偷，就返回nil或调用New函数。
  5. Put操作时，优先放private。
  6. private已经被放了，那就放到shared的最后。

### 用一张图来表示：

![](/img/in-post/golang-sync-pool/t1.png)

### sync.Pool的特性

- 无大小限制。
- **自动清理**，每次GC前会清掉Pool里的所有对象。所以不适合用来做连接池。
- 每个P都会有一个本地的poolLocal，Get和Put优先在当前P的本地poolLocal上操作，其次再进行跨P操作。
- 所以Pool的最大个数是runtime.GOMAXPROCS(0)。

### sync.Pool的缺点

pool的Get()并非成本低廉，最坏情况可能会上锁runtime.GOMAXPROCS(0)次。
所以，多goroutine与多P的情况下，使用Pool的效果才会突显。否则要经历无谓的锁成本。

### 简单的常用场景

bytes.Buffer作为临时对象放在池子里，这样减轻每次都需要创建的消耗。

```go
type Dao struct {
	bp	sync.Pool
}

func New(c *conf.Config) (d *Dao) {
	d = &Dao{
		bp: sync.Pool{
			New: func() interface{} {
				return &bytes.Buffer{}
			},
		},
	}
	return
}

func (d *Dao) Infoc(args ...string) (value string, err error) {
	if len(args) == 0 {
		return
	}

	// fetch a buf from bufpool
	buf, ok := d.bp.Get().(*bytes.Buffer)
	if !ok {
		return "", ErrType
	}

	// append first arg
	if _, err := buf.WriteString(args[0]); err != nil {
		return "", err
	}

	for _, arg := range args[1:] {
		// append arg
		if _, err := buf.WriteString(defaultSpliter); err != nil {
			return "", err	
		}

		if _, err := buf.WriteString(strings.Replace(arg, defaultSpliter, defaultReplacer, -1)); err != nil {
			return "", err
		}
	}

	value = buf.String()
	buf.Reset()
	d.bp.Put(buf)
	return
}
```
