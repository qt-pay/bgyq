## VPN基础-draft

VPN的英文全称是“Virtual Private Network”，即“虚拟专用网络”。VPN是为了替代物理专用网络而出现，物理专用网络是通过架设或者租用物理线路来实现跨地域的互联。

VPN = 隧道 + 安全

VPN可以减少企业在公网暴漏端口的风险呢
### VPN demo：h3c

#### 客户需求

客户要求在交换机管理口(管理口接管理网交换)绑定VPN实例，从而实现从不同地址段远程登录交换机.

#### 需求分析

可以定义一个vpn管理实例，然后把实例绑定到管理口，实现管理的路由表和业务的路由表隔开，同时又打通路由可以访问，配置vpn实例路由这样就可以了。

#### 配置示例

所有接入的switch的mgt-vpn，都写相同的默认路由（都执行网关）即可。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/switch-mgt-gw-and-connect.png)

```bash
## 有管理口是一个 三层接口可以直接配置IP
ip vpn-instance MGMT
description MGMT
interface M-GigabitEthernet0/0/0
ip binding vpn-instance MGMT
ip address 192.168.0.1 24
ip route-static vpn-instance MGMT 0.0.0.0 0  192.168.0.254 description TO-MGMT

## 无管理口，需要借助VLAN IF配置IP
ip vpn-instance MGMT
vlan 100
int vlan 100
ip binding vpn-instance Management
ip address  192.168.0.1   255.255.255.0

## 
interface g1/0/1
port access vlan 100
ip route-static vpn-instance MGMT 0.0.0.0 0  192.168.0.254 description TO-MGMT

## 显示vpn示例
display ip vpn-instance [ instance-name vpn-instance-name ]

```

#### VPN：路由表隔离

只要端口或vlan都加入到vpn实例，所有涉及到VPN网段的网段路由、snnp、ntp、acl策略里面都需要加入到vpn实例中，不然无法互通。即需要写基于vpn-instance的路由。

端口或vlan加了vpn实例，相当于从iPv4的网络隔离出来进入到了vpn的网络，因此涉及到的配置都需要加入vpn实例。

不同VPN之间的路由隔离通过VPN实例（VPN-instance）实现，VPN实例又称为VRF（Virtual Routing and Forwarding，虚拟路由和转发）实例。

VPN（Virtual Private Network）：也称VRF(Virtual Route Forwarding，虚拟路由及转发) ，目的是解决不同企业私网地址段相同，为了防止冲突，采用将相同私网地址放到不同的VRF表中。

一台设备由于可能同时连接了多个用户，这些用户（的路由）彼此之间需要相互隔离，那么这时候就用到了VRF，设备上每一个用户都对应一个VRF。设备除了维护全局IP路由表之外，还为每个VRF维护一张独立的IP路由表，这张路由表称为VRF路由表。要注意的是全局IP路由表，以及每一个VRF路由表都是相互独立或者说相互隔离的。

对于每一个VRF表，都具有路由区分符(Route Distinguisher：RD)和路由目标(Route Target：RT)两大属性。

### VRF:tada:

VRF（Virtual Routing and Rorwarding，虚拟路由转发）技术，华为 VRF 又称 VPN Instance（VPN 实例），是种虚拟化技术。

在物理设备上，创建多个 VRF-Instance（VRF 实例，VRF-INST），然后将物理接口划入不同的 VRP-INST 中。
简单理解是，将整个设备“分割”成多个小设备。

对于每一个VRF表，都具有路由区分符(Route Distinguisher：RD)和路由目标(Route Target：RT)两大属性。

#### 典型应用场景

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-demo-company.png)

某企业网络内有生产和管理两张网络，这两张网络独占接入和汇聚层交换机，共享核心交换机。
但是，核心交换机上同时连接了生产网络和管理网络的服务器群，两个网段均为192.168.100.0/24网段。
**需求：**实现生产和管理网络内部的数据通信，同时隔离两张网络之间的通信。

这时，通过两个VPN实例，可以实现这个需求即不同租户可以复用同一个网段、

#### 特性说明

每个 VPN Instance 拥有独立的接口、路由表、路由协议进程、路由转发表项等。

#### 应用场景

常用于 MPLS VPN、Firewall 等一些需要实现隔离的应用场景。VRF 能够实现网络层的隔离。

##### Firewall 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrf-in-fw.png)

