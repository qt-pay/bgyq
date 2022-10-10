## Golang Goroutine GPM模型

goroutine很特殊，它在系统分配的Threads上实现了自己的调度策略。

一个goroutine只占2KB的内存.

goroutine的退出机制设计是，goroutine退出只能由本身控制，不允许从外部强制结束该goroutine。只有两种情况例外，那就是main函数结束或者程序崩溃结束运行；所以，要实现主进程控制子goroutine的开始和结束，必须借助其它工具来实现。

大佬：https://zboya.github.io/post/go_scheduler/#%E6%B7%B1%E5%85%A5golang-runtime%E7%9A%84%E8%B0%83%E5%BA%A6

看：https://cloud.tencent.com/developer/article/1135275



### 调度器发展

#### 协程的优势

 Linux 操作系统来讲，cpu 对进程的态度和线程的态度是一样的，CPU 调度切换的是进程和线程。

进程/线程的创建、切换、销毁，都会占用很长的时间，从而消耗CPU性能资源。

多进程、多线程已经提高了系统的并发能力，但是在当今互联网高并发场景下，为每个任务都创建一个线程是不现实的，因为会消耗大量的内存 (进程虚拟内存会占用 4GB [32 位操作系统], 而线程也要大约 4MB)。

大量的进程 / 线程出现了新的问题

* 高内存占用
* 调度的高消耗 CPU

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/cpu-schedule.png)

一个 “用户态线程” 必须要绑定一个 “内核态线程”，但是 CPU 并不知道有 “用户态线程” 的存在，它只知道它运行的是一个 “内核态线程”(Linux 的 PCB 进程控制块)。

内核线程依然叫 “线程 (thread)”，用户线程叫 “协程 (co-routine)”.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/cpu-routine-1-1.png)

#### 协程与线程绑定

 协程跟线程是有区别的，线程由 CPU 调度是抢占式的，**协程由用户态调度是协作式的**，一个协程让出 CPU 后，才执行下一个协程。

既然一个协程 (co-routine) 可以绑定一个线程 (thread)，那么能不能多个协程 (co-routine) 绑定一个或者多个线程 (thread) 上呢。

* 线程与协程，1:1没意义。协程切换还是线程切换
* 线程与协程，1:N ，可能性能不行
* 线程与协程，M:N，效率和性能都可以

1:N关系

N 个协程绑定 1 个线程，优点就是协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速。但也有很大的缺点，1 个进程的所有协程都绑定在 1 个线程上

缺点：

* 某个程序用不了硬件的多核加速能力
* 一旦某协程阻塞，造成线程阻塞，本进程的其他协程都无法执行了，根本就没有并发的能力了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/cpu-routine-1-N.png)

M:N模型，借助调度器可以实现多核并发。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/cpu-M-N.png)



### 大佬

https://halfrost.com/go_channel/

### 最简单的goroutine+channel

想测试下golang channel随手写了下面代码，请问为什么会死锁？

```go
package main

import "fmt"

func main()  {
	ch := make(chan string)
	ch <- "test"
	fmt.Println(<-ch)

}
```

死锁提示：goroutine 1 [chan send]

#### v1: 还是死锁

```go
package main
import "fmt"

func main()  {
	ch := make(chan string)
	ch <- "test"
	go func(){
		fmt.Println(<-ch)
	}()
}
```

死锁提示：goroutine 1 [chan send]

这是因为 ch是一个无缓存的channel，当我们向它存入值的时候，会阻塞所在goroutine，这里阻塞的当然是main.main的goroutine，所以后面的语句根本不会执行到。

#### v2: 不死锁但没输出

```go
package main

import "fmt"

func main()  {
	ch := make(chan string)
	//ch <- "test"
	go func(){
		fmt.Println(<-ch)
	}()
	ch <- "test"
}
```

请稍微思考下

#### v3: <-chan

```go
package main

import (
	"fmt"
	"time"
)

func main()  {
	ch := make(chan string)
	//ch <- "test"
	go func(){
		fmt.Println(<-ch)
	}()
	ch <- "test"
	time.Sleep(time.Second)
}
```

