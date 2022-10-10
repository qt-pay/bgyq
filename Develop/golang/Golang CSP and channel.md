## Golang **CSP** and Channel

Each channel internally holds a mutex lock which is used to avoid data races in all kinds of operations.

### 大佬笔记

https://halfrost.com/go_channel/

### 饥饿 和 锁

线程需要资源没有拿到，无法进行下一步，就是饥饿。死锁（Deadlock）和活锁（Livelock）都是饥饿的一种形式。 非抢占的系统中，互斥的资源获取，形成循环依赖就会产生死锁。死锁发生后，如果利用抢占解决，导致资源频繁被转让，有一定概率触发活锁。死锁、活锁，都可以通过设计并发控制算法解决，比如哲学家就餐问题。

活锁、死锁本质上是一样的，原因是在获取临界区资源时，并发多个进程/线程声明资源占用(加锁)的顺序不一致，死锁是加不上就死等，活锁是加不上就放开已获得的资源，然后稍后重试。

#### 死锁条件

死锁的四个条件是：

- **禁止抢占**（no preemption）：系统资源不能被强制从一个线程中退出。如果哲学家可以抢夺，那么大家都去抢别人的筷子，也会打破死锁的局面，但这是有风险的，因为可能孔子还没吃饭就被老子抢走了。计算机资源如果不是主动释放而是被抢夺有可能出现意想不到的现象。
- **持有和等待**（hold and wait）：一个线程在等待时持有并发资源。持有并发资源并还等待其它资源。
- **互斥**（mutual exclusion）：资源只能同时分配给一个线程，无法多个线程共享。资源具有排他性。
- **循环等待**（circular waiting）：一系列线程互相持有其他进程所需要的资源。必须有一个循环依赖的关系。

死锁只有在四个条件同时满足时发生，预防死锁必须至少破坏其中一项。

### Hoare  I/O

C.A.R Hoare 1978 的萌芽论文，认为输入输出在一种模型编程语言中是基本要素。

What turned out after reading the CSP original [paper](https://dl.acm.org/citation.cfm?doid=359576.359585) was nothing fancy — a simple concept (which was later formulated into the [Process calculus](https://en.wikipedia.org/wiki/Process_calculus) to reason about the program correctness) solving concurrency through two primitives of programming.

1. **Input.**
2. **Output.**

The paper abbreviates the term *processes* to any individual logic that needs input to run and produce output. (You can visualize this as`Goroutine`)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/process-in-csp.png)

### GCL

The **Guarded Command Language** (**GCL**) is a language defined by Edsger Dijkstra for predicate transformer semantics.Its simplicity makes proving the correctness of programs easier, using Hoare logic.

```bash
## 伪代码
if error = True then x := 0

## 使用守卫命令语言
if error = true → x := 0
|  error = false → skip
fi
```

#### Syntax语法

A guarded command is a statement of the form G → S, where

- G is a proposition, called the guard
- S is a statement

If G is true, the guarded command may be written simply S.

#### Semantics语义

At the moment G is encountered in a calculation, it is evaluated.

- If G is true, execute S
- If G is false, look at the context to decide what to do (in any case, do not execute S)

### CSP

#### 定义

CSP 是 Communicating Sequential Process 的简称，中文可以叫做**通信顺序进程**，是一种并发编程模型，是一个很强大的并发数据模型，是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。

 **CSP 模型**，即在通信双方抽象出中间层，数据的流转由中间层来控制，通信双方只负责数据的发送和接收，从而实现了数据的共享，这就是所谓的**通过通信来共享内存**。

CSP 的模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信， 这里发送消息时使用的就是通道（channel）。也就是我们常说的 『 Don't communicate by sharing memory; share memory by communicating 』。

channel 在多并发操作里是属于协程安全的，并且遵循了 **FIFO** 特性。即先执行读取的 goroutine 会先获取到数据，先发送数据的 goroutine 会先输入数据。

#### 设计细节

CSP 语言的结构非常简单，极度的数学形式化、简洁与优雅。 

将Hoare I/O、GCL和自己声明运算符融合。

**Multiple concurrent processes are allowed to synchronize(\*communicate via named sources and destinations\*) with each other by synchronizing their I/O. The paper describes this with commands.**

* `!` for sending input into a process
* `?` for reading the output from a process

The main Concept CSP describe is Synchronization and Guarded command.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Sync-two-Process-communicating-under-CSP.png)

In the above example of Synchronization:

\1. Process **P1** output value of “a” via output command (!) to Process **P2**.
\2. Process **P2** input value from Process **P2** via input command (?) and assign it to “x”.

### Golang channel

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-inherently-interesting.jpg)

* goroutine-safe: hchan mutex
* store values, pass in FIFO: copying into and out of hchan buffer
* can cause goroutines to pause and resume:
  * hchan sudog queues
  * calls into the runtime scheduler(gopark, goread)

Do you find some similarity with Go’s channels? Apparently, it’s is. Though solutions in Go are a bit longer, **Hoare’s I/O commands** with **Dijkstra Guarded Command** forms the foundation stone of Go’s channels.

```go
ch <- v    // Send v to channel ch.
v := <-ch  // Receive from ch, and
           // assign value to v.
```

Golang，其实只用到了 CSP 的很小一部分，即理论中的 Process/Channel（对应到语言中的 goroutine/channel）：这两个并发原语之间没有从属关系， Process 可以订阅任意个 Channel，Channel 也并不关心是哪个 Process 在利用它进行通信；Process 围绕 Channel 进行读写，形成一套有序阻塞和可预测的并发模型。

#### channel数据通讯概述

简单来说，CSP 模型由并发执行的实体（线程或者进程或者协程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。

