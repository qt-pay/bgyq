## [LGTM]分布式限流算法

### (微)服务中的上下游
在由微服务（或者只是过时的分布式服务）构成的系统中，同样有上下游服务的讨论。意料之中，依赖原则和价值原则都可以运用到这个场景。服务 B 是上游服务因为服务 A 依赖它。服务 A 是下游服务因为它在服务 B 的基础上增加了价值。

请注意这里讨论的什么是上游什么是下游中的“游”不是通过服务 A 进入系统的数据流，而是从系统核心部分到面向用户服务的数据流。

**离用户（或者其他终端客户）越近的服务，它就越下游。**

### 软件依赖的上下游

很多软件模块会依赖其他的模块。那么什么是上游依赖和下游依赖呢？
考虑下面关系： 模块 C 依赖模块 B，模块 B 依赖模块 A。
运用依赖原则，我们可以有把握地说模块 A 是模块 B 的上游，模块 B 和模块 C 的上游（尽管箭头是相反的方向）

这里运用价值原则会有点抽象，但是我们可以认为模块 C 拥有最多的价值，因为它导入了模块 B 和 A 的所有功能，并且附加了自己独有的价值。所以模块 C 是下游模块。

Nginx请求流向是 client -> nginx -> server，因为Server是离用户最远的所以是最上游。



### 什么是限流

在开发高并发系统时，有三把利器用来保护系统：缓存、降级和限流。那么什么是限流呢？顾名思义，限流就是限制流量。通过限流，我们可以很好地控制系统的qps，从而达到保护系统的目的。

对一般的限流场景来说它具有两个维度的信息：

- **时间** 限流基于某段时间范围或者某个时间点，也就是我们常说的“时间窗口”，比如对每分钟、每秒钟的时间窗口做限定
- **资源** 基于可用资源的限制，比如设定最大访问次数，或最高可用连接数

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/limit-rate-method.webp)

### 分布式限流

所谓的分布式限流，其实道理很简单，一句话就可以解释清楚。分布式区别于单机限流的场景，它把整个分布式环境中所有服务器当做一个整体来考量。比如说针对IP的限流，我们限制了1个IP每秒最多10个访问，不管来自这个IP的请求落在了哪台机器上，只要是访问了集群中的服务节点，那么都会受到限流规则的制约。

从上面的例子不难看出，我们必须将限流信息保存在一个“中心化”的组件上，这样它就可以获取到集群中所有机器的访问状态，目前有两个比较主流的限流方案：

- **网关层限流** 将限流规则应用在所有流量的入口处
- **中间件限流** 将限流信息存储在分布式环境中某个中间件里（比如Redis缓存），每个组件都可以从这里获取到当前时刻的流量统计，从而决定是拒绝服务还是放行流量

#### 中心存储

分布式限流所需的是一个类似中心节点的地方存储限流数据。比如，如果希望控制接口的访问速率为每秒100个请求，那么就需要将当前1s内已经接收到的请求的数量保存在某个地方，并且可以让集群环境中所有节点都能访问。

一个全局的限流器，也就是说服务虽然部署在分布式系统中，但在外界看起来就像部署在单台机器上一样，那样就必须要一个中心化的存储设备去管理限流器，比如Redis。
一种直观的做法是，将限流器配置在数据库中，每个节点收到来自上游的请求后直接请求数据库，然后数据库根据限流器判断是否处理这个请求，最后返回给节点相关信息。如果用Redis实现，限流器的代码可以通过Lua脚本的方式放在Redis端，从而减少节点访问Redis的次数。
如果服务的流量不大的话，这种简单的基于中心化数据库的实现方法还能撑住。但如果服务的流量很大，这种方法则会有很大的成本和性能问题，每有一个上游的请求，节点就会请求一次数据库并等待数据库是否限流的回复，那么数据库的压力的特别大，会造成从数据库返回结果的延迟较高。并且为了得到正确的结果，每个节点访问数据库的时候还需要避免数据竞争，如果是支持事物的数据库还好，如果基于Redis做，这就需要对限流器加锁，Redis的延迟会更高，这样会导致服务处理请求的延迟很高。

#####  优化基于中心化数据库的限流器



#### 网关限流