防火墙虚拟系统，虚拟系统（Virtual System）是在一台物理设备上划分出的多台相互独立的逻辑设备，其中路由虚拟化依靠创建 VPN Instance 来实现。

虚拟系统主要具有以下特点：
1）资源虚拟化：每个虚拟系统都有独享的资源，包括接口、VLAN、策略和会话等。
2）路由虚拟化：每个虚拟系统都拥有各自的路由表，相互独立隔离。

##### BGP/MPLS VPN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrf-in-vpn.png)

BGP/MPLS IP VPN 是种基于 PE 的 L3VPN 技术。它使用BGP在服务提供商骨干网上发布VPN路由，使用MPLS在服务提供商骨干网上转发VPN报文。
通过创建 VPN Instance 的方式在PE上区别不同VPN的路由。

### VPN技术的定义：traffic separation

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



### MCE设备 VPN:thinking:

为方便记忆和管理，建议用户在MCE设备和PE设备上为同一个VPN实例配置相同的RD。

BGP/MPLS VPN以隧道的方式解决了在公网中传送私网数据的问题，但传统的BGP/MPLS VPN架构要求每个VPN实例单独使用一个CE与PE相连。

传统的MPLS L3VPN架构要求每个用户站点单独使用一个CE与PE相连。随着用户业务的不断细化和安全需求的提高，一个私有网络内的用户可能需要划分成多个VPN，不同VPN用户间的业务需要完全隔离。此时，为每个VPN单独配置一台CE将加大用户的设备开支和维护成本；而多个VPN共用一台CE，使用同一个路由表项，又无法保证数据的安全性。



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/pe-ce-1-1.png)

实现方法：

在MCE设备上为不同的VPN创建各自的路由转发表，并绑定到对应的接口(interface)。在接收路由信息时，MCE设备根据接收报文的接口，即可判断该路由信息的来源，并将其维护到对应VPN的路由转发表中。

同时，在PE上也需要将连接MCE的接口与VPN进行绑定，绑定的方式与MCE设备一致。PE根据接收报文的接口判断报文所属的VPN，将报文在指定的隧道内传输。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/pe-ce-1-N.png)



**MCE功能通过在CE设备上建立VPN实例，为不同的VPN提供逻辑独立的路由转发表和地址空间，使多个VPN可以共享一个CE**。该CE设备称为MCE设备。MCE功能有效地解决了多VPN网络带来的用户数据安全与网络成本之间的矛盾。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/MPLS-L3VPN.png)

随着用户业务的不断细化和安全需求的提高，很多情况下**一个私有网络内的用户需要划分成多个VPN**，不同VPN用户间的业务需要完全隔离。此时，为每个VPN单独配置一台CE将加大用户的设备开支和维护成本；而多个VPN共用一台CE，使用同一个路由表项，又无法保证数据的安全性。

交换机提供的MCE（Multi-VPN-Instance CE，具备多VPN实例功能的CE）功能，可以有效解决多VPN网络带来的用户数据安全与网络成本之间的矛盾，它使用CE设备本身的VLAN接口编号与网络内的VPN进行绑定，并为每个VPN创建和维护独立的路由转发表（Multi-VRF）。这样不但能够隔离私网内不同VPN的报文转发路径，而且通过与PE间的配合，也能够将每个VPN的路由正确发布至对端PE，保证VPN报文在公网内的传输。

#### 工作原理

左侧私网内有两个VPN站点：站点1和站点2，分别通过MCE设备接入MPLS骨干网，其中VPN1和VPN2的用户，需要分别与远端站点2内的VPN1用户和站点1内的VPN2用户建立VPN隧道。

通过配置MCE功能，可以在MCE设备上为VPN1和VPN2创建不同的VPN实例，为其维护各自独立的路由表，并使用Vlan-interface2接口与VPN1进行绑定、Vlan-interface3与VPN2进行绑定。在接收路由信息时，MCE设备根据接收接口的编号，即可判断该路由信息的来源，并将其维护到对应VPN实例的路由转发表中。

同时，MCE需要通过不同的路由协议或进程来向PE发布VPN1和VPN2的路由，以便PE能够识别并分别维护不同VPN的路由。因此，MCE需要通过两个接口连接到PE，并创建两个VPN实例与两个接口分别绑定。在向PE发布VPN路由时，将不同的路由协议或进程与VPN实例绑定，并在不同的协议或进程中分别引入VPN1和VPN2内的路由。

