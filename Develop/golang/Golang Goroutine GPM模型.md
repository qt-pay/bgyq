## Golang Goroutine GPM模型

goroutine很特殊，它在系统分配的Threads上实现了自己的调度策略。

一个goroutine只占2KB的内存.

goroutine的退出机制设计是，goroutine退出只能由本身控制，不允许从外部强制结束该goroutine。只有两种情况例外，那就是main函数结束或者程序崩溃结束运行；所以，要实现主进程控制子goroutine的开始和结束，必须借助其它工具来实现。

大佬：https://zboya.github.io/post/go_scheduler/#%E6%B7%B1%E5%85%A5golang-runtime%E7%9A%84%E8%B0%83%E5%BA%A6

看：https://cloud.tencent.com/developer/article/1135275

线程与协程，1:1没意义
线程与协程，1:N ，可能性能不行
线程与协程，M:N，效率和性能都可以

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

### sudog

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

channel 最核心的数据结构是 sudog。sudog 代表了一个在等待队列中的 g。sudog 是 Go 中非常重要的数据结构，因为 g 与同步对象关系是多对多的。一个 g 可以出现在许多等待队列上，因此一个 g 可能有很多sudog。并且多个 g 可能正在等待同一个同步对象，因此一个对象可能有许多 sudog。sudog 是从特殊池中分配出来的。使用 acquireSudog 和 releaseSudog 分配和释放它们。

### channel数据通讯

G1 负责给ch写数据，G2负责读数据。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/channel-communication-1.jpg)



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-2.jpg)



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-3.jpg)



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/goroutine-communication-4.jpg)

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

