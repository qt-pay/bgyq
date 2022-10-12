## OSPF-draft



 Open Shortest Path First

### RIP cons

RIP路由协议只是以简单度量为单位，无法感知整个路由链路的状态，从而会导致无法选择最优链路。

如下，RTB --> RTD只有一跳，RTB-->RTC-->RTD是两跳，RIP会选择B到D直连，但是B和D之间链路只有1M，

明显不是最优链路。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/rip-cons.png)



### what is OSPF:heavy_check_mark:基于AS又做了内部划分？

OSPF routing depends on the relationship that is defined between areas within a routing domain. CL commands are used to define area types and i5/OS routing neighbors.

For the OSPF protocol, an autonomous system (AS) is a collection of IP networks that are under a common administration, sharing a common routing strategy, and running only one routing protocol. **The routers and links that make up the AS are in logical groups called areas. Areas are identified by uniquely assigned numbers and an AS must define at least one area.**

When an AS is divided into multiple areas, the areas are interconnected by a router that is designated as an area border router (ABR). **By definition, an ABR has different OSPF interfaces that are attached to different OSPF areas so that it can operate in more than one area.** The ABR keeps a copy of the link-state database for each associated area.

All routers in an area have identical copies of the autonomous system topology database. Each router computes its own routing table using a spanning tree algorithm, called the Dijkstra algorithm, that computes the Shortest Path First.

OSPF supports multipath, which means it supports multiple routes to the same destination or system. When a new link-state update packet is received, the entire tree is recomputed. Other multipath calculation considerations include the following:

- The routing tree retains all multiple equal-cost routes.
- The routing tree selects only the shortest path for retention when multiple routes exists to the same destination.
- Both costs are added to the stack when two routes, that have the same costs, are to the same destination.
- OSPF always adds the route with the lowest cost to the TCP/IP stack.
- Multipath routes to the same destination are added to the TCP/IP routing table when those paths have the same cost.

**Each router in an OSPF area originates a link-state advertisement (LSA)**, which is a basic means of OSPF communication, to transport the topology of one router to all other routers in the same OSPF area. Each link-state originated advertisement, in each router, is stored in its link-state database. The database is synchronized between routers so that each router of the OSPF area, has an identical copy of the link-state database.

An ABR uses a separate, summary-LSA to advertise to all other destinations that are known to the ABR but that are outside of the area.

#### OSPF协议特点

OSPF协议具有以下特点：

- OSPF把自治系统AS（Autonomous System）划分成逻辑意义上的一个或多个区域；
- OSPF通过LSA（Link State Advertisement）的形式发布路由；
- OSPF依靠在OSPF区域内各设备间交互OSPF报文来达到路由信息的统一；
- OSPF报文封装在IP报文内，可以采用单播或组播的形式发送。

#### area

动态路由协议为了区别路由选择性连通划分的area的概念，同区域内声明的网段都互通，不同区域内的网段需要路由注入才能通.

#### OSPF路由器类型

##### IR

区域内路由器，Internal Router，该类路由器的所有接口都属于同一个OSPF区域。

##### ABR

Area Border Router，区域边界路由器。

该类路由器可以同时属于两个以上的区域，但其中一个必须是骨干区域。ABR用来连接骨干区域和非骨干区域，它与骨干区域之间既可以是物理连接，也可以是逻辑上的连接。

##### BR

Backbone Router，骨干路由器。

该类路由器至少有一个接口属于骨干区域。因此，所有的ABR和位于Area0的内部路由器都是骨干路由器。

*Backbone routers* are routing devices that have one or more interfaces connected to the OSPF backbone area (area ID 0.0.0.0)

##### ASBR

ASBR，Autonomous System Boundary Router，自治系统边界路由器。

与其他AS交换路由信息的路由器称为ASBR。ASBR并不一定位于AS的边界，它有可能是区域内路由器，也有可能是ABR。只要一台OSPF路由器引入了外部路由的信息，它就成为ASBR。

#### 骨干区域（Backbone Area）

OSPF划分区域之后，并非所有的区域都是平等的关系。其中有一个区域是与众不同的，它的区域号是0，通常被称为骨干区域。骨干区域负责区域之间的路由，非骨干区域之间的路由信息必须通过骨干区域来转发。对此，OSPF有两个规定：

* 所有非骨干区域必须与骨干区域保持连通；
* 骨干区域自身也必须保持连通。

在实际应用中，可能会因为各方面条件的限制，无法满足上面的要求。这时可以通过配置OSPF虚连接予以解决。

An OSPF *backbone area* consists of all networks in area ID 0.0.0.0, their attached routing devices, and all ABRs. The backbone itself does not have any ABRs.

```bash
## 配置area 0
ospf 1
 non-stop-routing
 area 0.0.0.0
## 但可以有多个ospf 进程
## OSPF Process 2 
ospf 2
 non-stop-routing
 area 0.0.0.0
```

end

#### 虚链接

虚连接（Virtual Link）

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/opsf-virtual-link.png)

虚连接是指在两台ABR之间通过一个非骨干区域而建立的一条逻辑上的连接通道。它的两端必须是ABR，而且必须在两端同时配置方可生效。为虚连接两端提供一条非骨干区域内部路由的区域称为传输区（Transit Area)

虚连接的另外一个应用是提供冗余的备份链路，当骨干区域因链路故障不能保持连通时，通过虚连接仍然可以保证骨干区域在逻辑上的连通

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/opsf-virtual-link-bk.png)

虚连接相当于在两个ABR之间形成了一个点到点的连接，因此，在这个连接上，和物理接口一样可以配置接口的各参数，如发送Hello报文间隔等。

两台ABR之间直接传递OSPF报文信息，它们之间的OSPF路由器只是起到一个转发报文的作用。由于协议报文的目的地址不是中间这些路由器，所以这些报文对于它们而言是透明的，只是当作普通的IP报文来转发。



### 多OSPF进程路由：路由重发

核心：

1. **不同进程之间不相互交换路由信息，默认是不通的**
2. **路由进程仅对本地路由器有意义，相连路由器的进程可以不同**
3. **同一个网络可以配置在多个OSPF路由进程中**

https://www.cnblogs.com/Wesuiliye/articles/13814796.html