通过上述过程，MCE即可将两个VPN内的路由信息分别发布到PE设备。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/MCE-vpn.png)

##### RD

RD(Route-Distinguisher，路由区分符)：RD用来区分本地VRF，该属性仅本地有效。8个字节的RD+4个字节的IPv4地址组成96位VPNv4路由，使不唯一的IPv4地址转化为唯一的VPN-IPv4地址，该VPNv4路由在ISP域内传递（区分），RD给某VRF里面的路由打上标签，进而实现地址的复用而不产生冲突。

##### RT

RT(Route Tagert)：是BGP的扩展团体属性，它分成Import RT和Export RT，分别用于路由的导入、导出策略。

通过配置import和export RT，来控制收发路由。

1.当从VRF表中导出路由时，要用export RT对VRF路由进行标记。

2.当往VRF表中导入路由时，只有所带RT标记与该VRF表中任意一个import RT相符的路由才会被导入到VRF表中。

##### VPN target

MPLS L3VPN使用BGP扩展团体属性——VPN Target（也称为Route Target）来控制VPN路由信息的发布。

VPN Target属性分为如下两类：

* Export Target属性：本地PE从与自己直接相连的站点学习到IPv4路由后，将其转换为VPN-IPv4路由，为VPN-IPv4路由设置Export Target属性并发布给其它PE。
* Import Target属性：PE在接收到其它PE发布的VPN-IPv4路由时，检查其Export Target属性。只有当此属性与PE上某个VPN实例的Import Target属性匹配时，才把路由加入到该VPN实例的路由表中。

例如：某VPN实例的Import Target包含100:1，200:1和300:1，当收到的路由信息的Export Target为100:1、200:1、300:1中的任意值时，都可以被注入到该VPN实例中。

#### demo

MCE和PE间使用OSPF引入VPN路由组网示意图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/MCE-VPN-demo.png)

为区分设备，假设MCE系统名为“MCE”，VPN1和VPN2的出口路由器分别名为“VR1”和“VR2”，PE设备系统名为“PE”。