CSP 模型的关键是关注 channel，而不关注发送消息的实体。Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。

Channel 在 gouroutine 间架起了一条管道，在管道里传输数据，实现 gouroutine 间的通信；由于它是线程安全的，所以用起来非常方便；channel 还提供“先进先出”的特性；它还能影响 goroutine 的阻塞和唤醒。

G1 负责给ch写数据，G2负责读数据。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-communication-1.jpg)

G1交接收到的数据直接写入到recv的内存地址

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-2.jpg)

基于值拷贝，实现G1和G2数值传递

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-3.jpg)

简单明了

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-4.jpg)



综合来说，CSP 1978 中描述的编程语言（与 Go 所构建的基于通道的 channel/select 同步机制进行对比）：

1. channel 没有被显式命名
2. channel 没有缓冲，对应 Go 中 unbuffered channel
3. buffered channel 不是一种基本的编程源语，并展示了一个使用 unbuffered channel 实现其作用的例子
4. guarded command 等价于 if 与 select 语句的组合，分支的随机触发性是为了提供公平性保障
5. guarded command 没有对确定性分支选择与非确定性（即多个分支有效时随机选择）分支选择进行区分
6. repetitive command 的本质是一个无条件的 for 循环，但终止性所要求的条件太苛刻，不利于理论的进一步形式化
7. CSP 1978 中描述的编程语言对程序终止性的讨论几乎为零
8. 此时与 Actor 模型进行比较，CSP 与 Actor 均在实体间直接通信，区别在于 Actor 支持异步消息通信，而 CSP 1978 是同步通信

#### struct hchan

channel 的底层源码和相关实现在 src/runtime/chan.go 中。

```go
type hchan struct {
	qcount   uint           // 队列中所有数据总数
	dataqsiz uint           // 环形队列的 size
	buf      unsafe.Pointer // 指向 dataqsiz 长度的数组
	elemsize uint16         // 元素大小
	closed   uint32
	elemtype *_type         // 元素类型
	sendx    uint           // 已发送的元素在环形队列中的位置
	recvx    uint           // 已接收的元素在环形队列中的位置
	recvq    waitq          // 接收者的等待队列
	sendq    waitq          // 发送者的等待队列

	lock mutex
}

// 排他锁
// Mutual exclusion locks.  In the uncontended case,
// as fast as spin locks (just a few user-level instructions),
// but on the contention path they sleep in the kernel.
// A zeroed Mutex is unlocked (no need to initialize each lock).
// Initialization is helpful for static lock ranking, but not required.
type mutex struct {
	// Empty struct if lock ranking is disabled, otherwise includes the lock rank
	lockRankStruct
	// Futex-based impl treats it as uint32 key,
	// while sema-based impl as M* waitm.
	// Used to be a union, but unions break precise GC.
	key uintptr
}
```

lock 锁保护 hchan 中的所有字段，以及此通道上被阻塞的 sudogs 中的多个字段。持有 lock 的时候，禁止更改另一个 G 的状态（特别是不要使 G 状态变成ready），因为这会因为堆栈 shrinking 而发生死锁。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-hchan-struct.png)

recvq 和 sendq 是等待队列，waitq 是一个双向链表：

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

end

##### chan 缓冲区

hchan内部实现了一个环形队列作为缓冲区，队列的长度是创建channel时指定的。下图展示了一个可缓存6个元素的channel的示意图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-buffer.png)



- dataqsiz指示队列长度为6，即可缓存6个元素
- buf指向队列的内存，队列中还剩余两个元素
- qcount表示队列中还有两个元素
- sendx指示后续写入的数据存储的位置，取值[0,6)
- recvx指示从该位置读取数据，取值[0,6)

##### chan 队列

sendx 和 sendq对应

从channel读消息，如果channel缓冲区为空或者没有缓存区，当前goroutine会被阻塞。
向channel读消息，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞。

被阻塞的goroutine将会封装成sudog，加入到channel的等待队列中：

- 因读消息阻塞的goroutine会被channel向channel写入数据的goroutine唤醒
- 因写消息阻塞的goroutine会从channel读消息的goroutine唤醒

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel queue.png)

一般情况下，recvq和sendq至少一个为空，只有一个例外，即同一个goroutine使用select语句向channel一边写数据，一个读数据。

#### sudog

channel 最核心的数据结构是 sudog。

sudog看goroutine那个笔记吧。

### channel lock:feet:

**Each channel internally holds a mutex lock which is used to avoid data races in all kinds of operations.**

The following will make more explanations for the four cases shown with superscripts (A, B, C and D).

To better understand channel types and values, and to make some explanations easier, looking in the raw internal structures of internal channel objects is very helpful.

We can think of each channel consisting of three queues (all can be viewed as FIFO queues) internally:

1. the receiving goroutine queue (generally FIFO). The queue is a linked list without size limitation. Goroutines in this queue are all in blocking state and waiting to receive values from that channel.
2. the sending goroutine queue (generally FIFO). The queue is also a linked list without size limitation. Goroutines in this queue are all in blocking state and waiting to send values to that channel. The value (or the address of the value, depending on compiler implementation) each goroutine is trying to send is also stored in the queue along with that goroutine.
3. the value buffer queue (absolutely FIFO). This is a circular queue. Its size is equal to the capacity of the channel. The types of the values stored in this buffer queue are all the element type of that channel. If the current number of values stored in the value buffer queue of the channel reaches the capacity of the channel, the channel is called in full status. If no values are stored in the value buffer queue of the channel currently, the channel is called in empty status. For a zero-capacity (unbuffered) channel, it is always in both full and empty status.

Each channel internally holds a mutex lock which is used to avoid data races in all kinds of operations.