　　配置OSPF路由时，首先是要在路由器上启用OSPF路由进程，而且在一个路由器上可以同时创建并运行多个OSPF进程。在创建OSPF路由进程的过程中，还需要指定与路由进程相关接口所连接的网络IP地址范围（也就是通告的网段），并分配与IP地址范围相关联的区域ID（area区域）。 

​    那这多个进程有什么用，而且不同进程之间又有什么区别和联系呢？相信很多知道熟悉OSPF的同学也不一定能够真正了解。

　　我们可以简单地把多个不同OSPF进程理解为多个不同的动态路由协议的进程。我们知道，**不同路由协议下的路由信息是不能直接进行交换的**，最终也造成通过不同路由协议学习到的动态路由都是不通的。OSPF上的不同进程也是如此，不同进程各自有不同的LSDB（链路状态数据库），彼此之间是不交换路由信息的，当然彼此之间的网络也就不会直接相通了。这就是相当于把一个物理网络划分成多个虚拟网络。但是，我们也要理解的另一个重点就是，**不同的OSPF进程仅对本地路由器有意义，也就它是仅将本地路由器划分成多个虚拟网络**。即同一网络下，不同的路由器的OSPF进程ID是没有关系的，可以相同也可以不同。 

> OSPF 进程ID只在本Router有作用，出了Router 数据包传输时，没有ospf process ID的概念了

　　下面是一个OSPF多进程的例子，假设有以下这样的一个OSPF网络，R1、R2和R3均会运行OSPF协议，但R2上配置了100和200两个进程，R1和R3各一个进程，并都宣告各自接口所在的网段。如下图所示

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/multi-ospf.png)

如果R2是一个OSPF进程的话，这4段网络都是可以互通的，但此时R2是2个OSPF进程，那192.168.1.0/24和192.168.3.0/24可以通么？答案是否定的，原因是R2上的OSPF100和OSPF200是不会互相通告路由信息的，但R2上12网段和23网段是可以通信的，因为它们在R2路由器上是直连路由，优先级最高，不需要OSPF协议的支持。所以如果想隔离这2段网络就可以采用此方法。  

　　我们又会想，那如何才能使得他们进行通信呢？可以采用**路由重分发（Redistribute）**。

```bash
router ospf 1
redistribute ospf 2 subnet

router ospf 2
redistribute ospf 1 subnet
```

　　如果R2路由器上将位于OSPF进程100的G1/0口学习到的OSPF路由和直连路由重分发到OSPF进程200，则R3路由器将学习到路由 192.168.1.0/24和192.168.12.0/24网络了。同理，如果R2路由器上将位于OSPF进程200的G2/0口学习到的OSPF路由和直连路由重分发到OSPF进程100，则R1路由器将学习到路由192.168.23.0/24和192.168.3.0/24网络了。 可能我们还会有疑问，那同一个网段是不是可以同时通告到不同的进程中呢，这样也是可以的，在上述案例中，这样做的话，也是可以做到网络互通的目的。

　　一般来说，为了进行路由备份，我们可以将**同一个网络可以配置在多个OSPF路由进程中，在不同的进程内发布了多条相同的路由条目**。在两个进程内发布了两条相同的路由条目，需要注意路由优先级的问题.

#### multi-ospf process 应用场景

OSPF多进程常用于：

1、每一个进程用来表示一类业务，如在某金融中心 ospf 1用来发布学习生产网网段，ospf 2用来发布学习各设备loopback地址，组网清晰

2、便于实现路由管理（所有网段都在同一个进程里，因为OSPF的链路状态算法特点，不便于过滤，做路由策略等，通过配置多个进程 可以在各进程灵活引入等

PS：通俗理解，两个ospf process 就是两个IGP路由

#### ospf进程号

https://forum.huawei.com/enterprise/zh/thread-326717.html

在配置OSPF时，我们采用的是router ospf命令，在该命令后面需要加上这个OSPF进程的进程号(Process-Id)，进程号用于在一台路由器上区分不同的OSPF进程。进程号的取值范围是1-65535。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ospf-process-ID.png)

上图所示的拓扑中，R1使用进程号10创建了一个OSPF进程，同时激活了自己的直连接口FE0/0，而R2使用进程号20创建了一个OSPF进程，同时也在自己的直连接口上激活了OSPF。虽然这两个进程号不一样，但是R1-R2之间的邻居关系建立是完全没有问题的。因为OSPF进程号只具有本地意义，路由器之间交互的所有OSPF报文中，都不会体现任何关于进程号的信息。进程号只在一台路由器上用于区分多个OSPF进程，因此对于R1而言，它并不关心它的直连OSPF邻居R2使用的是什么OSPF进程号。

#### multi-ospf-process：路由备份

OSPF是普遍应用的一种路由协议，在使用过程中我们经常会发现客户会遇到这样的问题。 当路由器B配置了两个OSPF进程（如下图A），为了实现路由的备份，用户在两个进程内发布了两条相同的路由条目，例如在如下图中分别在路由器A和路由器C上发布了路由条目：172.168.0.1/32

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ospf-backup.webp)



在客户原先的网络设计中，将路由器A发布的路由cost值为1，而路由器C发布的metric值被人为增加cost为200。客户认为根据OSPF协议的选路比较原则，在路由器B的路由表中应该优先选择metric最小的路由，因而会选择路由器A的接口作为该路由的下一跳，而路由器C发布的路由作为本条路由的备份.

按照客户的网络设计，当FE0/0端口故障中断的时候，172.168.0.1/32路由的下一跳将切换到RouterC上，直到FE0/0端口恢复，OSPF路由收敛完毕之后，OSPF路由表应该比较metric值而 重新选举RouterA作为该路由的下一跳。事实上，客户却在实际网络中发现，故障恢复后，下一跳依 然选择RouterC ! 这是为什么呢？

在做故障诊断之前，首先让我们复习一下OSPF协议选路的原则，当单进程的一台OSPF路由器检查一个数据包的目的地址的时候，它应该通过下面的步骤选择最优路由：

