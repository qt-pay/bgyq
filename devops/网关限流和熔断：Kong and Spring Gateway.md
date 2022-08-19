## 网关限流和熔断：Kong and Spring Gateway

### 微服务限流和熔断Demo:fire:

熔断+限流，应该从两个层次考虑：总流量阈值 + 单个实例阈值

每个应用实例本身配置Hystrix实现限流和熔断，甚至可以没有网关层面的整体流量熔断

网关配置熔断限流规则是全局的，即基于全部实例健康的前提下，设置的流量最大保护阈值。但是服务实例本身也要配置自身的最大安全阈值，不然会造成服务雪崩。

假设，业务A有N个实例，每个实例配置的最大容量阈值是200，这在网关层面可以给业务A的接口可以配置的最大流量阈值是N * 200。假如N个实例中有某个实例异常了，那么其余实例在满网关入口流量满负载的情况下，剩余的N-1个实例会收到大于200的流量请求，极端场景会导致整个服务雪崩。

此外，应用自身设置熔断和限流规则，可以防止不经过网关的东西流量（内部其他组件调用）。

比如，组件A的QPS是500，而它调用的上游服务B的QPS是200，如果服务B本身不做保护会直接被A击穿。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/spring-cloud-hystrix.png)

#### 应用 and 接口

一个springboot应用可以有N个接口，限流是针对应用/服务的某个接口来设置的。

如下，Hystrix hello world.

实现原理讲起来很简单，其实就是不让客户端“裸调“服务器的rpc接口，而是在客户端包装一层。就在这个包装层里面，实现熔断逻辑。 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/hystrix-流程.png)

```java
// HystrixConfig
@Configuration
public class HystrixConfig {

    /**
     * 声明一个HystrixCommandAspect代理类，现拦截HystrixCommand的功能
     */
    @Bean
    public HystrixCommandAspect hystrixCommandAspect() {
        return new HystrixCommandAspect();
    }

}
// 定义service
@Service
public class HelloService {

    @HystrixCommand(fallbackMethod = "helloError",
            commandProperties = {
                    @HystrixProperty(name = "execution.isolation.strategy", value = "THREAD"),
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000"),
                    @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "2")},
            threadPoolProperties = {
                    @HystrixProperty(name = "coreSize", value = "5"),
                    @HystrixProperty(name = "maximumSize", value = "5"),
                    @HystrixProperty(name = "maxQueueSize", value = "10")
            })
    public String sayHello(String name) {
        try {
            Thread.sleep( 15000 );
            return "Hello " + name + " !";
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return null;
    }

    public String helloError(String name) {
        return "服务器繁忙，请稍后访问~";
    }

}

// 启动类
@SpringBootApplication
@RestController
public class HystrixSimpleApplication {

    @Autowired
    //关键点：把一个RPC调用，封装在一个HystrixCommand里面
    private HelloService helloService;

    public static void main(String[] args) {
        SpringApplication.run( HystrixSimpleApplication.class, args );
    }

    @GetMapping("/hi")
    public String hi(String name) {
        return helloService.sayHello( name );
    }


}


//客户端调用：以前是直接调用远端RPC接口，现在是把RPC接口封装到HystrixCommand里面，它内部完成熔断逻辑
curl -X GET -d 'name=test' http://localhost:8080/hi

```



### 单机限流和分布式限流

由于请求倾斜的存在，分发到集群中每个节点上的流量不可能均匀，所以单机限流无法实现精确的限制整个集群的整体流量，导致总流量没有到达阈值的情况下一些机器就开始限流。例如服务 A 部署了 3 个节点，规则配置限流阈值为 200qps，理想情况下集群的限流阈值为 600qps，而实际情况可能某个节点先到达 200qps，开始限流，而其它节点还只有 100qps，此时集群的 QPS 为 400qps。

#### 区别

分布式区别于单机限流的场景，它把整个分布式环境中所有服务器当做一个整体来考量。即需要一个全局和中心的计数器来实现限流

分布式限流算法相较于单机的限流算法，最大的区别就是接口请求计数器需要中心化存储，比如我们开源限流项目 ratelimiter4j 就是基于 Redis 中心计数器来实现分布式限流算法。

