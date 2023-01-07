---
title: go 内存分配器
date: 2022-12-09 19:19:30
tags:
---
> 内存分配是 Go 语言运行时内存管理的核心逻辑，运行时的内存分配器使用类似 TCMalloc 的分配策略将对象根据大小分类，并设计多层级的组件提高内存分配器的性能。
<!-- more -->
## 设计原理
内存管理一般包含三个不同的组件，分别是用户程序（Mutator）、分配器（Allocator）和收集器（Collector）1，当用户程序申请内存时，它会通过内存分配器申请新内存，而分配器会负责从堆中初始化相应的内存区域。
![img_1.png](/images/go_memory_alllocator/img_1.png)
### 分配方法
> 编程语言的内存分配器一般包含两种分配方法，一种是线性分配器（Sequential Allocator，Bump Allocator），另一种是空闲链表分配器（Free-List Allocator），

#### 线性分配器
线性分配（Bump Allocator）是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置，即移动下图中的指针：
* 线性分配器无法在内存被释放时重用内存， 所以需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能了。
* 因为线性分配器需要与具有拷贝特性的垃圾回收算法配合，所以 C 和 C++ 等需要直接对外暴露指针的语言就无法使用该策略
![img_2.png](/images/go_memory_alllocator/img_2.png)

#### 空闲链表分配器
空闲链表分配器（Free-List Allocator）可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表
* 隔离适应（Segregated-Fit） 分配策略：  将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；
![img_3.png](/images/go_memory_alllocator/img_3.png)

### 分级分配 
> Go 语言的内存分配器就借鉴了 TCMalloc 的设计实现高速的内存分配，它的核心理念是使用**多级缓存**将对象根据**大小分类**，并按照类别实施不同的分配策略。
#### 多级缓存 #
内存分配器不仅会区别对待大小不同的对象，还会将内存分成不同的级别分别管理，TCMalloc 和 Go 运行时分配器都会引入线程缓存（Thread Cache）、中心缓存（Central Cache）和页堆（Page Heap）三个组件分级管理内存：
![img_4.png](/images/go_memory_alllocator/img_4.png)
线程缓存属于每一个独立的线程，它能够满足线程上绝大多数的内存分配需求，因为不涉及多线程，所以也不需要使用互斥锁来保护内存，这能够减少锁竞争带来的性能损耗。当线程缓存不能满足需求时，运行时会使用中心缓存作为补充解决小对象的内存分配，在遇到 32KB 以上的对象时，内存分配器会选择页堆直接分配大内存。

### 虚拟内存布局 
在 Go 语言 1.10 以前的版本，堆区的内存空间都是连续的；但是在 1.11 版本，Go 团队使用稀疏的堆内存空间替代了连续的内存，解决了连续内存带来的限制以及在特殊场景下可能出现的问题。
#### 线性内存
![img_5.png](/images/go_memory_alllocator/img_5.png)
在 C 和 Go 混合使用时会导致程序崩溃：

1. 分配的内存地址会发生冲突，导致堆的初始化和扩容失败；
2. 没有被预留的大块内存可能会被分配给 C 语言的二进制，导致扩容后的堆不连续；

#### 稀疏内存 
使用稀疏的内存布局不仅能移除堆大小的上限，还能解决 C 和 Go 混合使用时的地址空间冲突问题。不过因为基于稀疏内存的内存管理失去了内存的连续性这一假设，这也使内存管理变得更加复杂：
![img_6.png](/images/go_memory_alllocator/img_6.png)
如上图所示，运行时使用二维的 runtime.heapArena 数组管理所有的内存，每个单元都会管理 64MB 的内存空间：
``` go 
type heapArena struct {
    bitmap       [heapArenaBitmapBytes]byte
    spans        [pagesPerArena]*mspan
    pageInUse    [pagesPerArena / 8]uint8
    pageMarks    [pagesPerArena / 8]uint8
    pageSpecials [pagesPerArena / 8]uint8
    checkmarks   *checkmarksMap
    zeroedBase   uintptr
}
```
每个 runtime.heapArena 都会管理 64MB 的内存，整个堆区最多可以管理 256TB 的内存，这比之前的 512GB 多好几个数量级。

## 内存管理组件
> Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件，本节将介绍这几种最重要组件对应的数据结构 runtime.mspan、runtime.mcache、runtime.mcentral 和 runtime.mheap，我们会详细介绍它们在内存分配器中的作用以及实现。
![img_7.png](/images/go_memory_alllocator/img_7.png)

所有的 Go 语言程序都会在启动时初始化如上图所示的内存布局，每一个处理器都会分配一个线程缓存 runtime.mcache 用于处理微对象和小对象的分配，它们会持有内存管理单元 runtime.mspan。

每个类型的内存管理单元都会管理特定大小的对象，当内存管理单元中不存在空闲对象时，它们会从 runtime.mheap 持有的 134 个中心缓存 runtime.mcentral 中获取新的内存单元，中心缓存属于全局的堆结构体 runtime.mheap，它会从操作系统中申请内存。
### 内存管理单元
#### 页和内存


