## 注册中心 and 微服务框架--draft

### 问题场景

注册中心同时注册有虚拟机服务组件和容器服务组件，导致vm通过注册中心获取到container的provider IP时，无法访问。

因为不是calico/cilium组件，无法通过bgp协议或者静态路由，直接打通vm和contianer网络，导致vm IP和pod IP无法互通。

### 注册中心 and 配置中心

今天有点迷...搞不清zk是注册中心还是配置中心。

首先，明确一下，zk和ectd类似，都是保证了一致性的key-value数据库。

zk作为注册中心时，创建service path作为key，服务的provider IP作为value，用于其他服务调用。

有时候，配置中心会纳管zookeeper集群的注册信息，可以通过配置中心管理界面来修改zk上注册的各个服务地址或者端口等其他信息。

还是好迷....

注册中心不也要保障数据的一致性么....

#### 配置中心

配置中心主要看CAP吧...

C-数据一致性；A-服务可用性；P-服务对网络分区故障的容错性

#### 注册中心

服务注册，即把服务注册到服务中心，就是在注册中心进行一个登记，注册中心存储了该服务的IP、端口、调用方式(协议、序列化方式)等。

除了关注cap还要关心：负载策略、服务自动下线(心跳检测)、健康检测和支持访问协议等等这些吧

注册中心更复杂点，比如consul集群做注册中心，还需要在每个节点上部署consul agent。

#### zk作注册中心的弊端

作为一个分布式协同服务，ZooKeeper非常好，但是对于Service发现服务来说就不合适了；因为对于Service发现服务来说就算是返回了包含不实的信息的结果也比什么都不返回要好；再者，对于Service发现服务而言，宁可返回某服务5分钟之前在哪几个服务器上可用的信息，也不能 因为暂时的网络故障而找不到可用的服务器，而不返回任何结果。所以说，用ZooKeeper来做Service发现服务是肯定错误的。

当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。**但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。**在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

#### zk应用开发

Zookeeper应用开发，需要使用Zookeeper的java 客户端API ，去连接和操作Zookeeper 集群。

可以供选择的java 客户端API 有：Zookeeper 官方的 java客户端API，第三方的java客户端API。

Zookeeper官方的 客户端API提供了基本的操作，比如，创建会话、创建节点、读取节点、更新数据、删除节点和检查节点是否存在等。但对于开发人员来说，Zookeeper提供的基本操纵还是有一些不足之处。

Zookeeper API不足之处如下：

（1）Zookeeper的Watcher是一次性的，每次触发之后都需要重新进行注册；

（2）Session超时之后没有实现重连机制；

（3）异常处理繁琐，Zookeeper提供了很多异常，对于开发人员来说可能根本不知道该如何处理这些异常信息；

（4）只提供了简单的byte[]数组的接口，没有提供针对对象级别的序列化；

（5）创建节点时如果节点存在抛出异常，需要自行检查节点是否存在；

（6）删除节点无法实现级联删除；

第三方开源客户端主要有zkClient和Curator。

##### zkClient

ZkClient是一个开源客户端，在Zookeeper原生API接口的基础上进行了包装，更便于开发人员使用。zkClient客户端，在一些著名的互联网开源项目中，得到了应用，比如：阿里的分布式dubbo框架，对它进行了集成使用。

zkClient解决了Zookeeper原生API接口的很多问题。比如，zkClient提供了更加简洁的api，实现了session会话超时重连、Watcher反复注册等问题。虽然ZkClient对原生API进行了封装，但也有它自身的不足之处。

具体如下：

（1）zkClient社区不活跃，文档不够完善，几乎没有参考文档；

（2）异常处理简化（抛出RuntimeException）；

（3）重试机制比较难用；

（4）没有提供各种使用场景的参考实现；

##### Curator

Curator是Netflix公司开源的一套Zookeeper客户端框架，和ZkClient一样，解决了非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等。Curator是Apache基金会的顶级项目之一，Curator具有更加完善的文档，另外还提供了一套易用性和可读性更强的Fluent风格的客户端API框架。

不止上面这些，Curator中还提供了Zookeeper各种应用场景（Recipe，如共享锁服务、Master选举机制和分布式计算器等）的抽象封装。

另外，Curator供了一套非常优雅的链式调用api，对比ZkClient客户端 Api的使用，发现 Curator的api 优雅太多。

Elegant！Elegant！Elegant！

#### Nacos

Nacos致力于发现、配置和管理微服务。Nacos提供了一组简单易用的特性集，帮助您实现动态服务发现、服务配置管理、服务及流量管理。Nacos更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构(例如微服务范式、云原生范式)的服务基础设施。Nacos支持作为RPC注册中心，例如：支持Dubbo框架；也具备微服务注册中心的能力，例如：SpringCloud框架。



