## Golang GC and sync.Pool

## GC

### what is GC?

垃圾回收（英语：Garbage Collection，缩写为GC），在计算机科学中是一种自动的存储器管理机制。当一个计算机上的动态存储器不再需要时，就应该予以释放，以让出存储器，这种存储器资源管理，称为垃圾回收。垃圾回收器可以让程序员减轻许多负担，也减少程序员犯错的机会。

简单地说，**垃圾回收(GC)是在后台运行一个守护线程，它的作用是在监控各个对象的状态，识别并且丢弃不再使用的对象来释放和重用内存资源。**



### GC触发时机

GC 触发的场景主要分为两大类，分别是：

1. 系统触发：运行时自行根据内置的条件，检查、发现到，则进行 GC 处理，维护整个应用程序的可用性。
2. 手动触发：开发者在业务代码中自行调用 `runtime.GC` 方法来触发 GC 行为。

在系统触发的场景中，Go 源码的 `src/runtime/mgc.go` 文件，明确标识了 GC 系统触发的三种场景，分别如下：

```go
const (
 gcTriggerHeap gcTriggerKind = iota
 gcTriggerTime
 gcTriggerCycle
)
```

- gcTriggerHeap：当所分配的堆大小达到阈值（由控制器计算的触发堆的大小）时，将会触发。
- gcTriggerTime：当距离上一个 GC 周期的时间超过一定时间时，将会触发。-时间周期以 `runtime.forcegcperiod` 变量为准，默认 2 分钟。
- gcTriggerCycle：如果没有开启 GC，则启动 GC。
  - 在手动触发的 `runtime.GC` 方法中涉及。

### Golang 三色标记法

golang 的垃圾回收(GC)是基于标记清扫算法，这种算法需要进行 STW（stop the world)，这个过程就会导致程序是卡顿的，频繁的 GC 会严重影响程序性能.

GO STW的影响比较小，是因为**Golang GC的大部分处理是和用户代码并行的**。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Golang-GC-step.jpg)



Go 语言使用三色抽象作为其并发标记的实现，首先要理解三种颜色抽象：

- 黑：已经扫描完毕，子节点扫描完毕。(gcmarkbits = 1，且在队列外)
- 灰：已经扫描完毕，子节点未扫描完毕。(gcmarkbits = 1, 在队列内)
- 白：未扫描，collector 不知道任何相关信息。

三色标记法相对于普通标记清扫，减少了 STW 时间. 这主要得益于标记过程是 “on-the-fly” 的，在标记过程中是不需要 STW 的，它与程序是并发执行的，这就大大缩短了 STW 的时间.

GC工作的完整流程：

1. Mark: 包含两部分:

- Mark Prepare: 初始化GC任务，包括开启写屏障(write barrier)和辅助GC(mutator assist)，统计root对象的任务数量等。**这个过程需要STW**
- GC Drains: 扫描所有root对象，包括全局指针和goroutine(G)栈上的指针（扫描对应G栈时需停止该G)，将其加入标记队列(灰色队列)，并循环处理灰色队列的对象，直到灰色队列为空。**该过程后台并行执行**

1. Mark Termination: 完成标记工作，重新扫描(re-scan)全局指针和栈。因为Mark和用户程序是并行的，所以在Mark过程中可能会有新的对象分配和指针赋值，这个时候就需要通过写屏障（write barrier）记录下来，re-scan 再检查一下。**这个过程也是会STW的。**
2. Sweep: 按照标记结果回收所有的白色对象，**该过程后台并行执行**
3. Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC。

如果标记期间用户逻辑改变了刚打完标记的对象的引用状态，怎么办呢？



#### 写屏障 write barrier

barrier 本质是 : snippet of code insert before pointer modify。

在应用进入 GC 标记阶段前的 stw 阶段，会将全局变量 runtime.writeBarrier.enabled 修改为 true，这时所有的堆上指针修改操作在修改之前便会额外调用 runtime.gcWriteBarrier

早期 Go 只使用了 Dijistra 屏障，但因为会有上述对象丢失问题，需要在第二个 stw 周期进行栈重扫(stack rescan)。当 goroutine 数量较多时，会造成 stw 时间较长。

想要消除栈重扫，但单独使用任意一种 barrier 都没法满足 Go 的要求，所以最新版本中 Go 的混合屏障其实是 Dijistra Insertion Barrier  + Yuasa Deletion Barrier。