```bash
# VPN实例配置
# 在MCE设备上配置VPN实例，名称分别为VPN1和VPN2，RD分别取值为10:1和20:1。
<MCE> system-view
[MCE] ip vpn-instance vpn1
[MCE-vpn-instance-vpn1] route-distinguisher 10:1
[MCE-vpn-instance-vpn1] quit
[MCE] ip vpn-instance vpn2
[MCE-vpn-instance-vpn2] route-distinguisher 20:1

# 创建VLAN10，将端口GigabitEthernet 1/0/10加入VLAN10，并创建Vlan-interface10接口。
[MCE-vpn-instance-vpn2] quit
[MCE] vlan 10
[MCE-vlan10] port GigabitEthernet 1/0/10
[MCE-vlan10] quit
[MCE] interface Vlan-interface 10

# 配置Vlan-interface10接口与VPN1实例进行绑定，并配置IP地址为10.214.10.3，掩码为24位。
[MCE-Vlan-interface10] ip binding vpn-instance vpn1
[MCE-Vlan-interface10] ip address 10.214.10.3 24
# 使用类似步骤配置VLAN20，将端口GigabitEthernet 1/0/20加入VLAN20，配置接口与VPN2实例绑定并配置IP地址。
[MCE-Vlan-interface10] quit
[MCE] vlan 20
[MCE-vlan20] port GigabitEthernet 1/0/20
[MCE-vlan20] quit
[MCE] interface Vlan-interface 20
[MCE-Vlan-interface20] ip binding vpn-instance vpn2
[MCE-Vlan-interface20] ip address 10.214.20.3 24
[MCE-Vlan-interface20] quit

# 创建VLAN30和VLAN40，并创建相应的VLAN接口，配置IP地址，将VLAN30与VPN1绑定，VLAN40与VPN2绑定。

[MCE] vlan 30
[MCE-vlan30] quit
[MCE] interface Vlan-interface 30
[MCE-Vlan-interface30] ip binding vpn-instance vpn1
[MCE-Vlan-interface30] ip address 10.214.30.1 30
[MCE-Vlan-interface30] quit
[MCE] vlan 40
[MCE-vlan40] quit
[MCE] interface Vlan-interface 40
[MCE-Vlan-interface40] ip binding vpn-instance vpn2
[MCE-Vlan-interface40] ip address 10.214.40.1 30
[MCE-Vlan-interface40] quit

# MCE与站点间路由配置
# MCE与VPN1直接相连，且VPN1内未使用路由协议，因此可以使用静态路由进行配置。

# VR1设备的配置。这里假设VR1为S5500-EI交换机，配置与MCE连接的接口地址为10.214.10.2/24，连接VPN1接口的地址为192.168.0.1/24。向VLAN中增加端口和配置接口IP地址的配置这里省略。

# 在VR1上配置缺省路由，指定出方向报文的下一跳地址为10.214.10.3。

<VR1> system-view
[VR1] ip route-static 0.0.0.0 0.0.0.0 10.214.10.3 

# 在MCE上指定静态路由，去往192.168.0.0网段的报文，下一跳地址为10.214.10.2，并将此路由与VPN1实例绑定。
[MCE-Vlan-interface20] quit
[MCE] ip route-static vpn-instance vpn1 192.168.0.0 16 10.214.10.2

# 显示MCE上为VPN1实例维护的路由信息。

[MCE] display ip routing-table vpn-instance vpn1
Routing Tables: vpn1

         Destinations : 5        Routes : 5

 

Destination/Mask    Proto  Pre  Cost         NextHop         Interface

 

127.0.0.0/8         Direct 0     0            127.0.0.1       InLoop0

127.0.0.1/32        Direct 0     0            127.0.0.1       InLoop0

10.214.10.0/24      Direct 0     0            10.214.10.3     Vlan10

10.214.10.3/32      Direct 0     0            127.0.0.1       InLoop0

192.168.0.0/16      Static 60    0            10.214.10.2     Vlan10

可以看到，已经在MCE上为VPN1指定了静态路由。

# 在VR2上，配置与MCE连接的接口地址为10.214.20.2/24（配置过程略），配置RIP，将网段192.168.10.0和10.214.20.0发布。

<VR2> system-view
[VR2] rip 20
[VR2-rip-20] network 192.168.10.0
[VR2-rip-20] network 10.0.0.0

# VPN2内已经运行了RIP，可以在MCE上配置RIP路由协议，加入到站点内的路由运算中，自动更新路由信息。配置RIP进程为20，关闭路由自动聚合功能，引入OSPF（进程号20）的路由信息，并与VPN2实例进行绑定。

[MCE] rip 20 vpn-instance vpn2

# 将网段10.214.20.0和10.214.40.0发布。

[MCE-rip-20] network 10.0.0.0
[MCE-rip-20] undo summary
[MCE-rip-20] import-route ospf

# 在MCE上查看VPN2实例的路由信息。

[MCE-rip-20] display ip routing-table vpn-instance vpn2
Routing Tables: vpn2

         Destinations : 5        Routes : 5

 

Destination/Mask    Proto  Pre  Cost         NextHop         Interface

 

10.214.20.0/24      Direct 0    0            10.214.20.3     Vlan20

10.214.20.3/32      Direct 0    0            127.0.0.1       InLoop0

10.214.40.0/30      Direct 0    0            10.214.40.1     Vlan40

10.214.40.1/32      Direct 0    0            127.0.0.1       InLoop0 

127.0.0.0/8         Direct 0    0            127.0.0.1       InLoop0

127.0.0.1/32        Direct 0    0            127.0.0.1       InLoop0

192.168.10.0/24     RIP    100  1            10.214.20.2     Vlan20 

可以看到，MCE已经通过RIP学习到了VPN2内的私网路由，并与VPN1内的192.168.0.0路由信息分别维护在两个路由表内，有效进行了隔离。

# MCE与PE间路由配置
# MCE使用GigabitEthernet1/0/3端口连接到PE设备的GigabitEthernet1/0/18端口，需要配置这两个端口为Trunk端口，并允许VLAN30和VLAN40的报文携带Tag通过。

[MCE-rip-20] quit
[MCE] interface GigabitEthernet 1/0/3
[MCE-GigabitEthernet1/0/3] port link-type trunk
[MCE-GigabitEthernet1/0/3] port trunk permit vlan 30 40

# 配置PE的GigabitEthernet1/0/18端口。

<PE> system-view
[PE] interface GigabitEthernet 1/0/18
[PE-GigabitEthernet1/0/18] port link-type trunk
[PE-GigabitEthernet1/0/18] port trunk permit vlan 30 40

# 配置PE的Vlan-interface30和Vlan-interface40接口的IP地址分别为10.214.30.2和10.214.40.2，并将这两个接口分别与VPN1实例和VPN2实例进行绑定，配置步骤这里省略。

# 配置MCE和PE的Loopback0接口，用于指定MCE和PE的Router ID，地址分别为101.101.10.1和100.100.10.1。配置步骤这里省略。

# 配置MCE启动OSPF进程10，配置绑定到VPN1实例，域ID设置为10，并开启OSPF多实例功能。

[MCE-GigabitEthernet1/0/3] quit
[MCE] ospf 10 router-id 101.101.10.1 vpn-instance vpn1
[MCE-ospf-10] domain 10
[MCE-ospf-10] vpn-instance-capability simple

# 在Area0区域发布10.214.30.0网段，并引入VPN1的静态路由。

[MCE-ospf-10] area 0
[MCE-ospf-10-area-0.0.0.0] network 10.214.30.0 0.0.0.255
[MCE-ospf-10-area-0.0.0.0] quit
[MCE-ospf-10] import-route static

# 配置PE启动OSPF进程10，绑定到VPN1实例，域ID为10，开启OSPF多实例功能，并在Area0区域发布10.214.30.0网段。

[PE-GigabitEthernet1/0/18] quit
[PE] ospf 10 router-id 100.100.10.1 vpn-instance vpn1
[PE-ospf-10] domain-id 10
[PE-ospf-10] vpn-instance-capability simple
[PE-ospf-10] area 0
[PE-ospf-10-area-0.0.0.0] network 10.214.30.0 0.0.0.255

# 显示PE上的VPN1路由信息。

[PE-ospf-10-area-0.0.0.0] display ip routing-table vpn-instance vpn1
Routing Tables: vpn1

         Destinations : 6        Routes : 6

 

Destination/Mask    Proto  Pre  Cost         NextHop         Interface

 

127.0.0.0/8         Direct 0    0            127.0.0.1       InLoop0

127.0.0.1/32        Direct 0    0            127.0.0.1       InLoop0

10.214.30.0/24      Direct 0    0            10.214.30.1     Vlan30

10.214.30.2/32      Direct 0    0            127.0.0.1       InLoop0

100.100.10.1/32     Direct 0    0            127.0.0.1       InLoop0

192.168.0.0/16      O_ASE  150  1            10.214.30.1     Vlan30 

#VPN1内的静态路由已经引入到MCE与PE间的OSPF路由表中。
#MCE与PE间配置OSPF进程20，导入VPN2实例的路由信息的过程与上面介绍的配置基本一致，不同的是在MCE的OSPF中配置导入的是RIP路由，这里不再赘述，只以显示信息为例表示导入成功后的结果。

<PE> display ip routing-table vpn-instance vpn2
display ip routing-table vpn-instance vpn2
Routing Tables: vpn2

         Destinations : 6        Routes : 6

 

Destination/Mask    Proto  Pre  Cost         NextHop         Interface

 

127.0.0.0/8         Direct 0    0            127.0.0.1       InLoop0

127.0.0.1/32        Direct 0    0            127.0.0.1       InLoop0

10.214.40.0/24      Direct 0    0            10.214.40.1     Vlan40

10.214.40.2/32      Direct 0    0            127.0.0.1       InLoop0

200.200.20.1/32     Direct 0    0            127.0.0.1       InLoop0

192.168.10.0/24     O_ASE  150  1            10.214.40.1     Vlan40 


```