### 解决思路

#### hostnetwork

pod直接使用host网络，简单，但是端口可能冲突。

多副本情况，不能调度到同一个节点上

#### 将pod IP换成node IP

使用pod所在node IP作为service provider IP并将nodeport端口作为 service port，注册到注册中心。

实现vm和pod的相互访问

#### 弊端:synagogue:

通过NodePort方式暴露服务，可能有个问题，就是Dubbo的优雅停机，导致旧节点和新节点（因为旧的节点和新节点的IP端口是一样的，注册中心无法区分是新的节点还是旧的）在一瞬间是都被注销了（此时consumer调用会有短暂的异常），然后新节点再次注册，才可调用。

```yaml
# 端口要与Service中NodePort相同
env:
 - name: DUBBO_PORT_TO_REGISTRY
   value: "30002"
# 获取Node节点的IP
 - name: DUBBO_IP_TO_REGISTRY
   valueFrom:
   fieldRef:
   eldPath: status.hostIP	

```

...

##### Dubbo

```bash
DUBBO_IP_TO_REGISTRY — 注册到注册中心的IP地址
DUBBO_PORT_TO_REGISTRY — 注册到注册中心的端口
DUBBO_IP_TO_BIND — 监听IP地址
DUBBO_PORT_TO_BIND — 监听端口
```

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dubbo-注册ip地址.jpg)

Register & bind IP address for service provider, can be configured separately. Configuration priority: environment variables -> java system properties -> host property in config file -> /etc/hosts -> default network address -> first available network address

突然，感觉Dubbo挺好的，在注册服务时加个hook，可以修改服务的注册信息。

在多网卡环境或者需要nat的网络关键，非常有用呢。

Dubbo 选取注册地址的逻辑大致分成了两步

1. 先去 /etc/hosts 文件中找 hostname 对应的 IP 地址，找到则返回；找不到则转 2
2. 轮询网卡，寻找合适的 IP 地址，找到则返回；找不到返回 null，在 getLocalAddress0 外侧还有一段逻辑，如果返回 null，则注册 127.0.0.1 这个本地回环地址

但是在多网卡、nat环境下就会出现问题。

##### 原生zookeeper：修改hosts

zkClient默认向zk注册时会读取容器主机名对应的ip地址，具体内容一般/etc/hosts最后一行附加上的hostname和ip映射，只要在启动应用之前修改hostname映射ip为宿主机ip即可。

```bash
#!/bin/bash
cp /etc/hosts /tmp/
sed -i 's/10\.[0-9]*\.[0-9]*\.[0-9]*/${Node_IP}/g' /tmp/hosts
cp /tmp/hosts /etc/
```

脚本中`Node_IP`可以是k8s pod中的环境变量，它通过fieldRef获取到pod所在node的IP

至于为什么不直接执行`sed -i /etc/hosts`

因为pod中用sed操作`/etc`目录下文件回报错，如下`sed: cannot rename`

sed: cannot rename /etc/sedl8ySxL: Device or resource busy

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sed-in-k8s.jpg)

通过`strace`可以看到`sed`的系统调用...确实有`rename()`

```bash
$ echo "a" > /tmp/a.txt
$ strace sed -i 's/a/b/' /tmp/a.txt
execve("/bin/sed", ["sed", "-i", "s/a/b/", "/tmp/a.txt"], 0x7fff1b41e568 /* 21 vars */) = 0
brk(NULL)                               = 0x560fabc66000
...
openat(AT_FDCWD, "/tmp/a.txt", O_RDONLY) = 3
ioctl(3, TCGETS, 0x7fff04cca8d0)        = -1 ENOTTY (Inappropriate ioctl for device)
fstat(3, {st_mode=S_IFREG|0644, st_size=2, ...}) = 0
umask(0700)                             = 022
getpid()                                = 32198
openat(AT_FDCWD, "/tmp/sedOf7tu4", O_RDWR|O_CREAT|O_EXCL, 0600) = 4
umask(022)                              = 0700
...
## rename系统调用
rename("/tmp/sedOf7tu4", "/tmp/a.txt")  = 0
close(1)                                = 0
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++

```

原因么...如下别人说的--

Docker treats `/etc/hosts` differently. It overwrites it whenever it wants without any regard for your modifications.



##### Nacos

在springboot项目的配置文件bootstrap.properties里添加