分布式限流算法的性能瓶颈主要在中心计数器 Redis，从我们开源的 ratelimiter4j 压测数据来看，在没有做 Redis sharding 的情况下，基于单实例 Redis 的分布式限流算法的性能要远远低于基于内存的单机限流算法，基于我们的压测环境，单机限流算法可以达到 200 万 TPS，而分布式限流算法只能做到 5 万 TPS。所以，在应用分布式限流算法时，一定要考量限流算法的性能是否满足应用场景，如果微服务接口的 TPS 已经超过了限流框架本身的 TPS，则限流功能会成为性能瓶颈影响接口本身的性能。



### 分布式限流实现方式

分布式限流与微服务之间常见的部署架构有以下几种：

#### 在接入层（api-gateway）集成限流功能

这种集成方式是在微服务架构下，有 api-gateway 的前提下，最合理的架构模式。如果 api-gateway 是单实例部署，使用单机限流算法即可。如果 api-gateway 是多实例部署，为了做到服务级别的限流就必须使用分布式限流算法。

微服务框架配套开发的Gateway可以和自己的注册中心高度集成，Gateway可以从注册中心动态获取到服务的注册信息。但是开源的公共网关类似Kong不能直接从现有的注册中心读取已注册的服务，需要手动发布服务或者通过自己实现Lua插件自动读取注册中心内容。

#### 限流功能封装为 RPC 服务

当微服务接收到接口请求之后，会先通过限流服务暴露的 RPC 接口来查询接口请求是否超过限流阈值。这种架构模式，需要部署一个限流服务，增加了运维成本。这种部署架构，性能瓶颈会出现在微服务与限流服务之间的 RPC 通信上，即便单机限流算法可以做到 200 万 TPS，但经过 RPC 框架之后，做到 10 万 TPS 的请求限流就已经不错了。

#### 限流功能集成在微服务系统内

这种架构模式不需要再独立部署服务，减少了运维成本，但限流代码会跟业务代码有一些耦合，不过，可以将限流功能集成在切面层，尽量跟业务代码解耦。如果做服务级的分布式限流，必须使用分布式限流算法，如果是针对每台微服务实例进行单机限流，使用单机限流算法就可以。

### 限流

限流是微服务架构下保障系统的主要手段之一。通常来说系统的吞吐量是可以提前预测的，当请求量超过预期的伐值时可以采取一些限制措施来保障系统的稳定运行，比如延迟处理、拒绝服务等。

限流规则包含三个部分：时间粒度，接口粒度，最大限流值。限流规则设置是否合理直接影响到限流是否合理有效。

常见的限流算法有：计数器、漏桶、令牌桶。

#### 计数器

#### 漏桶算法

类似生活用到的漏斗，当请求进来时，相当于水倒入漏斗，然后从下端小口慢慢匀速的流出，算法描述如下：

- 一个固定容量的漏桶，按照固定的速率流出水滴（请求）。
- 如果桶是空的，则不需要流出水滴。
- 可以以任意的速率流入水滴到桶中，如果超出了桶的容量，则流入的水滴溢出（拒绝请求）。
- 桶的容量始终保持不变。

算法实现上，可以使用一个BlockingQueue表示漏桶，请求进来时放入这个BlockingQueue中。另起一个线程以固定的速率从BlockingQueue中取出请求，再提交给业务线程池处理。漏桶算法有个弊端：**无法应对短时间的突发流量。**即漏桶处理上限是固定的，无法处理偶尔的波动

#### 令牌桶算法

令牌桶算法是对漏桶算法的一种改进，漏桶算法能够限制请求流出的速率，而令牌桶算法能够在限制调用的平均速率的同时还允许一定程度的突发调用，算法描述如下：

- 有一个固定容量的桶，按照固定的速率向桶中添加令牌。
- 如果桶满了，则新添加的令牌被丢弃。
- 当请求进来时，必须从桶中获取一个令牌才能继续处理，否则拒绝请求，或者暂存到某个缓冲区等待可用令牌。

算法实现上，可以用一个BlockingQueue表示桶，起一个线程以固定的速率向桶中添加令牌。请求到达时需要先从桶中获取令牌，否则拒绝请求或等待。此外，Google开源的guava包也提供了很完善的令牌算法。

#### Redis+Lua

时间窗口算法

### 熔断

 circuit breaker 

在高可用设计中，除了流控外，对分布式系统调用链路中不稳定的资源(比如RPC服务等)进行熔断降级也是保障高可用的重要措施之一。

