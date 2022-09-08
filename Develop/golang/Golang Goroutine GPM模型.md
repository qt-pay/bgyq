## Golang Goroutine GPM模型

goroutine很特殊，它在系统分配的Threads上实现了自己的调度策略。

一个goroutine只占2KB的内存.

goroutine的退出机制设计是，goroutine退出只能由本身控制，不允许从外部强制结束该goroutine。只有两种情况例外，那就是main函数结束或者程序崩溃结束运行；所以，要实现主进程控制子goroutine的开始和结束，必须借助其它工具来实现。

大佬：https://zboya.github.io/post/go_scheduler/#%E6%B7%B1%E5%85%A5golang-runtime%E7%9A%84%E8%B0%83%E5%BA%A6

看：https://cloud.tencent.com/developer/article/1135275

线程与协程，1:1没意义
线程与协程，1:N ，可能性能不行
线程与协程，M:N，效率和性能都可以

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

这是因为 ch是一个无缓存的channel，当我们向它存入值的时候，会阻塞所在goroutine，这里当然是main.main的goroutine，后面的语句根本不会执行到。

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



### G

goroutines are user-space threads.

goroutines created and managed by the Go runtime， not the    OS. ligthweight compared to OS threads.



`G` 代表 goroutine，即用户创建的 goroutines

Goroutine is stackful coroutine. https://jiajunhuang.com/articles/2018_04_03-coroutine.md.html

#### G0

每个`M`都有一个`G0-goroutine`，`G0`负责调度goroutines, goroutine切换时，都会使用`G0`的stack即公共栈.

 `G0`仅负责调度`G`，而`G`则负责执行代码func.

### M

`M`是操作系统线程。在绝大多数时候，`P` 的数量和 `M` 的数量是相等的。每创建一个 `P`, 就会创建一个对应的 `M`.

在OS中，进程是资源分配单位，线程是最小的调度执行单位，这是大前提。所以，进程中至少有一个线程。

而协程则是只在用户态利用事件机制实现切换的低成本并发手段。

自旋线程会抢占P，但是不抢占G。

#### M0

Golang进程刚刚启动时，会启动一个M0，它完成初始化工作后就是正常的M了。

### P

`P` 代表 Logical Processor，是类似于 CPU 核心的概念，其用来控制并发的 M 数量

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
## 很恐怖啊, 代码在容器中读到的还是 host的全部cpu
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



### gopark：pausing goroutines

gopark函数在协程的实现上扮演着非常重要的角色，用于协程的切换，协程切换的原因一般有以下几种情况：

* 系统调用；
* channel读写条件不满足；
* 抢占式调度时间片结束；

gopark函数做的主要事情分为两点：

* 解除当前goroutine的m的绑定关系，将当前goroutine状态机切换为等待状态；
* 调用一次schedule()函数，在局部调度器P发起一轮新的调度。

waiting

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gopark-waiting.jpg)

scheduler

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