> 单个write barrier可能导致对象丢失...

#### 协助标记

当应用分配内存过快时，后台的 mark worker 无法及时完成标记工作，这时应用本身需要进行堆内存分配时，会判断是否需要适当协助 GC 的标记过程，防止应用因为分配过快发生 OOM。

协助标记会对应用的响应延迟产生影响，可以尝试降低应用的对象分配数量进行优化。



#### 内存碎片

Go 语言使用了并发标记与清扫算法作为其 GC 实现，并发标记清扫算法无法解决内存碎片问题，而 tcmalloc 恰好一定程度上缓解了内存碎片问题，两者配合使用相得益彰。

### 回收流程

相比复杂的标记流程，对象的回收和内存释放就简单多了。

进程启动时会有两个特殊 goroutine：

- 一个叫 sweep.g，主要负责清扫死对象，合并相关的空闲页
- 一个叫 scvg.g，主要负责向操作系统归还内存

### GC总结

Go 语言垃圾回收的关键点：

- 无分代
- 与应用执行并发
- 协助标记流程
- 并发执行时开启 write barrier

因为无分代，当我们遇到一些需要在内存中保留几千万 kv map 的场景(比如机器学习的特征系统)时，就需要想办法降低 GC 扫描成本。

因为有协助标记，当应用的 GC 占用的 CPU 超过 25% 时，会触发大量的协助标记，影响应用的延迟，这时也要对 GC 进行优化。

