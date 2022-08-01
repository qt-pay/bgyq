## VPN基础-draft

VPN的英文全称是“Virtual Private Network”，即“虚拟专用网络”。VPN是为了替代物理专用网络而出现，物理专用网络是通过架设或者租用物理线路来实现跨地域的互联。

VPN = 隧道 + 安全

VPN可以减少企业在公网暴漏端口的风险呢

### VPN技术的定义：

VPN: virtual private network

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-concept.png)

任何符合如下两个条件的网络我们都可以泛泛地称它为VPN网络：

1、使用共享的公共环境实现各个私有网络的连接；

2、不同的私有网络间（除非有特殊要求，如两个公司间的互访要求）是相互不可见的。

VPN具有以下两个基本特征：

- 专用（Private）：对于VPN用户，使用VPN与使用传统专网没有区别。VPN与底层承载网络之间保持资源独立，即VPN资源不被网络中非该VPN的用户所使用；且VPN能够提供足够的安全保证，确保VPN内部信息不受外部侵扰。

  **管理员可以自定义加密方式和账密密码**

- 虚拟（Virtual）：VPN用户内部的通信是通过公共网络进行的，而这个公共网络同时也可以被其他非VPN用户使用，VPN用户获得的只是一个逻辑意义上的专网。这个公共网络称为VPN骨干网（VPN Backbone）。

#### 核心技术：隧道+安全

公司内部对外访问肯定不能通过端口映射的方式来实现，这样一旦映射N多个端口会造成以下两个问题：

* 管理不便
* 增加被黑客扫描攻击风险

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VPN-core-tec.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-core-tec-2.jpg)

对于不同接入VPN server方式的client可以配置不同的地址池

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-core-auth.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-protocol.jpg)

* PPPoE: 以太网上的点对点协议，是将点对点协议（PPP）封装在以太网（Ethernet）框架中的一种网络隧道协议。

  实现出传统以太网不能提供的身份验证、加密以及压缩等功能，也可用于缆线调制解调器和数字用户线路等以以太网协议向用户提供接入服务的协议体系。

* PPTP:点对点隧道协议（英语：Point to Point Tunneling Protocol，缩写为PPTP）是实现虚拟专用网（VPN）的方式之一。PPTP是为两个对等结构之间的 IP 流量的传输提供一种封装协议

  PPTP没有内容加密。

* L2TP+ipSec ：Layer Two Tunneling Protocol

### MPLS VPN：运营商VPN

#### 关键组件

VPN网络的关键组件如下:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-basic.jpg)

* CE（Custom Edge）：直接与服务提供商相连的用户设备。
* PE（Provider Edge Router）：指骨干网上的边缘设备（如路由器、ATM交换机、帧中继交换机等），与CE相连,主要负责VPN业务的接入。
* P （Provider Router）：指骨干网上的核心路由器，主要完成路由和快速转发功能。P设备根据网络结构及规模可有可无。
* SP：Service Provider，服务提供商

任何一个VPN网络都是由这几个组件全部或部分组成的。

#### MPLS

MPLS全称Multi-Protocol Label Switching，目前来讲运营商网络中MPLS VPN被广泛的应用。所以我们现在通常讲的L3VPN什么的一般来讲都是基于MPLS的L3VPN。

一般VPN技术分为传统的VPN技术与虚拟的VPN技术；传统的VPN技术一般可以理解为基于静态的电路域的链接方式；虚拟的VPN技术，一般是指MPLS VPN，是一种基于MPLS技术的虚拟解决方案，可以实现底层标签的自动分配，在业务的提供上比传统的VPN技术更廉价，更快

### VPN实现分类

#### 建设单位

按建设单位分类分为运营商VPN和自建VPN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn分类-建设单位-1.jpg)

##### 运营商VPN

运营商网络设备会学习(通过不同的路由协议)客户内部网络(CE)路由，然后在PE设备上传输。需要先学习什么是MPLS才能掌握MPLS VPN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-分类-建设单位-运营商.jpg)

##### 自建VPN

购买厂商的安全设备和VPN，厂商回收取一定的授权费用。比如，深信服的SSL VPN按用户数量授权收费。

自建：甲方自建或乙方(厂商)搭建

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn分类-建设单位-自建.jpg)

###### PPTP和L2PT

PPTP和L2PT配置比较负载，通常使用专业的VPN设备提供的图形界面来配置，而不在专业网络设备上配置