**Channel operation case A**: when a goroutine `R` tries to receive a value from a not-closed non-nil channel

, **the goroutine `R` will acquire the lock associated with the channel firstly**, then do the following steps until one condition is satisfied.

1. If the value buffer queue of the channel is not empty, in which case the receiving goroutine queue of the channel must be empty, the goroutine `R` will receive (by shifting) a value from the value buffer queue. If the sending goroutine queue of the channel is also not empty, a sending goroutine will be shifted out of the sending goroutine queue and resumed to running state again. The value that the just shifted sending goroutine is trying to send will be pushed into the value buffer queue of the channel. The receiving goroutine `R` continues running. For this scenario, the channel receive operation is called a **non-blocking operation**.
2. Otherwise (the value buffer queue of the channel is empty), if the sending goroutine queue of the channel is not empty, in which case the channel must be an unbuffered channel, the receiving goroutine `R` will shift a sending goroutine from the sending goroutine queue of the channel and receive the value that the just shifted sending goroutine is trying to send. The just shifted sending goroutine will get unblocked and resumed to running state again. The receiving goroutine `R` continues running. For this scenario, the channel receive operation is called a **non-blocking operation**.
3. If value buffer queue and the sending goroutine queue of the channel are both empty, the goroutine `R` will be pushed into the receiving goroutine queue of the channel and enter (and stay in) blocking state. It may be resumed to running state when another goroutine sends a value to the channel later. For this scenario, the channel receive operation is called a **blocking operation**.

**Channel rule case B**: when a goroutine `S` tries to send a value to a not-closed non-nil channel

, the goroutine `S` will acquire the lock associated with the channel firstly, then do the following steps until one step condition is satisfied.

1. If the receiving goroutine queue of the channel is not empty, in which case the value buffer queue of the channel must be empty, the sending goroutine `S` will shift a receiving goroutine from the receiving goroutine queue of the channel and send the value to the just shifted receiving goroutine. The just shifted receiving goroutine will get unblocked and resumed to running state again. The sending goroutine `S` continues running. For this scenario, the channel send operation is called a **non-blocking operation**.
2. Otherwise (the receiving goroutine queue is empty), if the value buffer queue of the channel is not full, in which case the sending goroutine queue must be also empty, the value the sending goroutine `S` trying to send will be pushed into the value buffer queue, and the sending goroutine `S` continues running. For this scenario, the channel send operation is called a **non-blocking operation**.
3. If the receiving goroutine queue is empty and the value buffer queue of the channel is already full, the sending goroutine `S` will be pushed into the sending goroutine queue of the channel and enter (and stay in) blocking state. It may be resumed to running state when another goroutine receives a value from the channel later. For this scenario, the channel send operation is called a **blocking operation**.

Above has mentioned, once a non-nil channel is closed, sending a value to the channel will produce a runtime panic in the current goroutine. Note, sending data to a closed channel is viewed as a **non-blocking operation**.

**Channel operation case C**: when a goroutine tries to close a not-closed non-nil channel

, once the goroutine has acquired the lock of the channel, both of the following two steps will be performed by the following order.

1. If the receiving goroutine queue of the channel is not empty, in which case the value buffer of the channel must be empty, all the goroutines in the receiving goroutine queue of the channel will be shifted one by one, each of them will receive a zero value of the element type of the channel and be resumed to running state.
2. If the sending goroutine queue of the channel is not empty, all the goroutines in the sending goroutine queue of the channel will be shifted one by one and each of them will produce a panic for sending on a closed channel. This is the reason why we should avoid concurrent send and close operations on the same channel. In fact, when [the go command's data race detector option](https://golang.org/doc/articles/race_detector.html) (`-race`) is enabled, concurrent send and close operation cases might be detected at run time and a runtime panic will be thrown.

Note: after a channel is closed, the values which have been already pushed into the value buffer of the channel are still there. Please read the closely following explanations for case D for details.

**Channel operation case D**: after a non-nil channel is closed, channel receive operations on the channel will never block. The values in the value buffer of the channel can still be received. The accompanying second optional bool return values are still `true`. Once all the values in the value buffer are taken out and received, infinite zero values of the element type of the channel will be received by any of the following receive operations on the channel. As mentioned above, the optional second return result of a channel receive operation is an untyped boolean value which indicates whether or not the first result (the received value) is sent before the channel is closed. If the second return result is `false`, then the first return result (the received value) must be a zero value of the element type of the channel.

### sync and channel

Go 中的并发原语主要分为 2 大类，一个是 sync 包里面的，另一个是 channel。sync 包里面主要是 WaitGroup，互斥锁和读写锁，cond，once，sync.Pool 这一类。

 在 Go 语言的官方 FAQ 中，描述了如何选择这些并发原语：

> 为了尊重 mutex，sync 包实现了 mutex，但是我们希望 Go 语言的编程风格将会激励人们尝试更高等级的技巧。尤其是考虑构建你的程序，以便一次只有一个 goroutine 负责某个特定的数据

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sync-and-channel.png)

#### sync应用场景

在 2 种情况下推荐使用 sync 包：

- 对性能要求极高的临界区
- 保护某个结构内部状态和完整性

关于保护某个结构内部的状态和完整性。例如 Go 源码中如下代码：

```go
var sum struct {
	sync.Mutex
	i int
}

//export Add
func Add(x int) {
	defer func() {
		recover()
	}()
	sum.Lock()
	sum.i += x
	sum.Unlock()
	var p *int
	*p = 2
}
```

sum 这个结构体不想将内部的变量暴露在结构体之外，所以使用 sync.Mutex 来保护线程安全。

#### channel应用场景: 并发本质？