限流常在网关这一层做，比如Nginx、Openresty、Kong（需要手动发布服务或者通过自定义插件实现自动发现）、Zuul、Spring Cloud Gateway等，而像spring cloud - gateway网关限流底层实现原理，就是基于Redis + Lua，通过内置Lua限流脚本的方式。



服务网关，作为整个分布式链路中的第一道关卡，承接了所有用户来访请求.

在网关限流漏斗模型中做流量限制，最合适的环节就是网关层，因为它是整个访问链路的源头，是所有流量途径的第一站。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/网关限流-漏斗模型.webp)

上面是一个最普通的流量模型，从上到下的路径依次是：

1. 用户流量从网关层转发到后台服务
2. 后台服务承接流量，调用缓存获取数据
3. 缓存中无数据，则访问数据库

为什么说它是一个漏斗模型，因为流量自上而下是逐层递减的，在网关层聚集了最多最密集的用户访问请求，其次是后台服务。然后经过后台服务的验证逻辑之后，刷掉了一部分错误请求，剩下的请求落在缓存上，如果缓存中没有数据才会请求漏斗最下方的数据库，因此数据库层面请求数量最小（相比较其他组件来说数据库往往是并发量能力最差的一环，阿里系的MySQL即便经过了大量改造，单机并发量也无法和Redis、Kafka之类的组件相比）

#### 中间件限流

Redis简直就是为服务端限流量身打造的利器。利用Redis过期时间特性，我们可以轻松设置限流的时间跨度（比如每秒10个请求，或者每10秒10个请求）

分布式限流本质上是一个集群并发问题，Redis + Lua 的方案非常适合此场景：

1. Redis 单线程特性，适合解决分布式集群的并发问题
2. Redis 本身支持 Lua 脚本执行，可以实现原子执行的效果

### 分布式限流算法

#### 令牌桶算法：处理上限可以波动

Token Bucket令牌桶算法是目前应用最为广泛的限流算法，顾名思义，它有以下两个关键角色：

1. **令牌** 获取到令牌的Request才会被处理，其他Requests要么排队要么被直接丢弃
2. **桶** 用来装令牌的地方，所有Request都从这个桶里面获取令牌

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/token-bucket.webp)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/token-bucket-3.webp)



##### 令牌生成：控制速率

这个流程涉及到令牌生成器和令牌桶，前面我们提到过令牌桶是一个装令牌的地方，既然是个桶那么必然有一个容量，也就是说令牌桶所能容纳的令牌数量是一个固定的数值。

对于令牌生成器来说，它会根据一个预定的速率向桶中添加令牌，比如我们可以配置让它以每秒100个请求的速率发放令牌，或者每分钟50个。注意这里的发放速度是匀速，也就是说这50个令牌并非是在每个时间窗口刚开始的时候一次性发放，而是会在这个时间窗口内匀速发放。

在令牌发放器就是一个水龙头，假如在下面接水的桶子满了，那么自然这个水（令牌）就流到了外面。在令牌发放过程中也一样，令牌桶的容量是有限的，如果当前已经放满了额定容量的令牌，那么新来的令牌就会被丢弃掉。

##### 令牌获取

每个访问请求到来后，必须获取到一个令牌才能执行后面的逻辑。假如令牌的数量少，而访问请求较多的情况下，一部分请求自然无法获取到令牌，那么这个时候我们可以设置一个“缓冲队列”来暂存这些多余的请求。

缓冲队列其实是一个可选的选项，并不是所有应用了令牌桶算法的程序都会实现队列。当有缓存队列存在的情况下，那些暂时没有获取到令牌的请求将被放到这个队列中排队，直到新的令牌产生后，再从队列头部拿出一个请求来匹配令牌。

当队列已满的情况下，这部分访问请求将被丢弃。在实际应用中我们还可以给这个队列加一系列的特效，比如设置队列中请求的存活时间，或者将队列改造为PriorityQueue，根据某种优先级排序，而不是先进先出。算法是死的，人是活的，先进的生产力来自于不断的创造，在技术领域尤其如此。

##### 适用场景：突发流量

突发流量：因为令牌有冗余，可以容纳和处理超过平时流量的请求。但是漏桶不行，因为无论请求多少，漏桶处理的请求是固定的。