1. 选择可以和目的地址最精确匹配的路由，例如路由表中存在路由条目:172.168.64.0/18, 172.16.64.0/24 , 172.16.64.192.27，而目的地址是172.16.64.205,那么最后一条路由条目被选中。最精确的匹配总是应用最长匹配原则。
2. 区域内的路径总是优先于区域间的路径，E1的外部路径总是优先于E2外部路径
3. 不同的路由协议之间通过AD值进行比较路由，AD值相同的情况下会通过Metric进行比较

在我们的案例中，两台路由器发布的路由分别属于不同的OSPF进程，它们拥有相同的AD值和不同的metric值，

在RouterB上根据OSPF进程的启动收敛顺序，哪个进程率先把路由注入到ospf路由表，将优先选择它作为该路由的下一跳！这是因为METRIC值只会在同一进程内进行比较。在多进程OSPF的路由器上,不同进程的路由OSPF路由表只会注入OSPF路由表的顺序优先选择一条作为最佳路由放在全局路由表中，而不会针对metric属性进行选择，。

不同的OSPF进程在路由器上类似两个不同的路由协议，我们要进行路由比较的时候，可以根据AD值而不是metric值进行比较。因此我们再设计类似的网络的时候，可以通过修改不同进程的AD值达到用户期望的目的。



一、组网需求

如下图为网络拓扑，当设备无故障时，业务走主设备，当主设备或设备间链路故障时启用备份设备。要求内网设备中路由表规模及路由信息传递的数量尽可能少，不需要知道外部明细路由。

二、组网拓扑

六台路由设备，分别命名为1/2/3/4/5/6。1/3/5为主设备，2/4/6为备份设备。网段及路由分配如图所示（如路由器1的两个接口IP分别为10.0.0.1/24 、10.0.1.1/24）。内外网之间通过OSPF划分区域进行区分，对网络进行备份保护的同时，使用totally stub区域配置对内网设备隐藏外部明细路由，减少内网设备中路由表规模及路由信息传递的数量。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/路由备份-ospf.jpg)

```bash
[1]

interface LoopBack0
 ip address 1.1.1.1 255.255.255.255

#

interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 10.0.0.1 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 10.0.1.1 255.255.255.0

#

ospf 1
 area 0.0.0.1
  network 1.1.1.1 0.0.0.0
  network 10.0.0.0 0.0.0.255
  network 10.0.1.0 0.0.0.255
  stub no-summary      //配置stub no-summary区域，减小内部网络路由表数量

#

[2]

interface LoopBack0
 ip address 2.2.2.2 255.255.255.255

#

interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 10.0.0.2 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 10.0.2.2 255.255.255.0

#

ospf 1
 area 0.0.0.1
  network 2.2.2.2 0.0.0.0
  network 10.0.0.0 0.0.0.255
  network 10.0.2.0 0.0.0.255
  stub no-summary                //配置stub no-summary区域

#

[3]

 interface LoopBack0
 ip address 3.3.3.3 255.255.255.255

#

interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 20.0.0.2 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 10.0.1.2 255.255.255.0

#

interface GigabitEthernet0/2
 port link-mode route
 combo enable copper
 ip address 20.0.1.2 255.255.255.0

#

ospf 1
 area 0.0.0.0
  network 3.3.3.3 0.0.0.0
  network 20.0.0.0 0.0.0.255
  network 20.0.1.0 0.0.0.255
 area 0.0.0.1
  network 10.0.1.0 0.0.0.255
  stub no-summary                //配置stub no-summary区域

 

[4]

interface LoopBack0
 ip address 4.4.4.4 255.255.255.255

#

interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 20.0.0.1 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 10.0.2.1 255.255.255.0

#

interface GigabitEthernet0/2
 port link-mode route
 combo enable copper
 ip address 20.0.2.1 255.255.255.0
 ospf cost 100        //将路由4-6间的链路接口cost设为100，得链路无故障情况下路由

                                   优先使用主设备，即使用4-3之间的链路，通过路由3访问外网。

#

ospf 1

 area 0.0.0.0
  network 4.4.4.4 0.0.0.0
  network 20.0.0.0 0.0.0.255
  network 20.0.2.0 0.0.0.255

 area 0.0.0.1
  network 10.0.2.0 0.0.0.255
  stub no-summary              //配置stub no-summary区域

#

 

[5]

 interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 30.0.0.1 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 20.0.1.1 255.255.255.0

#

#

ospf 1

 area 0.0.0.0
  network 5.5.5.5 0.0.0.0
  network 20.0.1.0 0.0.0.255
  network 30.0.0.0 0.0.0.255

#

 

[6]

interface LoopBack0

 ip address 6.6.6.6 255.255.255.255

#

interface GigabitEthernet0/0
 port link-mode route
 combo enable copper
 ip address 30.0.0.2 255.255.255.0

#

interface GigabitEthernet0/1
 port link-mode route
 combo enable copper
 ip address 20.0.2.2 255.255.255.0

#

ospf 1
 area 0.0.0.0
  network 6.6.6.6 0.0.0.0
  network 20.0.2.0 0.0.0.255
  network 30.0.0.0 0.0.0.255

#
```



Running two OSPF processes on the same interface is not supported and does not really make any sense to me from a protocol perspective. You'll need to configure the networks such that the two processes are running on different interfaces.



### 作业

简单p2p拓扑

https://blog.csdn.net/weixin_40228200/article/details/118975991

```bash
 interface vlan 1
 ip address 192.168.1.254 255.255.255.0
 ip address 192.168.2.254 255.255.255.0
 
 interface GE1/0/1
 port link-mode route
 ip address unnumbered interface LoopBack0
 ospf network-type p2p
 ospf timer hello 3
 ospf 1 area 0.0.0.0
 save

[H3C-ospf-1-area-0.0.0.0]display ospf interface


         OSPF Process 1 with Router ID 2.2.2.2
                 Interfaces

 Area: 0.0.0.0
 IP Address      Type      State    Cost  Pri   DR              BDR
 192.168.2.1     PTP       P-2-P    1     1     0.0.0.0         0.0.0.0
 2.2.2.2         PTP       Loopback 0     1     0.0.0.0         0.0.0.0
## 配置静态路由
	
ip route-static dest-address { mask | mask-length } { next-hop-address | interface-type interface-number next-hop-address } [ preference preference-value ] [ description description-text ]
$ ip route-static 192.168.2.0 24 172.10.10.11 preference 80


===
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0 
 ospf network-type p2p
#
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 192.168.1.0 0.0.0.255
  network 0.0.0.0 255.255.255.255 // 这个不行呦
#
arp static 192.168.2.1 00e0-fc34-51e4
#

### arp
interface GigabitEthernet0/0
 ip address 192.168.2.1 255.255.255.0 
 ospf network-type p2p
#
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 192.168.2.0 0.0.0.255
  network 0.0.0.0 255.255.255.255 
#
arp static 192.168.1.1 00e0-fc37-3029
#

##　显示路由表
display ip routing-table

```

