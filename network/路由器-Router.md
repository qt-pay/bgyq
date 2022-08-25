## 路由器-Router

1. 路由器协议
2. 组网
3. end

### DCI

数据中心互联（DCI）是一种实现多个数据中心之间互联互通的网络解决方案。

### 路由

通讯是双向的，有去有回的路由条目才是完整的路由。

#### 路由条目组成

一条路由条目需要有可达的下一跳地址（通常是网关）和 正确的**出接口**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Router-interface.png)

#### 路由条目类型

路由表中存在的路由条目都是经过路由算法计算的最优路径了。

```bash
[Huawei]display ip routing-table pro ?
  bgp     Border Gateway Protocol (BGP) routes
  direct  Direct routes
  isis    IS-IS routing protocol defined by ISO
  ospf    Open Shortest Path First (OSPF) routes
  rip     Routing Information Protocol (RIP) routes
  static  Static routes
  unr     User network routes
```

复杂路由表会有各种路由协议类型的路由信息。

20.0.0.0/8 和 20.0.0.1/32 相当于直达列车和绕路列车，匹配优先级选路时，肯定32精确匹配的优先级高。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/huawei-ip-routing-table.png)

##### Preference 

Preference：优先级，比较**不同路由来源**到达相同目标网络的优先级，数值越小优先级越高。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/route-pre.png)

##### Cost

Cost：度量值，比较**相同路由来源**到达相同目标网络的不同路径的优先级，数值越小优先级越高。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/route-cost.png)





#### 路由匹配策略

路由条目越多，路由器遍历路由表的时间肯定越长。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由条目匹配算法.png)

每当路由选路时，路由器会遍历一边路由表，从而才能对比选择一个最优路径。

目标地址8.8.8.8 和 8.8.8.0/24网络地址去匹配，能匹配到28位一致。

即00001000匹配到8.8.8.0的00000000还能匹配4位。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由匹配demo.png)

#### 静态路由

使用场景：

比如k8s 集群 calico网络，想要将容器网段添加到核心交换机，如果不想使用动态路由协议，增加原有环境的变更风险，就可以在核心交换机上配置静态路由来实现容器网段的对外发现。

特点：

* 配置简单、开销小
* 需要管理员通过手动配置进行添加和维护
* 无法根据拓扑的变化进行动态的相应
* 适用于“组网规模较小”的场景，如网络规模大或者变化频繁，则维护成本很高
* 在大型网络中往往采取动、静结合方式

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ip-route-static.png)

静态路由支持load balancer，简单的round-robin

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ip-static-route-lb.png)

静态路由通过Preference实现主备链路

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ip-static-route-backup.png)

#### 缺省路由

缺省路由：一种特殊的路由，能匹配所有的网络。

缺省路由，可以通过静态路由配置，也可以通过动态路由协议发布。

在路由表中，以到网络0.0.0.0（掩码为0.0.0.0）的形式出现，通常用于末梢网络（如：家用上网（上网流量不确定去往哪里），企业出口）

eg：ip route-static 0.0.0.0 0.0.0.0 Next_hop_IP/Next_inf

#### 动态路由

动态路由，路由器自需要维护好自己的路由信息，然后通过路由协议来学习和同步其他路由器的路由信息。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dynamic-route-1.png)

### 路由协议总览：动态路由

0.0 静态路由没路由协议这回事

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dynamic-route-2.png)

路由协议分为IGP和EGP两大类：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dynamic-route-3.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dynamic-route-4.png)

BGP用于连接不同AS

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dynamic-route-5.png)

#### Autonomous system

Autonomous systems divide global external networks into individual routing domains, where local routing policies are applied. This organization simplifies routing domain administration and simplifies consistent policy configuration.



Each autonomous system can support multiple interior **routing protocols** that dynamically exchange routing information through route redistribution. The Regional Internet Registries assign a unique number to each public autonomous system that directly connects to the Internet. This autonomous system number (AS number) identifies both the routing process and the autonomous system.

#### DV and LS

Distance-Vector: 不了解链路状态（链路带宽等），仅局部最优，会导致无法选成最优路由

Link-State: 基于SPF算法，了解全局路径信息，类似地图。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dv-and-ls.png)

 #### 组播路由协议

使用场景，视频点播/直播或者地铁车站视频投放

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/multi-broadcast-route-protocol.png)

#### 路由协议操作规则