#### 组网方式：常见

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn组网方式分类.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VPN组网方式分类-点对点-3.jpg)

##### Remote-Access VPN

client IP不固定

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn组网方式分类-远程访问.jpg)

下图更清晰，可以有N个不定的client

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn组网方式分类-远程访问-2.jpg)

##### Site-to-Site VPN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn组网方式分类-点对点.jpg)

也被叫做LAN-to-LAN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn组网方式分类-点对点-2.jpg)

#### 实现协议:imp:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/VPN组网方式协议不同.jpg)

##### L7 VPN:ssl vpn

典型 SSL VPN：EasyConnect.... 因为server gateway地址是https协议，so L7 VPN？

通常使用`https+网关地址`拨入VPN

L7 VPN只能针对应用层数据、比如： chrome访问页面、远程到云桌面（H3C WorkSpace）

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ssl-vpn.jpg)

##### L3VPN: gre/ipsec

* GRE：不支持加密和身份认证，通常结合IPSec
* IPSec：六边形战士，可单独使用

##### L2VPN

* PPTP: 快淘汰，早期微软开发，windows系统都内置了。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/微软-pptp-vpn.png)

* L2F：快淘汰，**L2F**，或**第二层转发协议**（Layer 2 Forwarding Protocol），是由思科公司开发的，建立在互联网上的虚拟专用网络连接的隧道协议。L2F协议本身并不提供加密或保密；它依赖于协议被传输以提供保密。L2F是专为隧道点对点协议（PPP）通信。

* L2TP: 主流L2VPN

  https://cshihong.github.io/2019/08/21/L2TP-VPN%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/

  

### VPN常见形式

#### VPN服务器

* 服务器：自建使用
* 路由器：
* 防火墙
* VPN设备：厂商设备

#### VPN客户端

* 系统自带拨号工具
* 第三方拨号工具：各类client
* 浏览器

#### 拨入

接入VPN网络叫拨入...?

### VPN and Tunnel

A **virtual private network** (**VPN**) extends a private network across a public network, and enables users to send and receive data across shared or public networks as if their computing devices were directly connected to the private network. 

>  **private network** is a network that uses private IP address space.

VPN is Virtual Private Network. This is not a protocol. This is a concept. There are many different types of VPN technologies, but they have one thing in common: **they provide traffic separation.** The simplest example of VPN is a VLAN. Traffic inside the VLAN is segregated from traffic in other VLANs and to communicate with external networks (not belonging to this VLAN) it needs a layer 3 gateway. There are VPNs that provide data confidentiality via encryption, like IPSEC VPN. And there are VPNs that only provide traffic segregation, like VLAN or MPLS VPN (also called IP VPN). The main feature of any VPN is to make sure that all segments of the network stay under your administrative control. 

Tunneling is one of the VPN techniques that can be used with or without encryption.

 VPN是一个概念，任何一个可以是公网进行数据交流的私有网络都可以叫vpn. vpn 共同点： 流量分离

#### Tunnel

In computer networks, a **tunneling protocol** is a communications protocol that allows for the movement of data from one network to another.

典型应用场景：

ServerA 和 ServerB 可以想象成RouteA 和 RouteB.

```bash
                          |                                           |
                          |       1.1.1.1               2.2.2.2       |
             Piviate      |     +---------+  Public   +---------+     | Private
                          +-----| ServerA +-----------+ ServerB +-----+
             Networl      |     +---------+  Network  +---------+     | Network
                          |                                           |
192.168.0.0/24            |                                           | 192.168.1.0/24 
```

如果只通过，NAT手段，最多实现`192.168.0.0/24 ---> 2.2.2.2`,无法实现两个私有网段的直接通信即：

`192.168.0.0/24 <--> 192.168.1.0/24`.

假如需要将对外网ip202.103.96.112网口eth1的访问，映射到对内网ip192.168.0.112的访问；

`iptables -t nat -A PREROUTING -i eth1 -p tcp -d 202.103.96.112 --dport 80 -j DNAT --to-destination 192.168.0.112:80`

但通过ip tunnel建立ipip隧道，再通过iptables进行nat，便可以实现。

**Step 1. 建立ip隧道**
ServerA配置iptunnel,并给tunnel接口配置上ip

```bash
$ ip tunnel add a2b mode ipip remote 2.2.2.2 local 1.1.1.1
$ ifconfig a2b 192.168.2.1 netmask 255.255.255.0
```