end

```bash
### 华三设备上就是不行
[H3C]display ospf routing

         OSPF Process 1 with Router ID 192.168.1.1
                  Routing Table

                Topology base (MTID 0)

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 192.168.1.0/24     1        Stub    0.0.0.0         192.168.1.1     0.0.0.0
 192.168.2.0/24     2        Stub    192.168.2.1     192.168.2.1     0.0.0.0

 Total nets: 2
 Intra area: 2  Inter area: 0  ASE: 0  NSSA: 0


## ar2
[H3C]display  ospf routing

         OSPF Process 1 with Router ID 192.168.2.1
                  Routing Table

                Topology base (MTID 0)

 Routing for network
 Destination        Cost     Type    NextHop         AdvRouter       Area
 192.168.1.0/24     2        Stub    192.168.1.1     192.168.1.1     0.0.0.0
 192.168.2.0/24     1        Stub    0.0.0.0         192.168.2.1     0.0.0.0

 Total nets: 2
 Intra area: 2  Inter area: 0  ASE: 0  NSSA: 0
[H3C]

```



OSPF 路由协议是链路状态型路由协议，这里的链路指的是设备上的接口。链路状态型路由协议基于连接源和目标设备的链路状态作出路由的决定。链路状态是接口及其邻接网络设备的关系的描述，接口的信息即链路的信息，也就是链路的状态（信息）。这些信息包括接口的 IPv6 前缀（ prefix ）、子网掩码、接口连接的网络（链路）类型、与该接口在同一网络（链路）上的路由器等信息。这些链路状态信息由不同类型的 LSA 携带，在网络上传播。

### 组播地址

根据不同的数据链路层协议网络，OSPF的组播地址也不同。

**OSPF点到点网络**

是连接单独的一对路由器的网络,点到点网络上的有效邻居总是可以形成邻接关系的,在这种网络上,OSPF包的目标地址使用的是**224.0.0.5**
**OSPF广播型网络**

比如以太网,令牌环网和FDDI,这样的网络上会选举一个DR和BDR,DR/BDR的发送的OSPF包的目标地址为**224.0.0.5**;而除了DR/BDR以外的OSPF包的目标地址为**224.0.0.6**

在**广播型网络**中,**所有路由器**都以**224.0.0.5**的地址发送hello包,用来维持邻居关系,

**非DR/BDR**路由都以**224.0.0.6**的地址发送lsa更新,而只有**DR/BDR**路由**监听**这个地址,

反过来,**DR**路由使用**224.0.0.5**来发送更新**到非DR路由**

**AllDRouters**(224.0.0.6)

**AllSPFRouters** (224.0.0.5)

### 网络类型

https://blog.csdn.net/qq_44667101/article/details/105733924?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_default&utm_relevant_index=2



### 静态路由

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/static-route.png)

 说明：

　　AR1路由器直连A、B两个网段，C、D网段没有直连，你需要添加到C、D网段的路由。

　　AR2路由器直连B、C两个网段，A、D网段没有直连，你需要添加到A、D网段的路由。

　　AR3路由器直连C、D两个网段，A、B网段没有直连，你需要添加到A、B网段的路由。

要想实现全网络通信，必须添加路由条目，这里只是考虑添加静态路由

　　示列： ip route-static 到达的目标网络号 子网掩码 下一跳的路由器接口地址

　　[R1]ip route-static 172.16.1.0 24 172.16.0.2       到C段的路由条目

　　[R1]ip route-static 192.168.1.0 24 172.16.0.2　　　 到D段的路由条目

　　[R2]ip route-static 192.168.1.0 24 172.16.1.2　　   到A段的路由条目

　　[R2]ip route-static 192.168.0.0 24 172.16.0.1　　   到D段的路由条目

　　[R3]ip route-static 192.168.0.0 24 172.16.1.1      到A段的路由条目

　　[R3]ip route-static 172.16.0.0 24 172.16.1.1　　   到B段的路由条目 

如：上图里面的R1路由条目里到C段的路由条目的另一种写法

　　[R1]ip route-static 172.16.1.0 24 GigabitEthernet 1/1    到C段的路由条目  【GE接口写法】

　　[R1]ip route-static 172.16.1.0 24 Serial 1/1    到C段的路由条目  【Serial接口写法】

### Enabling OSPF on an interface

To enable OSPF on an interface, you can enable OSPF on the network where the interface resides or directly enable OSPF on that interface. If you configure both, the latter takes precedence

| Step                                          | Command                                                      | Remarks                                                      |
| --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1.   Enter system view.                       | **system-view**                                              | N/A                                                          |
| 2.   Enter interface view.                    | **interface** *interface-type interface-number*              | N/A                                                          |
| 3.   Enable an OSPF process on the interface. | **ospf** *process-id* **area** *area-id* [ **exclude-subip** ] | By default, OSPF is disabled on an interface.If the specified OSPF process and area do not exist, the command creates the OSPF process and area. Disabling an OSPF process on an interface does not delete the OSPF process or the area. |

#### 基于接口和进程配置ospf

OSPF两种配置方式，

1、接口配置

```bash
[H3C]interface l2
[H3C-LoopBack2]ip ad
[H3C-LoopBack2]ip address 1.1.1.8 255.255.255.255
# 这里栗子是loopback比较特殊，掩码是32位。
# 换成 ip addr 192.168.1.8 255.255.255.0 就比较直观了
# 直接将接口网段宣告到ospf
[H3C-LoopBack2]ospf 1 area 0.0.0.0
[H3C]display ospf

         OSPF Process 1 with Router ID 1.1.1.8
                 OSPF Protocol Information

 RouterID: 1.1.1.8          Router type:  ABR
 Route tag: 0
 ...

 Area: 0.0.0.0          (MPLS TE  not enabled)
 Authentication type: None    Area flag: Normal
 SPF scheduled count: 1
 ExChange/Loading neighbors: 0

 Interface: 1.1.1.8 (LoopBack2)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled
 Enabled by interface configuration (including secondary IP addresses)

 Area: 0.0.0.2          (MPLS TE  not enabled)
 Authentication type: None    Area flag: Normal
 SPF scheduled count: 1
 ExChange/Loading neighbors: 0

```