至此，通过配置，已经将两个VPN实例内的路由信息完整地传播到PE中，配置完成。

### VPLS网络

Virtual Private LAN Service (VPLS) delivers a point-to-multipoint L2VPN service over an MPLS or IP backbone. The provider backbone emulates a switch to connect all geographically dispersed sites of each customer network. The backbone is transparent to the customer sites. The sites can communicate with each other as if they were on the same LAN.

VPLS 是基于以太网的点对多点第 2 层 VPN。它允许您通过 MPLS 主干将地理上分散的以太网局域网 （LAN） 站点相互连接。对于实施 VPLS 的客户，尽管流量遍及服务提供商的网络，但所有站点似乎都在同一以太网 LAN 中。

L2 VPN可以避免客户内部路由信息被CE设备学习，更加安全。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpls-vpn.jpg)

#### 组件介绍

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpls-部署步骤.jpg)

主要组成部分：

*  CE（Customer Edge，用户网络边缘）设备，直接与服务提供商网络相连的用户网络侧设备。
* PE（Provider Edge，服务提供商网络边缘）设备，与CE相连的服务提供商网络侧设备。PE主要负责VPN业务的接入，完成报文从用户网络到公网隧道、从公网隧道到用户网络的映射与转发。在分层VPLS组网下，PE可以细分为UPE和NPE。
* AC（Attachment Circuit，接入电路）连接CE和PE的物理电路或虚拟电路，例如Ethernet接口、VLAN。
* PW（Pseudowire，伪线）两个PE之间的虚拟双向连接。MPLS PW由一对方向相反的单向LSP构成。
* 公网隧道（Tunnel）穿越IP或MPLS骨干网、用来承载PW的隧道。一条公网隧道可以承载多条PW，公网隧道可以是LSP、MPLS TE、GRE隧道等。
* VPLS实例 用户网络可能包括分布在不同地理位置的多个站点。在骨干网上可以利用VPLS技术将这些站点连接起来，为用户提供一个二层VPN。这个二层VPN称为一个VPLS实例。不同VPLS实例中的站点不能二层互通。
* VSI（Virtual Switch Instance，虚拟交换实例）VSI是PE设备上为一个VPLS实例提供二层交换服务的虚拟实例。VSI可以看作是PE设备上的一台虚拟交换机，它具有传统以太网交换机的所有功能，包括源MAC地址学习、MAC地址老化、泛洪等。VPLS通过VSI实现在VPLS实例内转发二层数据报文。

