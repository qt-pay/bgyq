## Golang **CSP** and Channel

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

#### 总结

简单来说，CSP 模型由并发执行的实体（线程或者进程或者协程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。

CSP 模型的关键是关注 channel，而不关注发送消息的实体。Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。

Channel 在 gouroutine 间架起了一条管道，在管道里传输数据，实现 gouroutine 间的通信；由于它是线程安全的，所以用起来非常方便；channel 还提供“先进先出”的特性；它还能影响 goroutine 的阻塞和唤醒。

综合来说，CSP 1978 中描述的编程语言（与 Go 所构建的基于通道的 channel/select 同步机制进行对比）：

1. channel 没有被显式命名
2. channel 没有缓冲，对应 Go 中 unbuffered channel
3. buffered channel 不是一种基本的编程源语，并展示了一个使用 unbuffered channel 实现其作用的例子
4. guarded command 等价于 if 与 select 语句的组合，分支的随机触发性是为了提供公平性保障
5. guarded command 没有对确定性分支选择与非确定性（即多个分支有效时随机选择）分支选择进行区分
6. repetitive command 的本质是一个无条件的 for 循环，但终止性所要求的条件太苛刻，不利于理论的进一步形式化
7. CSP 1978 中描述的编程语言对程序终止性的讨论几乎为零
8. 此时与 Actor 模型进行比较，CSP 与 Actor 均在实体间直接通信，区别在于 Actor 支持异步消息通信，而 CSP 1978 是同步通信



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