2、进程配置

```bash
[H3C]interface LoopBack 8
[H3C-LoopBack8]ipa
[H3C-LoopBack8]ip a
[H3C-LoopBack8]ip address 9.9.9.9 255.255.255.255
[H3C-LoopBack8]quit
# 使用ospf 进程宣告网络
[H3C]ospf 1
[H3C-ospf-1]ar
[H3C-ospf-1]area 0
[H3C-ospf-1-area-0.0.0.0]network 9.9.9.9 255.255.255.255
[H3C-ospf-1-area-0.0.0.0]quit

C-ospf-1]display ospf

         OSPF Process 1 with Router ID 1.1.1.8
                 OSPF Protocol Information

 RouterID: 1.1.1.8          Router type:  ABR
 Route tag: 0
 Multi-VPN-Instance is not enabled
 ...

 Area: 0.0.0.0          (MPLS TE  not enabled)
 Authentication type: None    Area flag: Normal
 SPF scheduled count: 1
 ExChange/Loading neighbors: 0

 Interface: 1.1.1.8 (LoopBack2)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled
 Enabled by interface configuration (including secondary IP addresses)

 Interface: 2.2.2.2 (LoopBack4)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled
 Enabled by interface configuration (including secondary IP addresses)

 Interface: 9.9.9.9 (LoopBack8)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled

```







### 基础名词

#### router-id

By default, no global router ID is configured.

If no global router ID is configured, the highest loopback interface IP address, if any, is used as the router ID. If no loopback interface IP address is available, the highest physical interface IP address is used, regardless of the interface status (up or down)

#### area-id

```bash
[H3C-ospf-3]area?
  area  Specify the OSPF area

[H3C-ospf-3]area ?
  INTEGER<0-4294967295>  OSPF area ID (Integer)
  X.X.X.X                OSPF area ID (IP address)
```

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ospf-package.jpg)



#### arp

arp没有跨网段的说法.ARP工作在一个广播域，负责二层寻址而且是在广播网络（对于非广播网络，是没有ARP，比如ATM网络）。它完成了硬件MAC地址和IP之间的翻译。

R3的37.1.1.1 ping R4的38.1.1.1，要封装二层的时候，发现没有38.1.1.1对应的MAC地址，先把目的IP和本出接口IP与运算，发现不同网段，就去找路由表，发现路由表没有路由，于是直接丢弃！

OSPF在P2P网络类型不检查掩码.

#### `?` and history

有点意思

```bash
[H3C]display interface ?
  >                    Redirect it to a file
  >>                   Redirect it to a file in append mode
  FortyGigE            FortyGigE interface
  GigabitEthernet      GigabitEthernet interface
  InLoopBack           InLoopBack interface
  M-GigabitEthernet    MGE interface
  NULL                 NULL interface
  Register-Tunnel      Register Tunnel interface
  Ten-GigabitEthernet  Ten-GigabitEthernet interface
  brief                Brief information of status and configuration for
                       interface(s)
  range                Display range information
  |                    Matching output
  <cr>

## 显示history
[ar1]display history-command
 sys
 interface vlan 1
 ip address 192.168.1.254 255.255.255.0
 save
 exit
 display  interface brief

```

end、

#### MGE0 port

管理用以太网口不受交换芯片工作状态的影响，**一般用于连接计算机以进行系统的程序加载、调试等工作，也可以连接远端的网管工作站等设备以实现系统的远程管理。**

这是个带外网管接口--

#### LSA

LSA（ Link-State Advertisement ，链路状态广播）是链路状态协议使用的一个分组，它包括有关邻居和链路成本的信息。LSAs 被路由器接收用于维护它们的 RIB（路由表）。

路由器把收集到的 LSA 存储在链路状态数据库中，然后运行 SPF 算法计算出路由表。链路状态数据库和路由表的不同在于：数据库中包含的是完整的链路状态原始数据，而路由表中列出的是到达所有已知目标网络的最短路径的列表。

LSA的新旧比较

(1) 会先比较序列号（sequence number），序列号越大越优

(2) 如果序列号相同，会比较校验值（checksum），越大越优

(3) 如果校验值也相同，会比较LSA Age时间，是否等于MAX-age时间（3600S）

(4) 如果age时间不等于MAX-age时间，会比较他们的差值，如果差值>15分钟（900S），越小越优

(5) 如果age时间不等于MAX-age时间，会比较他们的差值，如果差值<15分钟（900S），说明是同一条LSA，忽略其中一条

**LSDB**，Link State DataBase，链路状态数据库。



#### DR/BDR/Drother

Designated Router/Backup Designated Router/Designated Route other

通过在多路访问网段中选择出一个核心路由器，称为DR，网段中所有的OSPF路由器都和DR互换LSA，这样一来，DR就会拥有所有的LSA，并且将所有的LSA转发给每一台路由器；DR就像是该网段的LSA中转站，所有的路由器都与该中转站互换LSA，如果DR失效后，那么就会造成LSA的丢失与不完整，所以在多路访问网络中除了选举出DR之外，还会选举出一台路由器作为DR的备份，称为BDR，BDR在DR不可用时，代替DR的工作，而既不是DR，也不是BDR的路由器称为Drother，事实上，Drother除了和DR互换LSA之外，同时还会和BDR互换LSA。

在广播网和NBMA网络中，任意两台路由器之间都要交换路由信息。如果网络中有n台路由器，则需要建立n（n-1）/2个邻接关系。这使得任何一台路由器的路由变化都会导致多次传递，浪费了带宽资源。为解决这一问题，OSPF提出了DR的概念，所有路由器只将信息发送给DR，由DR将网络链路状态发送出去。