#### cons

CE接入多个不同PE时，需要配置xSTP，而且面对不同的PE设备（huawei or cisco），可能配置还相同。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpls-cons-1.png)

收敛慢，evpn可以快速收敛。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpls-cons-2.jpg)



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

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn运营模式分类-华为.jpg)

  1）、CPE-Based VPN: 设备放置在用户侧(Custom Premise Equipment)

由用户管理或委托运营商进行管理；
   2）、Network-Based VPN: 设备放置在运营商网络侧

用户设备不需要感知VPN由运营商管理，又称为Provider Provide VPN。

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

* MPLS L3 VPN： BGP MPLS VPN

  MPLS L3VPN是一种三层VPN技术，它使用BGP在服务提供商骨干网上发布用户站点的私网路由，使用MPLS在服务提供商骨干网上转发用户站点之间的私网报文，从而实现通过服务提供商的骨干网连接属于同一个VPN、位于不同地理位置的用户站点。

##### L2VPN

* PPTP: 快淘汰，早期微软开发，windows系统都内置了。

  ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/微软-pptp-vpn.png)

* L2F：快淘汰，**L2F**，或**第二层转发协议**（Layer 2 Forwarding Protocol），是由思科公司开发的，建立在互联网上的虚拟专用网络连接的隧道协议。L2F协议本身并不提供加密或保密；它依赖于协议被传输以提供保密。L2F是专为隧道点对点协议（PPP）通信。

* L2TP: 主流L2VPN

  https://cshihong.github.io/2019/08/21/L2TP-VPN%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/
  
* MPLS L2 VPN

  包含VLL、VPLS、PWE3。VLL是点到点的业务，可以将整个网络看成一条网线；VPLS是点到多点业务，可将整个网络看成是一个交换机；PWE3是端到端的仿真，其实就是VLL的Martini形式扩展。

### EVPN:popcorn:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn产生背景.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-组成.jpg)



EVPN实现VxLAN控制平面的自动配置。

EVPN不允许二层流量跨越VNI。

Overlay的实现常用：evpn+vxlan

EVPN参考了MP-BGP（MultiProtocol BGP）的机制。在深入理解EVPN的工作原理前，我们先对MP-BGP（MultiProtocol BGP）做下简单回顾。

传统的BGP-4使用Update报文在对等体之间交换路由信息。一条Update报文可以通告一类具有相同路径属性的可达路由，这些路由放在NLRI（**Network Layer Reachable Information**，网络层可达信息）字段中。因为BGP-4只能管理IPv4单播路由信息，为了提供对多种网络层协议的支持（例如IPv6、组播），发展出了MP-BGP。MP-BGP在BGP-4基础上对NLRI作了新扩展。玄机就在于新扩展的NLRI上，扩展之后的NLRI增加了地址族的描述，可以用来区分不同的网络层协议，例如IPv6单播地址族、VPN实例地址族等。

