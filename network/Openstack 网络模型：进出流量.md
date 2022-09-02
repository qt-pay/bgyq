## Openstack 网络模型

### 问题

Openstack VM访问集群外其他宿主机时，IP数据包中记录的源IP是vm的IP地址么？

答案：

在provider模式下，vlan组网，vm IP即floating IP，vm访问集群外网络就是vm source IP

在self-service模式下，无论vlan，vxlan等，vm访问集群外都不是源IP了，是floating IP或者SNAT后的fix ip。



### Openstack VS. k8s 网络模型

如果不允许同一子网跨多个物理节点，则每个节点上的路由器就不会冲突，设计起来就简单的多。

k8s的网络就是不允许一个子网跨多个物理节点，calico倒是通过`IP/32`静态路由方式实现了，多个节点共享一个网段的功能。

### Openstack 网络模式

#### provider模式

provider是一个半虚拟化的二层网络架构，只能通过桥接的方式实现，处于provider网络模式下vm获取到的ip地址与物理网络在同一网段，可以看成是物理网络的扩展，在该模式下，控制节点不需要安装L3 agent，也不需要网络节点，vm直接通过宿主机的NIC与物理网络通信,provider网络只支持flat和vlan两种模式。

对于neutron而言，**这种网络类型是“没有”三层路由功能的，或者说没有自主的路由功能**，他需要借助外部的网络，才能完成不同网络之间的路由。也就是说他的路由器或者三层网络服务是由openstack 之外的力量提供，因此被称为provider。

宿主机接入交换机的接口设置为Trunk模式即可。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-provider.png)

#### slef-service模式

self-service就是neutron不依赖外部的网络和三层路由，租户自己通过ovs或者Linux bridge创建虚拟的路由来进行交换。self-service也不需要配置provider。

self-service模式允许租户自己创建网络，最终租户创建的网络借助provider网络以NAT方式访问外网，所以self-service模式可以看成是网络层级的延伸，要实现self-service模式必须先创建provider网络,self-service网络支持flat、vlan、vxlan、gre模式。其架构如下:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-self-service.png)

#### fix ip and floating ip

传统SNAT只能实现vm访问外网，但是无法实现外网主动向vm发起通讯，因为SNAT无法为每个vm提供一个固定的静态对外IP。这就是floating IP做的事情了。

vm从self-service获取到的IP地址称为fix ip，vm从provider网络获取到的IP地址称为floating IP。

不管租户创建的网络类型为vxlan或者vlan（br-tun或br-vlan）租户vm之间通过fix ip之间的互访流量称为东西走向，只有当vm需要通过snat访问外网或者从通过fip访问vm时，此时的流量称为南北走向。

Note that the same network can be used to allocate floating IP addresses to instances, even if they have been added to private networks at the same time. The addresses allocated as floating IPs from this network will be bound to the *qrouter-xxx* namespace on the Network node, and will perform *DNAT-SNAT* to the associated private IP address. In contrast, the IP addresses allocated for direct external network access will be bound directly inside the instance, and allow the instance to communicate directly with external network.

### Network and Subnet

network 必须属于某个 Project（ Tenant 租户），Project 中可以创建多个 network。 network 与 Project 之间是 1对多 关系。

subnet 是一个 IPv4 或者 IPv6 地址段。instance 的 IP 从 subnet 中分配。每个 subnet 需要定义 IP 地址的范围和掩码。

subnet 与 network 是 1对多 关系。一个 subnet 只能属于某个 network；一个 network 可以有多个 subnet，这些 subnet 可以是不同的 IP 段，但不能重叠。下面的配置是有效的：

```bash
network A       subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
                subnet A-b: 10.10.2.0/24  {"start": "10.10.2.1", "end": "10.10.2.50"}
```

但下面的配置则无效，因为 subnet 有重叠

```bash
networkA        subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
                subnet A-b: 10.10.1.0/24  {"start": "10.10.1.51", "end": "10.10.1.100"}
```

这里不是判断 IP 是否有重叠，而是 subnet 的 CIDR 重叠（都是 10.10.1.0/24）

但是，如果 subnet 在不同的 network 中，CIDR 和 IP 都是可以重叠的，比如

```bash
network A       subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}

networkB        subnet B-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
```