另外，OSPF提出了BDR的概念。BDR是对DR的一个备份，在选举DR的同时也选举BDR，BDR也和本网段内的所有路由器建立邻接关系并交换路由信息。当DR失效后，BDR会立即成为新的DR。

OSPF网络中，既不是DR也不是BDR的路由器为DR Other。DR Other仅与DR和BDR建立邻接关系，DR Other之间不交换任何路由信息。这样就减少了广播网和NBMA网络上各路由器之间邻接关系的数量，同时减少网络流量，节约了带宽资源。



#### OSPF网络类型

OSPF根据链路层协议类型将网络四种类型:broadcast、nbma、p2mp和p2p

配置命令: `ospf network-type <type_name>`

##### 广播类型

广播（Broadcast）类型：当链路层协议是Ethernet、FDDI时，缺省情况下，OSPF认为网络类型是Broadcast。

在该类型的网络中，通常以组播形式（OSPF路由器的预留IP组播地址是224.0.0.5；OSPF DR的预留IP组播地址是224.0.0.6）发送Hello报文、LSU报文和LSAck报文；以单播形式发送DD报文和LSR报文。

##### NBMA

NBMA（Non-Broadcast Multi-Access，非广播多路访问）类型：当链路层协议是帧中继、ATM或X.25时，缺省情况下，OSPF认为网络类型是NBMA。

在该类型的网络中，以单播形式发送协议报文。

##### P2MP

P2MP（Point-to-MultiPoint，点到多点）类型：没有一种链路层协议会被缺省的认为是P2MP类型。P2MP必须是由其他的网络类型强制更改的，常用做法是将NBMA网络改为P2MP网络。

在该类型的网络中，缺省情况下，以组播形式（224.0.0.5）发送协议报文。可以根据用户需要，以单播形式发送协议报文。

NBMA与P2MP网络之间的区别如下：

* NBMA网络是全连通的；P2MP网络并不需要一定是全连通的。
* NBMA网络中需要选举DR与BDR；P2MP网络中没有DR与BDR。
* NBMA网络采用单播发送报文，需要手工配置邻居；P2MP网络采用组播方式发送报文，通过配置也可以采用单播发送报文

##### P2P：不同网段也互通

P2P（Point-to-Point，点到点）类型：当链路层协议是PPP、HDLC时，缺省情况下，OSPF认为网络类型是P2P。

在该类型的网络中，以组播形式（224.0.0.5）发送协议报文。

下面是什么意思：

OSPF在P2P网络类型不检查掩码、

P2P 网络中掩码之所以可以不一致是因为 P2P 中有 1LSA 的 stub 类型来描述每一个网络的掩码信息，并且在 PPP 链路中 NCP 阶段，两台路由器会互推自己的 IP地址，并且以 32 位主机路由的方式加入自己的路由表，所以 P2P 网络中建立邻居不需要掩码一致。



##### 实际应用

用户可以根据需要更改接口的网络类型，例如：

* 当NBMA网络通过配置地址映射成为全连通网络时（即网络中任意两台路由器之间都存在一条虚电路而直接可达），可以将网络类型更改为广播，不需要手工配置邻居，简化配置。
* 当广播网络中有部分路由器不支持组播时，可以将网络类型更改为NBMA。
* NBMA网络要求必须是全连通的，即网络中任意两台路由器之间都必须有一条虚电路直接可达；如果NBMA网络不是全连通而是部分连通时，可以将网络类型更改为P2MP，达到简化配置、节省网络开销的目的。
* 如果路由器在NBMA网络中只有一个对端，也可将接口类型配置为P2P，节省网络开销。

**如果接口配置为广播、NBMA或者P2MP网络类型，只有双方接口在同一网段才能建立邻居关系。**

#### 路由协议类型

```bash
$ display ip routing-table
# 发现该路由的路由协议类型：

·     O_INTRA：OSPF intra area

·     O_INTER：OSPF inter area

·     O_ASE1：OSPF external type 1

·     O_ASE2：OSPF external type 2

·     O_NSSA1：OSPF NSSA external type 1

·     O_NSSA2：OSPF NSSA external type 2

·     O_SUM：OSPF summary

·     IS_L1：IS-IS level-1

·     IS_L2：IS-IS level-2

·     IS_SUM：IS-IS summary
```

end

### 基础命令

#### 端口缩写

MGE为设备管理接口，XGE为万兆接口，FGE千兆接口



#### 交换机二层和三层

```bash
## 该交换机默认是三层网络
[H3C-GigabitEthernet0/0]port link-mode ?
  bridge  Switch to layer2 ethernet
  route   Switch to layer3 ethernet

## 切换成二层网络模式
## 命令都变了
[H3C-GigabitEthernet0/0]port link-mode bridge
[H3C-GigabitEthernet0/0]port ?
  access     Set access port attributes
  hybrid     Set hybrid port attributes
  link-mode  Switch the specified interface to layer2 or layer3 ethernet
  link-type  Set the link type
  trunk      Set trunk port attributes

[H3C-GigabitEthernet0/0]port access ?
  vlan  Assign the port to a VLAN
```

end

#### 查看当前配置

```bash
$ interface Vlanif 2
$ ip address 10.10.100.1 255.255.255.0
$ display current-configuration
```

end

#### 静态路由

命令解析学习，一步步的`?`真的好用呢

```bash
[Huawei]ip route-static ?
  X.X.X.X             Destination IP address
  default-preference  Preference-value for IPv4 static-routes
  selection-rule      Selection rule
  vpn-instance        VPN-Instance route information

[Huawei]ip route-static 0.0.0.0?
  X.X.X.X                                 
[Huawei]ip route-static 0.0.0.0 ?
  INTEGER<0-32>  Length of IP address mask
  X.X.X.X        IP address mask

[Huawei]ip route-static 0.0.0.0 0.0.0.0 ?
  GigabitEthernet  GigabitEthernet interface
  MEth             MEth interface
  NULL             NULL interface
  Vlanif           Vlan interface
  X.X.X.X          Gateway address
  vpn-instance     Destination VPN-Instance for Gateway address

[Huawei]ip route-static 0.0.0.0 0.0.0.0 10.10.10.1
```

实践例子--