输出: test

### GM模型：废弃deprecated

M 想要执行、放回 G 都必须访问全局 G 队列，并且 M 有多个，即多线程访问同一资源需要加锁进行保证互斥 / 同步，所以全局 G 队列是有互斥锁进行保护的。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/GM模型-deprecated.png)

老调度器有几个缺点：

1. 创建、销毁、调度 G 都需要每个 M 获取锁，这就形成了**激烈的锁竞争**。
2. M 转移 G 会造成**延迟和额外的系统负载**。比如当 G 中包含创建新协程的时候，M 创建了 G’，为了继续执行 G，需要把 G’交给 M’执行，也造成了很差的局部性，因为 G’和 G 是相关的，最好放在 M 上执行，而不是其他 M’。**即M没有自己的本地队列**
3. 系统调用 (CPU 在 M 之间的切换) 导致频繁的线程阻塞和取消阻塞操作增加了系统开销。



### GPM模型：P引入本地Queue

G 来表示 Goroutine，用 M即Machine，或 worker thread，代表系统线程，

**Processor，它包含了运行 goroutine 的资源**，如果线程想运行 goroutine，必须先获取 P，P 中还包含了可运行的 G 队列。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/GPM-arch.png)

在 Go 中，**线程是运行 goroutine 的实体，调度器的功能是把可运行的 goroutine 分配到工作线程上**。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/GPM-schedule.jpeg)

1. **全局队列（Global Queue）**：存放等待运行的 G。
2. **P 的本地队列**：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G’时，G’优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
3. **P 列表**：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
4. **M**：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

Goroutine 调度器和 OS 调度器是通过 M 结合起来的，每个 M 都代表了 1 个内核线程，OS 调度器负责把内核线程分配到 CPU 的核上执行。

### GPM调度策略

**复用线程**：避免频繁的创建、销毁线程，而是对线程的复用。

1）work stealing 机制

 当本线程(M)无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。

2）hand off 机制

 当本线程(M)因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

**利用并行**：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度，比如 GOMAXPROCS = 核数/2，则最多利用了一半的 CPU 核进行并行。

**抢占**：在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方。

**全局 G 队列**：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。

#### 调度场景举例

 假定当前除了 M3 和 M4 为自旋线程，还有 M5 和 M6 为空闲的线程 (没有得到 P 的绑定，注意我们这里最多就只能够存在 4 个 P，所以 P 的数量应该永远是 M>=P, 大部分都是 M 在抢占需要运行的 P)，G8 创建了 G9，G8 进行了阻塞的系统调用，M2 和 P2 立即解绑，P2 会执行以下判断：如果 P2 本地队列有 G、全局队列有 G 或有空闲的 M，P2 都会立马唤醒 1 个 M 和它绑定，否则 P2 则会加入到空闲 P 列表，等待 M 来获取可用的 p。本场景中，P2 本地队列有 G9，可以和其他空闲的线程 M5 绑定。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/G阻塞和M调度.png)

#### goroutine调度流程

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-schedule-workflow.jpeg)

从上图我们可以分析出几个结论：

 1、我们通过 go func () 来创建一个 goroutine；

 2、有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

 3、G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想其他的 MP 组合偷取一个可执行的 G 来执行；

 4、一个 M 调度 G 执行的过程是一个循环机制；

 5、当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

 6、当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。



### G

goroutines are user-space threads.

goroutines created and managed by the Go runtime， not the    OS. ligthweight compared to OS threads.



`G` 代表 goroutine，即用户创建的 goroutines

Goroutine is stackful coroutine. https://jiajunhuang.com/articles/2018_04_03-coroutine.md.html

#### G0

每个`M`都有一个`G0-goroutine`，`G0`负责调度goroutines, goroutine切换时，都会使用`G0`的stack即公共栈.

 `G0`仅负责调度`G`，而`G`则负责执行代码func.

G0 是每次启动一个 M 都会第一个创建的 goroutine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0。

#### goroutine state

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine_state.jpg)



### M