现代微服务架构基本都是分布式的，整个分布式系统由非常多的微服务组成。不同服务之间相互调用，组成复杂的调用链路。前面描述的问题在分布式链路调用中会产生放大的效果。整个复杂链路中的某一环如果不稳定，就可能会层层级联，最终可能导致整个链路全部挂掉。因此我们需要对不稳定的 **弱依赖服务调用** 进行 **熔断降级**，暂时切断不稳定的服务调用，避免局部不稳定因素导致整个分布式系统的雪崩。熔断降级作为保护服务自身的手段，通常在客户端（调用端）进行配置。

#### 静默期

静默期是指一个最小的静默请求数，在一个统计周期内，如果对资源的请求数小于设置的静默数，那么熔断器将不会基于其统计值去更改熔断器的状态。静默期的设计理由也很简单，举个例子，假设在一个统计周期刚刚开始时候，第 1 个请求碰巧是个慢请求，这个时候这个时候的慢调用比例就会是 100%，很明显是不合理，所以存在一定的巧合性。所以静默期提高了熔断器的精准性以及降低误判可能性。

#### `SlowRequestRatio`慢调用比

熔断器不在静默期，并且慢调用的比例大于设置的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。

#### `ErrorRatio`错误比

熔断器不在静默期，并且在统计周期内资源请求访问异常的比例大于设定的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。

#### `ErrorCount`错误计算

熔断器不在静默期，并且在统计周期内资源请求访问异常数大于设定的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。

### Sentinel

Sentinel的官方标题是：分布式系统的流量防卫兵。从名字上来看，很容易就能猜到它是用来作服务稳定性保障的。对于服务稳定性保障组件，如果熟悉Spring Cloud的用户，第一反应应该就是Hystrix。但是比较可惜的是Netflix已经宣布对Hystrix停止更新。

> 之前Spring Cloud就是使用Hystrix实现限流和熔断的

除了Spring Cloud官方推荐的resilience4j之外，目前Spring Cloud Alibaba下整合的Sentinel也是用户可以重点考察和选型的目标。

#### 实现原理

Sentinel的使用分为两部分：

1. sentinel-dashboard：与hystrix-dashboard类似，但是它更为强大一些。除了与hystrix-dashboard一样提供实时监控之外，还提供了流控规则、熔断规则的在线维护等功能。

2. 客户端整合：每个微服务客户端都需要整合sentinel的客户端封装与配置，才能将监控信息上报给dashboard展示以及实时的更改限流或熔断规则等。

   即首先在pom.xml文件中添加依赖，然后通过@SentinelResource 注解使用Sentinel

### Spring Gateway + Nacos + Sentinel

#### 网关限流

网关限流是通过网关层对我们的服务进行限流，达到保护后端服务的作用。在微服务架构的系统中，网关层可以屏蔽外部的请求直接对服务进行调用，网关层可以对内部服务进行隔离，保护的作用。

Sentinel通过引入 Sentinel API Gateway Adapter Common 模块，以此实现了网关规则管理、自定义API分组管理，进而对网关进行限流操作。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel-adapter-gateway.png)

#### Nacos+Gateway

使用nacos作为服务的注册中心，实现服务的负载均衡。使用gateway作为网关，配置断言，`/order/**`进行匹配路由，进而访问nacos-order-provider服务的接口。网关转发请求是不是可以增加额外的负载策略、

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nacos-and-gateway.png)

这样的好处，就是网关拿到后端服务提供地址就可以做限流、认证和熔断等措施了。

这个网关描述的也挺清晰

Apache APISIX + Nacos 可以将各个微服务节点中与业务无关的各项控制，集中在 Apache APISIX 中进行统一管理，即**通过 Apache APISIX 实现接口服务的代理和路由转发的能力**。在 Nacos 上注册各个微服务后，Apache APISIX 可以通过 Nacos 的服务发现功能获取服务列表，查找对应的服务地址从而实现动态代理。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gateway-and-nacos.webp)

#### 服务内部限流:field_hockey:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/internal-limit-rate.jpg)

商品服务不仅仅被网关层调用，还被内部订单服务调用，这时候仅仅在网关层限流，那么商品服务还安全吗？

一旦大量的请求订单服务，比如大促秒杀，商品服务不做限流会被瞬间击垮。

业务场景对自己负责的服务也要进行限流兜底，最常见的方案：**网关层集群限流+内部服务的单机限流兜底**，这样才能保证不被流量冲垮。

#### 总结

1、Spring Gateway结合Sentinel完美实现网关限流

2、Sentinel搭配Nacos实现配置的持久化