```bash
## 添加静态路由
[Huawei]ip route-static 0.0.0.0 0.0.0.0 192.168.1.1
[Huawei]display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 5        Routes : 5        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

        0.0.0.0/0   Static  60   0          RD   192.168.1.1     Vlanif100
      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.1.0/24  Direct  0    0           D   192.168.1.2     Vlanif100
    192.168.1.2/32  Direct  0    0           D   127.0.0.1       Vlanif100
## 删除静态路由
[Huawei]undo ip route-static 0.0.0.0 0.0.0.0 192.168.1.1
[Huawei]display ip routing-table
Route Flags: R - relay, D - download to fib
Routing Tables: Public
         Destinations : 4        Routes : 4        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.1.0/24  Direct  0    0           D   192.168.1.2     Vlanif100
    192.168.1.2/32  Direct  0    0           D   127.0.0.1       Vlanif100

```

end

#### 删除IP地址

How do I remove a VLANIF IP from my Huawei?

```bash
# 查看 ip 地址
$ display ip interface brief

# physical port
$ undo -bind static ip-address 192.168.1.2 interface GigabitEthernet 0/0/1
Info: The operation may take a few seconds. Please wait for a moment.done.
0 item(s) deleted.

# svi
[H3C-Vlan-interface20]undo ipaddress //删除IP地址

[Huawei-Vlanif100]undo ip address 192.168.1.2 255.255.255.0
[Huawei-Vlanif100]
Apr  8 2022 15:47:28-08:00 Huawei %%01IFNET/4/LINK_STATE(l)[4]:The line protocol
 IP on the interface Vlanif100 has entered the DOWN state.
[Huawei-Vlanif100]display current-configuration
```

end

#### Serial接口

Ser口是serial接口的意思，也叫高速异步串口，主要是连接广域网的V.35线缆用的，即路由器和路由器连接时候用的，可以用命令设置带宽，一般也就在10M、8M左右，不过现在的路由器都没有这些口了，都用光口代替了。

#### LLDP

LLDP是一种邻近发现协议。它为以太网网络设备，如交换机、路由器和无线局域网接入点定义了一种标准的方法，使其可以向网络中其他节点公告自身的存在，并保存各个邻近设备的发现信息。例如设备配置和设备识别等详细信息都可以用该协议进行公告。

具体来说，LLDP定义了一个通用公告信息集、一个传输公告的协议和一种用来存储所收到的公告信息的方法。要公告自身信息的设备可以将多条公告信息放在一个局域网数据包内传输，传输的形式为类型长度值（TLV）域。

```bash
 lldp management-address arp-learning
 lldp tlv-enable basic-tlv management-address-tlv interface LoopBack0

## ?是查看啊
[H3C]lldp ?
  compliance     Enable compliance with another link layer discovery  protocol
  fast-count     The fast-start times of transmitting frames
  global         Specify global
  hold-multiplier   Hold multiplicator for TTL
  ignore-pvid-inconsistency  Ignore PVID inconsistency
  max-credit        Specify LLDP maximum transmit credit
  mode              Specify LLDP bridge mode
  timer             Timer of LLDP

```

end

#### 创建ospf

```bash
# 启动OSPF进程，进入OSPF视图
ospf [ process-id | router-id router-id | vpn-instance vpn-instance-name ... ]

# 创建opsf进程1，并设置router-id为1.1.1.1
$ ospf 1 router-id 1.1.1.1
```

* **process-id** 为进程号，缺省值为1。

  路由器支持OSPF多进程，可以根据业务类型划分不同的进程。进程号是本地概念，不影响与其它路由器之间的报文交换。因此，不同的路由器之间，即使进程号不同也可以进行报文交换.。

* **router-id** ： router-id为路由器的ID号。

  缺省情况下，路由器系统会从当前接口的IP地址中自动选取一个最大值作为Router ID。手动配置Router ID时，必须保证自治系统中任意两台Router ID都不相同。通常的做法是将Router ID配置为与该设备某个接口的IP地址一致。

* **vpn-instance** : vpn-instance-name 表示VPN实例。

#### 创建ospf interface

```bash
# 将一个interface 
[H3C]interface LoopBack 4
[H3C-LoopBack4]ip add
[H3C-LoopBack4]ip address 2.2.2.2 255.255.255.255
[H3C-LoopBack4]ospf 1 area 0.0.0.0

[H3C-LoopBack4]display ospf

         OSPF Process 1 with Router ID 1.1.1.8
                 OSPF Protocol Information

 RouterID: 1.1.1.8          Router type:  ABR
 Route tag: 0
 Multi-VPN-Instance is not enabled
 Ext-community type: Domain ID 0x5, Route Type 0x306, Router ID 0x107
 Domain ID: 0.0.0.0
 Opaque capable
 ISPF is enabled
 SPF-schedule-interval: 5 50 200
 LSA generation interval: 5 50 200
 LSA arrival interval: 1000 500 500
 Transmit pacing: Interval: 20 Count: 3
 Default ASE parameters: Metric: 1 Tag: 1 Type: 2
 Route preference: 10
 ASE route preference: 150
 SPF calculation count: 2
 RFC 1583 compatible
 Graceful restart interval: 120
 SNMP trap rate limit interval: 10  Count: 7
 Area count: 4   NSSA area count: 0
 ExChange/Loading neighbors: 0

 Area: 0.0.0.0          (MPLS TE  not enabled)
 Authentication type: None    Area flag: Normal
 SPF scheduled count: 1
 ExChange/Loading neighbors: 0

 Interface: 1.1.1.8 (LoopBack2)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled
 Enabled by interface configuration (including secondary IP addresses)
 # 对应
 Interface: 2.2.2.2 (LoopBack4)
 Cost: 0       State: Loopback    Type: PTP       MTU: 1536
 Timers: Hello 10, Dead 40, Poll 40, Retransmit 5, Transmit Delay 1
 FRR backup: Enabled
 Enabled by interface configuration (including secondary IP addresses)
 ....

```



#### 有向图？？？

LSDB有向图

Link State DataBase，链路状态数据库，OSPF干脆就给区域内每台路由器都搞一张地图，这样就不会上当受骗了，这个地图就是LSDB，这样就使得OSPF可以保证区域内无环，区域间无环，通过一些规则来限制，这样区域内外都能保证无环。