* 路由协议是**接口上**运行的（类似连接器需要两端一致才能接通）；
* 只能学习和发布相同协议已知的路由信息；
* 如果不同的路由协议间需要交换路由信息，就需要注入（import）。类似强行加入静态路由。

如下，通过路由注入功能，R2将OSPF路由信息转成RIP路由信息注入给R1。

操作命令：import-route ospf 1

这就是类似通过编程实现数据格式的转换和插入。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/route-info-import.jpg)

#### 路由器收敛

* 当前所有路由表包含相同网络可达性信息
* 网络（路由）进入一个稳定状态
* 网络在达到收敛前无法完全正常工作

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由器-收敛.png)

#### 路由协议衡量指标

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由器衡量指标.png)

普适性：比如BGP协议很厉害，什么网络架构规模都支棱得起来，但是在小型网络环境没必要使用BGP。因为BGP对路由设备有较高要求，性能要求也高。有时，你想使用BGP但是你的网络设备可能不支持BGP。

### 策略路由 and 路由策略

**策略路由**，即**PBR（Policy-Based Routing）**，与传统的基于IP报文目的地址查找路由表进行转发不同，它可以依据用户自定义的策略进行报文转发。这个自定义的策略，可以根据报文的来源、应用、协议以及报文长度等特征来进行定义。报文可以优先根据设置的PBR进行转发，因此策略路由给组网提供了极大的灵活性。

路由策略和策略路由都是对数据控制转发起一定作用，不同就在于路由策略作用对象是“**路由**”信息，主要通过改特定路由的相关属性，进而对路由的学习，发布等进行控制。而策略路由的作用对象是“**数据包**”，主要是对转发报文的报文本身特征来进行转发控制。

#### **策略路由分类**

策略路由一般分为**前策略路由**和**后策略路由**，其中前策略路由又根据策略路由配置的下一跳不生效时对报文的不同处理方式分为**强策略路由**和**弱策略路由**。

- 前策略路由：报文首先按照策略路由的下一跳进行转发。

  1）强策略路由：当策略路由下一跳不可达时，报文直接被丢弃。

  2）弱策略路由：当策略路由下一跳不可达时，报文不会直接被丢弃，而是重新查询路由表，根据路由表转发报文，没有路由表匹配，按缺省下一跳转发（apply ip-address default next-hop）。

- 路由表中明细路由>后策略路由>路由表中的默认路由，所以后策略路由优先级是介于明细路由和默认路由之间。

一般我们所说的策略路由都是指接口策略路由，策略路由应用在接口仅对进入该接口的报文生效，而对该接口出方向报文无效（包括本设备产生的报文）。而本地策略路由刚好是仅对本设备产生的报文生效，而对于转发的报文不生效。

#### 策略路由组网

【举例】某单位外网有两条企业宽带接入，但是发现在外网使用公网地址管理设备时经常出现管理不到，或者打开路由器管理界面非常慢问题。

【涉及的组网】

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/策略路由组网.jpg)

**对于这种多WAN上网的组网方式，一般建议客户使用负载均衡或者策略路由将由同一源地址发出的数据始终走其中一条外网。两条等价默认路由的设置显然不合理。**

**等价路由（Equal Cost Multi-Path）**：同一个路由协议，到同一个目的地有几条相同度量值的路由时，这些路由都会被加入到路由表中， IP包会在这几个链路上负载分担。

我们来简单分析下为什么会造成无法管理到设备的问题。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/策略路由-case.jpg)

我们从联通网络访问设备时数据到达ge2接口，我们期望路由器回复的数据也是从ge2出。但是由于之前我们是设置的**等价默认路由**。因此数据的回包变成了不确定事件，路由器在回复ping报文时会从两个出接口轮流回复ICMP应答报文。如果回包是从ge1出那么这个数据报文的源地址就是ge1接口的IP地址，那么很显然对端收到这个报文时是会丢弃的，这就是我们之前说的有时能管理设备，有时又不能的原因。

懂了懂了，从ge3回包时，可能从ge1回去了，就不能响应给ge2了。

下面配置就是确保设备的数据来回路由一致、

```bash
# 创建ACL匹配源地址为1.1.1.2的数据
acl number 3000
rule permit ip source 1.1.1.2 0

# 创建策略节点，并进入策略节点视图。
# policy-based-route policy-name [ deny | permit ] node node-number
# permit：指定策略节点的匹配模式为允许模式。缺省匹配模式为permit。
policy-based-route 1 node 0
if-match acl 3000
apply ip-address next-hop 1.1.1.1

# 全局调用本地策略路由
ip local policy-based-route 1
```