令牌桶在流量入口就做了把控，所以大量的请求涌入会给client响应提交失败等，类似jd抢mt，提示下单失败。

令牌桶的本质是速率控制。

令牌桶的算法原本是用于网络设备控制传输速度的，而且它控制的目的是保证一段时间内的平均速率控制，之所以说令牌桶适合突发流量，是指在网络传输的时候，可以允许某段时间内（一般就几秒）超过平均传输速率，这在网络环境下常见的情况就是“网络抖动”，但这个短时间的突发流量是不会导致雪崩效应，网络设备也能够处理得过来。对应到令牌桶应用到业务处理的场景，就要求即使有突发流量来了，系统自己或者下游系统要真的能够处理的过来，否则令牌桶允许突发流量进来，结果系统或者下游处理不了，那还是会被压垮。

即生成令牌的速率要低于下游服务的处理极限，避免打垮服务。

#### 漏桶算法：处理上限固定

Leaky Bucket，漏桶算法的前半段和令牌桶类似，但是操作的对象不同，令牌桶是将令牌放入桶里，而漏桶是将访问请求的数据包放到桶里。同样的是，如果桶满了，那么后面新来的数据包将被丢弃（这里是不是也可以自己修改算法，增加请求缓存队列？）。

漏桶算法的后半程是有鲜明特色的，它永远只会以一个恒定的速率将数据包从桶内流出。打个比方，如果我设置了漏桶可以存放100个数据包，然后流出速度是1s一个，那么不管数据包以什么速率流入桶里，也不管桶里有多少数据包，漏桶能保证这些数据包永远以1s一个的恒定速度被处理。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/leaky-bucket.webp)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/leaky-bucket-2.webp)

##### 适用场景：秒杀

类似，场景小米抢购时，一直提示排队中....应该就是在等漏桶执行--

漏桶算法更适合“突发流量”，是指秒杀、抢购、整点打卡签到、微博热点事件这种业务高并发场景，它不是由于“**XX 抖动**”引起的，而是**由业务场景引起**的，并且持续的事件可能是几分钟甚至几十分钟，这种业务场景为了用户体验和业务尽量少受损，优先采取的不是丢弃大量请求，而是缓存请求，避免系统出现雪崩效应。因此我们会看到，漏桶和令牌桶都有保护作用，但**漏桶的保护是尽量缓存请求（缓存不下才丢），令牌桶的保护主要是丢弃请求（即使系统还能处理，只要超过指定的速率就丢弃，除非此时动态提高速率）**。

所以如果在秒杀、抢购、整点打卡签到、微博热点事件这些业务场景用**令牌桶**的话，会出现大量用户访问出错，因为请求被直接丢弃了；而用漏桶的话，处理可能只是会慢一些，用户体验会更好一些，所以我认为漏桶更适合“突发流量”。



#### 滑动窗口

Rolling Window

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rolling-window.webp)

上图中黑色的大框就是时间窗口，我们设定窗口时间为5秒，它会随着时间推移向后滑动。我们将窗口内的时间划分为五个小格子，每个格子代表1秒钟，同时这个格子还包含一个计数器，用来计算在当前时间内访问的请求数量。那么这个时间窗口内的总访问量就是所有格子计数器累加后的数值。

比如说，我们在第一秒内有5个用户访问，第5秒内有10个用户访问，那么在0到5秒这个时间窗口内访问量就是15。如果我们的接口设置了时间窗口内访问上限是20，那么当时间到第六秒的时候，这个时间窗口内的计数总和就变成了10，因为1秒的格子已经退出了时间窗口，因此在第六秒内可以接收的访问量就是20-10=10个。

滑动窗口其实也是一种计算器算法，它有一个显著特点，当时间窗口的跨度越长时，限流效果就越平滑。打个比方，如果当前时间窗口只有两秒，而访问请求全部集中在第一秒的时候，当时间向后滑动一秒后，当前窗口的计数量将发生较大的变化，拉长时间窗口可以降低这种情况的发生概率。

### 桶算法核心