```bash
[ar1]display  ospf lsdb router

         OSPF Process 1 with Router ID 1.1.1.1
                 Link State Database

                         Area: 0.0.0.0

    Type      : Router
    LS ID     : 192.168.2.1
    Adv Rtr   : 192.168.2.1
    LS age    : 208
    Len       : 60
    Options   : O E
    Seq#      : 8000000c
    Checksum  : 0xed3d
    Link Count: 3
       Link ID: 1.1.1.1
       Data   : 192.168.2.1
       Link Type: P-2-P
       Metric : 1
       Link ID: 2.2.2.2
       Data   : 255.255.255.255
       Link Type: StubNet
       Metric : 0
       Link ID: 192.168.2.0
       Data   : 255.255.255.0
       Link Type: StubNet
       Metric : 1

    Type      : Router
    LS ID     : 1.1.1.1
    Adv Rtr   : 1.1.1.1
    LS age    : 207
    Len       : 48
    Options   : O E
    Seq#      : 8000000a
    Checksum  : 0xac03
    Link Count: 2
       Link ID: 192.168.2.1
       Data   : 192.168.1.1
       Link Type: P-2-P
       Metric : 1
       Link ID: 192.168.1.0
       Data   : 255.255.255.0
       Link Type: 
```

end

### 基础组网配置

网络中有三台交换机。现在需要实现三台交换机之间能够互通，且以后能依据SwitchA和SwitchB为主要的业务设备来继续扩展整个网络。

![ospf-base](D:\temp_files\download\ospf-base.png)

采用如下的思路配置OSPF基本功能：

1. 在各交换机的VLANIF接口上配置IP地址并配置各接口所属VLAN，实现网段内的互通。
2. 在各交换机上配置OSPF基本功能，并且以SwitchA为ABR将OSPF网络划分为Area0和Area1两个区域，实现后续以SwitchA和SwitchB所在区域为骨干区域来扩展整个OSPF网络

https://support.huawei.com/enterprise/zh/doc/EDOC1000141427/d55d1d8c

```bash
# //创建进程号为1，Router ID为10.1.1.1的OSPF进程
[SwitchA] ospf 1 router-id 10.1.1.1   
# //创建area 0区域并进入area 0视图
[SwitchA-ospf-1] area 0   
# //配置area 0所包含的网段
[SwitchA-ospf-1-area-0.0.0.0] network 192.168.0.0 0.0.0.255   
[SwitchA-ospf-1-area-0.0.0.0] quit

## switch B
[SwitchB] ospf 1 router-id 10.2.2.2
[SwitchB-ospf-1] area 0
[SwitchB-ospf-1-area-0.0.0.0] network 192.168.0.0 0.0.0.255
[SwitchB-ospf-1-area-0.0.0.0] return

## switch C
[SwitchC] ospf 1 router-id 10.3.3.3
[SwitchC-ospf-1] area 1
[SwitchC-ospf-1-area-0.0.0.1] network 192.168.1.0 0.0.0.255
[SwitchC-ospf-1-area-0.0.0.1] return
```

配置总览：

Switch A

```bash
#
sysname SwitchA
#
vlan batch 10 20
#
interface Vlanif10
 ip address 192.168.0.1 255.255.255.0
#
interface Vlanif20
 ip address 192.168.1.1 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 20
#
ospf 1 router-id 10.1.1.1
 area 0.0.0.0
  network 192.168.0.0 0.0.0.255
 area 0.0.0.1
  network 192.168.1.0 0.0.0.255
#
return
```

Switch B

```bash
#
sysname SwitchB
#
vlan batch 10
#
interface Vlanif10
 ip address 192.168.0.2 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10
#
ospf 1 router-id 10.2.2.2
 area 0.0.0.0
  network 192.168.0.0 0.0.0.255
#
return
```

Switch C

```bash
#
sysname SwitchC
#
vlan batch 20
#
interface Vlanif20
 ip address 192.168.1.2 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 20
#
ospf 1 router-id 10.3.3.3
 area 0.0.0.1
  network 192.168.1.0 0.0.0.255
#
return
```

end

### 交换机和路由器互联：basic

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/switch-router.png)

#### 交换机配置svi

```bash
[Switch] vlan batch 100
[Switch] interface gigabitethernet 0/0/1
[Switch-GigabitEthernet0/0/1] port link-type access
[Switch-GigabitEthernet0/0/1] port default vlan 100   
[Switch-GigabitEthernet0/0/1] quit
[Switch] interface vlanif 100
[Switch-Vlanif100] ip address 192.168.100.2 24
[Switch-Vlanif100] quit
```

end

#### 路由器接口

```bash
<Huawei> system-view
[Huawei] sysname Router
[Router] interface gigabitethernet 0/0/1
## 配置的IP地址192.168.100.1为交换机缺省路由的下一跳IP地址
[Router-GigabitEthernet0/0/1] ip address 192.168.100.1 255.255.255.0   
[Router-GigabitEthernet0/0/1] quit
```

end

#### 缺省和回程路由

```bash
## 交换机缺省路由
##  缺省交换机路由的下一跳是路由器接口的IP地址192.168.100.1
##　vlanIF IP is 192.168.100.2
[Switch] ip route-static 0.0.0.0 0.0.0.0 192.168.100.1 
## 路由器缺省和回程路由
## 配置路由器回程路由的下一跳就指向交换机上行接口的IP地址192.168.100.2
[Router] ip route-static 192.168.0.0 255.255.0.0 192.168.100.2  
## 配置路由器静态缺省路由的下一跳指向公网提供的IP地址200.0.0.1，路由器一端连着公网负责提供对外访问
[Router] ip route-static 0.0.0.0 0.0.0.0 200.0.0.1   

```

end







### OSPF排查思路

https://support.huawei.com/enterprise/en/knowledge/EKB1000116686



### 引用

1. https://ccie.lol/knowledge-base/ospf-ospfv3-lsa/
2. http://www.h3c.com/cn/d_201312/810777_30005_0.htm#_Toc373336023
3. https://blog.csdn.net/qq_38265137/article/details/80390740
4. https://blog.51cto.com/u_3258715/1735713
5. https://blog.51cto.com/u_12347226/2435104
6. https://blog.csdn.net/Bert_Wang/article/details/106869834