这里大家不免会疑惑： 如果上面的IP地址是可以重叠的，那么就可能存在具有相同 IP 的两个 instance，这样会不会冲突？ 简单的回答是：不会！

具体原因**： 因为 Neutron 的 router 是通过 Linux network namespace 实现的。network namespace 是一种网络的隔离机制。通过它，每个 router 有自己独立的路由表。**

上面的配置有两种结果：

1. 如果两个 subnet 是通过同一个 router 路由，根据 router 的配置，只有指定的一个 subnet 可被路由。
2. 如果上面的两个 subnet 是通过不同 router 路由，因为 router 的路由表是独立的，所以两个 subnet 都可以被路由。

#### port

port 可以看做虚拟交换机上的一个端口。port 上定义了 MAC 地址和 IP 地址，当 instance 的虚拟网卡 VIF（Virtual Interface） 绑定到 port 时，port 会将 MAC 和 IP 分配给 VIF。

port 与 subnet 是 1对多 关系。一个 port 必须属于某个 subnet；一个 subnet 可以有多个 port。

```bash
$ openstack port list

# 可以查看Router Port或者VM Port
$ neutron port-show Port_ID
```

### Router

#### 界面创建Router

点击 “router_100_101” 链接进入 router 的配置页面，在 “Interfaces” 标签中点击 “Add Interface” 按钮

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-add-router-1.png)

选择 vlan100 的 subnet_172_16_100_0，点击 “Add Interface” 确认

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-add-router-2.png)

用同样的方法添加 vlan101 的 subnet_172_16_101_0

完成后，可以看到 router_100_101 有了两个 interface，其 IP 正好是 subnet 的 Gateway IP 172.16.100.1 和 172.16.101.1。router_100_101 已经连接了 subnet_172_16_100_0 和 subnet_172_16_101_0。
两个不同subnet下的 cirros-vm1 和 cirros-vm3 应该可以通信了。

#### 东西流量

##### 不跨节点

对于同一主机上的不同子网之间访问，路由器直接在 br-int 上转发即可，不需要经过外部网桥。

##### 同网段跨节点：br-tun mac

相同subnet之间通信不需要借助L3 Agent，vm之间流量如下图所示

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/one-subnet-on-multi-node.png)

1.vm1向vm2发起请求，通过目的IP地址得知vm2与自己在同一网段。
2.数据包经过linux bridge，进行安全策略检查，进入br-int打上内部vlan号。
3.数据包脱掉br-int的内部vlan号进入br-tun/br-vlan。
4.进入br-tun的数据包此时vxlan封装并打上vni号，进入br-vlan的数据包此时打上外部vlan号，通过nic离开compute1。
5.数据包进入compute2，经过br-tun的数据包完成vxlan的解封装去掉vni号，经过br-vlan的数据包去掉vlan号。
6.数据包进入br-int，此时会被打上内部vlan号。
7.数据包经过离开br-int并去掉内部vlan号，送往linux bridge通过其上的iptables安全策略检查。

8.最后数据包送到vm2。

##### 不同网段跨节点

无论tenant创建的网络类型是隧道还是vlan，不同subnet之间的通信必须借助L3 Agent完成，而在集中式网络节点架构中，只有网络节点部署了该角色，vm之间的流量如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/multi-subnet-on-multi-node-openstack.png)

1.vm1向vm2发出通信请求，根据目的IP地址得知vm2和自己不在同一网段，将数据包送往网关。
2.数据包经过linux bridge通过其上的iptables安全策略检查，按后送往br-int并打上内部vlan号。
3.数据包脱掉br-int的内部vlan号进入br-tun/br-vlan。
4.进入br-tun的数据包此时vxlan封装并打上vni号，进入br-vlan的数据包此时打上外部vlan号，通过nic离开compute1。
5.数据包进入网络节点，经过br-tun的数据包完成vxlan的解封装去掉vni号，经过br-vlan的数据包去掉vlan号。
6.数据包进入br-int，此时会被打上内部vlan号。
7.进入router namespace路由空间找到网关，vm1的网关配置在此路由器接口qr-1口上。
8.将数据包路由到vm2的网关，vm2的网关配置在qr-2口上，送回br-int并打上内部vlan号。
9.数据包脱掉br-int的内部vlan号进入br-tun/br-vlan。
10.进入br-tun的数据包此时vxlan封装并打上vni号，进入br-vlan的数据包此时打上外部vlan号，通过nic离开network node。
11.数据包进入compute2，经过br-tun的数据包完成vxlan的解封装去掉vni号，经过br-vlan的数据包去掉vlan号。
12.数据包进入br-int，此时会被打上内部vlan号。
13.数据包经过离开br-int并去掉内部vlan号，送往linux bridge通过其上的iptables安全策略检查。