有一点问题需要给大家解释下，为什么ACL要匹配源地址为公网地址的数据。是因为访问路由器时目的地址为路由器其中一个接口的公网地址，路由器回包时必定源地址是以此公网口地址，目的地址是访问发起方进行回包。这样我们就可以让访问设备的数据来回路由一致。是不是学到了什么了？

### IGP

内部网关协议(IGP：Interior GatewayProtocol)，适用于单个ISP的统一路由协议的运行，一般由一个ISP运营的网络位于一个AS(自治系统)内，有统一的ASnumber(自治系统号)，用来处理内部路由。

**OSPF比RIP强大的地方是，OSPF对整网的拓扑结构了如指掌，一旦某一条路径断了，可以及时选择备份链路，对通信的影响小。**

作为目前主流的IGP协议，OSPF主要是为了解决RIP的三大问题而出现的，比如：收敛很慢、容易产生路由环路以及可延展性差等。

#### OSPF

https://www.cfdzsw.com/2021/06/23/%E7%B2%BE%EF%BC%81%E4%B8%87%E5%AD%9715%E5%9B%BE%E8%AF%A6%E8%A7%A3ospf%E8%B7%AF%E7%94%B1%E5%8D%8F%E8%AE%AE/

开放式最短路径优先（英语：Open Shortest Path First，缩写为 OSPF）是一种基于IP协议的路由协议。

它是大中型网络上使用较为广泛的IGP协议。OSPF是对链路状态路由协议的一种实现，运作于自治系统内部

##### OSPF工作原理

在OSPF网络中，每一台运行OSPF协议的网络设备都会根据自己周围网络拓扑生成链路状态通告LSA（Link State Advertisement），并通过更新报文将LSA发送给网络中的其它设备。

LSA更新完成后，每台设备都会生成自己的LSDB（Link State DataBase）。

LSA是对路由器周围网络拓扑结构的描述，LSDB则是对整个自治系统的网络拓扑结构的描述。

路由器将LSDB转换成一张带权的有向图，这张图便是对整个网络拓扑结构的真实反映。在网络拓扑稳定的情况下，各个路由器得到的有向图是完全相同的。

路由器根据最短路径优先(Shortest Path First)算法计算到达目的网络的路径，而不是根据路由通告来获取路由信息。

每台路由器根据有向图，使用SPF算法计算出一棵以自己为根的最短路径树，这棵树给出了到自治系统中各节点的路由。相对于RIP，这种机制极大地提升了路由器的自主选路能力，使得路由器不再依靠路由通告进行选路。

> 我们需要注意的是，运行OSPF的设备交互的不是路由表，而是带有链路状态。路由信息是每台设备根据自己的LSDB计算出来的。设备自主选路
> SPF算法只会算出最短的路径，也就决定了他无法想EIGRP那样，实现不等价路由负载均衡。

根据上面描述的工作原理，OSPF运行机制大概可以分为如下几步

1. 通过交互Hello报文形成邻居关系
2. 通过泛洪LSA通告链路状态信息
3. 通过组建LSDB形成带权有向图
4. 通过SPF算法计算并形成路由
5. 维护和更新路由表

### EGP

EGP是相对于IGP来说的话，那么EGP实际上有两种：一种是BGP，另一种是EGP（没错，重名而已）。

AS之间交换路由信息的路由协议称为外部路由协议（缩写是EGP），最早的时候因为只有一种协议，所以被命名为EGP。

### BGP



IGP发现中，从RIP到后来的EIGRP，OPPF，ISIS，协议发展迅速。但是总体来讲，IGP认为同一个AS之间的路由器是可以相关信息的，所以，IPG的自动发现和路由计算大多使用完全开放状态，人工干预较少。

不同的AS互联的需求推动产生了EGP（外部网关协议），EGP的主要目的是在不同的AS之间传递路由。最早的EGP只有单纯的发布网络可达信息，不做路由优选和环路保护设计等缺陷，很快就被BGP取代了。

所以，BGP的产生，根源上是为了解决不同区域之间的路由传递问题。

#### 协议

<https://cshihong.github.io/2018/01/23/BGP%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/>

0.0



#### BGP防环

https://wangshichao.com/f5f977db/