漏桶的本质是总量控制，令牌桶的本质是速率控制。至于这个控制，你可以用来保护自己，也可以用来保护别人。
有的技术人员对于“保护别人”可能不太理解，这个主要应用于访问第三方，最典型的例子就是双十一的支付宝，支付的时候要访问银行的接口，银行的接口实际上处理高并发的能力并不强（例如澳门某银行对外提供的移动支付接口，峰值 TPS 是 30 次/s），这个时候支付宝自己处理性能再高都没用，而且越高越容易把下游的银行给压垮，所以支付宝不得不针对下游接口请求做限流。

令牌桶就是一个速率控制，你可以用来控制自己的处理速度，也可以控制请求别人的处理速度，都可以起到保护作用；

其实漏桶也可以既保护自己又保护下游，因为请求太多的时候把请求先缓存到漏桶里面了，漏桶放不下就丢弃新的请求，这也是保护机制。

#### 总结

根据它们各自的特点不难看出来，这两种算法都有一个“恒定”的速率和“不定”的速率。令牌桶是以恒定速率创建令牌，但是访问请求获取令牌的速率“不定”，反正有多少令牌发多少，令牌没了就干等。而漏桶是以“恒定”的速率处理请求，但是这些请求流入桶的速率是“不定”的。

从这两个特点来说，漏桶的天然特性决定了它不会发生突发流量，就算每秒1000个请求到来，那么它对后台服务输出的访问速率永远恒定。而令牌桶则不同，其特性可以“预存”一定量的令牌，因此在应对突发流量的时候可以在短时间消耗所有令牌，其突发流量处理效率会比漏桶高，但是导向后台系统的压力也会相应增多。

1）如果要让自己的系统不被打垮，用令牌桶。如果保证别人的系统不被打垮，用漏桶算法；

2）在“令牌桶算法”中，只要令牌桶中存在令牌，那么就允许突发地传输数据直到达到用户配置的门限，所以它适合于具有突发特性的流量。

### code demo

#### leaky bucket

