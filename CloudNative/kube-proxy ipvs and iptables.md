## kube-proxy: ipvs and iptables

### iptables模式

在 Linux 操作系统中使用 `netfilter` 处理防火墙工作。这是一个内核模块，决定是否放行数据包。iptables 是 netfilter 的前端。

概念：`man iptables`  `iptables-save`很好用啊把规则转成对应的命令

应用例子：`man iptables-extensions` 还可以配置route tables，很野。

核心：iptables 中的 rules 位置越靠前优先级越高。

`chain.tables` 即默认表包含链，`rule.DefaultChain.table`,因为表有优先级，所以不同表中的规则有了优先级。

即： `rule_A.PREROUNTING.mangle` 与`rule_B.PREROUNTING.nat` 为互斥条件，则`rule_A`生效.

 Default_Chain是Class_iptables里面的全局变量，tables is instance of Class_iptables Object。而[自定义链][User-Defined Chains]则是各个实例中的成员变量。且自定义链必须被默认链引用才能生效。Target类似每个实例的成员变量。

> User-Defined Chains (sub chain) 用户定义的链被称为子链

#### 链

每条链负责一种特定任务。

- `PREROUTING`：决定数据包刚刚进入网络端口时的对策。有几种不同的选择，例如修改数据包（NAT），丢弃数据包或者什么都不做使其通过；
- `INPUT`：其中经常包含一些用于防止恶意行为的严格规则，防止系统遭到入侵。开放或者屏蔽端口的行为就是在这里进行的；
- `FORWARD`：顾名思义，负责数据包的转发。在将服务器作为路由器的时候，就需要在这里完成任务。
- `OUTPUT`：这里负责所有的网络浏览的行为。这里可以限制所有数据包的发送。
- `POSTROUTING`：发生在数据包离开服务器之前，数据包最后的可跟踪位置。

#### 表

- `Filter`：缺省表，这里决定是否允许数据包出入本机，因此可以在这里进行屏蔽等操作；
- `Nat`：是网络地址转换的缩写。下面会有例子说明；
- `Mangle`：仅对特定包有用。它的功能是在包出入之前修改包中的内容；
- `RAW`：用于处理原始数据包，主要用在跟踪连接状态，下面有一个放行 SSH 连接的例子。
- `Security`：负责在 Filter 之后保障安全。

#### k8s iptables流量

基于iptables实现负载方式只有random

为了能够进行包过滤和 NAT，Kubernetes 会创建一个 `KUBE-SERVICES` 链，把所有 `PREROUTING` 和 `OUTPUT` 流量转发给 `KUBE-SERVICES`：

```bash
$ iptables -t nat -L PREROUTING | column -t
Chain            PREROUTING  (policy  ACCEPT)                                                                    
target           prot        opt      source    destination                                                      
cali-PREROUTING  all         --       anywhere  anywhere     /*        cali:6gwbT8clXdHdC1b1  */                 
KUBE-SERVICES    all         --       anywhere  anywhere     /*        kubernetes             service   portals  */
DOCKER           all         --       anywhere  anywhere     ADDRTYPE  match                  dst-type  LOCAL
```

使用 `KUBE-SERVICES` 介入包过滤和 NAT 之后，Kubernetes 会监控通向 Service 的流量，并进行 SNAT/DNAT 的处理。在 `KUBE-SERVICES` 链尾部，会写入另一个链 `KUBE-SERVICES`，用于处理 `NodePort` 类型的 Service。

`KUBE-SVC-2IRACUALRELARSND` 链会处理针对 `ClusterIP` 的流量，否则的话就会进入 `KUBE-NODEPORTS`：

```bash
$ iptables -t nat -L KUBE-SERVICES | column -t
Chain                      KUBE-SERVICES  (2   references)                                                                                                                                                                             
target                     prot           opt  source          destination                                                                                                                                                             
KUBE-MARK-MASQ             tcp            --   !10.244.0.0/16  10.103.46.104   /*  default/webapp                   cluster  IP          */     tcp   dpt:www                                                                          
KUBE-SVC-2IRACUALRELARSND  tcp            --   anywhere        10.103.46.104   /*  default/webapp                   cluster  IP          */     tcp   dpt:www                                                                                                                                             
KUBE-NODEPORTS             all            --   anywhere        anywhere        /*  kubernetes                       service  nodeports;  NOTE:  this  must        be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

看看 `KUBE-NODEPORTS` 的内容：

```bash
$ iptables -t nat -L KUBE-NODEPORTS | column -t
Chain                      KUBE-NODEPORTS  (1   references)                                            
target                     prot            opt  source       destination                               
KUBE-MARK-MASQ             tcp             --   anywhere     anywhere     /*  default/webapp  */  tcp  dpt:31380
KUBE-SVC-2IRACUALRELARSND  tcp             --   anywhere     anywhere     /*  default/webapp  */  tcp  dpt:31380
```

看起来 `ClusterIP` 和 `NodePort` 处理过程是一样的，那么看看下面的处理流程：

```bash
# statistic  mode  random -> Random load-balancing between endpoints.
$ iptables -t nat -L KUBE-SVC-2IRACUALRELARSND | column -t
Chain                      KUBE-SVC-2IRACUALRELARSND  (2   references)                                                                             
target                     prot                       opt  source       destination                                                                
KUBE-SEP-AO6KYGU752IZFEZ4  all                        --   anywhere     anywhere     /*  default/webapp  */  statistic  mode  random  probability  0.50000000000
KUBE-SEP-PJFBSHHDX4VZAOXM  all                        --   anywhere     anywhere     /*  default/webapp  */