AS_Path属性按矢量顺序记录了某条路由从本地到目的地址所要经过的所有AS编号。在接收路由时，设备如果发现AS_Path列表中有本AS号，则不接收该路由，**AS_Path属性解决了EBGP环路问题**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/AS_PATH防环.png)

如图，假设RTC从右边AS100收到一条路由信息，记录为99.0.3.0/22（100，65530，65101），发现自己的AS号在里面，则丢弃。

#### 黑洞



#### iBGP 替代 IGP

先说结论，iBGP不能替代IGP，但是eBGP可以，并且已经有一个非常有说服力的实践案例——阿里云。

路由协议的选择是一个工程问题，协议本身的设计和特性则是一个历史问题，没有绝对的谁行或者谁不行，并且也不能因为C厂可劲收割用户智商税就说这个东西成本太高不合适。

工程问题就是说，脱离实际的需求和工程谈协议对比和优劣都是扯淡。

我虽然吹了一波全eBGP，但是到了自己的生产网的环网骨干上，老老实实的用起来了IGP+iBGP的MPLS VPN方案，除了多租户/流量工程/SR这些考虑之外，也想要从基础路由的路径规划中解脱出来——现在基础路由我只需要规划IGP Metric值就可以了。

我们先来看一下题主的问题——

> 既然我们已经有了IBGP用于AS内部路由，那么OSPF与RIP这些内部路由IGP为啥还要支持呢？通用BGP岂不是最方便省事？

我们也可以完全不使用iBGP于AS内部路由，你完全可以设计IGP和BGP之间的路由重分发，并且这很常见。（笑......



题主的这个问题是许多网络初学者都会犯的一个误区，那就是**忽略了路由协议的“历史性”。**路由协议不是一夜之间冒出来的，仅BGP本身的RFC都更新了三个版本，这里引用下我自己的文章——

> 最老的RFC是RFC1654，后被RFC1771废弃，再后来是RFC4271，发布于2006年，所以今天我们看BGP的RFC，直接奔着RFC4271去就可以了。

我就不提BGP前身还有个EGP了吧……所以题主首先要明白，你今天看到的BGP的样子是发展了二十多年后的样子，而不是一开始的样子。

阿拉伯网工对于iBGP提到了这样一点——

> iBGP有个天生的短板是eBGP用到的AS-PATH防环机制对它无效，只能用传一跳的方式来防环，导致iBGP路由器之间必须是full mesh的逻辑拓扑，或者使用RR，扩展性不如IGP。

首先无法同意的是，这不是一个“短板”，你只能说它就是这么设计的，如果说这是个短板，那么在广域互连场景上，IGP简直浑身都是短板一无是处了，然而实际上，SP的网络中也会使用IGP来搭建一个骨架——传loopback路由。其次无法同意的是，都提到RR了，还说扩展性不如IGP，RR这个冤啊，

BGP是一个典型的分布式系统，所以它依赖各种路径属性来防止环路，控制选路，其中AS-PATH这个属性，让BGP即使不使用泛洪机制，也能知道收到的路由经过了哪些AS。精妙的属性构建起来的分布式系统，让BGP有能力支撑一个非常庞大的网络——我无法想象在整个Internet范围上去跑个泛洪型的协议会发生什么事情。

所以当我们将BGP用在一个AS内部的时候（在同一个AS内部的IBGP，AS-PATH是一样，无法通过AS-PATH防环），因为没有AS-PATH了，防环就没了，又没有泛洪型的路径学习，只好引入单跳限制了。其实如果你认真研究引入RR后BGP所额外使用的那些路径属性就会发现，它就是一种水平分割的实现——收到的更新包含自己的`ORIGINATOR_ID`时就丢弃嘛。

那么怎么才能在内部使用到AS-PATH属性呢——打破常规思路，用eBGP不就好了，不就传个路由嘛。在数据中心内部，有着严格的拓扑层次设计，使用eBGP作为整网唯一的路由协议，在路由协议层面，简化了网络构建的复杂性。

所以我一开始才说，这是个工程问题，你不能脱离实际情况来讨论。比如同样是数据中心，为什么我们使用的就是iBGP & RR呢？因为我们跑了VXLAN/EVPN，Cisco就是这么实现的。

关于数据中心内部使用eBGP的案例，至少阿里公开写书说自己是这么干的，国内国外能跟阿里有同样网络规模的，十个指头可以数过来吧？