每个 runtime.mspan 都管理 npages 个大小为 8KB 的页，这里的页不是操作系统中的内存页，它们是操作系统内存页的整数倍，该结构体会使用下面这些字段来管理内存页的分配和回收：
```go
type mspan struct {
	startAddr uintptr // 起始地址
	npages    uintptr // 页数
	freeindex uintptr

	allocBits  *gcBits
	gcmarkBits *gcBits
	allocCache uint64
	.
}
```

* startAddr 和 npages — 确定该结构体管理的多个页所在的内存，每个页的大小都是 8KB；
* freeindex — 扫描页中空闲对象的初始索引；
* allocBits 和 gcmarkBits — 分别用于标记内存的占用和回收情况；
* allocCache — allocBits 的补码，可以用于快速查找内存中未被使用的内存；

#### 跨度类
```go
type mspan struct {
	.
	spanclass   spanClass
	.
}
```
* Go 语言的内存管理模块中一共包含 67 种跨度类，每一个跨度类都会存储特定大小的对象并且包含特定数量的页数以及对象，所有的数据都会被预选计算好并存储在 runtime.class_to_size 和 runtime.class_to_allocnpages 等变量中：

* 除了上述 67 个跨度类之外，运行时中还包含 ID 为 0 的特殊跨度类，它能够管理大于 32KB 的特殊对象，我们会在后面详细介绍大对象的分配过程

### 线程缓存 
runtime.mcache 是 Go 语言中的线程缓存，它会与线程上的处理器一一绑定，主要用来缓存用户程序申请的微小对象。每一个线程缓存都持有 68 * 2 个 runtime.mspan，
这些内存管理单元都存储在结构体的 alloc 字段中
![img.png](/images/go_memory_alllocator/img.png)
#### 微分配器 
* 线程缓存中还包含几个用于分配微对象的字段
* 微分配器只会用于分配非指针类型的内存

### 中心缓存
* 与线程缓存不同，访问中心缓存中的内存管理单元需要使用互斥锁
```go
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet
	full     [2]spanSet
}
```
每个中心缓存都会管理某个跨度类的内存管理单元**，它会同时持有两个 runtime.spanSet，分别存储包含空闲对象和不包含空闲对象的内存管理单元。
#### 内存管理单元分配
线程缓存会通过中心缓存的 runtime.mcentral.cacheSpan 方法获取新的内存管理单元，该方法的实现比较复杂，我们可以将其分成以下几个部分：

1. 调用 runtime.mcentral.partialSwept 从清理过的、包含空闲空间的 runtime.spanSet 结构中查找可以使用的内存管理单元；
2. 调用 runtime.mcentral.partialUnswept 从未被清理过的、有空闲对象的 runtime.spanSet 结构中查找可以使用的内存管理单元；
3. 调用 runtime.mcentral.fullUnswept 获取未被清理的、不包含空闲空间的 runtime.spanSet 中获取内存管理单元并通过 runtime.mspan.sweep 清理它的内存空间；
4. 调用 runtime.mcentral.grow 从堆中申请新的内存管理单元；
更新内存管理单元的 allocCache 等字段帮助快速分配内存；


### 页堆 
runtime.mheap 是内存分配的核心结构体，Go 语言程序会将其作为全局变量存储，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表 central，另一个是管理堆区内存区域的 arenas 以及相关字段。
![img_8.png](/images/go_memory_alllocator/img_8.png)

#### 内存管理单元分配
1. 从堆上分配新的内存页和内存管理单元 runtime.mspan；
2. 初始化内存管理单元并将其加入 runtime.mheap 持有内存单元列表；

#### 申请内存
1. 如果申请的内存比较小，获取申请内存的处理器并尝试调用 runtime.pageCache.alloc 获取内存区域的基地址和大小；
2. 如果申请的内存比较大或者**线程的页缓存**中内存不足，会通过 runtime.pageAlloc.alloc **在页堆上申请内存**；
3. 如果发现页堆上的内存不足，会尝试通过 runtime.mheap.grow 扩容并重新调用 runtime.pageAlloc.alloc 申请内存；
   1. 如果申请到内存，意味着扩容成功；
   2. 如果没有申请到内存，意味着扩容失败，**宿主机可能不存在空闲内存**，运行时会直接中止当前程序；


## 内存分配
微对象 (0, 16B) — 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；
小对象 [16B, 32KB] — 依次尝试使用线程缓存、中心缓存和堆分配内存；
大对象 (32KB, +∞) — 直接在堆上分配内存；

### 小对象

> 小对象是指大小为 16 字节到 32,768 字节的对象以及所有小于 16 字节的指针类型的对象，小对象的分配可以被分成以下的三个步骤：
1. 确定分配对象的大小以及跨度类 runtime.spanClass；
2. 从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
3. 调用 runtime.memclrNoHeapPointers 清空空闲内存中的所有数据；

### 大对象 
运行时对于大于 32KB 的大对象会单独处理，我们不会从线程缓存或者中心缓存中获取内存管理单元

## 参考
[go-内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)