ServerB配置iptunnel,并给tunnel接口配置上ip

```bash
$ ip tunnel add a2b mode ipip remote 1.1.1.1 local 2.2.2.2
$ ifconfig a2b 192.168.2.2 netmask 255.255.255.0
```

隧道配置完成后，请在ServerA上192.168.2.2，看是否可以ping通，ping通则继续，ping不通需要再看一下上面的命令执行是否有报错
**Step 2. 添加路由和nat**
ServerA上，添加到192.168.1.0/24的路由

```bash
$ /sbin/route add -net 192.168.1.0/24 gw 192.168.2.2
```

ServerB上，添加iptables nat，将ServerA过了访问192.168.1.0/24段的包进行NAT，并开启ip foward功能

```bash
$ iptables -t nat -A POSTROUTING -s 192.168.2.1 -d 192.168.1.0/24 -j MASQUERADE
$ sysctl -w net.ipv4.ip_forward=1
$ sed -i '/net.ipv4.ip_forward/ s/0/1/'  /etc/sysctl.conf
```

至此，完成了两端的配置，ServerA可以直接访问ServerB 所接的私网了。

#### **What's the difference between tunneling and VPN?**

Tunnelling is actually a protocol that allows secure data transfer from one network to another. It uses a process called ‘encapsulation’ through which the private network communications are sent to the public networks. On the other hand, **VPN is based on the idea of tunnelling. It involves establishing and maintaining a logical network connection.** On this connection, packets constructed in a specific VPN protocol format are encapsulated within some other base or carrier protocol, then transmitted between VPN client and server, and finally de-encapsulated on the receiving side. Hope this helps.

A VPN is a secure, encrypted connection over a publicly shared network. **Tunneling is the process by which VPN packets reach their intended destination, which is typically a private network**. Many VPNs use the IPsec protocol suite.

有些隧道协议可能是不安全的比如,IPIP Tunnel，但是VPN都是安全加秘密的传输协议。



### VPN运营模式分类

VPN的分类，VPN的分类有许多种方式。我们一般常接触的分类方式是按照运营模式进行分类:
  1）、CPE-Based VPN: 设备放置在用户侧(Custom Premise Equipment)

由用户管理或委托运营商进行管理；
   2）、Network-Based VPN: 设备放置在运营商网络侧

用户设备不需要感知VPN由运营商管理，又称为Provider Provide VPN。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn运营模式分类.jpg)



### VPN技术的分类

VPN的概念早在10多年前便已经提出，至今经历了较长的演进过程。

随着新技术（如加密技术、隧道技术、MPLS技术）的不断涌现，及新的客户需求的出现，目前VPN的概念变得越来越复杂。所以引入了VPN的分类，以区分各种不同的VPN技术。

VPN技术主要从四个方面进行分类：

1、VPN要解决的业务问题；

2、服务提供商在哪一层与客户交换拓扑信息；

3、在服务提供商中用于实现VPN服务的第二层或第三层技术；

4、网络的拓扑结构。

每一个分类与其它的分类间都不是独立的，都是有相互关联的，它们共同决定了VPN网络的架构。

#### 专线网络

VPN的概念最早是从专线引发的。专线网络具有以下特点：

1、安全性高，线路为客户专用，不同用户间是物理隔离的；

2、价格昂贵；

3、带宽浪费严重。

正因为专线网络具有的一些固有缺陷，随着统计复用技术的出现，一些新的共享带宽技术逐渐替代了专用线路，并可以提供与专线相同的服务。如帧中继、X.25技术等，通过VC在公共的交换设备上提供虚链路，实现用户的私有通信。由于是共享带宽，所以价格比专线便宜很多。这些技术构成了早期的VPN网络。

VPN是为了替代物理专用网络而出现，物理专用网络是通过架设或者租用物理线路来实现跨地域的互联。

#### Overlay VPN

Overlay VPN的主要特点是客户的路由协议总是在客户设备之间交换，而服务提供商对客户网络的内部结构一无所知。

典型的Overlay VPN技术有：

第二层隧道技术如X.25、帧中继(Frame Relay)、ATM；

> 需要ATM交换机和帧中继交换机支持

第三层隧道技术如IP-over-IP隧道技术等。

其中IP-over-IP隧道技术通过专用IP主干或INTERNET来实现覆盖VPN网络，最常用的技术是GRE VPN和IP See VPN等。

