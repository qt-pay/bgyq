## HTTP and RPC

而在分布式系统中，因为每个服务的边界都很小，很有可能调用别的服务提供的方法。这就出现了服务A调用服务B中方法的需求，即远程过程调用。

### Restful实现远程调用

要想让服务A调用服务B中的方法，最先想到的就是通过HTTP请求实现。是的，这是很常见的，例如服务B暴露Restful接口，然后让服务A调用它的接口。基于Restful的调用方式因为可读性好（服务B暴露出的是Restful接口，可读性当然好）而且HTTP请求可以通过各种防火墙，因此非常不错。

```bash
## http 调用
client： http://IP:PORT/Interface_func_name
```

基于Restful的远程过程调用有着明显的缺点，主要是效率低、封装调用复杂。当存在大量的服务间调用时，这些缺点变得更为突出。

仅使用SpringBoot开发N个应用时，不同应用直接的访问就是通过Rest方式，简单但是低效。

#### Rest需要考虑

##### 异步请求

使用微服务架构首先要考虑的是异步通信方式，因为异步通信的调用者不需要考虑等待服务的响应时间、

使用异步方式的好处不仅提升了整体性能，还增加了一些可靠性的因素，另外也不需要担心超时问题和在程序中设置断路器模式。

#####  广播能力

这个最典型的就是消息的“发布-订阅”模式。

#####  事务请求

服务消费者发送一个消息到第一个服务，然后发送另一个消息的第二个服务。在服务使用者执行提交之前，这些消息都保存在队列中。一旦服务使用者执行提交，两个消息就会被释放。

服务消费者将消息发送到第一个队列中，然后服务消费者业务报错， 这时可以在消息事务中进行回滚，从消息系统的队列中删除掉刚才发的消息。

使用REST实现这种事务能力就非常困难，其实就是要求服务使用者使用TCC、或者补偿方式来达到最终一致性。

### RPC远程调用

```python
### RPC
$ cat rpc_server.py
from xmlrpc.server import SimpleXMLRPCServer

def add(x, y):
    return  x + y

if __name__ == '__main__':
    server =  SimpleXMLRPCServer(('localhost', 28888))
    ## 将add()注册到rpc框架中，并取名为add_num()
    server.register_function(add, "add_num")
    print("Listening for Client")
    server.serve_forever()

$ cat rpc_client.py
from xmlrpc.client import ServerProxy

if __name__ == '__main__':
    ## 借助注册中心可以实现server provider IP的动态获取
    server = ServerProxy("http://localhost:28888")
    ## client 直接调用约定好的func名字即可
    print("2 + 3 =",server.add_num(2, 3))

## Running
$  python3.5 rpc_server.py
Listening for Client
127.0.0.1 - - [29/Mar/2020 18:51:59] "POST /RPC2 HTTP/1.1" 200 -
127.0.0.1 - - [29/Mar/2020 18:52:11] "POST /RPC2 HTTP/1.1" 200 -
...
$ python3.5 rpc_client.py
2 + 3 = 5
```



服务A调用服务B的过程是应用间的内部过程，**牺牲可读性提升效率、易用性是可取的**。基于这种思路，RPC产生了。

通常，RPC要求在调用方中放置被调用的方法的接口。**调用方只要调用了这些接口，就相当于调用了被调用方的实际方法，十分易用**。于是，调用方可以像调用内部接口一样调用远程的方法，而不用封装参数名和参数值等操作。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rpc调用原理.jpg)

RPC之间的**通信数据可读性不需要好**，只需要RPC框架能读懂即可，因此**效率可以更高**。通常使用UDP或者TCP作为通讯协议，当然也可以使用HTTP。

再说一次，不是说RPC好，也不是说HTTP好，两者各有千秋。本质上，两者是**可读性和效率之间的抉择**，**通用性和易用性之间的抉择**。

### gRPC

### 引用

1. https://www.zhihu.com/question/41609070/answer/1030913797