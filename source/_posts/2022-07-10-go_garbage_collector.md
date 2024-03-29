---
title: go垃圾收集器
date: 2022-07-10 16:32:55
tags: 
 - gc
categories:
 - go
 - 内存
---
## 设计原理
### 标记清除
> 标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

1. 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
2. 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表；
* 问题：两个阶段都需要STW
<!-- more -->

### 三色抽象 
> 为了解决原始标记清除算法带来的长时间 STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短 STW 的时间。三色标记算法将程序中的对象分成白色、黑色和灰色三类：
* 白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收；
* 黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
* 灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；
![img.png](/images/go_garbage_collector/img.png)
  在垃圾收集器开始工作时，程序中不存在任何的黑色对象，**垃圾收集的根对象会被标记成灰色**，**垃圾收集器只会从灰色对象集合中取出对象开始扫描，当灰色集合中不存在任何对象时，标记阶段就会结束**。


三色标记垃圾收集器的工作原理很简单，我们可以将其归纳成以下几个步骤：

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
2. **将黑色对象指向的所有对象都标记成灰色**，保证该对象和被该对象引用的对象都不会被回收；
3. 重复上述两个步骤直到对象图中不存在灰色对象；

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象**，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾**，
* 问题: 因为用户程序可能在标记执行的过程中修改对象的指针，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW

### 屏障技术

> 内存屏障技术是一种屏障指令，它可以让 CPU 或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，
但是该技术能够保证内存操作的顺序性，**在内存屏障前执行的操作一定会先于内存屏障后执行的操作**。
 想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性（Tri-color invariant）中的一种：

1. 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
2. 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径7；
![img_1.png](/images/go_garbage_collector/img_1.png)

上图分别展示了遵循强三色不变性和弱三色不变性的堆内存，遵循上述两个不变性中的任意一个，我们都能保证垃圾收集算法的正确性，而屏障技术就是在并发或者增量标记过程中保证三色不变性的重要技术。

垃圾收集中的屏障技术更像是一个钩子方法，**它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码**，根据操作类型的不同，我们可以将它们分成读屏障（Read barrier）和写屏障（Write barrier）两种，
因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以**编程语言往往都会采用写屏障保证三色不变性**。

#### Dijkstra 插入写屏障
``` go 
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```
每当执行类似 *slot = ptr 的表达式时，我们会执行上述写屏障通过 shade 函数尝试改变指针的颜色。**如果 ptr 指针是白色的，那么该函数会将该对象设置成灰色**，其他情况则保持不变。
![img_2.png](/images/go_garbage_collector/img_2.png)

Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将有存活可能的对象都标记成灰色以满足强三色不变性。

它也有明显的缺点。因为**栈上的对象在垃圾收集中也会被认为是根对象**，所以为了保证内存的安全，Dijkstra **必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描**，

#### Yuasa 删除写屏障 
一旦该写屏障开始工作，它会保证开启写屏障时堆上所有对象的可达，所以也被称作快照垃圾收集（Snapshot GC）
```go
writePointer(slot, ptr)
    shade(*slot)
    *slot = ptr
```
上述代码会在老对象的引用被删除时，**将白色的老对象涂成灰色**，这样删除写屏障就可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用。

![img_3.png](/images/go_garbage_collector/img_3.png)


假设我们在应用程序中使用 Yuasa 提出的删除写屏障，在一个垃圾收集器和用户程序交替运行的场景中会出现如上图所示的标记过程：

1. 垃圾收集器将根对象指向 A 对象标记成黑色并将 A 对象指向的对象 B 标记成灰色；
2. 用户程序将 A 对象原本指向 B 的指针指向 C，触发删除写屏障，但是因为 B 对象已经是灰色的，所以不做改变；
3. **用户程序将 B 对象原本指向 C 的指针删除，触发删除写屏障，白色的 C 对象被涂成灰色；**
4. 垃圾收集器依次遍历程序中的其他灰色对象，将它们分别标记成黑色；
上述过程中的第三步触发了 Yuasa 删除写屏障的着色，因为用户程序删除了 B 指向 C 对象的指针，所以 C 和 D 两个对象会分别违反强三色不变性和弱三色不变性：

* 强三色不变性 — 黑色的 A 对象直接指向白色的 C 对象；
* 弱三色不变性 — 垃圾收集器无法从某个灰色对象出发，经过几个连续的白色对象访问白色的 C 和 D 两个对象；
**Yuasa 删除写屏障通过对 C 对象的着色，保证了 C 对象和下游的 D 对象能够在这一次垃圾收集的循环中存活**，避免发生悬挂指针以保证用户程序的正确性。