多个goroutines时，一个goroutine使用channel时会lock。

channel 也有 2 种情况：

- 输出数据给其他使用方
- 组合多个逻辑

输出数据给其他使用方的目的是转移数据的使用权。并发安全的实质是保证同时只有一个并发上下文拥有数据的所有权。channel 可以很方便的将数据所有权转给其他使用方。另一个优势是组合型。如果使用 sync 里面的锁，想实现组合多个逻辑并且保证并发安全，是比较困难的。但是使用 channel + select 实现组合逻辑实在太方便了。



### channel发送数据:+1:

#### send异常检查

chansend() 函数一开始先进行异常检查：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 判断 channel 是否为 nil
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// 简易快速的检查
	if !block && c.closed == 0 && full(c) {
		return false
	}
......
}
```

chansend() 一上来对 channel 进行检查，如果被 GC 回收了会变为 nil。朝一个为 nil 的 channel 发送数据会发生阻塞。gopark 会引发以 waitReasonChanSendNilChan 为原因的休眠，并抛出 unreachable 的 fatal error。当 channel 不为 nil，再开始检查在没有获取锁的情况下会导致发送失败的非阻塞操作。

当 channel 不为 nil，并且 channel 没有 close 时，还需要检查此时 channel 是否做好发送的准备，即判断 full(c)

```go
// full reports whether a send on c would block (that is, the channel is full).
// It uses a single word-sized read of mutable state, so although
// the answer is instantaneously true, the correct answer may have changed
// by the time the calling function receives the return value.
func full(c *hchan) bool {
	if c.dataqsiz == 0 {
		// 假设指针读取是近似原子性的
		return c.recvq.first == nil
	}
	// 假设读取 uint 是近似原子性的
	return c.qcount == c.dataqsiz
}
```

full() 方法作用是判断在 channel 上发送是否会阻塞（即通道已满）。它读取单个字节大小的可变状态(recvq.first 和 qcount)，尽管答案可能在一瞬间是 true，但在调用函数收到返回值时，正确的结果可能发生了更改即它刚检测到chan is full，其他goroutine又从chan读取了一个数据，此时chan is not full，需要等待下次检测。值得注意的是 dataqsiz 字段，它在创建完 channel 以后是不可变的，因此它可以安全的在任意时刻读取。

> 那么多个goroutine对一个channel写数据，肯定存在g_1刚检测channel不满，准备发数据，此时G_2抢先一步写入数据，G_1将何去何从？
>
> 真正发送前应该再检查一下吧--

回到 chansend() 异常检查中。一个已经 close 的 channel 是不可能从“准备发送”的状态变为“未准备好发送”的状态。所以在检查完 channel 是否 close 以后，就算 channel close 了，也不影响此处检查的结果。可能有读者疑惑，“能不能把检查顺序倒一倒？先检查是否 full()，再检查是否 close？”。这样倒过来确实能保证检查 full() 的时候，channel 没有 close。但是这种做法也没有实质性的改变。channel 依旧可以在检查完 close 以后再关闭。其实我们依赖的是 chanrecv() 和 closechan() 这两个方法在锁释放后，它们更新这个线程 c.close 和 full() 的结果视图。

#### 同步发送

channel 异常状态检查以后，接下来的代码是发送的逻辑

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
......
    // Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
    if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// 如果channel 关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

......

}
```

在发送之前，先上锁，保证线程安全。并再一次检查 channel 是否关闭。如果关闭则抛出 panic。加锁成功并且 channel 未关闭，开始发送。如果有正在阻塞等待的接收方，通过 dequeue() 取出头部第一个非空的 sudog，调用 send() 函数:

```go
// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

send() 函数主要完成了 2 件事：

- 调用 sendDirect() 函数将数据拷贝到了接收变量的内存地址上
- 调用 goready() 将等待接收的阻塞 goroutine 的状态从 Gwaiting 或者 Gscanwaiting 改变成 **Grunnable**。下一轮调度时会唤醒这个接收的 goroutine。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel同步发送.png)

可以看下goready() 的实现。理解了它的源码，就能明白为什么往 channel 中发送数据并非立即可以从接收方获取到。



#### 异步发送

如果初始化 channel 时创建的带缓冲区的异步 Channel，当接收者队列为空时，这是会进入到异步发送逻辑：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
......

	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	
......
}
```

如果 qcount 还没有满，则调用 chanbuf() 获取 sendx 索引的元素指针值。调用 typedmemmove() 方法将发送的值拷贝到缓冲区 buf 中。拷贝完成，需要维护 sendx 索引下标值和 qcount 个数。这里将 buf 缓冲区设计成环形的，索引值如果到了队尾，下一个位置重新回到队头。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel异步发送.png)

#### 阻塞发送