漏桶法的关键点在于漏桶始终按照固定的速率运行，关于漏桶的实现，uber团队有一个开源的[github.com/uber-go/ratelimit](https://github.com/uber-go/ratelimit)库。 这个库的使用方法比较简单。

```go
func main() {
	// rl := ratelimit.New(100,ratelimit.Per(20 * time.Second)) // per second
    rl := ratelimit.New(100)
	prev := time.Now()
	for i := 0; i < 10; i++ {
		now := rl.Take()
		fmt.Println(i, now.Sub(prev))
		prev = now
	}

}
// output
0s
1 10ms
...
99 10ms
```



#### token bucket

令牌桶其实和漏桶的原理类似，令牌桶按固定的速率往桶里放入令牌，并且只要能从桶里取出令牌就能通过，令牌桶支持突发流量的快速处理。

```go
func main() {
	//b := newBucket(1*time.Second, 100)
	b := ratelimit.NewBucket(1*time.Second, 100)
	blockNum := 0
	acceptNUm := 0
	for i := 0; i < 1000; i++ {
		before := b.Available()
		tokenGet := b.TakeAvailable(1)
		fmt.Println("tokenGet num", tokenGet)
		if tokenGet != 0 {
			acceptNUm++
			fmt.Println("获取到令牌 index=", i+1, "前后数量-> 前：", before, ", 后: ", b.Available(), ", tokenGet=", tokenGet, acceptNUm)
		} else {
			blockNum++
			fmt.Println("未获取到令牌，拒绝", i+1, blockNum)
		}
		// 1 Second = 1000 * Millisecond
		time.Sleep(1*time.Millisecond)
	}

}
// output
tokenGet num 1
获取到令牌 index= 90 前后数量-> 前： 11 , 后:  10 , tokenGet= 1 90
...
tokenGet num 1
获取到令牌 index= 100 前后数量-> 前： 1 , 后:  0 , tokenGet= 1 100
tokenGet num 0
未获取到令牌，拒绝 101 1
tokenGet num 0
未获取到令牌，拒绝 102 2
...
```

对于令牌桶的Go语言实现，大家可以参照[github.com/juju/ratelimit](https://github.com/juju/ratelimit)库。这个库支持多种令牌桶模式，并且使用起来也比较简单。

创建令牌桶的方法：

```go
// 创建指定填充速率和容量大小的令牌桶
func NewBucket(fillInterval time.Duration, capacity int64) *Bucket
// 创建指定填充速率、容量大小和每次填充的令牌数的令牌桶
func NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64) *Bucket
// 创建填充速度为指定速率和容量大小的令牌桶
// NewBucketWithRate(0.1, 200) 表示每秒填充20个令牌
func NewBucketWithRate(rate float64, capacity int64) *Bucket
```

取出令牌的方法如下：

```go
// 取token（非阻塞）
func (tb *Bucket) Take(count int64) time.Duration
// TakeAvailable takes up to count immediately available tokens from the
// bucket. It returns the number of tokens removed, or zero if there are
// no available tokens. It does not block.
// 返回0就是没可用token了
func (tb *Bucket) TakeAvailable(count int64) int64

// 最多等maxWait时间取token
func (tb *Bucket) TakeMaxDuration(count int64, maxWait time.Duration) (time.Duration, bool)

// 取token（阻塞）
func (tb *Bucket) Wait(count int64)
func (tb *Bucket) WaitMaxDuration(count int64, maxWait time.Duration) bool
```

虽说是令牌桶，但是我们没有必要真的去生成令牌放到桶里，我们只需要每次来取令牌的时候计算一下，当前是否有足够的令牌就可以了，具体的计算方式可以总结为下面的公式：

```go
当前令牌数 = 上一次剩余的令牌数 + (本次取令牌的时刻-上一次取令牌的时刻)/放置令牌的时间间隔 * 每次放置的令牌数
```

#### 计数器

计数器是一种最简单限流算法，其原理就是：在一段时间间隔内，对请求进行计数，与阀值进行比较判断是否需要限流，一旦到了时间临界点，将计数器清零。

```go

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"sync"
	"time"
)

type Counter struct {
	rate  int           //计数周期内最多允许的请求数
	begin time.Time     //计数开始时间
	cycle time.Duration //计数周期
	count int           //计数周期内累计收到的请求数
	lock  sync.Mutex
}

func (l *Counter) Allow() bool {
	l.lock.Lock()
	defer l.lock.Unlock()

	if l.count == l.rate-1 {
		now := time.Now()
		if now.Sub(l.begin) >= l.cycle {
			//速度允许范围内， 重置计数器
			l.Reset(now)
			return true
		} else {
			return false
		}
	} else {
		//没有达到速率限制，计数加1
		l.count++
		return true
	}
}

func (l *Counter) Set(r int, cycle time.Duration) {
	l.rate = r
	l.begin = time.Now()
	l.cycle = cycle
	l.count = 0
}

func (l *Counter) Reset(t time.Time) {
	l.begin = t
	l.count = 0
}

func main() {
	var wg sync.WaitGroup
	var lr Counter
	lr.Set(3, time.Second) // 1s内最多请求3次
	resultReq := make([]bool, 10, 10)

	for i := 0; i < 10; i++ {
		wg.Add(1)
		log.Debug("创建请求:", i)
		go func(i int) {
			if lr.Allow() {
				resultReq[i]=true
				log.Debug("响应请求:", i)
			}
			wg.Done()
		}(i)
		// 1s = 1000 Millisecond
		time.Sleep(200 * time.Millisecond)
	}
	wg.Wait()
	for num, res := range resultReq{
		if res{
			fmt.Println("请求-", num, ":", " 响应")
		} else {
			fmt.Println("请求-", num, ":", " 拒绝")
		}
	}
}

// output
请求- 0 :  响应
请求- 1 :  响应
请求- 2 :  拒绝
请求- 3 :  拒绝
请求- 4 :  拒绝
// 新的一秒开始了-
请求- 5 :  响应
请求- 6 :  响应
请求- 7 :  响应
请求- 8 :  拒绝
请求- 9 :  拒绝
```



##### 流量突刺:时间边界

如果有个需求对于某个接口 `/query` 每分钟最多允许访问 200 次，假设有个用户在第 59 秒的最后几毫秒瞬间发送 200 个请求，当 59 秒结束后 `Counter` 清零了，他在下一秒的时候又发送 200 个请求。那么在 1 秒钟内这个用户发送了 2 倍的请求，这个是符合我们的设计逻辑的，这也是计数器方法的设计缺陷，系统可能会承受恶意用户的大量请求，甚至击穿系统。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/计数器-时间边界.webp)

这种方法虽然简单，但也有个大问题就是没有很好的处理单位时间的边界。

其实计数器也是滑动窗口啊，只不过只有一个格子而已，

#### rolling windows

滑动窗口算法将一个大的时间窗口分成多个小窗口，每次大窗口向后滑动一个小窗口，并保证大的窗口内流量不会超出最大值，这种实现比固定窗口的流量曲线更加平滑。

○ 问题：没有根本解决固定窗口算法的临界突发流量问题

所谓 `滑动窗口（Sliding window）` 是一种流量控制技术，这个词出现在 `TCP` 协议中。`滑动窗口`把固定时间片进行划分，并且随着时间的流逝，进行移动，固定数量的可以移动的格子，进行计数并判断阀值。

红色的虚线代表一个时间窗口（`一分钟`），每个时间窗口有 `6` 个格子，每个格子是 `10` 秒钟。每过 `10` 秒钟时间窗口向右移动一格，可以看红色箭头的方向。我们为每个格子都设置一个独立的计数器 `Counter`，假如一个请求在 `0:45` 访问了那么我们将第五个格子的计数器 `+1`（也是就是 `0:40~0:50`），在判断限流的时候需要把所有格子的计数加起来和设定的频次进行比较即可。

**确实，计数器是只有一个格子的滑动窗口。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/slidingwindows实现.webp)