14.最后数据包送到vm2

------





![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/不同节点东西流量-openstack.jpg)

如图所示，租户 T1 的两台虚拟机 VM1（计算节点 1）和 VM4（计算节点 2）分别属于不同子网，位于不同的计算节点。VM1 要访问 VM4，由计算节点 1 上的 IR1 起到路由器功能。返程的网包，则由计算节点 2 上的路由器 IR2 起作用。

两个路由器的 id、内部接口、功能等其实都是一样的。即同一个路由器，但是实际上在多个计算节点上同时存在。

这里可能有人会想到，多台同样的路由器，如果都暴露在外部网络上，会出现冲突。例如当 VM1 的请求包离开计算节点 1 时，带的源 mac 是路由器目标接口的 mac，而这个 mac 在计算节点 2 上的路由器上同样存在。

> 源mac和目标mac相同，会导致mac地址震荡，造成二层环路

因此，需要拦截路由器对外的暴露信息。一个是让每个路由器只应答本机的 mac 请求；另一个是绝对不让带着路由器 mac 地址的包直接扔出去。实现上在 br-int 上进行拦截，修改其源 mac 为 tunnel 端口的 mac。同样的，计算节点 2 在 br-int 上拦截源 mac 为这个 tunnel 端口的 mac，替换为正常的子网网关的 mac，直接扔给目标虚拟机所在的主机。



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/How_DVR_works-东西流量.png)

每个东西向路由器有自己的命名空间，负责跨子网的转发。

如上图所示，租户两个子网，红色和绿色，分别有 vm1 和 vm2，位于节点 cn1 和 cn2 上。

vm1 访问 vm2 的网包如步骤 1-6，整个过程 ip 保持不变。

- 原始包，vm1 访问 vm2，目的 mac 为本地（红网）的路由器网关接口 `r1 red mac`；
- 经过 br-int-cn1 转发，该网包通过本地（红网）网关接口扔给本地路由器 r1。
- r1 根据路由规则，经过到绿网的接口发出，此时网包的源 mac 改为绿网的网关接口 `r1 grn mac`，目的 mac 改为 `vm2 mac`，并且带上绿网的本地 vlan tag；
- 网包发给 br-tun-cn1 进行 tunnel，扔出去之前，将源 mac 替换为跟节点相关的特定 mac `dvr cn1 mac`，之后带着目标子网（绿网）的 外部 tunnel id 扔出去（实现可以为 vlan、vxlan、gre 等，功能都是一样的）；
- 节点 cn2 的网桥 br-tun-cn2 会从 tunnel 收到这个包，解封包，带上本地 vlan tag，最终抵达网桥 br-int-cn2；
- br-int-cn2 上替换网包的源 mac（此时为 dvr-cn1-mac）为本地路由器的绿网网关接口，然后发给 vm2。

返回包的过程正好是反过来。虽然实现上略复杂，但整个过程还是比较清晰的，保证 vm1 和 vm2 感觉到的都是直接跟路由器的接口相连（分别为红网网关接口和绿网网关接口）。





#### 南北流量

南北流量也分为有floating ip和无floating ip（fix ip）两种情况，唯一的区别在于vm最终离开network node访问internet时，有floating ip的vm源地址为floating ip，而使用fix ip的vm通过snat方式，源地址为network node的ip。

##### SNAT fix IP

流量必须依赖网络节点，这就会造成网络节点单点

没有绑定floating ip的vm在访问外网时需要通过网络节点的SNAT Router NameSpace进行地址转换，其流量走向如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-snat-fix-ip.png)