当 channel 处于打开状态，但是没有接收者，并且没有 buf 缓冲队列或者 buf 队列已满，这时 channel 会进入阻塞发送。

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
......

	if !block {
		unlock(&c.lock)
		return false
	}
	
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	KeepAlive(ep)
......
}
```

- 调用 getg() 方法获取当前 goroutine 的指针，用于绑定给一个 sudog。
- 调用 acquireSudog() 方法获取一个 sudog，可能是新建的 sudog，也有可能是从缓存中获取的。设置好 sudog 要发送的数据和状态。比如发送的 Channel、是否在 select 中和待发送数据的内存地址等等。
- 调用 c.sendq.enqueue 方法将配置好的 sudog 加入待发送的等待队列。
- 设置原子信号。当栈要 shrink 收缩时，这个标记代表当前 goroutine 还 parking 停在某个 channel 中。在 g 状态变更与设置 activeStackChans 状态这两个时间点之间的时间窗口进行栈 shrink 收缩是不安全的，所以需要设置这个原子信号。
- 调用 gopark 方法挂起当前 goroutine，状态为 waitReasonChanSend，阻塞等待 channel。
- 最后，KeepAlive() 确保发送的值保持活动状态，直到接收者将其复制出来。 sudog 具有指向堆栈对象的指针，但 sudog 不能被当做堆栈跟踪器的 root。发送的数值是分配在堆上，这样可以避免被 GC 回收。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel阻塞发送.png)

chansend() 函数最后返回 true 表示成功向 Channel 发送了数据。

#### channel发送总结

 channel 各个状态做一个小结。

|       |               Channel Status               |    Result     |
| :---: | :----------------------------------------: | :-----------: |
| Write |                    nil                     |     阻塞      |
| Write | 打开但填满（无缓存channel为0，写入即阻塞） |     阻塞      |
| Write |                 打开但未满                 |  成功写入值   |
| Write |                    关闭                    |   **panic**   |
| Write |                    只读                    | Compile Error |

channel 发送过程中包含 2 次有关 goroutine 调度过程：

- 当接收队列中存在 sudog 可以直接发送数据时，执行 `goready()`将 g 插入 runnext 插槽中，状态从 Gwaiting 或者 Gscanwaiting 改变成 Grunnable，等待下次调度便立即运行。
- 当 channel 阻塞时，执行 `gopark()` 将 g 阻塞，让出 cpu 的使用权。

**需要强调的是，**通道并不提供跨 goroutine 的数据访问保护机制。如果通过通道传输数据的一份副本，那么每个 goroutine 都持有一份副本，各自对自己的副本做修改是安全的。当传输的是指向数据的指针时，如果读和写是由不同的 goroutine 完成的，那么每个 goroutine 依旧需要额外的同步操作。

### channel接收数据:+1:

2 种不同的 channel 接收方式会转换成 runtime.chanrecv1 和 runtime.chanrecv2 两种不同函数的调用，但是最终核心逻辑还是在 runtime.chanrecv 中。

#### recv异常检查

chanrecv() 函数一开始先进行异常检查：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 简易快速的检查
	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 {
			return
		}
		if empty(c) {
			// channel 不可逆的关闭了且为空
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}
```

chanrecv() 一上来对 channel 进行检查，如果被 GC 回收了会变为 nil。从一个为 nil 的 channel 中接收数据会发生阻塞。gopark 会引发以 waitReasonChanReceiveNilChan 为原因的休眠，并抛出 unreachable 的 fatal error。当 channel 不为 nil，再开始检查在没有获取锁的情况下会导致接收失败的非阻塞操作。

这里进行的简易快速的检查，检查中状态不能发生变化。这一点和 chansend() 函数有区别。在 chansend() 简易快速的检查中，改变顺序对检查结果无太大影响，但是此处如果检查过程中状态发生变化，如果发生了 racing，检查结果会出现完全相反的错误的结果。例如以下这种情况：channel 在第一个和第二个 if 检查时是打开的且非空，于是在第二个 if 里面 return。但是 return 的瞬间， channel 关闭且空。这样判断出来认为 channel 是打开的且非空。明显是错误的结果，实际上 channel 是关闭且空的。同理检查是否为空的时候也会发生状态反转。为了防止错误的检查结果，c.closed 和 empty() 都必须使用原子检查。

```go
func empty(c *hchan) bool {
	// c.dataqsiz 是不可变的
	if c.dataqsiz == 0 {
		return atomic.Loadp(unsafe.Pointer(&c.sendq.first)) == nil
	}
	return atomic.Loaduint(&c.qcount) == 0
}
```

这里总共检查了 2 次 empty()，因为第一次检查时， channel 可能还没有关闭，但是第二次检查的时候关闭了，在 2 次检查之间可能有待接收的数据到达了。所以需要 2 次 empty() 检查。

不过就算按照上述源码检查，细心的读者可能还会举出一个反例，例如，关闭一个已经阻塞的同步的 channel，最开始的 !block && empty(c) 为 false，会跳过这个检查。这种情况不能算在正常 chanrecv() 里面。上述是不获取锁的情况检查会接收失败的情况。接下来在获取锁的情况下再次检查一遍异常情况。

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......
	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
......
```

如果 channel 已经关闭且不存在缓存数据了，则清理 ep 指针中的数据并返回。这里也是从已经关闭的 channel 中读数据，读出来的是该类型零值的原因。

#### 同步接收

同 chansend 逻辑类似，检查完异常情况，紧接着是同步接收。

```go
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......

	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
......
```

在 channel 的发送队列中找到了等待发送的 goroutine。取出队头等待的 goroutine。如果缓冲区的大小为 0，则直接从发送方接收值。否则，对应缓冲区满的情况，从队列的头部接收数据，发送者的值添加到队列的末尾（此时队列已满，因此两者都映射到缓冲区中的同一个下标）。同步接收的核心逻辑见下面 recv() 函数：

```go
// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// 从 sender 里面拷贝数据
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
	    // 这里对应 buf 满的情况
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
		}
		// 将数据从 buf 中拷贝到接收者内存地址中
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 将数据从 sender 中拷贝到 buf 中
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

需要注意的是由于有发送者在等待，所以**如果存在缓冲区，那么缓冲区一定是满的**。这个情况对应发送阶段阻塞发送的情况，如果缓冲区还有空位，发送的数据直接放入缓冲区，只有当缓冲区满了，才会打包成 sudog，插入到 sendq 队列中等待调度。注意理解这一情况。

接收时主要分为 2 种情况，有缓冲且 buf 满和无缓冲的情况：