**本质思想**是转换概念，将原本问题的确定时间大小，进行次数限制。转换成确定次数大小，进行时间限制。

```go
package main

import (
	"log"
	"net/http"
	"time"
)

var LimitQueue map[string][]int64
var ok bool

//单机时间滑动窗口限流法
func LimitFreqSingle(queueName string, count uint, timeWindow int64) bool {
	currTime := time.Now().Unix()
	if LimitQueue == nil {
		LimitQueue = make(map[string][]int64)
	}
	if _, ok = LimitQueue[queueName]; !ok {
		LimitQueue[queueName] = make([]int64, 0)
	}
	//队列未满
	if uint(len(LimitQueue[queueName])) < count {
		LimitQueue[queueName] = append(LimitQueue[queueName], currTime)
		return true
	}
	//队列满了,取出最早访问的时间
	earlyTime := LimitQueue[queueName][0]
	//说明最早期的时间还在时间窗口内,还没过期,所以不允许通过
	if currTime-earlyTime <= timeWindow {
		return false
	} else {
		//说明最早期的访问应该过期了,去掉最早期的
		LimitQueue[queueName] = LimitQueue[queueName][1:]
		LimitQueue[queueName] = append(LimitQueue[queueName], currTime)
	}
	return true
}
func main()  {
	http.HandleFunc(
		"/",
		func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("httpserver v1"))
		},
	)
	http.HandleFunc("/bye", sayBye)
	log.Println("Starting v1 server ...")
	log.Fatal(http.ListenAndServe(":1210", nil))
}

func sayBye(w http.ResponseWriter, r *http.Request) {
	if !LimitFreqSingle(string(r.Host), 2, 10){
		log.Println("limit")
	}else {
		log.Println("Allow")
	}
	w.Write([]byte("bye bye ,this is v1 httpServer"))
}

// output
// client curl http://127.0.0.1:1210/bye
2022/08/10 23:29:35 Starting v1 server ...
2022/08/10 23:30:02 Allow
2022/08/10 23:30:03 Allow
2022/08/10 23:30:04 limit
2022/08/10 23:30:04 limit
2022/08/10 23:30:06 limit
```



### 引用

1. https://segmentfault.com/a/1190000023126434
2. https://xie.infoq.cn/article/4a0acdd12a0f6dd4a53e0472c
3. https://blog.csdn.net/ternence_hsu/article/details/109844697
4. https://www.codeleading.com/article/90595528948/
5. https://www.cnblogs.com/liwenzhou/p/13670165.html#autoid-0-2-1
6. https://segmentfault.com/a/1190000039958561
7. https://cloud.tencent.com/developer/article/1761700