3、Spring Gateway搭配Nacos实现服务的负载均衡



### Kong

#### Nginx、OpenRestry and Kong

Nginx、OpenRestry、Kong 这三个项目关系比较紧密：

- Nginx 是模块化设计的反向代理软件，C语言开发；
- OpenResty 是以 Nginx 为核心的 Web 开发平台，可以解析执行 Lua 脚本
- Kong 是 OpenResty 的一个应用，是一个 API 网关，具有API管理和请求代理的功能。

#### Kong持久化

有两种趋势，第一种 Less DB 即不需要第三方数据库持久化；另外一种是 通过 Cassandra or PostgreSQL 协同完成持久化。

如果后续要记录 Kong 的 dao 层，动态生成表等功能，可以选择 PostgreSQL 做持久化，这样 Kong 则接近于无状态服务。

#### Kong自动读取注册中心数据

Spring Gateway属于Spring Cloud中的一个，可以和Eureka、Spring Config等高度集成，Spring Gateway可以直接读取注册中心中存在的服务信息，然后配置限流和熔断策略。

插件实现功能：

- 调用kong原生api，进行upstream，target，service，route，plugins等创建，更新，及删除操作
- 由于原生konga上大部分功能都包括，所以不做过多的功能嵌入，以防止配置的复杂性，只需做简单配置
- 此服务皆在于实现服务动态路由，服务动态上下线，服务动态寻址，根据配置动态刷新服务信息等

#### Kong 负载均衡

Kong 提供了非常优雅的一套 Admin API, 可以进行上述的负载均衡的配置, 不用像之前那样, 改 Nginx 配置文件, 还生怕出错啦.

Kong 主要四对象, 当前还有其他的 Consumer、Plugin、Tag、Certificate、Target 等等, 但这里没有用到,暂且不表

| 组件名称 | 说明                                                         |
| :------- | :----------------------------------------------------------- |
| service  | service 对应服务，可以直接指向一个 API 服务节点(host 参数设置为 ip + port)，也可以指定一个 upstream 实现负载均衡。简单来说，服务用于映射被转发的后端 API 的节点集合 |
| route    | route 对应路由，它负责匹配实际的请求，映射到 service 中      |
| upstream | upstream 对应一组 API 节点，实现负载均衡                     |
| target   | target 对应一个 API 节点                                     |

组件调用关系:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/kong-负载均衡.png)

#### Kong限流限速

*这个是 Kong 提供的优秀插件功能之一. Kong 提供了 Rate Limiting 插件，实现对请求的限流功能，通过量化请求,精准化管理.*

Rate Limiting 支持秒/分/小时/日/月/年多种时间维度的限流，并且可以组合使用。例如说：限制每秒最多 100 次请求，并且每分钟最多 1000 次请求

Rate Limiting 支持 consumer、credential、ip 三种基础维度的限流，默认为 consumer。例如说：设置每个 IP 允许每秒请求的次数。计数的存储，支持使用 local、cluster、redis 三种方式进行存储，默认为 cluster：

| 存储方式 | 说明                                                |
| :------- | :-------------------------------------------------- |
| local    | 存储在 Nginx 本地，实现单实例限流                   |
| cluster  | 存储在 Cassandra 或 PostgreSQL 数据库，实现集群限流 |
| redis    | 存储在 Redis 数据库，实现集群限流                   |

Rate Limiting 采用的限流算法是计数器的方式，所以无法提供类似令牌桶算法的平滑限流能力

#### Kong访问控制

*使用 Kong 提供的JWT 插件,实现使用 JWT 进行认证，保护后端服务的安全性。*



### 引用

1. https://sb-woms.gitbook.io/sb/ratelimit/sentinel/sentinel-01
1. https://sentinelguard.io/zh-cn/docs/golang/circuit-breaking.html
1. https://cloud.tencent.com/developer/article/1808112
1. https://juejin.cn/post/7026520607978160142
1. https://www.gotkx.com/?p=82
1. https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Sentinel%EF%BC%88%E5%AE%8C%EF%BC%89/18%20Sentinel%20%E9%9B%86%E7%BE%A4%E9%99%90%E6%B5%81%E7%9A%84%E5%AE%9E%E7%8E%B0%EF%BC%88%E4%B8%8A%EF%BC%89.md
1. https://blog.csdn.net/en_joker/article/details/108644174
1. https://blog.51cto.com/u_5650011/5393118