> OpenStack中的VPNaaS就是基于IPSec实现的。

Overlay VPN的主要缺陷是：

* 连接性比较复杂时管理开销非常大；
* 用户需要在自己的设备上管理各种路由，例如到Beta的路由，从哪个VPN tunnel出，去Gamma的路由，又该从哪个VPN tunnel出。当用户规模增大的时候，这尤其不方便。
* 要正确提供VC的容量，必须了解站点间的流量情况，比较难统计。

##### Overlay VPN模型

VPN最开始的实现，就是服务提供商（Service provider）向用户提供模拟的线路，用户基于模拟线路来实现跨地域互连，这里的模拟线路也叫做VC（Virtual circuit）。这种VPN的实现方式，就是overlay 模型。VC的提供者，也就是服务提供商不参与用户网络的组建。用户需要在自己的设备CPE（Customer Premises Edge）上，建立跨地域的连接。服务提供商的角色如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay-vpn-arc.png)

从用户的角度来说，逻辑上，用户各个地域之间的设备，是按照下图方式连接的。用户的CPE之间路由协议数据通常是在用户设备之间直接交换。例如，Alpha要访问Beta，那用户需要自己在Alpha网络里面配置好目的地是Beta的路由（或者通过IGP，BGP配置），再通过VC线路转发。

#### Peer-to-Peer VPN

Peer-to-Peer VPN的主要特点是服务提供商的PE设备直接参与CE设备的路由交换。该VPN模型的实现依据是：如果去往某一特定网络的路由未被安装在路由器的转发表中，在那台路由器上，该网络不可达。实施Peer-to-Peer VPN的前提是所有CE端的地址是全局唯一的！

MPLS L3 VPN是一种peer-to-peer模式的实现，但不是唯一的实现。实际上，现在的服务提供商网络里面，存活着其他的peer-to-peer模型的VPN网络，这里就不展开了。相比其他的peer-to-peer 模式VPN，MPLS L3 VPN的优势在于控制简单，成本低，支持IP地址重叠等等其他。总之就是那么好，所以现在MPLS L3 VPN已经成为peer-to-peer 模式VPN的主流实现。

典型的Peer-to-Peer VPN技术有：

1、专用PE接入技术

2、共享PE接入技术

##### 共享PE p2p

其中，**共享PE Peer-to-Peer VPN**（如下图所示）的主要特点是：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/p2p-vpn-share.jpg)

1、所有VPN用户的CE都连到同一台PE上，PE与不同的CE之间运行不同的路由协议（或者是相同路由协议的不同进程，比如OSPF）。

2、由始发PE将VPN路由发布到公网上，在接收端的PE上将这些路由过滤后再发给相应的CE设备。

3、共享路由接入方式为避免不同VPN用户间的路由外漏，需要配置大量的ACL进行过滤，对于设备的性能及管理开销都会有很大的负担。

##### 专用PE p2p

**专用PE Peer-to-Peer VPN**（如下图所示）的主要特点是：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/p2p-vpn-private.jpg)

为每一个VPN单独准备一台PE路由器，PE和CE之间可以运行任意的路由协议，与其它VPN无关。PE与P之间运行BGP或其他路由协议，并使用路由属性进行过滤。

优点：无需配置任何ACL,大大减轻了管理员的配置工作量。

缺点：每一个VPN用户都要新增一台专用的PE，代价过于昂贵。

##### P2P VPN 模型

peer-to-peer被提出。peer-to-peer model下，服务商的edge设备是一个路由器（PE），直接与用户的edge设备（CE）交换路由。也就是说，服务提供商不仅参与数据的传递，同时也参与路由协议数据的传递。 peer-to-peer模式下，服务商的角色如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/P2P-vpn-arc.jpg)



这与最开始的租用线路已经很不一样了。同时服务提供商也参与用户网络的构建，两者分工变得模糊。但是peer-to-peer解决了overlay模型的实际问题。

1. 用户在Delta新开公司，用户只需要与Delta对应的PE连接和配置即可。服务提供商再对相应的PE配置做修改，即可完成Delta与其他所有分公司的连接。
2. 用户CE只与PE连接，用户可以把路由都指向相应的PE。同时因为这个原因，对用于CE的要求降低。
3. 服务提供商可以提供最优的路由路径来传递用户的私有网络服务。