### 增量和并发 
为了减少应用程序暂停的最长时间和垃圾收集的总暂停时间，我们会使用下面的策略优化现代的垃圾收集器：

* 增量垃圾收集 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；
  * 增量式的垃圾收集需要与三色标记法一起使用，为了保证垃圾收集的正确性，我们需要在垃圾收集开始前**打开写屏障**，这样用户程序修改内存都会先经过写屏障的处理，保证了堆内存中对象关系的强三色不变性或者弱三色不变性。
* 并发垃圾收集 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；
  * **通过开启读写屏障**、**利用多核优势与用户程序并行执行**

## 演进过程
最开始的垃圾收集器是不精确的单线程 STW 收集器，最新版本的垃圾收集器支持**并发垃圾收集、去中心化协调**等特性

### 并发垃圾收集
> Go 语言在 v1.5 中引入了并发的垃圾收集器，该垃圾收集器使用了我们上面提到的**三色抽象和写屏障技术**保证垃圾收集器执行的正确性


Go 语言的并发垃圾收集器会在扫描对象之前暂停程序做一些标记对象的准备工作，其中包括**启动后台标记的垃圾收集器以及开启写屏障**

### 回收堆目标

Go 语言 v1.5 引入并发垃圾收集器的同时使用**垃圾收集调步（Pacing）算法计算触发的垃圾收集的最佳时间**，**确保触发的时间既不会浪费计算资源，也不会超出预期的堆大小**。
![img_4.png](/images/go_garbage_collector/img_4.png)

### 混合写屏障
在 Go 语言 v1.7 版本之前，运行时会使用 **Dijkstra 插入写屏障保证强三色不变性**，但是运行时**并没有在所有的垃圾收集根对象上开启插入写屏障**。
因为应用程序可能包含成百上千的 Goroutine，而垃圾收集的根对象一般包括全局变量和**栈对象**，如果运行时需要在几百个 Goroutine 的栈上都开启写屏障，会带来巨大的额外开销，
所以 Go 团队在实现上选择了**在标记阶段完成时暂停程序、将所有栈对象标记为灰色并重新扫描**

Go 语言在 v1.8 **组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障**构成了如下所示的混合写屏障，**该写屏障会将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色**：
```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```
为了移除栈的重扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，**我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收**，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。

## 实现原理
Go 语言的垃圾收集可以分成清除终止、标记、标记终止和清除四个不同阶段，它们分别完成了不同的工作
![img.png](/images/go_garbage_collector/img_5.png)
1. 清理终止阶段；
   1. **暂停程序**，所有的处理器在这时会进入安全点（Safe point）；
   2. 如果当前垃圾收集循环是强制触发的，我们还需要处理还未被清理的内存管理单元；
2. 标记阶段；
   1. 将状态切换至 _**GCmark**、**开启写屏障**、**用户程序协助**（Mutator Assists）并将根对象入队；
   2. 恢复执行程序，标记进程和用于协助的用户程序会开始并发标记内存中的对象，写屏障会将被覆盖的指针和新指针都标记成灰色，**而所有新创建的对象都会被直接标记成黑色**；
   3. 开始扫描根对象，包括所有 Goroutine 的栈、全局对象以及不在堆中的运行时数据结构，扫描 Goroutine 栈期间会暂停当前处理器；
   4. 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色；
   5. 使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段；
3. 标记终止阶段；
   1. **暂停程序**、将状态切换至 _**GCmarktermination** 并关闭辅助标记的用户程序；
   2. 清理处理器上的线程缓存；
4. 清理阶段；
   1. 将状态切换至 _**GCoff** 开始清理阶段，初始化清理状态并**关闭写屏障；**
   2. **恢复用户程序，所有新创建的对象会标记成白色**；
   3. 后台并发清理所有的内存管理单元，**当 Goroutine 申请新的内存管理单元时就会触发清理（惰性清除）**；
运行时虽然只会使用 _GCoff、_GCmark 和 _GCmarktermination 三个状态表示垃圾收集的全部阶段，但是在实现上却复杂很多

### 触发时机
![img_1.png](/images/go_garbage_collector/img_6.png)
#### 后台触发