类似的，EVPN也是借用了MP-BGP的机制，在L2VPN地址族下定义了新的子地址族——EVPN地址族，在这个地址族下又新增了一种NLRI，即EVPN NLRI。EVPN NLRI定义了几种BGP EVPN路由类型，这些路由可以携带主机IP、MAC、VNI、VRF等信息。这样，当一个VTEP学习到下挂的主机的IP、MAC地址信息后，就可以通过MP-BGP路由将这些信息发送给其他的VTEP，从而在控制平面实现主机IP、MAC地址的学习，抑制了数据平面的泛洪。

采用EVPN作为VXLAN的控制平面具有以下优势：

- 可实现VTEP自动发现、VXLAN隧道自动建立，从而降低网络部署、扩展的难度。
- EVPN可以同时发布二层MAC信息和三层路由信息。
- 可以减少网络中的泛洪流量。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-nlri.jpg)

#### mac learn

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-mac-learn.jpg)

#### 集中式网关跨网段

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-集中式网关-1.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-集中式网关2.png)



#### 分布式网关跨网段

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-1.jpg)

L3 VNI 和 L3 VPN 有映射关系

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-2.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-3.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-4.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-5.jpg)

隧道传输和解封装

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/evpn-分布式网关-6.jpg)

##### L3 VPN info

VPN用来区别不同租户的路由表，因为一个leaf下面可能



华为配置L3 VPN

配置与EVPN实例交互路由的VPN实例。

**当Overlay网络是IPv4网络时，执行如下操作步骤：**

1. 执行命令**system-view**，进入系统视图。

2. 执行命令**ip vpn-instance vpn-instance-name**，创建VPN实例，并进入VPN实例视图。

   缺省情况下，未创建VPN实例。

3. 执行命令[**vxlan vni**] *vni-id*，创建VXLAN网络标识VNI并关联VPN实例。

   缺省情况下，VNI与VPN实例没有绑定。

4. 执行命令[**ipv4-family**]，使能VPN实例IPv4地址族，并进入VPN实例IPv4地址族视图。

   缺省情况下，未使能VPN实例的IPv4地址族。

5. 执行命令[**route-distinguisher**] *route-distinguisher*，配置VPN实例IPv4地址族的RD。

   缺省情况下，没有为VPN实例IPv4地址族配置RD。

6. （可选）执行命令[**vpn-target**]*vpn-target* &<1-8> [ **both** | **export-extcommunity** | **import-extcommunity** ]，为VPN实例IPv4地址族配置VPN-target扩展团体属性。

   缺省情况下，没有为VPN实例IPv4地址族配置入方向和出方向VPN-Target扩展团体属性列表。

   如果当前节点还需要与同一VPN实例的其他节点交换L3VPN路由，则需要执行此步骤，配置VPN实例本身的VPN-Target值。

7. 执行命令[**vpn-target**]*vpn-target* &<1-8> [ **both** | **export-extcommunity** | **import-extcommunity** ] **evpn**，为VPN实例IPv4地址族配置EVPN路由的VPN-target扩展团体属性。

   当本端设备向邻居发布EVPN的IP前缀路由时，该路由携带此步骤为EVPN路由设置的出方向VPN-Target属性列表中的所有VPN-Target属性；当本端设备向邻居发布EVPN的IRB路由时，该路由携带的是BD下EVPN实例的出方向VPN-Target属性列表中的所有VPN-Target属性。

   当本端设备收到的IRB路由或IP前缀路由携带的VPN-Target属性，与此步骤为EVPN路由设置的入方向VPN-Target属性列表中的值存在交集时，才允许该IRB路由或IP前缀路由交叉到本地VPN实例IPv4地址族路由表。

8. （可选）执行命令[**import route-policy**]policy-name **evpn**，将当前VPN实例IPv4地址族与一条入方向Route-Policy进行关联，该Route-Policy用来过滤引入到当前VPN实例IPv4地址族的EVPN路由。

   如果需要更精确地控制进入到当前VPN实例IPv4地址族的EVPN路由，可以执行此步骤指定入方向Route-Policy来过滤EVPN路由信息，以及为通过过滤条件的路由设置路由属性。

9. （可选）执行命令[**export route-policy**] *policy-name* **evpn**，将当前VPN实例IPv4地址族与一条出方向Route-Policy进行关联，该Route-Policy用来过滤当前VPN实例IPv4地址族发布的EVPN路由。

   如果需要更精确地控制当前VPN实例IPv4地址族发布的EVPN路由，可以执行此步骤指定出方向Route-Policy来过滤发布的EVPN路由信息，以及为通过过滤条件的路由设置路由属性。

