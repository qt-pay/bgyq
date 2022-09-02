## VxLAN 应用： Openstack

Openstack的租户网络使用VxLAN，可以提供1600w，满足各种需求，主机层面仍使用vlan即可，因为一个host上不可能跑4000个虚拟机。

### 实现不同租户隔离的是vlan或vxlan

不同的虚拟网络是以vlan/vxlan id区分的，而不是3层的网络cidr。

不同租户可以创建相同的网络，网络隔离本质上还是ovs流表做的。

#### VLAN多租户

VLAN创建Network时，要手动指定VLAN tag。

VLAN模型引入了多租户机制，虚拟机可以使用不同的私有IP网段，一个租户可以拥有多个IP网段。虚拟机IP通过DHCP消息获取IP地址（Nova-network实现为nova-network主机中的dnsmaq，Neutron实现为网络节点上的dhcp-agent）。网段内部虚拟机间的通信直接通过HyperVisor中的网桥转发，同一租户跨网段通信通过网关路由，不同租户通过网关上的ACL进行隔离，公网流量在该网段的网关上进行NAT（Nova-network实现为开启nova-network主机内核的iptables，Neutron实现为网络节点上的l3-agent）**。如果不同租户逻辑上共用一个网关，则无法实现租户间IP地址的复用。**

#### VxLAN多租户

VxLAN创建Network时，从Neutron配置vxlan区间中，自动递增。

Overlay模型（包括GRE和VxlAN隧道技术），相比于VLAN模型有以下改进。

1）租户数量从4K增加到16million；

2）租户内部通信可以跨越任意IP网络，支持虚拟机任意迁移；

3）一般来说每个租户逻辑上都有一个网关实例，IP地址可以在租户间进行复用；

4）能够结合SDN技术对流量进行优化。

### VLAN on Openstack

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vlan-demo.jpg)



vlan模式虚拟机通信需要经过的所有端口，数据流向如下：

- 数据帧从VM出来，经过TAP提供的虚拟网口vNIC，再经过Linux网桥qrb安全验证，走到qvm，会打上一个内部的VLANtag，成为主机节点内部的local id。这个id的作用是区分同一个主机内部的不同VM;

- 继续南下到ply，资料显示作用是过滤掉非本机的MAC，主要作用是为了方便访问同一个主机的其他VM，如果目的源是同一台主机则直接访问，不用br-int转发;

- 继续走到br-int会实现转发到目的帧主机，南下到patch port（一对patch-port连接br-int和br-eth），br-eth会把帧中间的之前打的内部VLAN ID即local id删除，换成外部的VLAN ID;

  eg：设置 br-eth1 中的 flow rules。从 patch port 进入的数据帧，将内部 VLAN ID 1 修改为 101，内部 VLAN ID 2 修改为 102，再从 eth1 端口发出。对从 eth1 进入的数据帧做相反的处理。

- 之后帧走到实际的外部物理交换机网口，发送到目的地

  物理交换机与eth网卡相连的 port 设置成 trunk 模式，实现同一块物理网卡上面通过多个不同vlan 的数据。

#### vlan tag 转换

vlan转换是因为br-int上的vlan tag是做宿主机内部不同租户网络隔离用的，这个Tag只在这个宿主机内部有效。

Tenant在创建private network时，需要分配一个vlanid，此vlanid是全局的。

举个例子，Tenant 在A主机上的打的标签是tag1，在B主机上打的标签是tag2， tag1在访问B上的虚拟机时，需要将tag1转换成tag2. 这个转换需要用到全局vlanid

A主机上，数据帧从br-int经过br-eth1然后通过网口发出， tag1->vlanid1   B主机接收到vlanid1的数据帧，转换为tag2发送到br-int上， vlanid1->tag2 ， 完成转换。A主机上标签为TAG1的VM，能够与B主机上标签为TAG2的VM通信。


如果是GRE模式，也有这个标识网络的全局ID，segmentID

#### 带vlan tag的流量

默认br-int 只能接收虚机发出的网络包没有加 vlan tag，也就是 untagged。