1.vm向外网发起请求，数据报文送往linux bridge。
2.进入linux bridge的数据报文经过iptables安全策略检查后将报文送往br-int，此时打上内部vlan号。
3.数据报文从br-int送往Router NameSpace的qr口，该接口配置了vm的网关地址，在Router NameSpace内对Snet NameSpace的sg口的mac地址进行解析，sg接口为vm所在子网的接口，该接口上的ip地址与vm在同一网段。然后将报文送往br-tun。****
4.数据报文进入br-tun后脱掉内部vlan号，进行vxlan封装，打上vni号，离开conpute1.
5.数据报文进入Network节点，脱掉vni号，进行vxlan解封装，送往br-int交换机，进入br-int交换机后打上内部vlan号。
6.数据报文进入sg后，进行路由查表，将数据发往fg口，fg口上配置的是可被路由的公网ip。

7.数据报文在fg口上进行SNAT地址转换，转换后的源ip地址为fg口上配置的公网ip访问公网。



##### floating IP

**没有floating IP，外部网络的主机无法主动发起与内网的通信。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dvr_compute_node-1.png)

单独有一个 qfloat-XXX 路由器（也在一个独立命名空间中）来负责处理带有 floating IP 的南北向流量。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dvr_compute_node_flow.png)



启用DVR功能后每台计算节点主机都安装了L3 Agent，绑定了floating ip的vm不再需要绕行到网络节点，直接由计算节点主机访问呢公网，其流量走向如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-dvr-floating-ip.png)

1.vm向外网发起访问，由于vm是provider类型（provider类型？）的私网地址，所以首先要去找vm地址所在的网关。
2.数据报文经过linux bridge和br-int后进入Distribute NameSpace的qr口，该接口配置的ip地址为vm的网关地址。
3.数据报文从qr口流出，进入rfp口，该接口上配置有2个ip地址，**其中3为vm绑定的floating ip地址，在此处进行SNAT地址转换，外网流量访问vm时在此名称空间利用iptables做DNAT地址转换。**
4.通过qrouter与fip内部通信的直连接口（4），接口地址由L3 Agent自行维护，ip为169.254.x.x/31格式，将数据包发往fip名称空间。
5.fip空间的直连接口fpr接收到数据包后，转发给外网网关fg口。

6.fip名称空间外网网关接口将数据包发到br-ex交换机最后通过物理网卡访问internet，外网访问vm的数据流向为该过程的逆方向，此处不再赘述。

**针对使用floating ip的数据包进出时需要注意的地方是：**
1.fg接口上会额外配置一个外网ip地址，这也是为什么公有云场景下不会将vm外网ip直接设置成公网ip地址的原因，因为每个计算主机都需要一个额外的地址作为fg网关地址。

2.当外部网络访问vm时，请求的ip地址是qrouter名称空间中rfp接口上做SNAT的ip地址，但此时fg接口会响应rfp接口上外网ip的arp地址解析请求，所以通常认为fg接口是floating ip的arp代理接口。



#### 网络节点单点

由于openstack默认部署模式下，计算节点通过ml2插件实现二层互通，所有三层流量都要经过网络节点，如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-网络节点.png)

同一tenant下有2个不同子网，vm1/2和vm3/4分别属于不同的subnet，通过上图可以看出不同子网之间通信，以及未绑定fip的vm与公网间通信，和拥有fip的vm通过公网访问都需要经过网络节点，网络节点存在单点隐患，对此需要通过L3 HA来对该节点进行高可用。

##### 集中式路由好处

neutron路由器支持一对一的静态NAT，和多对一的PNAT。默认情况下路由器执行PNAT操作，租户网络内的主机经路由器与外部通信时，源地址都会修改为路由器路由器外部接口的地址。这也被称作源地址转换SNAT。是内网主机上网最常用的一种方式。**这种情况下所有的主机共用一个外网地址，外部网络的主机无法主动发起与内网的通信**。这时就需要静态NAT功能，将内网主机映射为特定的外网地址。即给主机绑定一个fip。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-集中式路由-snat-fix-IP.png)

通过路由的NAT功能，只要统一做好外部网络的地址规划后，即可实现租户内网IP子网的复用，不同的租户之间能够复用完全相同的子网网段，且不会冲突。这也是公有云上能够容纳大量租户的原因之一。