所以答主想的是没错的，国内最厉害的网工也是这么想的，全用BGP肯定更省事。我自己的网络里也是iBGP+eBGP混合，那些跑不动BGP的设备逐渐都被淘汰掉了。

当然由于我们使用的是VXLAN/EVPN，没有阿里的魄力，所以仍要使用IGP去传loopback路由，从这个角度来说，确实无法完全抛弃IGP。但也仅此而已，IGP沦为了给BGP打工的。

------

我们再来看一下阿拉伯网工的其它说法，不是为了对线啊，只是我觉得说的不妥——

> iBGP不像IGP具有自动发现邻居以及和peer自动建立邻居关系的能力。

完全的自动发现iBGP做不到，但是`listen range`也可以让你的网络实现基本自动化上线，不是什么大问题。 

IGP的自动发现和自动建立邻居既是优势也是缺陷，在设计不当的时候，或者在本就应该控制邻居建立的情况下使用了IGP，反而会带来额外的麻烦。

一个生产上的例子是，在一个共享介质的多路访问网络里，一堆路由器A要相互建立邻居，一堆路由器B要相互建立邻居，堆A和堆B不要建立邻居，你咋办……？两个骚操作——

- password配置不一样
- 单播指neighbor

都不是好办法。

自动发现邻居和自动建立邻居关系只是一个特定场景的需求，BGP作为一个域间路由协议，其最初被设计出来的时候就没有这个需求，后来有这个需求后，就有了`listen range`的功能。

`listen range`大概是这么配的——

```text
router bgp 65100
 bgp listen range 10.1.0.0/24 peer-group TEST
```

> 默认状态下iBGP的收敛速度不如IGP， 比如OSPF的hello timer和dead timer分别是10秒和40秒，BGP则是60秒和180秒。快速收敛本来就不是BGP设计之初的目的**（这也是iBGP和IGP相比最大的短板）**，如果你手动把BGP的hello和dead timer调低，比如分别改成5秒和15秒，思科的路由器还会提示你这么做会导致peer flapping发生概率升高的问题：

hello timer和dead timer跟收敛有什么关系？邻居建立了BGP就马上收敛了吗？hello timer和dead timer只是和BGP邻居本身出现问题时的收敛有关，这个对于IGP来说也是一样的，设成多少完全取决于你的链路质量，至于flapping的问题，如果你的链路已经flapping到了BGP邻居在反复震荡了，你这链路真的还能用吗？这已经不是一个BGP可以解决的问题了，先去解决链路质量问题吧。

要在邻居出现问题时快速收敛，还有比调timer更快的方案——BFD，由BFD引起的抖动，比调低timer严重多了，在我们使用的一些虚拟二层专线上，个别线路上因为存在难以被观察到的丢包引起了BFD抖动，进而引起了BGP的震荡，我们在这些链路上关掉了BFD，但是即便BFD无法工作了，调低timer仍然是有效的。

不是说我调了timer它就一定会震荡，不一定的，古时候广域网链路窄的很，当然我们不会喜欢BGP老是去发hello，但是现在广域网链路随随便便上百M，Keepalive那点儿数据包，说难听的，够干啥的啊？

(要素插入)

BGP其实有非常多的收敛优化功能，比如这个——

这都2020年了，别担心什么CPU扛不住之类的问题了，去看看ISR4K路由器跑IWAN时的扩展性，4451作HUB路由器，全功能IWAN，带个几百个Site都是洒洒水啦，ASR1K直接1000个Site起步，全功能什么概念？BGP算个啥？NHRP/IPSec/NBAR/PFR/NETFLOW哪个不消耗CPU？

> \6. iBGP的同步（Synchronization）机制注定了它和IGP有缘，在不使用IGP的时候，你如果不关掉Synchronization的话，iBGP表中的前缀根本不能进入路由表，反观IGP就没有类似这样的顾虑。

现在默认关，此题结束。

> \5. 在选路规则方面，IGP通常只有一个metric，比如OSPF和ISIS看链路开销，RIP看跳数，EIGRP有一套通过链路带宽和延时计算metric的公式（我承认也很复杂，但是还是比不上BGP），而BGP选路规则比IGP复杂很多（但从某个角度来说这也是它的优势，更方便我们手动做TE）。

不是从某个角度来说，而是这本来就是优势。我们不能因为一个东西“挺复杂的”就说这是劣势，BGP的选路原则和路径属性设计非常精妙，这套东西不管怎么比都会是个优势，而不是劣势。