没有悬念，MPLS L3 VPN是一种peer-to-peer模式的实现，但不是唯一的实现



##### n2n

[n2n](https://www.ntop.org/products/n2n/#)是一个开源的`2`层`P2P`架构`VPN`，有如下特点：

1. 基于`P2P`协议的加密`2`层专用网络
2. 边缘节点(`edge node`)的加密是使用带有用户自定义密码的开放协议
3. 每个`n2n`用户可以同时加入不同的网络（或称为社区）
4. `n2n`能够以反向流量方向（即从外部到内部）跨越`NAT`和防火墙。防火墙不再是`IP`级别直接通信的障碍。
5. `n2n`网络并不意味着是独立的：可以通过`n2n`和非`n2n`网络连接。

### 其他分类方式



##### L2 vs L3

L3VPN与L2VPN的本质区别就是运营商边界设备PE是否参与客户的路由；通俗来讲就是L3VPN对于用户来讲像一个大的路由器，需要L3VPN网络提供路由，而L2VPN对于用户则像一个交换机。



##### L2 VPN

With L2 VPN, you can stretch multiple logical networks (both VLAN and VXLAN) across geographical sites. In addition, you can configure multiple sites on an L2 VPN server.

###### QinQ

QinQ是 802.1Q in 802.1Q 的简称，是基于 IEEE 802.1Q 技术的一种比较简单的二层 VPN 协议。

通过将一层 VLAN Tag 封装到私网报文上，使其携带两层 VLAN Tag 穿越运营商的骨干网络(又称公网)，从而使运营商能够利用一个 VLAN 为包含多个 VLAN 的用户网络提供服务。

QinQ报文在运营商网络中传输时带有双层VLAN Tag：

- 内层 VLAN Tag：为用户的私网 VLAN Tag，Customer VLAN Tag (简称 CVLAN)。设备依靠该 Tag 在私网中传送报文。
- 外层 VLAN Tag：为运营商分配给用户的公网 VLAN Tag， Service VLAN Tag(简称 SVLAN)。设备依靠该 Tag 在公网中传送 QinQ 报文。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/QinQ.jpeg)

在公网的传输过程中，设备只根据外层 VLAN Tag 转发报文，而内层 VLAN Tag 将被当作报文的数据部分进行传输。

##### L3 VPN

L3VPN为切入点，对VPN进行一个由点到面的介绍。

L3VPN中部署最为广泛的就是MPLS BGP VPN了，MPLS提供公网隧道的转发，BGP提供私网路由的扩散。下图为MPLS BGP VPN的网络结构。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142611g1gx7lglbxzwksnp.png)

这里VPN的实际目的就很明确，两侧VPNA的网络用户需要互联互通，而同侧的VPNA和VPNB网络需要相互隔离。简单介绍一下各个网元概念，CE指的是用户边缘设备，通常是用户侧的交换机、路由器或者是台主机；PE指运营商边缘设备，主要负责与CE通过路由协议进行对接，将CE侧的路由存入自己的VPN路由表中，并且将用户数据传入网络侧的隧道中去。P指的是运营商核心设备，主要负责公网的隧道转发。从各个设备的主要功能，就可以基本了解到这个L3VPN网络的基本工作原理了，CE与PE通过路由协议直接连接，PE设备存储CE的VPN与路由信息，入口PE到出口PE通过P设备则通过MPLS进行公网的转发传输，通过BGP进行路由的扩散；报文到达出口PE后通过扩散得到的路由信息发送至相应的出口CE这样就完成了VPN报文的转发；具体的报文实现上通常是对VPN报文进行两层标签的封装，外层的标签用于报文在公网上的传输，而内层的标签用于指定报文送达哪个VPN网络。也正是由于这两层标签的封装，实际讲用户的标识丰富成为了IP+VPN的二维信息，这样就解决了不同的用户私网中可能存在的地址重叠的问题。下图为VPN报文转发的简图。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142632z6c8he4zhfvrlsuh.png)

图中，10.1.5.1这一层表示的为报文的IP层；15362这一层表示的是2层VPN标签封装中的内侧标签，实际意义为VPNA的标志；1024与3这一层表示的是2层VPN标签封装中的外层标签，用于PEB到PEA通过P传输时的MPLS标签；报文到达PEA后我们可以得到VPNA的信息与报文的IP信息，这样就可以精确的送达到VPNA网络中的目的地，完成报文的传输。