Machine，或 worker thread，代表系统线程。

`M`是操作系统线程。在绝大多数时候，`P` 的数量和 `M` 的数量是相等的。每创建一个 `P`, 就会创建一个对应的 `M`.

在OS中，进程是资源分配单位，线程是最小的调度执行单位，这是大前提。所以，进程中至少有一个线程。

而协程则是只在用户态利用事件机制实现切换的低成本并发手段。

自旋线程会抢占P，但是不抢占G。

#### M0

Golang进程刚刚启动时，会启动一个M0，它完成初始化工作后就是正常的M了。

#### M数量

`M`数量设置：

* go 语言本身的限制：go 程序启动时，会设置 M 的最大数量，默认 10000. 但是内核很难支持这么多的线程数，所以这个限制可以忽略。
* runtime/debug 中的 SetMaxThreads 函数，设置 M 的最大数量
* 一个 M 阻塞了，会创建新的 M。

M 何时创建：没有足够的 M 来关联 P 并运行其中的可运行的 G。比如所有的 M 此时都阻塞住了，而 P 中还有很多就绪任务，就会去寻找空闲的 M，而没有空闲的，就会去创建新的 M。

### P

`P` 代表 Logical Processor，是类似于 CPU 核心的概念，其用来控制并发的 M 数量.

一个 M 只有绑定 P 才能执行 goroutine，当 M 被阻塞时，整个 P 会被传递给其他 M ，或者说整个 P 被接管。

#### P数量

由启动时环境变量 $GOMAXPROCS 或者是由 runtime 的方法 GOMAXPROCS() 决定。这意味着在程序执行的任意时刻都只有 $GOMAXPROCS 个 goroutine 在同时运行。

#### container P number

https://pandaychen.github.io/2020/02/28/GOMAXPROCS-POT/

```bash
# golang
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Printf("NumCPU: %d, GOMAXPROCS: %d\n", runtime.NumCPU(), runtime.GOMAXPROCS(-1))
}
## 代码在容器中读到的还是 host的全部cpu
[root@dodo awesomeProject]# kt exec -it echo-group3147-847c585846-d7h6n -- sh
/takumi # /tmp/lol
NumCPU: 32, GOMAXPROCS: 32

## 加入 uber的库
[root@dodo awesomeProject]# cat lol.go
package main

import (
        "fmt"
        "runtime"
        // 很强
        _ "go.uber.org/automaxprocs"
)

func main() {
        fmt.Printf("NumCPU: %d, GOMAXPROCS: %d\n", runtime.NumCPU(), runtime.GOMAXPROCS(-1))
}[root@dodo awesomeProject]#go build -o lol lol.go
[root@dodo awesomeProject]# kt cp ./lol echo-group3147-847c585846-d7h6n:/tmp/
[root@dodo awesomeProject]# kt exec -it echo-group3147-847c585846-d7h6n -- sh
/takumi # /tmp/lol
2021/03/21 11:50:04 maxprocs: Updating GOMAXPROCS=1: determined from CPU quota
NumCPU: 32, GOMAXPROCS: 1
/takumi #

```

end

### 调度器

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/调度器生命周期.png)



### gopark：pausing goroutines

gopark函数在协程的实现上扮演着非常重要的角色，用于协程的切换，协程切换的原因一般有以下几种情况：

* 系统调用；
* channel读写条件不满足；
* 抢占式调度时间片结束；

gopark函数做的主要事情分为两点：

* 解除当前goroutine的m的绑定关系，将当前goroutine状态机切换为等待状态；
* 调用一次schedule()函数，在局部调度器P发起一轮新的调度。

#### g waiting

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gopark-waiting.jpg)

#### g scheduler

**This is neat.**

**G1 is blocked as needed, but not the OS thread.**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gopark-schedule.jpg)



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-resume-1.jpg)



### goready: resume

```go
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}

func ready(gp *g, traceskip int, next bool) {
......

	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	wakep()
	releasem(mp)
}
```