- 无缓冲。ep 发送数据不为 nil，调用 recvDirect() 将发送队列中 sudog 存储的 ep 数据直接拷贝到接收者的内存地址中。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel同步接收.png)

- 有缓冲并且 buf 满。有 2 次 copy 操作，先将队列中 recvx 索引下标的数据拷贝到接收方的内存地址，再将发送队列头的数据拷贝到缓冲区中，释放一个 sudog 阻塞的 goroutine。

  有缓冲且 buf 满的情况需要注意，取数据从缓冲队列头取出，发送的数据放在队列尾部，由于 buf 装满，取出的 recvx 指针和发送的 sendx 指针指向相同的下标。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel同步接收-2.png)

  最后调用 goready() 将等待接收的阻塞 goroutine 的状态从 Gwaiting 或者 Gscanwaiting 改变成 Grunnable。下一轮调度时会唤醒这个发送的 goroutine。这部分逻辑和同步发送中一致，关于 goready() 底层实现的代码不在赘述。



#### 异步接收

如果 Channel 的缓冲区中包含一些数据时，从 Channel 中接收数据会直接从缓冲区中 recvx 的索引位置中取出数据进行处理：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......

	if c.qcount > 0 {
		// 直接从队列中接收
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}
......
```

上述代码比较简单，如果接收数据的内存地址 ep 不为空，则调用 runtime.typedmemmove() 将缓冲区内的数据拷贝到内存中，并通过 typedmemclr() 清除队列中的数据

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-异步接收.png)

维护 recvx 下标，如果移动到了环形队列的队尾，下标需要回到队头。最后减少 qcount 计数器并释放持有 Channel 的锁。



#### 阻塞接收

如果 channel 发送队列上没有待发送的 goroutine，并且缓冲区也没有数据时，将会进入到最后一个阶段阻塞接收：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......

	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
......
```

- 调用 getg() 方法获取当前 goroutine 的指针，用于绑定给一个 sudog。
- 调用 acquireSudog() 方法获取一个 sudog，可能是新建的 sudog，也有可能是从缓存中获取的。设置好 sudog 要发送的数据和状态。比如发送的 Channel、是否在 select 中和待发送数据的内存地址等等。
- 调用 c.recvq.enqueue 方法将配置好的 sudog 加入待发送的等待队列。
- 设置原子信号。当栈要 shrink 收缩时，这个标记代表当前 goroutine 还 parking 停在某个 channel 中。在 g 状态变更与设置 activeStackChans 状态这两个时间点之间的时间窗口进行栈 shrink 收缩是不安全的，所以需要设置这个原子信号。
- 调用 gopark 方法挂起当前 goroutine，状态为 waitReasonChanReceive，阻塞等待 channel。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel阻塞接收.png)

上面这段代码与 chansend() 中阻塞发送几乎完全一致，区别在于最后一步没有 KeepAlive(ep)。

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......

	// 被唤醒
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

goroutine 被唤醒后会完成 channel 的阻塞数据接收。接收完最后进行基本的参数检查，解除 channel 的绑定并释放 sudog。

#### channel接收总结

关于 channel 接收的源码实现已经分析完了，针对 channel 各个状态做一个小结。

|      | Channel status |     Result      |
| :--: | :------------: | :-------------: |
| Read |      nil       |      阻塞       |
| Read |   打开且非空   |    读取到值     |
| Read |   打开但为空   |      阻塞       |
| Read |      关闭      | <默认值>, false |
| Read |      只读      |  Compile Error  |

chanrecv 的返回值有几种情况：

```go
tmp, ok := <-ch
```

|   Channel status   | Selected | Received |
| :----------------: | :------: | :------: |
|        nil         |  false   |  false   |
|     打开且非空     |   true   |   true   |
|     打开但为空     |  false   |  false   |
| 关闭且返回值是零值 |   true   |  false   |

received 值会传递给读取 channel 外部的 bool 值 ok，selected 值不会被外部使用。

channel 接收过程中包含 2 次有关 goroutine 调度过程：

1. 当 channel 为 nil 时，执行 gopark() 挂起当前的 goroutine。
2. 当发送队列中存在 sudog 可以直接接收数据时，执行 goready()将 g 插入 runnext 插槽中，状态从 Gwaiting 或者 Gscanwaiting 改变成 Grunnable，等待下次调度便立即运行。
3. 当 channel 缓冲区为空，且没有发送者时，这时 channel 阻塞，执行 gopark() 将 g 阻塞，让出 cpu 的使用权并等待调度器的调度。

### 优雅关闭channel:star2:

“Channel 有几种优雅的关闭方法？” 这种问题常常出现在面试题中，究其原因是因为 Channel 创建容易，但是关闭“不易”：

- 在不改变 Channel 自身状态的条件下，无法知道它是否已经关闭。“不易”之一，关闭时机未知。
- 如果一个 Channel 已经关闭，重复关闭 Channel 会导致 panic。“不易”之二，不能无脑关闭。
- 往一个 close 的 Channel 内写数据，也会导致 panic。“不易”之三，写数据之前也需要关注是否 close 的状态。

|       | Channel Status |                            Result                            |
| :---: | :------------: | :----------------------------------------------------------: |
| close |      nil       |                          **panic**                           |
| close |   打开且非空   | 关闭 Channel；读取成功，直到 Channel 耗尽数据，然后读取产生值的默认值 |
| close |   打开但为空   |               关闭 Channel；读到生产者的默认值               |
| close |      关闭      |                          **panic**                           |
| close |      只读      |                        Compile Error                         |

那究竟什么时候关闭 Channel 呢？由上面三个“不易”，可以浓缩为 2 点：

- 不能简单的从消费者侧关闭 Channel。
- 如果有多个生产者，它们不能关闭 Channel。

