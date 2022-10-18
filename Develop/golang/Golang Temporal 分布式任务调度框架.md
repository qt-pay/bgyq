## Golang Temporal 分布式任务调度框架

### Temporal 架构设计

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Temporal-arch.png)

| 角色       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| Activity   | 一段可运行代码（function），可以包含任何逻辑。由于Temporal提供了各种语言的SDK（go、java、python、php等等）所以Activity是不限制语言的。 |
| Workflow   | Activity的集合，多个Activity可以构成一个Workflow，也是调度的最小单位。 |
| Workers    | 不同语言写的Workflow可以注册到对应语言的Worker中，Worker是代码的真正执行者 |
| Temporal   | 管理注册到自己的Workers，向Workers下发任务，监听任务状态等等 |
| CLI or SDK | 任务的发起者、监控任务进度等                                 |
| Web        | 负责任务的监控、查询等。                                     |

这个架构是不是有点类似生产者-消费者模型，`CLI or SDK `是生产者，负责启动某个Workflow；`Temporal`类似 kafka\RocktMQ ，负责把请求调度某个Worker；`Workers`是消费者，负责执行请求。但深入思考一下，我还是觉得跟消息队列有很大差别：

  1. 更关注任务的执行
        消息队列的核心是消息，生产者不关心这个消息被谁消费，消息队列关注的是消息是否送达。
        而 Temporal ，关注的是任务，关注任务执行进度和结果，是否需要重启等等。
  1. 需要提前设计好Workflow
        消息队列不需要关注消费者怎么消费消息的。而 Temporal ，你必须先把Workflow的逻辑
        写好。

所以这两个系统还是不太一样的，选择 `Temporal`，是因为我们想执行某个预定任务，并保证它执行成功。比如，交易系统每晚定时对账；注册系统过段时间自动向用户发送短信等。虽然用MQ也能实现，但可能要做很多错误处理等工作。





### 原文

1. https://juejin.cn/post/7009164386648457229