```bash
spring.cloud.nacos.discovery.ip=${NODE_IP}
spring.cloud.nacos.discovery.port=8766

## NODE_IP是pod动态获取的node ip
env:
  - name: NODE_IP   //此配置项通过k8s自带的 downward Api获取节点的Ip
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
```

这样注册到nacos里的ip与port就被手动指定好了。
更多Nacos配置，比如多网卡的指定

```yaml
# 如果选择固定Ip注册可以配置
spring.cloud.nacos.discovery.ip = 10.2.11.11
spring.cloud.nacos.discovery.port = 9090

# 如果选择固定网卡配置项
spring.cloud.nacos.discovery.networkInterface = eth0

# 如果想更丰富的选择，可以使用spring cloud 的工具 InetUtils进行配置
# 具体说明可以自行检索: https://github.com/spring-cloud/spring-cloud-commons/blob/master/docs/src/main/asciidoc/spring-cloud-commons.adoc
spring.cloud.inetutils.default-hostname
spring.cloud.inetutils.default-ip-address
spring.cloud.inetutils.ignored-interfaces[0]=eth0   # 忽略网卡，eth0
spring.cloud.inetutils.ignored-interfaces=eth.*     # 忽略网卡，eth.*，正则表达式
spring.cloud.inetutils.preferred-networks=10.34.12  # 选择符合前缀的IP作为服务注册IP
spring.cloud.inetutils.timeout-seconds
spring.cloud.inetutils.use-only-site-local-interfaces
```

end

##### consul



### 应用迁移至k8s

## 迁移到kubernetes的方案【转】

https://www.pianshen.com/article/7526377303/

#### dubbo注册中心部署

​    dubbo的注册中心有很多种，主流的用法是将zookeeper作为dubbo的注册中心。Dubbo的注册中心可以放在集群内部部署也可以放在集群外部部署。

​    在这里考虑到注册中心是有状态的服务，我们可以在kubernetes上部署，也可以在集群外的ECS上部署。

##### 在集群外部部署

​    如果dubbo部署在集群之外，需要集群中的部署的应用（服务提供者和服务消费者）需要能够访问dubbo所在的网络即可。

##### 在集群内部部署

​    如果dubbo部署在集群内部需要做以下事情：

​          i.      制作dubbo的相关镜像并部署在kubernetes平台；

​         ii.      为dubbo应用创建service，可以为clusterIP和LoadBalancer类型，如果用到LoadBalancer类型，可以创建内网slb；

​        iii.      应用可以通过dubbo的service name、clusterIP或者内网SLB地址进行访问。

#### 服务提供者和服务消费者全部在集群内调用

##### 应用改造及部署

​    在该种情况下，应用不需要进行改造，在application.properties文件中需要更改下dubbo.registry.address中dubbo的地址。

同时在该种情况下，应用不需要关心自己注册到dubbo中的地址，因为在同一个集群中所有的pod地址默认是通的，并且该默认情况下，应用会将pod的地址作为注册地址。 

#### 服务提供者需要提供服务给集群外部的服务消费者

##### 服务提供者与kubernetes集群都部署在阿里云上

###### 在同一VPC中

​    如果服务消费者与kubernetes集群在同一个VPC，那么只需要将将两者放到一个安全组中，保证能够互相访问即可。此时，部署在kubernetes集群上的应用是将pod IP地址注册在dubbo上。

​    注意，此时pod的IP地址规划不能与其他VPC环境的IP段冲突。

###### 在不同VPC中或线下IDC

​    如果服务消费者与kubernetes集群不在同一个VPC中，则需要先打通两者的VPC环境，使两方的应用能够互通即可，显现IDC同理。此时，部署在kubernetes集群上的应用也是将pod IP地址注册在dubbo上。

​    注意，此时pod的IP地址规划不能与其他VPC环境或者线下IDC环境的IP段冲突。

###### 线下IDC内网（无法与VPC打通）应用改造及部署

   首先，在该种情况下，应用不需要进行改造，在application.properties文件中需要更改下dubbo.registry.address中dubbo的地址。如果dubbo部署在集群内，应用应该根据情况填写能够访问到的地址。

​    其次，在部署应用前，需要先为该应用创建外网LoadBalancer类型的service，以便先获取到外网SLB的地址，在下面注册地址到dubbo的时候需要用。

最后，由于存在集群外部的应用交互调用，所以作为服务提供者不能使用pod地址，需要使用外网LoadBalancer类型service的地址。需要在application.properties中进行注册IP地址的更改。

 ```yam
 dubbo.protocol.host = EXTERNAL_SLB_IP
 dubbo.registry.address = zookeeper://IP:Port
 ```