解释一下这 2 个问题。第一个问题，消费者不知道 Channel 何时该关闭。如果关闭了已经关闭的 Channel 会导致 panic。而且分布式应用通常有多个消费者，每个消费者的行为一致，这么多消费者都尝试关闭 Channel 必然会导致 panic。第二个问题，如果有多个生产者往 Channel 内写入数据，这些生产者的行为逻辑也都一致，如果其中一个生产者关闭了 Channel，其他的生产者还在往里写，这个时候会 panic。所以为了防止 panic，必须解决上面这 2 个问题。

关闭 Channel 的方式就 2 种：

- Context
- done channel

Context 的方式在本篇文章不详细展开，这里使用 done channel 的做法。假设有多个生产者，有多个消费者。在生产者和消费者之间增加一个额外的辅助控制 channel，用来传递关闭信号。

```go
type session struct {
	done     chan struct{}
	doneOnce sync.Once
	data     chan int
}

func (sess *session) Serve() {
	go sess.loopRead()
	sess.loopWrite()
}

func (sess *session) loopRead() {
	defer func() {
		if err := recover(); err != nil {
			sess.doneOnce.Do(func() { close(sess.done) })
		}
	}()

	var err error
	for {
		select {
		case <-sess.done:
			return
		default:
		}

		if err == io.ErrUnexpectedEOF || err == io.EOF {
			goto failed
		}
	}
failed:
	sess.doneOnce.Do(func() { close(sess.done) })
}

func (sess *session) loopWrite() {
	defer func() {
		if err := recover(); err != nil {
			sess.doneOnce.Do(func() { close(sess.done) })
		}
	}()

	var err error
	for {
		select {
		case <-sess.done:
			return
		case sess.data <- rand.Intn(100):
		}
		
		if err != nil {
			goto done
		}
	}
done:
	if err != nil {
		log("sess: loop write failed: %v, %s", err, sess)
	}
}
```

消费者侧发送关闭 done channel，由于消费者有多个，如果每一个都关闭 done channel，会导致 panic。所以这里用 doneOnce.Do() 保证只会关闭 done channel 一次。这解决了第一个问题。生产者收到 done channel 的信号以后自动退出。多个生产者退出时间不同，但是最终肯定都会退出。当生产者全部退出以后，data channel 最终没有引用，会被 gc 回收。这也解决了第二个问题，生产者不会去关闭 data channel，防止出现 panic。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/how-to-close-channel.png)

总结一下 done channel 的做法：消费者利用辅助的 done channel 发送信号，并先开始退出协程。生产者接收到 done channel 的信号，也开始退出协程。最终 data channel 无人持有，被 gc 回收关闭。

#### context and channel

**官方推荐通过 `context`配合来关闭chan（主要）**

我们换一个思路，你其实并不是一定要判断 channel 是否 close，真正的目的是：安全的使用 channel，避免使用到已经关闭的 closed channel，从而导致 panic 。

这个问题的本质上是保证一个事件的时序，官方推荐通过 `context` 来配合使用，我们可以通过一个 ctx 变量来指明 close 事件，而不是直接去判断 channel 的一个状态。举个栗子：

```go
select {
case <-ctx.Done():
    // ... exit
    return
case v, ok := <-c:
    // do something....
default:
    // do default ....
}
```

`ctx.Done()` 事件发生之后，我们就明确不去读 channel 的数据。

或者

```perl
select {
case <-ctx.Done():
    // ... exit
    return
default:
    // push 
    c <- x
}
```

`ctx.Done()` 事件发生之后，我们就明确不写数据到 channel ，或者不从 channel 里读数据，那么保证这个时序即可。就一定不会有问题。

我们只需要确保一点：触发时序保证：一定要先触发 ctx.Done() 事件，再去做 close channel 的操作，保证这个时序的才能保证 select 判断的时候没有问题；只有这个时序，才能保证在获悉到 Done 事件的时候，一切还是安全的。

**上述的顺序指的是触发顺序，而不是select里case的顺序，case是没有顺序的，他是随机的。**



### 如何优雅关闭channel：参考二

特别注意：

1. 接收者关闭channel要小心，因为关闭后发送者继续发送会panic
2. 当有多个发送者时，在一个发送者协程中关闭channel要小心，因为关闭后其他发送者继续发送会panic

复杂情况下的参考思路：

1. channel的关闭并非必须的，只要channel对象不再被持有，垃圾回收器会清理它
2. 可使用原子变量等同步原语保证close有且只有发生一次
3. 除了传输数据的channel，可以再增加channel配合select使用，用于取消生产、消费
4. 接收端也可以通过其他channel发出消息，反向通知发送端
5. 

#### 通道关闭原则

一般原则上使用通道是不允许接收方关闭通道和不能关闭一个有多个并发发送者的通道。 换而言之， 你只能在发送方的 goroutine 中关闭只有该发送方的通道。

当然，对于关闭通道来说这不是一个普遍的原则。 普遍原则是不发送数据给（或关闭）一个关闭状态通道。 如果任意一个 goroutine 能保证没有 goroutine 会再发送数据给（或关闭）一个没有关闭不为空的通道， 这样的话 goruoutine 可以安全地关闭通道。然而，从通道接受方或者多个发送方之一实现这样的保证需要花费很大的工夫，而且通常会把代码写得很复杂。相反，遵守上文所提到的 通道关闭原则 更加简单。

#### panic-recover(不推荐)

关闭一个 channel 直接调用 close 即可，但是关闭一个已经关闭的 channel 会导致 panic，怎么办？panic-recover 配合使用即可。