10. 执行命令quit，退出VPN实例IPv4地址族视图。

11. 执行命令quit，退出VPN实例视图。

```bash
[~NPE2] ip vpn-instance vpn1
[*NPE2-vpn-instance-vpn1] ipv4-family
[*NPE2-vpn-instance-vpn1-af-ipv4] route-distinguisher 20:2
[*NPE2-vpn-instance-vpn1-af-ipv4] vpn-target 1:1 both
[*NPE2-vpn-instance-vpn1-af-ipv4] vpn-target 2:2 both evpn
[*NPE2-vpn-instance-vpn1-af-ipv4] evpn mpls routing-enable
[*NPE2-vpn-instance-vpn1-af-ipv4] quit
[*NPE2-vpn-instance-vpn1] quit
[*NPE2] interface GigabitEthernet 0/2/0
[*NPE2-GigabitEthernet0/2/0] ip binding vpn-instance vpn1
[*NPE2-GigabitEthernet0/2/0] ip address 192.168.30.1 24
[*NPE2-GigabitEthernet0/2/0] quit
[*NPE2] bgp 100
[*NPE2-bgp] ipv4-family vpn-instance vpn1
[*NPE2-bgp-vpn1] advertise l2vpn evpn
[*NPE2-bgp-vpn1] import-route direct
[*NPE2-bgp-vpn1] quit
[*NPE2-bgp] quit
[*NPE2] commit
```



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

### VPN and Tunnel:fire:

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

在L3VPN中，不同VPN之间的路由隔离通过VPN实例（VPN-instance）实现，VPN实例又称为VRF（Virtual Routing and Forwarding，虚拟路由和转发）实例。

L3VPN为切入点，对VPN进行一个由点到面的介绍。

L3VPN中部署最为广泛的就是MPLS BGP VPN了，MPLS提供公网隧道的转发，BGP提供私网路由的扩散。下图为MPLS BGP VPN的网络结构。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142611g1gx7lglbxzwksnp.png)

这里VPN的实际目的就很明确，两侧VPNA的网络用户需要互联互通，而同侧的VPNA和VPNB网络需要相互隔离。简单介绍一下各个网元概念，CE指的是用户边缘设备，通常是用户侧的交换机、路由器或者是台主机；PE指运营商边缘设备，主要负责与CE通过路由协议进行对接，将CE侧的路由存入自己的VPN路由表中，并且将用户数据传入网络侧的隧道中去。P指的是运营商核心设备，主要负责公网的隧道转发。从各个设备的主要功能，就可以基本了解到这个L3VPN网络的基本工作原理了，CE与PE通过路由协议直接连接，PE设备存储CE的VPN与路由信息，入口PE到出口PE通过P设备则通过MPLS进行公网的转发传输，通过BGP进行路由的扩散；报文到达出口PE后通过扩散得到的路由信息发送至相应的出口CE这样就完成了VPN报文的转发；具体的报文实现上通常是对VPN报文进行两层标签的封装，外层的标签用于报文在公网上的传输，而内层的标签用于指定报文送达哪个VPN网络。也正是由于这两层标签的封装，实际讲用户的标识丰富成为了IP+VPN的二维信息，这样就解决了不同的用户私网中可能存在的地址重叠的问题。下图为VPN报文转发的简图。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/142632z6c8he4zhfvrlsuh.png)

图中，10.1.5.1这一层表示的为报文的IP层；15362这一层表示的是2层VPN标签封装中的内侧标签，实际意义为VPNA的标志；1024与3这一层表示的是2层VPN标签封装中的外层标签，用于PEB到PEA通过P传输时的MPLS标签；报文到达PEA后我们可以得到VPNA的信息与报文的IP信息，这样就可以精确的送达到VPNA网络中的目的地，完成报文的传输。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpls-部署步骤.jpg)



### 引用

1. https://bbs.huaweicloud.com/blogs/281600
2. https://www.bilibili.com/video/BV1Eh411179P/?spm_id_from=333.788.recommend_more_video.-1&vd_source=2795986600b37194ea1056cddb9856fa
3. https://blog.k4nz.com/93455d77477135edab934f318e2d0e60/
4. https://blog.csdn.net/qq_45959697/article/details/123913729