$ iptables -t nat -L KUBE-SEP-AO6KYGU752IZFEZ4 | column -t
Chain           KUBE-SEP-AO6KYGU752IZFEZ4  (1   references)                                               
target          prot                       opt  source          destination                               
KUBE-MARK-MASQ  all                        --   10.244.120.102  anywhere     /*  default/webapp  */       
DNAT            tcp                        --   anywhere        anywhere     /*  default/webapp  */  tcp  to:10.244.120.102:80

$ iptables -t nat -L KUBE-SEP-PJFBSHHDX4VZAOXM | column -t
Chain           KUBE-SEP-PJFBSHHDX4VZAOXM  (1   references)                                               
target          prot                       opt  source          destination                               
KUBE-MARK-MASQ  all                        --   10.244.120.103  anywhere     /*  default/webapp  */       
DNAT            tcp                        --   anywhere        anywhere     /*  default/webapp  */  tcp  to:10.244.120.103:80

$ iptables -t nat -L KUBE-MARK-MASQ | column -t
Chain   KUBE-MARK-MASQ  (24  references)                         
target  prot            opt  source       destination            
MARK    all             --   anywhere     anywhere     MARK  or  0x4000
```

> 注意：输出内容已经被精简

可以通过`iptables -t nat -L KUBE-SERVICES`和`iptables -t nat -L KUBE-NODEPORTS`详细查看

- ClusterIP：`KUBE-SERVICES` → `KUBE-SVC-XXX` → `KUBE-SEP-XXX`
- NodePort：`KUBE-SERVICES` → `KUBE-NODEPORTS` → `KUBE-SVC-XXX` → `KUBE-SEP-XXX`

> NodePort 服务会有一个 ClusterIP 用于处理内外部通信。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/kube-proxy-iptables-traffic.png)

#### masqueradeALL参数

`--masquerade-all`纯 iptables 代理，对所有通过集群 service IP发送的流量进行 SNAT（通常不配置）。

没开启的时候，在node_A上ping k8s_svc_B，只有pod_B恰好在node_A上时才能ping同。

https://developer.jdcloud.com/article/1466

https://developer.jdcloud.com/article/1470

### ipvs模式

基于ipvs的cluster ip类型service的实现原理，本质上是在iptable的PREROUTING chain以及相关target中利用ipset来匹配cluster ip，完成对即将做MASQUERADE伪装的items的mark标记，同时结合ipset也减少了iptable中的entry数量。

#### ipvs cluster ip

流量测试的时候，将cluster-ip 绑在的网络设备kube-ipvs0上就可以ping通了。

https://mp.weixin.qq.com/s?__biz=MzI0MDE3MjAzMg==&mid=2648393263&idx=1&sn=d6f27c502a007aa8be7e75b17afac42f&chksm=f1310b40c64682563cfbfd0688deb0fc9569eca3b13dc721bfe0ad7992183cabfba354e02050&scene=21#wechat_redirect

#### ipvs nodeport

https://cloud.tencent.com/developer/article/1607777



#### no route to host

[https://mp.weixin.qq.com/s?\_\_biz=MzU5Mzc0NDUyNg==&mid=2247483806&idx=1&sn=7450d83002e4220354dda7dcfab033ab&chksm=fe0a867fc97d0f692dafa94b6e0c7323a9d72a8eec4cca000d85dda84d73148dab930d900d8f&scene=21\#wechat\_redirect](https://mp.weixin.qq.com/s?__biz=MzU5Mzc0NDUyNg==&mid=2247483806&idx=1&sn=7450d83002e4220354dda7dcfab033ab&chksm=fe0a867fc97d0f692dafa94b6e0c7323a9d72a8eec4cca000d85dda84d73148dab930d900d8f&scene=21#wechat_redirect)

这个问题通常发生的场景就是类似于我们测试环境这种：ServiceA 对外提供服务，当外部发起请求，ServiceA 会通过 RPC 或 HTTP 调用 ServiceB，如果外部请求量变大，ServiceA 调用 ServiceB 的量也会跟着变大，大到一定程度，ServiceA 所在节点源端口不够用，复用`TIME_WAIT`状态连接的源端口，导致五元组跟 IPVS 里连接转发表中的`TIME_WAIT`连接相同，IPVS 就认为这是一个存量连接的报文，就不判断权重直接转发给之前的 rs，导致转发到已销毁的 Pod，从而发生 “No route to host”。

如何规避？集群规模小可以使用 iptables 模式，**如果需要使用 ipvs 模式，可以增加 ServiceA 的副本，并且配置反亲和性 \(podAntiAffinity\)，让 ServiceA 的 Pod 部署到不同节点，分摊流量，避免流量集中到某一个节点，导致调用 ServiceB 时源端口复用。

#### **ipvs pro.**

ipvs 在大型集群规模部署时，性能更好；

ipvs 可以动态修改 ipset 集合；

ipvs 支持更为复杂的调度算法（最小负载、最少连接、加权等）；

ipvs 支持服务器健康检查和连接重试等；



#### **ipvs con.**

ipvs 无法提供包过滤、SNAT、masquared等功能，在某些场景下，还是需要与 iptables 搭配使用（如 Service 类型为 NodePort 时）



### 引用

1. https://cloud.tencent.com/developer/article/1607777
2. https://blog.fleeto.us/post/life-of-a-packet-in-k8s-3/