```go
func SafeClose(ch chan int) (closed bool) {
 defer func() {
  if recover() != nil {
   closed = false
  }
 }()
 // 如果 ch 是一个已经关闭的，会 panic 的，然后被 recover 捕捉到；
 close(ch)
 return true
}
```

很不优雅

#### sync.Once

使用 `sync.Once` 去关闭通道

https://learnku.com/go/t/23459/how-to-close-the-channel-gracefully

#### sync.Mutex

使用 `sync.Mutex` 去避免多次关闭一个通道

#### 优雅关闭基本思路

1. M 个接受者，1 个发送者， 发送者通过关闭数据通道说 「不要再发送了」

这是最简单的情况，只需让发送者在不想再发送数据的时候关闭数据通道：

2. 1 个接收者，N 个发送者，接收者通过关闭一个信号通道说 「请不要再发送数据了」
这个情况比上面的要复杂一点。 我们不能让接收者关闭数据通道，for 不然就会违反了 通道关闭原则。但是我们可以让接收者关闭一个信号通道去通知接收者停止发送数据：

3. M 个接收者，N 个发送者， 其中，任意一个通过通知一个主持人去关闭一个信号通道说「让我们结束这场游戏吧」
    这是最复杂的情况。 我们不能让任意一个接收者或者发送者关闭数据通道。我们也不能让任意一个接收者通过关闭信号通道去通知所有的发送者和接收者退出这场游戏。 实现任意一个都会违反 通道关闭原则。 然而，我们可以创建一个主持人角色去关闭信号通道

  ```golang
  package main
  
  import (
      "time"
      "math/rand"
      "sync"
      "log"
      "strconv"
  )
  
  func main() {
      rand.Seed(time.Now().UnixNano())
      log.SetFlags(0)
  
      // ...
      const MaxRandomNumber = 100000
      const NumReceivers = 10
      const NumSenders = 1000
  
      wgReceivers := sync.WaitGroup{}
      wgReceivers.Add(NumReceivers)
  
      // ...
      dataCh := make(chan int, 100)
      stopCh := make(chan struct{})
          // stopCh 是一个信号通道。
          // 它的发送者是下面的主持人 goroutine。
          // 它的接收者是 dataCh的所有发送者和接收者。
      toStop := make(chan string, 1)
          // toStop 通道通常用来通知主持人去关闭信号通道( stopCh )。
          // 它的发送者是 dataCh的任意发送者和接收者。
          // 它的接收者是下面的主持人 goroutine
  
      var stoppedBy string
  
      // 主持人
      go func() {
          stoppedBy = <-toStop
          close(stopCh)
      }()
  
      // 发送者
      for i := 0; i < NumSenders; i++ {
          go func(id string) {
              for {
                  value := rand.Intn(MaxRandomNumber)
                  if value == 0 {
                      // 这里，一个用于通知主持人关闭信号通道的技巧。
                      select {
                      case toStop <- "sender#" + id:
                      default:
                      }
                      return
                  }
  
                  // 这里的第一个 select 是为了尽可能早的尝试退出 goroutine。
                  //这个 select 代码块有 1 个接受行为 的 case 和 一个将作为 Go 编译器的 try-receive 操作进行特别优化的默认分支。
                  select {
                  case <- stopCh:
                      return
                  default:
                  }
  
                  // 即使 stopCh 被关闭， 如果 发送到 dataCh 的操作没有阻塞，那么第二个 select 的第一个分支可能会在一些循环（和在理论上永远） 不会执行。
                  // 这就是为什么上面的第一个 select 代码块是必须的。
                  select {
                  case <- stopCh:
                      return
                  case dataCh <- value:
                  }
              }
          }(strconv.Itoa(i))
      }
  
      // 接收者
      for i := 0; i < NumReceivers; i++ {
          go func(id string) {
              defer wgReceivers.Done()
  
              for {
                  // 和发送者 goroutine 一样， 这里的第一个 select 是为了尽可能早的尝试退出 这个 goroutine。
                  // 这个 select 代码块有 1 个发送行为 的 case 和 一个将作为 Go 编译器的 try-send 操作进行特别优化的默认分支。
                  select {
                  case <- stopCh:
                      return
                  default:
                  }
  
                  //  即使 stopCh 被关闭， 如果从 dataCh 接受数据不会阻塞，那么第二个 select 的第一分支可能会在一些循环（和理论上永远）不会被执行到。
                  // 这就是为什么第一个 select 代码块是必要的。
                  select {
                  case <- stopCh:
                      return
                  case value := <-dataCh:
                      if value == MaxRandomNumber-1 {
                          // 同样的技巧用于通知主持人去关闭信号通道。
                          select {
                          case toStop <- "receiver#" + id:
                          default:
                          }
                          return
                      }
  
                      log.Println(value)
                  }
              }
          }(strconv.Itoa(i))
      }
  
      // ...
      wgReceivers.Wait()
      log.Println("stopped by", stoppedBy)
  }
  
  ```

  end





### 引用

1. https://golang.design/under-the-hood/zh-cn/part1basic/ch01basic/csp/
2. https://levelup.gitconnected.com/communicating-sequential-processes-csp-for-go-developer-in-a-nutshell-866795eb879d
3. https://en.wikipedia.org/wiki/Guarded_Command_Language
4. https://zhuanlan.zhihu.com/p/313763247
5. https://learnku.com/go/t/23459/how-to-close-the-channel-gracefully
6. https://blog.csdn.net/chushoufengli/article/details/114966927
7. https://go101.org/article/channel.html
8. https://ustack.io/2019-10-04-Golang%E6%BC%AB%E8%B0%88%E4%B9%8B%E7%BB%93%E6%9E%84%E4%BD%93.html