简单的业务使用 sync.Pool 就可以带来较好的优化效果，若碰到一些复杂的业务场景，还要考虑 offheap 之类的欺骗 GC 的方案，比如 [dgraph 的方案](https://dgraph.io/blog/post/manual-memory-management-golang-jemalloc/)。

## GC 优化

### 分代：GC分代收集时选择不同算法

就JAVA来举例，JAVA GC堆内存分为了老年代和新生代。

如果要选择算法，那么一定得从它们的本质去入手。

* 老年代：存活数量多，需要被处理的对象少

* 新生代：存活数量少，需要被处理的对象多

复制算法因为每次复制的都是存活的对象，而新生代的存活对象都是比较少的，所以这个时候就可以采用复制算法来实现。

也就是说，新生代中划分出来的大Eden 区 和两个 Survivor区，每次使用的时候都是 Eden 区和其中的一块 Survivor 区，当进行回收时，将该两块空间中还存活的对象复制到另一块 Survivor 区中。

所以因为新生代的这种特性，所以使用复制算法。

而老年代因为每次只回收少量对象，因而采用 Mark-Compact 算法。

**So，Golang GC不分代，当内存中保留几千万 kv map 的场景时，Golang只能采取MARK算法，GC成本就会很大。**

### sync.Pool优化GC:maple_leaf:

一句话总结：保存和复用临时对象，减少内存分配，降低 GC 压力。

Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector. That is, it makes it easy to build efficient, thread-safe free lists. However, it is not suitable for all free lists. A Pool is safe for use by multiple goroutines simultaneously.

sync.Pool 是一个并发安全的缓存池，能够并发且安全地存储、获取元素/对象。常用于对象实例创建会占用较多资源的场景。但是它不具有严格的缓存作用，因为 Pool 中的元素/对象的释放时机是随机的。
 作为缓存的一种姿势，sync.Pool 能够避免元素/对象的申请内存操作和初始化过程，以提高性能。当然，这里有个 trick，释放元素/对象的操作是直接将元素/对象放回池子，从而免去了真正释放的操作。
 另外，不考虑内存浪费和初始化消耗的情况下，“使用 sync.Pool 管理多个对象”和“直接 New 多个对象”两者的区别在于后者会创建出更多的对象，并发高时会给 GC 带来非常大的负担，进而影响整体程序的性能。因为 Go 申请内存是程序员触发的，而回收却是 Go 内部 runtime GC 回收器来执行的。即，使用 sync.Pool 还可以减少 GC 次数。

#### 使用场景

1、增加临时对象的重用率（sync.Pool 本质用途）。在高并发业务场景下出现 GC 问题时，可以使用 sync.Pool 减少 GC 负担。

2、不适合存储带状态的对象，因为获取对象是随机的（Get 到的对象可能是刚创建的，也可能是之前创建并 cache 住的），并且缓存对象的释放策略完全由 runtime 内部管理。像 socket 长连接、数据库连接，有人说适合有人说不适合。

3、不适合需要控制缓存元素个数的场景，因为无法知道 Pool 池里面的对象个数，以及对象的释放时机也是未知的。

4、典型的应用场景之一是在网络包收取发送的时候，用 sync.Pool 会有奇效，可以大大降低 GC 压力。

#### Pool new()并发安全

没有配置 New 方法时，如果 Get 操作多于 Put 操作，继续 Get 会得到一个 nil interface{} 对象，所以需要代码进行兼容。
 配置 New 方法后，Get 获取不到对象时（Pool 池中已经没有对象了），会调用自定义的 New 方法创建一个对象并返回。

```go
// A Pool must not be copied after first use.
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
```

sync.Pool 本身数据结构是并发安全的，但是 Pool.New 函数（用户自定义的）不一定是线程安全的。并且 Pool.New 函数可能会被并发调用，如果 New 函数里面的实现逻辑是非并发安全的，那就会有问题。

```go

// 用来统计实例真正创建的次数
var numCalcsCreated int32

// 创建实例的函数
func createBuffer() interface{} {
    // 这里要注意下，非常重要的一点。这里必须使用原子加，不然有并发问题；
    atomic.AddInt32(&numCalcsCreated, 1)
    buffer := make([]byte, 1024)
    return &buffer
}

func main() {
    // 创建实例
    bufferPool := &sync.Pool{
        New: createBuffer,
    }
}
```



#### demo

https://geektutu.com/post/hpg-sync-pool.html

```bash
$ mv sync_pool.go sync_pool_test.go
$ go test -bench .
goos: linux
goarch: amd64
pkg: bench/test
cpu: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
BenchmarkUnmarshal-2                8922            131496 ns/op
BenchmarkUnmarshalWithPool-2        9127            131004 ns/op
PASS
ok      bench/test      2.414s
$ go test -bench . -benchmem
goos: linux
goarch: amd64
pkg: bench/test
cpu: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
## 内存差了不少--
BenchmarkUnmarshal-2         8485      131982 ns/op     1392 B/op          8 allocs/op
BenchmarkUnmarshalWithPool-2    8816   130281 ns/op     240 B/op          7 allocs/op
PASS
ok      bench/test      2.306s

```



#### 思考总结

1、Pool 是如何实现并发安全的？
 某一时刻 P 只会调度一个 G，那么对于生产者而言，当前 P 操作的都是本 P 的 poolLocal。相当于是通过隔离避免了数据竞争，即调用 pushHead 和 popHead 并不需要加锁。

当消费者是其他 P（窃取时），会进行 popTail 操作，这时会和 pushHead/popHead 操作形成数据竞争。pushHead 的流程是先取 slot，再判断是否可插入，最后修改 headTail；而 popTail 的流程是先修改headTail，再取 slot，然后重置 slot。pushHead 修改 head 位置，popTail修改 tail 位置，并且都是用的公共字段 headTail，这样的话使用 CAS 原子操作就可以避免读写冲突。

2、pinSlow() 函数中为什么先执行了 runtime_procUnpin，随后又执行了 runtime_procPin？
 目的是避免获取不到全局锁 allPoolsMu 却一直占用当前 P 的情况。而 runtime_procUnpin 和 runtime_procPin 之间，本 G 可能已经切换到其他 P 上，而 []poolLocal 也可能已经被初始化。

3、为什么需要将 poolDequeue 串成链表？
 因为 poolDequeue 的实现是固定大小的。

4、为什么要禁止 copy sync.Pool 实例？
 因为 copy 后，对于同一个 Pool 实例中的 cache 对象，就有了两个指向来源。原 Pool 清空之后，copy 的 Pool 没有清理掉，那么里面的对象就全都泄露了。并且 Pool 的无锁设计的基础是多个 Goroutine 不会操作到同一个数据结构，Pool 拷贝之后则不能保证这点（因为存储的成员都是指针）。





### dgraph优化GC



### 引用

1. https://segmentfault.com/a/1190000020102375
2. https://www.yisu.com/zixun/451546.html
3. https://xargin.com/impl-of-go-gc/
4. https://juejin.cn/post/6989798306440282148
5. https://geektutu.com/post/hpg-sync-pool.html
6. https://cloud.tencent.com/developer/article/1847917