只要肯下点功夫理解透了，其实十三条选路原则很少会比到后面几条去，以我的工程经验来看，最多比到IGP度量去，如果要施加路由策略，那就比到AS-PATH或者MED就差不多出结果了。

这可能是路由上最有价值去钻研的东西了，在任何情况下都没理由说它是劣势。

> \7. 预算和成本考虑，低端、便宜的路由器都能支持一些IGP，但不一定支持BGP，小微企业没必要花那个冤枉钱去买更贵的支持BGP的路由器。

在2020年，我们建议小微企业全面上云，解雇网工。

不好意思，又皮了。



iBGP requires a full mesh or use of mitigation like confederations or route reflectors, BGP doesn't converge with anything like the speed of OSPF, etc.

Each OSPF router would have a full understanding of all the routes that are in the area in which it resides without needing a full mesh, and it converges very, very quickly.

Using an IGP is recommended with iBGP. Without the IGP, iBGP must neighbor on external-facing interfaces, with an IGP, iBGP can neighbor on loopback interfaces which never go down, and can have multiple paths to reach.

I have seen iBGP-only for local routing, but it is more difficult and fragile.



### VRF

Virtual Routing Forwarding

Tips: Virtualization is VRF in the router, VLAN in the switch, trunk (dot1q tagging) on the Ethernet link, context or VDOM on the firewall and VM on the server.

> 虚拟化 是 VRF之于路由器， VLAN之于交换机，trunk之于以太网连接，VDOM之于防火墙，VM之于服务器 

VRF将一个路由器分隔成两个，使其用于独立的路由表，类似VLAN隔离广播域。

场景如下，可以补个图：

假设PC1与R2这一侧的网络属于一个独立的业务；PC2与R3这一侧的网络属于另一个独立的业务，由于设备资源有限或者其他方面的原因，这两个独立的业务的相关节点连接在R1上，也就是同一台设备上。那么在完成相关配置后，R1的路由表如上图所示。
现在如果PC1要发一个数据包到2.2.2.2，那么这个数据包在到达R1后，R1就会去查看自己的路由表，发现有一条2.2.2.0/24的路由匹配，因此将这个IP包从GE0/0/2口转发给192.168.100.2。这是没有问题的，然而如果PC1要访问3.3.3.0/24网络呢？也是无压力的，因为数据包到达R1后，她照样查找路由表结果发现有匹配的路由，因此将数据包转给R3。但是实际上，从业务的角度考虑，我们禁止PC1访问3.3.3.0/24网络。

VRF1及VRF2有了自己的接口，也有了自己的路由表。并且相互之间是隔离的。

### 名词

#### RP

Root Port



### 路由协议

##### HSRP

 热备份路由协议（CISCO私有）

作用：网关冗余，一个组中有一个活跃路由器，实现负载分担时，可配置多个热备组。

原理：通过虚拟IP、虚拟MAC地址在多个网关间切换

##### VRRP

虚拟路由冗余协议（IETF开发，公有）

作用：网关冗余，一个组中有一个活跃路由器，实现负载分担时，可配置多个热备组。

原理：通过虚拟IP、虚拟MAC地址在多个网关间切换。

> 虚拟IP可以和某个实际IP相同

##### GLBP

 网关负载分担协议（CISCO私有）

作用：网关冗余，一个组中可以有多个转发者，实现负载分担，配置工作量小

原理：通过活跃网关给转发者分配虚拟MAC地址，不同客户端通过不同转发者转发数据

##### PBR

 基于策略的路由

作用： PBR让管理员能够指定路由策略，而不是使用路由表进行基于目的地的路由

原理： 要配置PBR，需要配置一个路由映射表，然后将该路映射表应用于接口（用于入站分组）

##### RIP

 路由信息协议

作用：距离矢量路由协议（动态路由学习）

原理：通过广播或组播交换路由表

##### OSPF

 开放式最短路径优先

作用： 链路状态路由协议（动态路由学习）

原理：通过hello建立并维护邻接关系；更新LSA生成链路状态数据库；以自己为中心计算生成路由表

##### EIGRP

增强型内部网关路由协议

作用：高级距离矢量路由协议（动态路由学习）

原理：

##### BGP



作用： 路径矢量路由协议（动态路由学习）

### 引用

1. https://www.zhihu.com/question/411029743/answer/1516720777
2. https://wangshichao.com/6590a70/
2. https://zhuanlan.zhihu.com/p/36945542
3. end