gcTriggerTime — 如果一定时间内没有触发，就会触发新的循环，该触发条件由 runtime.forcegcperiod 变量控制，默认为 2 分钟；

#### 手动触发 
用户程序会通过 runtime.GC 函数在程序运行期间主动通知运行时执行，该方法在调用时会阻塞调用方直到当前垃圾收集循环完成，在垃圾收集期间也可能会通过 STW 暂停整个程序：

#### 申请内存 
1. 当前线程的内存管理单元中不存在空闲空间时，创建微对象和小对象需要调用 runtime.mcache.nextFree **从中心缓存或者页堆中获取新的管理单元，在这时就可能触发垃圾收集**；
2. **当用户程序申请分配 32KB 以上的大对象时，一定会构建 runtime.gcTrigger 结构体尝试触发垃圾收集**；

### 垃圾收集启动
1. 两次调用 runtime.gcTrigger.test 检查是否满足垃圾收集条件；
2. **暂停程序**、在后台**启动用于处理标记任务的工作 Goroutine**、确定所有内存管理单元都被清理以及其他标记阶段开始前的准备工作；
3. 进入标记阶段、准备后台的标记工作、根对象的标记工作以及微对象、恢复用户程序，进入并发扫描和标记阶段；

### 并发扫描与标记辅助
1. 获取当前处理器以及 Goroutine 打包成runtime.gcBgMarkWorkerNode 类型的结构并主动陷入休眠等待唤醒；
2. 根据处理器上的 gcMarkWorkerMode 模式决定扫描任务的策略；
3. 所有标记任务都完成后，调用 runtime.gcMarkDone 方法完成标记阶段；

#### 工作池
![img_2.png](/images/go_garbage_collector/img_7.png)
4. 写屏障、根对象扫描和栈扫描都会向工作池中增加额外的灰色对象等待处理，而对象的扫描过程会将灰色对象标记成黑色，同时也可能发现新的灰色对象，当工作队列中不包含灰色对象时，整个扫描过程就会结束。
5. 为了减少锁竞争，运行时在每个处理器上会保存独立的待扫描工作，然而这会遇到与调度器一样的问题 — 不同处理器的资源不平均，导致部分处理器无事可做，调度器引入了工作窃取来解决这个问题，垃圾收集器也使用了差不多的机制平衡不同处理器上的待处理任务。

#### 写屏障
```azure
if writeBarrier.enabled {
  gcWriteBarrier(ptr, val)
} else {
  *ptr = val
}
```
`gcWriteBarrier` 该函数会覆盖原来的值并通过 runtime.wbBufFlush **通知垃圾收集器将原值和新值加入当前处理器的工作队列**

#### 标记辅助 
为了保证用户程序分配内存的速度不会超出后台任务的标记速度，运行时还引入了标记辅助技术，它遵循一条非常简单并且朴实的原则，**分配多少内存就需要完成多少标记任务**。
![img_3.png](/images/go_garbage_collector/img_8.png)
用户程序辅助标记的核心目的是避免用户程序分配内存影响垃圾收集器完成标记工作的期望时间，它通过维护账户体系保证用户程序不会对垃圾收集造成过多的负担，一旦用户程序分配了大量的内存，该用户程序就会通过辅助标记的方式平衡账本，这个过程会在最后达到相对平衡，保证标记任务在到达期望堆大小时完成。

### 标记终止

### 内存清理
> 用户程序在申请内存时才会惰性回收内存。
> 垃圾收集的清理中包含**对象回收器（Reclaimer）和内存单元回收器**，这两种回收器使用不同的算法清理堆内存：

1. 对象回收器在内存管理单元中查找并释放未被标记的对象，**但是如果 runtime.mspan 中的所有对象都没有被标记，整个单元就会被直接回收**，该过程会被 runtime.mcentral.cacheSpan 或者 runtime.sweepone 异步触发；
2. 内存单元回收器会在内存中查找所有的对象都未被标记的 runtime.mspan，该过程会被 runtime.mheap.reclaim 触发；

## 总结
Go 语言为了实现高性能的并发垃圾收集器，使用**三色抽象、并发增量回收、混合写屏障、调步算法以及用户程序协助**等机制将垃圾收集的暂停时间优化至毫秒级以下，从早期的版本看到今天，我们能体会到其中的工程设计和演进，作者觉得研究垃圾收集的是实现原理还是非常值得的。
## 参考
[go垃圾收集器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collecto)