在某些场景中，虚机发出的包需要有 vlan tag。 Neutron 的 BP https://blueprints.launchpad.net/neutron/+spec/vlan-aware-vms 实现了一种方案，而且代码已经合并。『This blueprint proposes how to incorporate VLAN aware VM:s into OpenStack. In this document a VLAN aware VM is a VM that sends and receives VLAN tagged frames over its vNIC.』

### VxLAN on Openstack

####  VNI 和 VID 转换意义: vlan类似

在openstack中，尽管br-tun上的vni数量增多，但br-int上的网络类型只能是vlan，所有vm都有一个内外vid（vni）转换的过程，将用户层的vni转换为本地层的vid。
尽管br-tun上vni的数量为16777216个，但br-int上vid只有4096个，那引入vxlan是否有意义？

答案是肯定的，以目前的物理机计算能力来说，假设每个vm属于不同的tenant，1台物理机上也不可能运行4094个vm，所以这么映射是有意义的。

**一个host上的vm可能属于不同的vlan. openstack 同一个vxlan下可能对应不同了vlan.**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vxlan-vlan.png)



有的vm属于同一个tenant，即在用户层vni一致（同一个vxlan），但在本地层，同一tenant由nova-compute分配的vid可以不一致，同一宿主机上同一tenant的相同subnet(相同VLAN)之间的vm相互访问不需要经过内外vid（vni）转换，不同宿主机上相同tenant的vm之间相互访问则需要经过vid（vni）转换。**如果所有宿主机上vid和vni对应关系一致，整个云环境最多只能有4094个tenant，引入vxlan才真的没有意义。**

这就是，Openstack提出的层次化绑定即租户使用配置的静态vni，vm在host则动态分配vlan

#### br-tun：6

Neutron如何做VID的转换：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-br-tun.jpg)

把br-tun一分为二，设想为两部分：

* 上层是VTEP，对VXLAN隧道进行了终结；

* 下层是一个普通的VLAN Bridge。

所以，对于Host来说，它有两重网络，如图所示，虚线以上是VXLAN网络，虚线以下是VLAN网络。如此一来，VXLAN内外VID的转换则变成了不得不做的工作，因为它不仅仅是表面上看起的那样，仅仅是VID数值的转变，而且背后还蕴含着网络类型的转变。

**VLAN类型的网络并不存在VXLAN这样的问题**。当Host遇到VLAN时，它并不会变成两重网络，可为什么也要做内外VID的转换呢？这主要是为了避免内部VLAN ID的冲突。内部VLAN ID是体现在br-int上的，而每一个Host内装有一个br-int，也就是说VLAN和VXLAN是共用一个br-int。假设VLAN网络不做内外VID的转换，则很可能引发br-int上的内部VLAN ID冲突。

如下图所示，Openstack 有两个类型的租户网络一个是VLAN类型，一个是VXLAN类型的网络，但是host上都只有由br-int提供的VLAN类型网络，所以必然发生VID转化。当不由Neutron同一管理时，可能发生冲突。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vlan-and-vxlan.png)

VXLAN做内外VID转换时，并不知道VLAN的外部VID是什么，所以它就根据自己的算法将内部VID转换为100，结果很不幸中枪，与VLAN网络的外部VID相等。**因为VLAN的内部VID没有做转换，仍然是等于外部VID**，所以两者的内部VID在br-int上产生了冲突。正是这个原因，所以VLAN类型的网络，也要做内外VID的转换，而且是所有网络类型都需要做内外VID的转换。这样的话Neutron就能统一掌控，从而避免内部VID的冲突。

##### why need turn VID？

原因1： 满足租户自定义网络需求

vlan 网络也得转换，因为要对租户提供可自定义的VLAN网络，如果租户想创建自定义创建VLAN ID为`111`的网络，他提交了申请单，但是Openstack集群创建的时候VLAN网络模式没有预留`VLAN 111`，怎么办，这不是没有办法满足客户自定义的要求了吗？

这时，就需要VID 的转换了。管理员可以根据租户申请单，给租户创建`VLAN 111`网络，将这个`VLAN 111`映射到Openstack集群预留的`VLAN 217`，即可满足租户自定义网络的需求。