#### DVR 网络节点HA

开启DVR，服务基本没变动，除了 L3 服务需要配置为 dvr_snat 模式。

命名空间上会多一个专门的 snat-xxx 命名空间，处理来自计算节点的无 floating IP 的南北向流量。

为了降低网络节点的负载，同时提高可扩展性，OpenStack 自 Juno 版本开始正式引入了分布式路由（Distributed Virtual Router，DVR）特性（用户可以选择使用与否），来让计算节点自己来处理原先的大量东西向流量和非 SNAT 南北流量（有 floating IP 的 vm 跟外面的通信）。

这样网络节点只需要处理占到一部分的 SNAT （无 floating IP 的 vm 跟外面的通信）流量，大大降低了负载和整个系统对网络节点的依赖。很自然的，FWaaS 也可以跟着放到计算节点上。

DHCP 服务、VPN 服务目前仍然需要集中在网络节点上进行。

DVR有两种运行模式。一个是 dvr-snat 模式，该模式的下，主机上的 dvr_snat 不仅要负责不同子网之间东西向的流量，还要作为租户网络与外部网络通信的网关。内网与外部网络所有的南北向流量都需要经过该主机上的 dvr 进行传输，并启用路由器SNAT功能。另一种则是 dvr 模式，这一模式下，dvr 仅处理租户网络内部子网的东西向流量，即租户在物理主机与物理主机之间的内网流量。该配置由由L3agent的配置文件实现

```ini
agent_mode = dvr_snat/dvr
```

开启DVR模式下的网络节点只是针对没有绑定floating ip的vm进行SNAT地址转换，并且qrouter名称空间只处理元数据，所以不同于传统L3 HA对Router NameSpace的高可用，DVR下的L3 HA是对SNAT NameSpace进行的高可用，仍采用vrrp实现，如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dvr-ha-1.png)

从部署结构来看，分别要对SNAT外网ip地址和子网接口ip地址做高可用，所以当使用keepalive时，此时架构如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dvr-ha-2.png)

### Using VLAN provider networks：X电测试环境

how to use provider networks to connect instances directly to an external network.

官网：https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/10/html/networking_guide/sec-connect-instance#doc-wrapper

https://blog.51cto.com/u_15642673/5299348



This procedure creates VLAN provider networks that can connect instances directly to external networks. You would do this if you want to connect multiple VLAN-tagged interfaces (on a single NIC) to multiple provider networks. This example uses a physical network called `physnet1`, with a range of VLANs (`171-172`). The network nodes and compute nodes are connected to the physical network using a physical interface on them called `eth1`. The switch ports to which these interfaces are connected must be configured to trunk the required VLAN ranges.

需要把一个独立 NIC 上的多个 VLAN-tagged 接口连接到多个供应商网络。如下例子，一个名为 `physnet1` 的网络，它具有一组 VLAN（`171-172`）。网络节点和计算节点使用名为 `eth1` 的物理接口连接到物理网络。连接这些接口的交换机网络被配置为对所需的 VLAN 进行端口汇聚（trunk）。
以下过程使用上面提供的 VLAN ID 和名称对 VLAN 供应商网络进行配置。

#### 控制器节点修改

1）L2 Plugin启用vlan插件

编辑 `/etc/neutron/plugin.ini` （到 `/etc/neutron/plugins/ml2/ml2_conf.ini` 文件的符合链接）文件来启用 *vlan* 机制驱动，把 *vlan* 添加到存在的值列表中。如：

```bash
[ml2]
type_drivers = vxlan,flat,vlan
```

2）配置 `network_vlan_ranges` 的设置来反映使用的物理网络和 VLAN。并重启 *neutron-server* 服务以使所做的修改生效

```bash
[ml2_type_vlan]
network_vlan_ranges=physnet1:171:172

$ systemctl restart neutron-server
```

3）创建外部网络作为 *vlan* 类型，并把它们与配置的 `physical_network` 相关联。把它创建为一个 `--shared`网络来允许其它用户直接连接到实例。这个例子会创建两个网络：一个为 VLAN 171，另一个为 VLAN 172：

> “--shared”属性创建租户网络，允许其他租户将自己的实例附加到该网络。

