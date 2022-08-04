## Rabbit MQ--draft

### 基础概念

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rabbitmq-arc.png) 

* Message（消息)： 消息由消息头和消息体组成。消息头用于存储与消息相关的元数据：如目标交换器的名字 (exchange_name) 、路由键 (RountingKey) 和其他可选配置 (properties) 信息。消息体为实际需要传递的数据。
* Exchange（交换器）：类似路由表，将不同Messages安装规则给不同的Queues。交换器负责接收来自生产者的消息，并将将消息路由到一个或者多个队列中，如果路由不到，则返回给生产者或者直接丢弃，这取决于交换器的 mandatory 属性：

  * 当 mandatory 为 true 时：如果交换器无法根据自身类型和路由键找到一个符合条件的队列，则会将该消息返回给生产者；
  * 当 mandatory 为 false 时：如果交换器无法根据自身类型和路由键找到一个符合条件的队列，则会直接丢弃该消息。 
* BindingKey (绑定键）：交换器与队列通过 BindingKey 建立绑定关系。
* Binding：绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以把交换器理解成一个由绑定构成的路由表。Exchange和Queue的绑定可以是多对多的关系。
* Routingkey（路由键）：生产者将消息发给交换器的时候，一般会指定一个 RountingKey，用来指定这个消息的路由规则。当 RountingKey 与 BindingKey 基于交换器类型的规则相匹配时，消息被路由到对应的队列中。
* Queue（消息队列）：用于存储路由过来的消息。多个消费者可以订阅同一个消息队列，此时队列会将收到的消息将以轮询 (round-robin) 的方式分发给所有消费者。即每条消息只会发送给一个消费者，不会出现一条消息被多个消费者重复消费的情况。
* Connection（连接）：用于传递消息的 TCP 连接。
*  Channel（信道）：RabbitMQ 采用类似 NIO (非阻塞式 IO ) 的设计，通过 Channel 来复用 TCP 连接，并确保每个 Channel 的隔离性，就像是拥有独立的 Connection 连接。当数据流量不是很大时，采用连接复用技术可以避免创建过多的 TCP 连接而导致昂贵的性能开销。



### 应用场景

异步处理

应用解耦

流量削峰

### 常用问题定位

#### 思路

1. 生产者是否连接到mq
2. exchange是否将消息发送到指定Queue
3. Queue有木有consumer去消费message
4. consumer执行消费逻辑是否异常

#### Overview

5672端口是客户端和RabbitMQ及通信的接口，15672端口为管理界面访问web界面的端口。

登陆管理界面，点开`Overview`可以看到Queued Messages，即确实有积压的消息。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rabbitmq-overview.jpg)

#### exchange

查看Queue的Binding信息，确定该Queue可以收到来自生产者的Message即创建jenkins-slave请求。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rabbitmq-exchange.png)

#### Queue

点击界面Queues，点击出问题的应用组件对应的Queue，查看消息积压情况。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rabbitmq-queues.png)

本次故障现象是Jenkins slave pod提示已调度，但是没有任何jenkins-jnlp pod被创建。这说明任务创建成功了，Message也发出去了但是没有被消费。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rabbitmq-queue-consumer.png)

##### 解决

点击`Purge Messages`，清空积压消息，然后重启对应服务（使用fluxes.queue的应用）。

因为这些调度信息不重要，所以可以直接清空历史消息。



### 引用

1. https://blog.51cto.com/zero01/2553646
2. https://juejin.cn/post/6844904039193280526