原因2： 避免VxLAN转换VLAN ID时冲突

假如`VxLAN VNI 1000`在br-tun时，转换时转换成了`VLAN 100`，这时候不巧的是underlay网络正好有`Vlan 100`这时候数据包怎么传输？？？？







#### VTEP Trunk网络规划

公有云架构中，vtep角色通过宿主机上的ovs实现，宿主机上联至接入交换机的接口类型为trunk，在物理网络中为vtep专门规划出一个网络平面。

vm在经过vtep时，通过流表规则，去除vid（Ethernet和vlan两种模式），添加上vni.

交换机Trunk互联，VLAN Trunk是一种网络设备间的point-to-point的连接。一条VLAN Trunk线路可以同时传输多个甚至所有的VLAN数据。

```bash
[SW1] interface gigabitEthernet0/0/24
[SW1-GigabitEthernet0/0/24] port link-type trunk
[SW1-GigabitEthernet0/0/24] port trunk allow-pass vlan 10 20
```

通过VLAN Trunk port，两个交换机之间不论需要连接多少个VLAN，只需要一个VLAN Trunk连接（一对Trunk port）即可。

如果有多个交换机需要连接，通常会用另外一个交换机连接它们，如下图所示，交换机之间都通过Trunk port相连。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/swtich-trunk.jpeg)

有两个交换机，各有两个VLAN，为了让VLAN能正常工作，将VLAN1和VLAN2分别连接起来。如果是两个交换机，两个VLAN还好，如果是100个VLAN，那么这里需要有100条线路，200个交换机端口。这么连接可以吗？可以，只要有钱，因为这里的线路和交换机端口都是钱。这种方法，运维和成本支出比较大，不实用。



### 层次和非层次化VxLAN

华三的定义：

层次化VxLAN VPC组网与非层次化VxLAN VPC组网实现方式相同，区别在于非层次化VxLAN VPC组网中二层网络可实现VxLAN与VLAN之间的1:1映射，最多仅支持4K个网络。

而**层次化VxLAN（overlay vxlan）** VPC组网的二层网络，VxLAN最多可支持1600万个，且实现了层次化多出口端口绑定。

#### openstack vxlan层次化绑定：突破4k vlan限制

Neutron ML2 先调用到物理交换机对 应的 Mechanism driver 进行端口绑定（port binding），将 VxLAN A 与网络接口进 行绑定。

ML2 因为知道了网络接口还需要绑定到对应的 VLAN 上，再调用 OpenVSwitch 的 Mechanism Driver，将 VLAN B 与网络接口进行绑定。

分配动态 的 VLAN 和静态的 VxLAN 过程为：使用两个 ML2 Driver 基于 port 为计算节点 进行多级绑定；

第一个 switch 类型的 driver（ml2_h3c driver）使 port 绑定上层的 静态 VXLAN VNI；

第二个 hypervisor 类型的 driver（openvswitch driver）使 port 绑定底层动态分配的 VLAN tag。

将动态的 VLAN 映射到静态的 VxLAN 过程为： LLDP 配置及作用；

计算节点和 Leaf 开启 LLDP 服务，计算节点 LLDP 报文携带 主机名信息至 Leaf；

Leaf 将 LLDP 报文上送控制器；控制器记录计算节点、Leaf 设备和端口之间的对应关系；

VLAN-VXLAN 映射下发(虚拟网络，第一个 port 上 线)；

Neutron 通过两个 ML2 Driver 下发 VxLAN 和 VLAN，并通告 SDN控制器；

虚拟机上线，Leaf 将上线信息通告给对应的模块；向 Leaf 下发 VLAN-VxLAN 映射， 此时完成层次化端口映射。



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-layer-vxlan.png)

步骤如下：

1. 用户用Nova boot创建一个虚拟机， vm 的网络设定为VXLAN A；

2. Neutron创建一个VXLAN A网络接口，并将请求发送到ML2组件；

3. Neutron ML2先调用物理交换机(TOR)的Mechanism Driver进行端口绑定（port binding），将VxLAN A 绑定到物理交换机的网络接口 ；

4. 物理交换机Mechanism Driver**再申请一个VLAN B并通知ML2**，告诉ML2当前这个VM的网络接口还需要绑定VLAN B