```bash
$ neutron net-create provider-vlan171 \
			--provider:network_type vlan \
			--router:external true \
			--provider:physical_network physnet1 \
			--provider:segmentation_id 171 --shared

$ neutron net-create provider-vlan172 \
			--provider:network_type vlan \
			--router:external true \
			--provider:physical_network physnet1 \
			--provider:segmentation_id 172 --shared
```

4） Create a number of subnets and configure them to use the external network. This is done using either `neutron subnet-create` or the dashboard. You will want to make certain that the external subnet details you have received from your network administrator are correctly associated with each VLAN. In this example, VLAN 171 uses subnet *10.65.217.0/24* and VLAN 172 uses *10.65.218.0/24*:

```bash
$ neutron subnet-create \
			--name subnet-provider-171 provider-171 10.65.217.0/24 \
			--enable-dhcp \
			--gateway 10.65.217.254 \

$ neutron subnet-create \
			--name subnet-provider-172 provider-172 10.65.218.0/24 \
			--enable-dhcp \
			--gateway 10.65.218.254 \
```



#### 计算和网络节点修改

必须在 network 节点和 compute 节点上都进行。这将把节点连接到外部网络，并使实例可以直接和外部网络进行通讯。

1）创建一个外部网桥（*br-ex*），并把它与端口（*eth1*）进行关联

```bash
# 配置网桥使用eth1
$ vim /etc/sysconfig/network-scripts/ifcfg-eth1

DEVICE=eth1
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none

# 配置br-ex网桥
$ vim /etc/sysconfig/network-scripts/ifcfg-br-ex:

DEVICE=br-ex
TYPE=OVSBridge
DEVICETYPE=ovs
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
```

2）在 `/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini` 中配置物理网络，把网桥映射到相关的物理网络：

```bash
bridge_mappings = physnet1:br-ex
```

3）重启服务

```bash
systemctl restart neutron-openvswitch-agent
```

#### 网络节点L3 agent修改

把 `/etc/neutron/l3-agent.ini` 文件中的 `external_network_bridge =` 设置`br-ex`

```bash
# Name of bridge used for external network traffic. This should be set to
# empty value for the linux bridge
# When external_network_bridge is set, each L3 agent can be associated with no
# more than one external network. This value should be set to the UUID of that
# external network. To allow L3 agent support multiple external networks, both
# the external_network_bridge and gateway_external_network_id must be left
# empty. (string value)
# 当neutron启用L3 agent时，如果在配置文件中配置了external_network_bridge，从这个bridge上出去的包只能是untag的。但在DC中，极有可能被分配的是某一vlan
external_network_bridge =

## 重启 neutron-l3-agent 以使所做的修改生效。
systemctl restart neutron-l3-agent
```



#### 流量分析

经过上面配置创建两个`sharded vlan network`，并配置完`br-ex`后，形成如下所示的网络结构：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vlan-provider-arc.jpg)

##### 出站流量

**1**. eth0 接口的数据会首先到达与实例相连接的 linux 网桥 qbr-xx。

**2.** qbr-xx 使用 qvb-xx <--> qvo-xxx连接到 br-int。(这是个patch port吧)

**3.** qvb-xx 被连接到 linux 网桥 qbr-xx；qvo-xx连接到 Open vSwitch 网桥 br-int。

**4.** br-int到br-ex简单

`qvo-xx`的配置，`qvo-xx` 使用与 VLAN 供应商网络相关联的内部 VLAN tag 进行标记（tag）。

```bash
$ brctl show XXX         
         options: {peer=phy-br-ex}
        Port "qvo86257b61-5d"
            tag: 3

            Interface "qvo86257b61-5d"
        Port "qvo84878b78-63"
            tag: 2
            Interface "qvo84878b78-63"
```



 phy-br-ex（`in_port=4`）的数据包带有 vlan tag 2（`dl_vlan=2`）。Open vSwitch 会把这个 VLAN tag 替换为 171（`actions=mod_vlan_vid:171,NORMAL`）后转发数据包。 它还显示了到达 phy-br-ex（`in_port=4`）的数据包带有 vlan tag 3（`dl_vlan=3`）。Open vSwitch 会把这个 VLAN tag 替换为 172（`actions=mod_vlan_vid:172,NORMAL`）后转发数据包。这些规则由 neutron-openvswitch-agent 自动添加。