在 runqput() 函数的作用是把 g 绑定到本地可运行的队列中。此处 next 传入的是 true，将 g 插入到 runnext 插槽中，等待下次调度便立即运行。因为这一点导致了虽然 goroutine 保证了线程安全，但是在读取数据方面比数组慢了几百纳秒。

|    Read     |       Channel        | Slice |
| :---------: | :------------------: | :---: |
|    Time     | x * 100 * nanosecond |   0   |
| Thread safe |         Yes          |  No   |

所以在写测试用例的某些时候，需要考虑到这个微弱的延迟，可以适当加 sleep()。再比如刷 LeetCode 题目的时候，并非无脑使用 goroutine 就能带来 runtime 的提升，例如 [509. Fibonacci Number](https://leetcode.com/problems/fibonacci-number/)，感兴趣的同学可以用 goroutine 来写一写这道题，笔者这里实现了[goroutine 解法](https://books.halfrost.com/leetcode/ChapterFour/0500~0599/0509.Fibonacci-Number/)，性能方面完全不如数组的解法



goready函数相比gopark函数来说简单一些，主要功能就是唤醒某一个goroutine，该协程转换到runnable的状态，并将其放入P的local queue，等待调度。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-goready-waiting.jpg)

set runnable

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-set-runnable.jpg)

runnable queue

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-runnable-queue.jpg)

G2 读取完channle数据，也被gopark设置为waiting

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-running-to-waiting.jpg)

创建sudog

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-waiting-recv.jpg)



### sudog:没看懂

The general consensus is that a `sudog` is a pseudo-`G`, as it is used to keep a list of G's. 

channel 最核心的数据结构是 sudog

sudog represents a g in a wait list, such as for sending/receiving on a channel.

sudog is necessary because the g ↔ synchronization object relation is many-to-many. A g can be on many wait lists, so there may be many sudogs for one g; and many gs may be waiting on the same synchronization object, so there may be many sudogs for one object.

sudogs are allocated from a special pool. Use `acquireSudog` and `releaseSudog` to allocate and free them.

sudog 代表了一个在等待队列中的 g。sudog 是 Go 中非常重要的数据结构，因为 g 与同步对象关系是多对多的。一个 g 可以出现在许多等待队列上，因此一个 g 可能有很多sudog。并且多个 g 可能正在等待同一个同步对象，因此一个对象可能有许多 sudog。sudog 是从特殊池中分配出来的。使用 acquireSudog 和 releaseSudog 分配和释放它们。

```go
type sudog struct {

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // 指向数据 (可能指向栈)

	acquiretime int64
	releasetime int64
	ticket      uint32

	isSelect bool
	success bool

	parent   *sudog     // semaRoot 二叉树
	waitlink *sudog     // g.waiting 列表或者 semaRoot
	waittail *sudog     // semaRoot
	c        *hchan     // channel
}
```

该结构中主要的就是一个g和一个elem。elem用于存储goroutine的数据。读通道时，数据会从Hchan的队列中拷贝到SudoG的elem域。写通道时，数据则是由SudoG的elem域拷贝到Hchan的队列中。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-sudog-struct.png)



sudog 中所有字段都受 hchan.lock 保护。acquiretime、releasetime、ticket 这三个字段永远不会被同时访问。对 channel 来说，waitlink 只由 g 使用。对 semaphores 来说，只有在持有 semaRoot 锁的时候才能访问这三个字段。isSelect 表示 g 是否被选择，g.selectDone 必须进行 CAS 才能在被唤醒的竞争中胜出。success 表示 channel c 上的通信是否成功。如果 goroutine 在 channel c 上传了一个值而被唤醒，则为 true；如果因为 c 关闭而被唤醒，则为 false。

> CAS,compare and swap,一种无锁算法。



**sudog 的二级缓存复用体系**。在 acquireSudog() 方法中：