5. 物理交换机Mechanism Driver通过Netconf接口告诉物理交换机设定VLAN B和VXLAN A的映射关系；

6. ML2知道网络接口还需要绑定到对应的VLAN上，所以ML2调用OVS的Mechanism Driver，在OVS添加VLAN B，并将该VLAN配置到VM对应的接口上，

7. OVS的Mechanism Driver会通过相应的API，告知位于计算节点的OpenVSwitch，OVS将对VM发出的数据包打上TAG=VLAN B 并转发到物理交换机的接口，物理交换机将带有TAG=VLAN B的数据包转换为TAG=VXLAN A的数据包；

   > br-int上不可能超过4094个vms--

-  层次化端口绑定的逻辑，一半是在Neutron ML2里面，有另一半是在物理交换机对应的Mechanism driver里面。
-  硬件交换机与服务器要运行LLDP之类的协议，来获取连接关系

相关配置

```bash
# 将/etc/neutron/plugins/ml2/ml2_conf.ini修改为
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider,net172_16
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
#vxlan的ID范围,最多到1677万
#值若给的太大,重启会特别慢,而且可能会报错
vni_ranges = 1:100000
[securitygroup]
enable_ipset = True

```

end

**OpenStack层次化端口绑定在 SDN中host和交换机如何协同？**

**虚拟机**

1、计算节点开启lldp服务，从underlay nic接口发送携带主机名的lldp报文，作为VTEP设备的交换机接收后上报控制器，控制器就记录计算节点所属的交换机端口，

当计算节点启动虚机的时候，虚机的vport中会携带主机名(计算节点的名字，也就是lldp报文中携带的主机名)，控制器就知道了这个虚拟机的报文将进入交换机的哪个端口。

2、Openstack创建port在bind_port时，根据port所属计算节点动态分配一个这个计算节点的physical network的vlan，然后赋值给框架的next_segments_to_bind以便框架进行openvswitch vlan绑定的绑定，如果这个port是这个这个网络的第一个port的话，会通知这个计算节点添加vlan和vxlan的映射关系。

3、虚机上线时，上线信息上报给控制器VSM模块，VSM模块再给NEM模块上报激活事件，使得NEM向交换机下发vlan和vxlan映射关系。

 **裸金属**

裸金属通过nuetron传递的local_link_information来感知服务器连接交换机的端口。在inspect阶段，裸金属用的deploy镜像里会包含lldp服务，ironic-agent会收集lldp信息，告知ironic。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/665372-20200216170045491-999112139.png)



 host对不同租户的虚拟机的报文 Tag 不同的VLAN ，host连接到交换机的port设为Trunk口，这样host可以把不同VLAN Tag的网络数据送到交换机vetp。vetp根据VLAN Tag映射到不同的vxlan tag，进而封装成相应的VxLAN数据。

 ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/665372-20200507152504620-1651570246.png)

##### why Hierarchical Port Binding

https://specs.openstack.org/openstack/neutron-specs/specs/kilo/ml2-hierarchical-port-binding.html

The ML2 plugin does not adequately support hierarchical network topologies. A hierarchical virtual network uses different network segments, potentially of different types (VLAN, VXLAN, GRE, proprietary fabric, …), at different levels. It might be made up of one or more top-level static network segments along with dynamically allocated network segments at lower levels. 

For example, virtual network traffic between ToR and core switches could be encapsulated using VXLAN segments, while traffic for those same virtual networks between the ToR switches and the compute nodes could use dynamically allocated VLANs segments.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-vxlan-bind-vlan.png)

Dynamically allocating segments at lower levels of the hierarchy is particularly important in allowing Neutron deployments to scale beyond the 4K limit per physical network for VLANs. VLAN allocations can be managed at lower levels of the hierarchy, allowing many more than 4K virtual networks to exist and be accessible to compute nodes as VLANs, as long as each link from ToR switch to compute node needs no more than 4K VLANs simultaneously

### 引用

1. https://www.codeleading.com/article/28243150268/
2. https://www.cnblogs.com/sammyliu/p/4626419.html