VLAN tag 转换完成后，这个数据包会被发送到物理接口 *eth1*。

```bash
# phy-br-ex 端口号是 4：
$ ovs-ofctl show br-ex
 4(phy-br-ex): addr:32:e7:a1:6b:90:3e
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
## 流表分析
# ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=6527.527s, table=0, n_packets=29211, n_bytes=2725576, idle_age=0, priority=1 actions=NORMAL
 cookie=0x0, duration=2939.172s, table=0, n_packets=117, n_bytes=8296, idle_age=58, priority=4,in_port=4,dl_vlan=3 actions=mod_vlan_vid:172,NORMAL
 cookie=0x0, duration=6111.389s, table=0, n_packets=145, n_bytes=9368, idle_age=98, priority=4,in_port=4,dl_vlan=2 actions=mod_vlan_vid:171,NORMAL
 cookie=0x0, duration=6526.675s, table=0, n_packets=82, n_bytes=6700, idle_age=2462, priority=2,in_port=4 actions=drop
```

##### 入站流量

入站网络数据的传输

- 从外部网络发送到实例的数据包会首先到达 *eth1*，然后到达 *br-ex*。
- 从 br-ex 上，数据包通过 patch-peer `phy-br-ex <-> int-br-ex` 被移到 br-int。

当数据包到达 *int-br-ex* 时，*br-int* 内的一个 OVS flow 规则会在数级包内为 `provider-171` 添加 VLAN tag 2，为 `provider-172` 添加 VLAN tag 3，当内部的 VLAN tag 被加入到数据包后，qvo-xxx 就可以接受它，并在删除 VLAN tag 后把数据包转发到 qvb-xx。最后，数据包就可以到达相关的实例。

```bash
## br-int端口是18
$ ovs-ofctl show br-int
 18(int-br-ex): addr:fe:b7:cb:03:c5:c1
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
     
$ ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=6770.572s, table=0, n_packets=1239, n_bytes=127795, idle_age=106, priority=1 actions=NORMAL
 cookie=0x0, duration=3181.679s, table=0, n_packets=2605, n_bytes=246456, idle_age=0, priority=3,in_port=18,dl_vlan=172 actions=mod_vlan_vid:3,NORMAL
 cookie=0x0, duration=6353.898s, table=0, n_packets=5077, n_bytes=482582, idle_age=0, priority=3,in_port=18,dl_vlan=171 actions=mod_vlan_vid:2,NORMAL
 cookie=0x0, duration=6769.391s, table=0, n_packets=22301, n_bytes=2013101, idle_age=0, priority=2,in_port=18 actions=drop
 cookie=0x0, duration=6770.463s, table=23, n_packets=0, n_bytes=0, idle_age=6770, priority=0 actions=drop
```

第 2 个规则表示，一个带有 VLAN tag 172（`dl_vlan=172`）的数据包到达 int-br-ex（`in_port=18`），把 VLAN tag 替换为 3（`actions=mod_vlan_vid:3,NORMAL`）并转发。第 3 个规则表示，一个带有 VLAN tag 171（`dl_vlan=171`）的数据包到达 int-br-ex（`in_port=18`），把 VLAN tag 替换为 2（`actions=mod_vlan_vid:2,NORMAL`）并转发。







### 引用

1. https://www.cnblogs.com/boshen-hzb/p/9870345.html
1. https://blog.51cto.com/u_15642673/5299348
1. https://www.bookstack.cn/read/openstack_understand_Neutron/dvr-compute_node.md#%E5%8D%97%E5%8C%97%E6%B5%81%E9%87%8F
1. https://zhuanlan.zhihu.com/p/273421961?utm_campaign=shareopn&utm_medium=social&utm_oi=982533992406560768&utm_psn=1549022408904011777&utm_source=wechat_session
1. https://blog.51cto.com/arkling/2409789
1. https://www.bookstack.cn/read/openstack_understand_Neutron/dvr-compute_node.md#%E5%8D%97%E5%8C%97%E6%B5%81%E9%87%8F
1. https://blog.51cto.com/arkling/2406590
1. https://www.xjimmy.com/openstack-5min-141.html