```go
func acquireSudog() *sudog {
	mp := acquirem()
	pp := mp.p.ptr()
	// 如果本地缓存为空
	if len(pp.sudogcache) == 0 {
		lock(&sched.sudoglock)
		// 首先尝试将全局中央缓存存一部分到本地
		for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
			s := sched.sudogcache
			sched.sudogcache = s.next
			s.next = nil
			pp.sudogcache = append(pp.sudogcache, s)
		}
		unlock(&sched.sudoglock)
		// 如果全局中央缓存是空的，则 allocate 一个新的
		if len(pp.sudogcache) == 0 {
			pp.sudogcache = append(pp.sudogcache, new(sudog))
		}
	}
	// 从尾部提取，并调整本地缓存
	n := len(pp.sudogcache)
	s := pp.sudogcache[n-1]
	pp.sudogcache[n-1] = nil
	pp.sudogcache = pp.sudogcache[:n-1]
	if s.elem != nil {
		throw("acquireSudog: found s.elem != nil in cache")
	}
	releasem(mp)
	return s
}
```

上述代码涉及到 2 个新的重要的结构体，由于这 2 个结构体特别复杂，暂时此处只展示和 acquireSudog() 有关的部分：

```go
type p struct {
......
	sudogcache []*sudog
	sudogbuf   [128]*sudog
......
}

type schedt struct {
......
	sudoglock  mutex
	sudogcache *sudog
......
}
```

sched.sudogcache 是全局中央缓存，可以认为它是“一级缓存”，它会在 GC 垃圾回收执行 clearpools 被清理。p.sudogcache 可以认为它是“二级缓存”，是本地缓存不会被 GC 清理掉。

chansend 最后的代码逻辑是当 goroutine 唤醒以后，解除阻塞的状态：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
......

	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

sudog 算是对 g 的一种封装，里面包含了 g，要发送的数据以及相关的状态。goroutine 被唤醒后会完成 channel 的阻塞数据发送。发送完最后进行基本的参数检查，解除 channel 的绑定并释放 sudog。

```go
func releaseSudog(s *sudog) {
	if s.elem != nil {
		throw("runtime: sudog with non-nil elem")
	}
	if s.isSelect {
		throw("runtime: sudog with non-false isSelect")
	}
	if s.next != nil {
		throw("runtime: sudog with non-nil next")
	}
	if s.prev != nil {
		throw("runtime: sudog with non-nil prev")
	}
	if s.waitlink != nil {
		throw("runtime: sudog with non-nil waitlink")
	}
	if s.c != nil {
		throw("runtime: sudog with non-nil c")
	}
	gp := getg()
	if gp.param != nil {
		throw("runtime: releaseSudog with non-nil gp.param")
	}
	// 防止 rescheduling 到了其他的 P
	mp := acquirem() 
	pp := mp.p.ptr()
	// 如果本地缓存已满
	if len(pp.sudogcache) == cap(pp.sudogcache) {
		// 转移一半本地缓存到全局中央缓存中
		var first, last *sudog
		for len(pp.sudogcache) > cap(pp.sudogcache)/2 {
			n := len(pp.sudogcache)
			p := pp.sudogcache[n-1]
			pp.sudogcache[n-1] = nil
			pp.sudogcache = pp.sudogcache[:n-1]
			if first == nil {
				first = p
			} else {
				last.next = p
			}
			last = p
		}
		lock(&sched.sudoglock)
		// 将提取的链表挂载到全局中央缓存中
		last.next = sched.sudogcache
		sched.sudogcache = first
		unlock(&sched.sudoglock)
	}
	pp.sudogcache = append(pp.sudogcache, s)
	releasem(mp)
}
```

releaseSudog() 虽然释放了 sudog 的内存，但是它会被 p.sudogcache 这个“二级缓存”缓存起来。



### 调度策略

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-and-thread.jpg)

0,0

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gmp调度.jpg)



send on a full channel



#### 饥饿



### 引用

1. https://blog.csdn.net/u010853261/article/details/85887948
2. https://www.youtube.com/watch?v=KBZlN0izeiY&ab_channel=GopherAcademy
3. https://zhuanlan.zhihu.com/p/27917262
4. https://halfrost.com/go_channel/
5. https://tiancaiamao.gitbooks.io/go-internals/content/zh/07.1.html
6. https://learnku.com/articles/41728

