## OVN

### OVN and Geneve

Generic Network Virtualization Encapsulation的简称，对应中文是通用网络虚拟化封装，由IETF草案定义。

OpenVSwitch的衍生项目OVN（Open Virtual Network）应该是GENEVE的最大拥趸。

OVN只支持GENEVE和STT作为网络虚拟化协议。这是因为OVN除了24bit的VNI之外，还要在overlay数据中传输15bit的源网络端口，和16bit的目的网络端口，以支持更高效的ACL和组播。GENEVE因为是可扩展的，自然是支持传递额外的元数据。STT因为本身的元数据是64bit的，也放得下OVN想要传递的内容。至于其他的协议，例如VXLAN，NVGRE，是没有可能满足OVN的需求。

GENEVE能很好的兼容VXLAN，因为就算是VXLAN的主场，GENEVE最后还是赢了。但是兼容性并不能解释最后的现象，文章本身也没有分析原因，只是提到了UDP checksum。OVN默认打开了GENEVE上的UDP checksum。因为Linux系统内核的一些优化，使得GENEVE数据包被网卡收到之后，网卡会计算并验证外层UDP的checksum。如果验证通过了，网卡会汇报给系统内核。这样系统内核在解析GENEVE时，将不再计算内层报文的任何checksum。相应的网络数据处理会更快一些。而VXLAN协议规定外层UDP的checksum应该为0，这样外层UDP的checksum就没有办法被验证，而内层报文的checksum需要再计算一遍，相应的网络数据处理就要慢一点。

### OVN Gateway routers 

A gateway router is a logical router that is **bound** to a physical location.

网关路由和逻辑路由都是通过`ovn-nbctl lr-add`添加的，只是网关路由绑定了physical location（chassis）

 Gateway routers are typically used in between distributed  logical  routers  and  physical networks.  The distributed logical router and the logical switches behind it, to which VMs and containers attach, effectively reside on each hypervisor. The **distributed  router**  and the  **gateway  router**  are  connected by another logical switch, sometimes referred to as **a join logical switch.** 

  When  using  gateway  routers, DNAT and SNAT rules are associated with the gateway router, which provides a central location that can handle one-to-many SNAT (aka IP masquerading).

```bash
         _________ 
        |  client | 172.18.1.10/16 Physical Network
         ---------
         ____|____ 
        |  switch | outside
         ---------
             |
         ____|____ 
        |  router | gw1 port 'gw1-outside': 172.18.1.2/16
         ---------      port 'gw1-join':    192.168.255.1/24
         ____|____ 
        |  switch | join  192.168.255.0/24 
         ---------  
         ____|____ 
        |  router | router1 port 'router1-join':  192.168.255.2/24
         ---------          port 'router1-ls1': 192.168.100.1/24
             |
         ____|____ 
        |  switch | ls1 192.168.100.0/24
         ---------  
         /       \
 _______/_       _\_______  
|  vm1    |     |   vm2   |
 ---------       ---------
192.168.100.10  192.168.100.11
```



### OVN distributed  logical  routers  

a distributed logical router (DLR), 分布在每个ovn host上，负责该host上different vswitch的连通。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpc-混合overlay-流量模型.jpg)

列举几种流量：

* cvk 1 上 vm 1 访问 cvk 1 上 vm 3：vm 1和vm 3在不用的vSwitch（不同vni）上，需要借助VPC下Router转发流量，但是此时还是L2 VxLAN数据包

  即cvk 1上的Router已经知道了去往vm3的下一跳了、

* cvk 1 上 vm 1 访问 cvk 2 上vm2： vm 1和vm 2不在同一个宿主机上，但是在同一个Switch上（同一个vni），流量不经过Router，由两个Switch 所在的节点 cvk 1 和 cvk 2建立的Tunnel传输，下一跳就是vm所在cvk 节点IP，这也是L2 VxLAN数据包

  PS：宿主机上只要有某个Switch下的一个vm，就会生产一个Switch，并知道整个Switch的路由信息。

  即cvk 1上的Router已经知道了去往vm2的下一跳了、

* cvk 1 上 vm 1 访问 cvk 2 上vm4：vm 1和vm 4不在同一个宿主机上，也不在同一个Switch上（不同vni），流量要经过Router转发，确定vm4所在Switch2的cvk节点，然后通过两个CVK建立的Tunnel传输L2 VxLAN数据包。

  **PS: 三层转发已经在CVK 1上通过Router 1完成了，vm 1 和 vm 2通讯时，只需两个switch 建立二层VxLAN隧道即可。**

  即cvk 1上的Router已经知道了去往vm4的下一跳了、

* CVK 1上 vm 1 访问网络overlay下CVK N上的vm 5： vm 5 在 Switch 2上，由于CVK 1上有vm3，所以可以看到Switch 2上的全部路由信息。VM 1 访问 VM 5 的流量就是VM 1先到网关Switch 1 再到路由器 Router 1（因为跨vni，跨子网了），然后到达Switch 2，在Switch 2上查到VM的下一跳是网络Overlay物理Leaf设备，然后下一跳到达Leaf，Leaf对Overlay解封装(应该是VxLAN协议了)，将数据转发给VM5。因为没过GWSwitch还是L2 VxLAN信息。

  PS：如果VM 5即不在switch 1 和 switch2 就要走GWSwitch了。

### OVN：as k8s L2 or L3 drive

You can use the `ovn-nbctl` utility to see an overview of the logical topology.

The `ovn-sbctl` utility can be used to see into the state stored in the `OVN_Southbound` database. 

Where Open vSwitch (OVS) provides a virtual switch on a single host, OVN extends this abstraction to span multiple hosts. You can create virtual switches that span many physical nodes, and OVN will take care of creating overlay networks to support this abstraction. While OVS is primarily just a layer 2 device, OVN also operates at layer 3: you can create virtual routers to connect your virtual networks as well a variety of access control mechanisms such as security groups and ACLs.

OVS只能处理single node 流量，而且需要手工配置。OVN是OVS的SDN controller，实现海量ovs nodes流量管理。

[OVN (Open Virtual Network)](http://openvswitch.org/support/dist-docs/ovn-architecture.7.html) 是OVS提供的原生虚拟化网络方案，旨在解决传统SDN架构（比如Neutron DVR）的性能问题。

OVS 社区觉得从长远来看，Neutron 应该让一个其它的项目来做虚拟网络的控制平面，Neutron 只需要提供 API 的处理，于是 OVS 社区推出了 OVN（Open Virtual Switch）这个项目，OVN 是 OVS 的控制平面，它给 OVS 增加了对虚拟网络的原生支持，大大提高了 OVS 在实际应用环境中的性能和规模。

**OVN是OpenvSwitch项目组为OpenvSwitch开发SDN控制器**，同其他SDN产品相比，OVN对OpenvSwitch 及OpenStack有更好的兼容性和性能

OVN, the Open Virtual Network, is an open source project that was originally developed by the Open vSwitch (OVS) team at Nicira.

It complements the existing capabilities of OVS and provides virtual network abstractions, such as virtual L2 and L3 overlays, security groups, and DHCP services. Just like OVS, OVN was designed to support highly scalable and production-grade implementations. It provides an open approach to virtual networking capabilities for any type of workload on a virtualised platform (virtual machines and containers) using the same API.

OVN gives admins the control over network resources by connecting groups of VMs or containers into private L2 and L3 networks, quickly, programmatically, and without the need to provision VLANs or other physical network resources.

#### OVN架构： man

https://man7.org/linux/man-pages/man7/ovn-architecture.7.html

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovn-is-sdn-controller-of-ovs.webp)

The main components of OVN can be seen either under a database or a daemon category:

The OVN databases:

- **Ovn-northbound ovsdb** represents the OpenStack/CMS integration point and keeps an intermediate and high-level of the desired state configuration as defined in the CMS. It keeps track of QoS, NAT, and ACL settings and their parent objects (logical ports, logical switches, logical routers).
- **Ovn-southbound ovsdb** holds a more hypervisor specific representation of the network and keeps the run-time state (i.e., location of logical ports, location of physical endpoints, logical pipeline generated based on configured and run-time state).

The OVN daemons:

- **ovn-northd** converts from the high-level northbound DB to the run-time southbound DB, and generates logical flows based on high-level configurations.
- **ovn-controller** is a local SDN controller that runs on every host and manages each OVS instance. It registers chassis and VIFs to the southbound DB and converts logical flows into physical flows (i.e., VIF UUIDs to OpenFlow ports). It pushes physical configurations to the local OVS instance through OVSDB and OpenFlow and uses SDN for remote compute location (VTEP). All of the controllers are coordinated through the southbound database.

OVN operates with a pair of databases. The *Northbound* database contains the *logical* structure of your networks: this is where you define switches, routers, ports, and so on.

The *Southbound* database is concerned with the *physical* structure of your network. This database maintains information about which ports are realized on which hosts.

The `ovn-northd` service “translates the logical network configuration in terms of conventional network concepts, taken from the OVN North‐ bound Database, into logical datapath flows in the OVN Southbound Database below it.”

The `ovn-controller` service running on each host connects to the Southbound database and is responsible for configuring OVS as instructed by the database configuration.

OVN逻辑流表会由ovn-northd分发给每台机器的ovn-controller，然后ovn-controller再把它们转换为物理流表。 

```
                                        CMS
                                          |
                                          |
                              +-----------|-----------+
                              |           |           |
                              |     OVN/CMS Plugin    |
                              |           |           |
                              |           |           |
                              |   OVN Northbound DB   |
                              |           |           |
                              |           |           |
                              |       ovn-northd      |
                              |           |           |
                              +-----------|-----------+
                                          |
                                          |
                                +-------------------+
                                | OVN Southbound DB |
                                +-------------------+
                                          |
                                          |
                       +------------------+------------------+
                       |                  |                  |
         HV 1          |                  |    HV n          |
       +---------------|---------------+  .  +---------------|---------------+
       |               |               |  .  |               |               |
       |        ovn-controller         |  .  |        ovn-controller         |
       |         |          |          |  .  |         |          |          |
       |         |          |          |     |         |          |          |
       |  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
       |                               |     |                               |
       +-------------------------------+     +-------------------------------+
```

从图表的最上面开始，主要有：

- **云管系统(Cloud Management System)**: 即最上面的部分

- **OVN/CMS 插件（Plugin）**：插件是用作OVN接口的CMS组件。在`OpenStack`中，就是一个`Neutron插件`。
  插件的主要目的就是将CMS的逻辑网络配置概念（以CMS可识别的特定格式存储在CMS的配置数据库中），转换为OVN可以理解的中间格式。

  此组件必须是CMS特定的，因此，需要为每个与OVN集成的CMS开发一个新插件。上图中此组件下面的所有组件都是独立于CMS的。

- **OVN北向数据库（Northbound Database）**：接收OVN/CMS插件传递的逻辑网络配置的中间表示。
  数据库模式与CMS中使用的概念`阻抗匹配（impedance matched）`，因此它直接支持逻辑交换机、路由器、ACL等概念。详见ovn nb。

  OVN北向数据库有两个客户端：它上面的`OVN/CMS插件`还有它下面的`ovn-northd`

- **ovn-northd**: 它上连OVN北向数据库，下连OVN南向数据库。
  它将从北向数据库中获得的传统网络概念中的逻辑网络配置，转换为它下面的南向数据库所能理解的`逻辑数据路径流`（logical datapath flow）。

- **OVN南向数据库（Southbound Database）**：这是本系统的中心。
  它的客户端(client)包括其上的ovn-northd，以及其下每个传输节点上的`ovn-controller`。

  OVN南向数据库包含三种数据：

  - **物理网络(Physical Network (PN))表**：指明如何访问hypervisor和其他节点
  - **逻辑网络(Logical Network (LN))表**：用`逻辑数据路径流（logical datapath flows）`来描述逻辑网络，
  - **绑定表**：将逻辑网络组件的位置链接到物理网络

  OVN南向数据库的性能必须随着传输节点的变化而变化。这就需要在遇到瓶颈的时候在`ovsdb-server`上进行一些工作。
  可能需要引入集群来提升可用性。

其余组件将被复制到每个虚拟机监控程序（hypervisor）上：

- **ovn-controller** ：是每个hypervisor和软件网关上的agent。
  - 北向而言，它连接到南向数据库来获取OVN的配置及状态，并用hypervisor的状态填充绑定表中的PN表和Chassis列。
  - 南向而言，它作为OpenFlow controller连接到 ovs-vswitchd 来控制网络流量 ，
    并连接到本地`ovsdb-server`来监控和控制Open vSwitch的配置。
- **ovs-vswitchd 和 ovsdb-server**：Open vSwitch的传统组件。

OVN引入了两个全新的OVSDB，

- 一个叫Northbound DB（北向数据库，NB），
- 一个叫Southbound DB（南向数据库，SB）

##### CMS（云管理系统）

CMS : Cloud Management System ???   

这是OVN的最终用户（通过其用户和管理员）。与OVN集成需要安装与CMS特定的插件和相关软件。OVN最初的目标CMS是OpenStack。我们通常会说一个CMS，但也可能出现多个CMS也可以管理一个OVN的不同部分。

##### OVN/CMS插件: adaptor

Plugin是连接到OVN的CMS组件。在OpenStack中，**这是一个Neutron插件**。该插件的主要目的是转换CMS中的逻辑网络的配置为OVN可以理解的中间表示。这个组件是必须是CMS特定的，所以对接一个新的CMS需要开发新的插件对接到OVN。所有在这个组件下面的其他组件是与CMS无关的。

从 OVN 的架构可以看出，OVN 里面数据的读写都是通过`OVSDB`来做的，**取代了** Neutron 的消息队列机制，所以有了 OVN 之后，Neutron 里面所有的 agent 都不需要了，Neutron 变成了一个 API server 来处理用户的 REST 请求，其他的功能都交给 OVN 来做，只需要在 Neutron 里面加一个 plugin 来调用配置 OVN。

Neutron 里面的子项目`networking-ovn`就是实现 OVN 的 plugin，取代了之前的python agent。Plugin 使用 OVSDB 协议来把用户的配置写在 Northbound DB 里，ovn-northd 监听到 Northbound DB 配置发生改变，然后把配置翻译到 Southbound DB 里面。 ovn-controller 监控到 Southbound DB 数据的发生变化之后，进而更新本地的流。

##### northbound database

ovn-nb - OVN_Northbound database schema

This database is the interface between OVN and the cloud management system (CMS), such as OpenStack, running above it. The CMS produces almost all of the contents of the database. The ovn-northd program monitors the database contents, transforms it, and stores it into the OVN_Southbound database.



存储逻辑交换机、路由器、ACL、端口等的信息，目前基于ovsdb-server，未来可能会支持etcd v3。类似 apiserver，提供了一组高层次的网络抽象，这样在真正创建网络资源时无需关心负责的逻辑规则，只需要通过 Northoboud DB 的接口创建对应实体即可。

Northbound DB 里面存的都是一些逻辑的数据，大部分和物理网络没有关系，比如 logical switch，logical router，ACL，logical port，和传统网络设备概念一致。

> 开放虚拟交换机数据库（OpenvSwitch Database,OVSDB）是开放虚拟交换机中保存的各种配置信息（如网桥、端口）的数据库，是针对OpenvSwitch开发的轻量级数据库。

> OVSDB是一个轻量级的数据库，其实它只是一个JSON文件，默认路径为/etc/openvswitch/conf.db。记录的网桥、端口、QOS等网络配置信息是以JSON格式（schema）保存的，通常schema在/usr/share/openvswitch/vswitch.ovsschema中。

##### southbound database

类似于etcd(不太准确)，存储集群视角下的逻辑规则。

基于ovsdb-server（未来可能会支持etcd v3），包含三类数据

- 物理网络数据，比如VM的IP地址和隧道封装格式
- 逻辑网络数据，比如报文转发方式
- 物理网络和逻辑网络的绑定关系，比如逻辑端口关联到哪个 HV 上面。

##### ovn-northd

ovn-northd - Open Virtual Network central control daemon。

集中式控制器，负责把northbound database数据分发到各个ovn-controller

OVN-northd 类似于一个集中的控制器，它把 Northbound DB 里面的数据翻译一下，写到 Southbound DB 里面。

ovn-northd is a centralized daemon responsible for translating the high-level OVN configuration into logical configuration consumable by daemons such as ovn-controller. It translates the logical network configuration in terms of conventional network concepts, taken from the OVN Northbound Database (see ovn-nb(5)),into logical datapath flows in the OVN Southbound Database (see ovn-sb(5)) below it.



##### ovn-controller：类似kubelet

ovn-controller - Open Virtual Network local controller

ovn-controller is the local controller daemon for OVN, the Open Virtual Network. It connects up to the OVN Southbound database over the OVSDB protocol, and down to the Open vSwitch database  over the OVSDB protocol and to ovs-vswitchd(8) via OpenFlow. **Each hypervisor and software gateway in an OVN deployment runs its own independent copy of ovn-controller**; thus, ovn-controller’s downward connections are machine-local and do not run over a physical network.

ovn-controller是每个hypervisor和软件网关上的OVN代理。北向，它连接到OVN南行数据库以了解OVN配置和状态，并把hypervisor的状态填充绑定表中的Chassis列以及PN表。南向，它连接到ovs-vswitchd作为OpenFlow控制器用于控制网络通信，并连接到本地ovsdb-server以允许它监视和控制Open vSwitch的配置。

ovs-vswitchd和ovsdb-server是标准的Open vSwitch组件。

运行在每台机器上的本地SDN控制器，类似于kubelet，负责和中心控制节点通信获取整个集群的网络信息，并更新本机的流量规则。ovs-vswitchd 和 ovsdb-server 可以理解为单机的docker 负责单机虚拟网络的真实操作。

ovn-controller 是 OVN 里面的 agent，类似于 neutron 里面的 ovs-agent，它也是运行在每个 HV (Hypervisor)上面，北向，ovn-controller 会把物理网络的信息写到 Southbound DB 里面，南向，它会把 Southbound DB 里面存的一些数据转化成 Openflow flow 配到本地的 OVS table 里面，来实现报文的转发。

ovn-controller是每个hypervisor和软件网关上的OVN代理。

- 北向，它连接到OVN南行数据库以了解OVN配置和状态，并把hypervisor的状态填充绑定表中的Chassis列以及PN表。
- 南向，它连接到ovs-vswitchd作为OpenFlow控制器用于控制网络通信，并连接到本地ovsdb-server以允许它监视和控制Open vSwitch的配置。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovn-arch.jpg)

补充说明

- Data Path：OVN的实现简单有效，都是基于OVS原生的功能特性来做的（由于OVN的实现不依赖于内核特性，这些功能在OVS+DPDK上也完全支持），比如
  - security policies 基于 OVS+conntrack 实现
  - 分布式L3路由基于OVS flow实现
- Logical Flows：逻辑流表，会由ovn-northd分发给每台机器的ovn-controller，然后ovn-controller再把它们转换为物理流表

**ovn-controller** is a local SDN controller that runs on every host and manages each OVS instance. It registers chassis and VIFs to the southbound DB and converts logical flows into physical flows (i.e., VIF UUIDs to OpenFlow ports). It pushes physical configurations to the local OVS instance through OVSDB and OpenFlow and uses SDN for remote compute location (VTEP). All of the controllers are coordinated through the southbound database.

```bash
                                  CMS
                                   |
                                   |
                       +-----------|-----------+
                       |           |           |
                       |     OVN/CMS Plugin    |
                       |           |           |
                       |           |           |
                       |   OVN Northbound DB   |
                       |           |           |
                       |           |           |
                       |       ovn-northd      |
                       |           |           |
                       +-----------|-----------+
                                   |
                                   |
                         +-------------------+
                         | OVN Southbound DB |
                         +-------------------+
                                   |
                                   |
                +------------------+------------------+
                |                  |                  |
 HV 1           |                  |    HV n          |
+---------------|---------------+  .  +---------------|---------------+
|               |               |  .  |               |               |
|        ovn-controller         |  .  |        ovn-controller         |
|         |          |          |  .  |         |          |          |
|         |          |          |     |         |          |          |
|  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
|                               |     |                               |
+-------------------------------+     +-------------------------------+

```

end

##### HV

HV是Hypervisor ...

##### VIF生命周期

https://www.dounaite.com/article/62cfff19f4ab41be487d8a7a.html

hypervisor上的VIF是连接到在该hypervisor上直接运行的虚拟机或容器的虚拟网络接口（这与运行在虚拟机内的容器的接口不同）

1. 当CMS管理员使用CMS用户界面或API创建新的VIF并将其添加到交换机（由OVN作为逻辑交换机实现的交换机）时，VIF的生命周期开始。CMS更新其自己的配置，主要包括将VIF唯一的持久标识符vif-id和以太网地址mac相关联。
2. CMS插件通过向Logical_Switch_Port表添加一行来更新OVN北向数据库以包括新的VIF信息。在新行中，名称是vif-id，mac是mac，交换机指向OVN逻辑交换机的Logical_Switch记录，而其他列被适当地初始化。
3. ovn-northd收到OVN北向数据库的更新。然后通过添加新行到OVN南向数据库Logical_Flow表中反映新的端口来对OVN南向数据库进行相应的更新，例如，添加一个流来识别发往新端口的MAC地址的数据包应该传递给它，并且更新传递广播和多播数据包的流以包括新的端口。它还在“绑定”表中创建一个记录，并填充除识别chassis列之外的所有列。
4. 在每个hypervisor上，ovn-controller接收上一步ovn-northd在Logical_Flow表所做的更新。但只要拥有VIF的虚拟机关机，ovn-controller也无能为力。例如，它不能发送数据包或从VIF接收数据包，因为VIF实际上并不存在于任何地方。
5. 最终，用户启动拥有该VIF的VM。在VM启动的hypervisor上，hypervisor与Open vSwitch（IntegrationGuide.rst中所描述的）之间的集成是通过将VIF添加到OVN集成网桥上，并在external_ids：iface-id中存储vif-id，以指示该接口是新VIF的实例。（这些代码在OVN中都不是新功能;这是已经在支持OVS的虚拟机管理程序上已经完成的预先集成工作。）
6. 在启动VM的hypervisor上，ovn-controller在新的接口中注意到external_ids：iface-id。作为响应，在OVN南向数据库中，它将更新绑定表的chassis列中链接逻辑端口从external_ids：iface-id到hypervisor的行。之后，ovn-controller更新本地虚拟机hypervisor的OpenFlow表，以便正确处理去往和来自VIF的数据包。
7. 一些CMS系统，包括OpenStack，只有在网络准备就绪的情况下才能完全启动虚拟机。为了支持这个功能，ovn-northd发现Binding表中的chassis列的某行更新了，则通过更新OVN北向数据库的Logical_Switch_Port表中的up列来向上指示这个变化，以指示VIF现在已经启动。如果使用此功能，CMS则可以通过允许VM执行继续进行后续的反应。
8. 在VIF所在的每个hypervisor上，ovn-controller注意到绑定表中完全填充的行。这为ovn-controller提供了逻辑端口的物理位置，因此每个实例都会更新其交换机的OpenFlow流表（基于OVN数据库Logical_Flow表中的逻辑数据路径流），以便通过隧道正确处理去往和来自VIF的数据包。
9. 最终，用户关闭拥有该VIF的VM。在VM关闭的hypervisor中，VIF将从OVN集成网桥中删除。
10. 在VM关闭的hypervisor上，ovn-controller注意到VIF已被删除。作为响应，它将删除逻辑端口绑定表中的Chassis列的内容。
11. 在每个hypervisor中，ovn-controller都会注意到绑定表行中空的Chassis列。这意味着ovn-controller不再知道逻辑端口的物理位置，因此每个实例都会更新其OpenFlow表以反映这一点。
12. 最终，当VIF（或其整个VM）不再被任何人需要时，管理员使用CMS用户界面或API删除VIF。CMS更新其自己的配置。
13. CMS插件通过删除Logical_Switch_Port表中的相关行来从OVN北向数据库中删除VIF。
14. ovn-northd收到OVN北向数据库的更新，然后相应地更新OVN南向数据库，方法是从OVN南向d数据库Logical_Flow表和绑定表中删除或更新与已销毁的VIF相关的行。
15. 在每个hypervisor上，ovn-controller接收在上一步中的Logical_Flow表更新并更新OpenFlow表。尽管可能没有太多要做，因为VIF已经变得无法访问，它在上一步中从绑定表中删除。

##### OVN Chassis

`Chassis` 是 `OVN` 新增的概念，`OVS` 里面没有这个概念，`Chassis` 可以是 `HV`，也可以是 `VTEP` 网关（网络硬件设备）。 `Chassis` 的信息保存在 `Southbound DB` 里面，由 `ovn-controller/ovn-controller-vtep` 来维护。

以 `ovn-controller` 为例，当 `ovn-controller` 启动的时候，它去本地的数据库 `Open_vSwitch` 表里面读取`external_ids:system_id`，`external_ids:ovn-remote`，`external_ids:ovn-encap-ip` 和`external_ids:ovn-encap-type`的值，然后它把这些值写到 `Southbound DB` 里面的表 `Chassis` 和表 `Encap` 里面：

- `external_ids:system_id`表示 `Chassis` 名字
- `external_ids:ovn-remote`表示 `Sounthbound DB` 的 `IP` 地址
- `external_ids:ovn-encap-ip`表示 `tunnel endpoint IP` 地址，可以是 `HV` 的某个接口的 `IP` 地址
- `external_ids:ovn-encap-type`表示 `tunnel` 封装类型，可以是 `VXLAN/Geneve/STT`

`external_ids:ovn-encap-ip`和`external_ids:ovn-encap-type`是一对，每个 `tunnel IP` 地址对应一个 `tunnel` 封装类型，如果 `HV` 有多个接口可以建立 `tunnel`，可以在 `ovn-controller` 启动之前，把每对值填在 `table Open_vSwitch` 里面。

##### OVN Tunnel：支持vxlan

  OVN 支持的 tunnel 类型有三种，分别是 Geneve，STT 和 VXLAN。HV 与 HV 之间的流量，只能用 Geneve 和 STT 两种，HV 和 VTEP 网关之间的流量除了用 Geneve 和 STT 外，还能用 VXLAN，这是为了兼容硬件 VTEP 网关（网络Overlay），因为大部分硬件 VTEP 网关只支持 VXLAN。虽然 VXLAN 是数据中心常用的 tunnel 技术，但是 VXLAN header 是固定的，只能传递一个 VNID（VXLAN network identifier），如果想在 tunnel 里面传递更多的信息，VXLAN 实现不了。所以 OVN 选择了 Geneve 和 STT，Geneve 的头部有个 option 字段，支持 TLV 格式，用户可以根据自己的需要进行扩展，而 STT 的头部可以传递 64-bit 的数据，比 VXLAN 的 24-bit 大很多。

  OVN tunnel 封装时使用了三种数据：

- Logical datapath identifier（逻辑的数据通道标识符）：datapath 是 OVS 里面的概念，报文需要送到 datapath 进行处理，一个 datapath 对应一个 OVN 里面的逻辑交换机或者逻辑路由器，类似于 tunnel ID。这个标识符有 24-bit，由 ovn-northd 分配的，全局唯一，保存在 Southbound DB 里面的表 Datapath_Binding 的列 tunnel_key 里。

- Logical input port identifier（逻辑的入端口标识符）：进入 logical datapath 的端口标识符，15-bit 长，由 ovn-northd 分配的，在每个 datapath 里面唯一。它可用范围是 1-32767，0 预留给内部使用。保存在 Southbound DB 里面的表 Port_Binding 的列 tunnel_key 里。

- Logical output port identifier（逻辑的出端口标识符）：出 logical datapath 的端口标识符，16-bit 长，范围 0-32767 和 logical input port identifier 含义一样，范围 32768-65535 给组播组使用。对于每个 logical port，input port identifier 和 output port identifier 相同。

  如果 tunnel 类型是 Geneve，Geneve header 里面的 VNI 字段填 logical datapath identifier，Option 字段填 logical input port identifier 和 logical output port identifier，TLV 的 class 为 0xffff，type 为 0，value 为 1-bit 0 + 15-bit logical input port identifier + 16-bit logical output port identifier。

OVS 的 tunnel 封装是由 Openflow 流表来做的，所以 ovn-controller 需要把这三个标识符写到本地 HV 的 Openflow flow table 里面，对于每个进入 br-int 的报文，都会有这三个属性，logical datapath identifier 和 logical input port identifier 在入口方向被赋值，分别存在 openflow metadata 字段和 Nicira 扩展寄存器 reg14 里面。报文经过 OVS 的 pipeline 处理后，如果需要从指定端口发出去，只需要把 Logical output port identifier 写在 Nicira 扩展寄存器 reg15 里面。

   **OVN tunnel 里面所携带的 logical input port identifier 和 logical output port identifier 可以提高流表的查找效率，OVS 流表可以通过这两个值来处理报文，不需要解析报文的字段。** OVN 里面的 tunnel 类型是由 HV 上面的 ovn-controller 来设置的，并不是由 CMS 指定的，并且 OVN 里面的 tunnel ID 又由 OVN 自己分配的，所以用 neutron 创建 network 时指定 tunnel 类型和 tunnel ID（比如 vnid）是无用的，OVN 不做处理。

##### ovn-controller-vtep

ovn-controller-vtep是可以配置支持OVSDB协议的物理交换机。

a physical switch that connects to an OVN deployment through a simple OVSDB schema.

ovn-controller-vtep retrieves its configuration information from both the ovnsb and the vtep database. 

OVN does support VXLAN for use with ASIC-based top of rack switches, using `ovn-controller-vtep(8)` and the OVSDB VTEP schema described in `vtep(5)`, but this limits the features available from OVN to the subset available from the VTEP schema.

###### vtep

vtep - hardware_vtep database schema
This schema specifies relations that a VTEP can use to integrate physical ports into logical switches maintained by a network virtualization controller such as NSX

##### Datapath: forwarding plane

https://arthurchiao.art/blog/ovs-deep-dive-3-datapath/

Datapath is the forwarding plane of OVS.

Initially, it is implemented as a kernel module, and kept as small as possible. Apart from the datapath, other components are implemented in userspace, and have little dependences with the underlying systems. That means, porting ovs to another OS or platform is simple (in concept): just porting or re-implement the kernel part to the target OS or platform. As an example of this, ovs-dpdk is just an effort to run OVS over Intel [DPDK](https://arthurchiao.art/blog/ovs-deep-dive-3-datapath/dpdk.org). For those who do, there is an official [porting guide](https://github.com/openvswitch/ovs/blob/master/Documentation/topics/porting.rst) for porting OVS to other platforms.

In fact, in recent versions (I’m not sure since which version, but according to my tests, 2.3+ support this) of OVS, there are already two type of datapath that you could choose from: **kernel datapath and userspace datapath**.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/dpif_providers.png)

which discusses OVS hardware offloading, reveals even more datapath types (enterprise solution). In this article, we only focus on kernel datapath and userspace datapath, which are provided in stock openvswitch.

**Open vSwitch supports different datapaths on different platforms[6]:**

- **Linux upstream**

  The datapath implemented by the kernel module shipped with Linux upstream. Since features have been gradually introduced into the kernel, the table mentions the first Linux release whose OVS module supports the feature.

- **Linux OVS tree**

  The datapath implemented by the Linux kernel module distributed with the OVS source tree. Some features of this module rely on functionality not available in older kernels: in this case the minumum Linux version (against which the feature can be compiled) is listed.

- **Userspace**

  Also known as DPDK, dpif-netdev or dummy datapath. It is the only datapath that works on NetBSD and FreeBSD.

- **Hyper-V**

  Also known as the Windows datapath.

###### Kernel Datapath

Here we only talk about the kernel datapath on Linux platform.

On Linux, kernel datapath is the default datapath type. It needs a kernel module `openvswitch.ko` to be loaded:

```bash
$ lsmod | grep openvswitch
openvswitch            98304  3
```

If it is not loaded, you need to install it manually:

```bash
$ find / -name openvswitch.ko
/usr/lib/modules/3.10.0-514.2.2.el7.x86_64/kernel/net/openvswitch/openvswitch.ko

$ modprobe openvswitch.ko
$ insmod /usr/lib/modules/3.10.0-514.2.2.el7.x86_64/kernel/net/openvswitch/openvswitch.ko
$ lsmod | grep openvswitch
```

Creating an OVS bridge:

```bash
$ ovs-vsctl add-br br0

$ ovs-vsctl show
05daf6f1-da58-4e01-8530-f6ec0d51b4e1
    Bridge br0
        Port br0
            Interface br0
                type: internal
```

###### Userspace Datapath

Userspace datapath differs from the traditional datapath in that its packet forwarding and processing are done in userspace. Among those, **netdev-dpdk** is one of the implementations, which is supported since OVS 2.4.

Commands for creating an OVS bridge using userspace datapath:

```bash
$ ovs-vsctl add-br br0 -- set Bridge br0 datapath_type=netdev
```

Note that you must specify the `datapath_type` to be `netdev` when creating a bridge, otherwise you will get an error like ***ovs-vsctl: Error detected while setting up ‘br0’\***.

#### OVN Logical flows

You can use the `ovn-nbctl` utility to see an overview of the logical topology.

The `ovn-sbctl` utility can be used to see into the state stored in the `OVN_Southbound` database. 

Here is how north-south DB flow population works:

OVN introduces an intermediary representation of the system’s configuration, called logical flows. Logical network configurations are stored in Northbound DB (e.g. defined and written down by CMS-Neutron ML2). The logical flows have a similar expressiveness to physical OpenFlow flows, but they only operate on logical entities. Logical flows for a given network are identical across the whole environment. 

Ovn-northd translates these logical network topology elements to the southbound DB into logical flow tables. With all the other tables that are in the southbound DB, they are pushed down to all the compute nodes and the OVN controllers by an OVSDB monitor request. Therefore, all the changes that happen in the Southbound DB are pushed down to the OVN controllers where relevant (this is called ‘conditional monitoring’) and the chassis hypervisors accordingly generate physical flows. On the hypervisor, ovn-controller receives an updated Southbound DB data, updates the physical flows of OVS, and updates the running configuration version ID. 

On the hypervisor level, when a new VM is added to a compute node, the OVN controller on this specific compute node sends (pushes up) a new condition to the Southbound DB via the OVSDB protocol. This eliminates irrelevant update flows.

Ovn-northd keeps monitoring nb_cfg globally and per chassis, where nb_cfg provides the current requested configuration, sb_cfg provides flow configuration, and hv_cfg provides the chassis running version.

#### OVN Overlay:heavy_check_mark:

OVN 支持三种隧道模式，Geneve，STT 和 VxLAN，但是其中 VxLAN 并不是什么情况下就能用的，Hypervisor 到 Hypervisor 之间的隧道模式只能走 Geneve 和 STT，到 GW 和 Vtep GW 的隧道才能用 VxLAN。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/geneve-vxlan.jpg)

##### 比ovs多了一个gwswitch？

？？？

##### Why Geneve & STT

核心：STT和Geneve可以携带Metadata方便路由、HV直接传输数据，OVS 流表可以通过这两个值来处理报文，不需要解析报文的字段。

因为只有 STT 和 Geneve 支持携带大于 32bit 的 Metadata，VxLAN 并不支持这一特性。并且 STT 和 Geneve 支持使用随机的 UDP 和 TCP 源端口，这些包在 ECMP 里更容易被分布到不同的路径里，VxLAN 的固定端口很容易就打到一条路径上了。

STT 由于是 fake 出来的 TCP 包，网卡只要支持 TSO，就很容易达到高性能。VxLAN 现在一般网卡也都支持 Offloading 了，但是就笔者经验，可能还有各种各样的问题。Geneve 比较新，也有新网卡支持了.

#### Geneve in OVN:factory:

OVSDB 里的 Geneve tunnel 长这样

```
Port "ovn-711117-0"
            Interface "ovn-711117-0"
                type: geneve
                options: {csum="true", key=flow, remote_ip="172.18.3.153"}
```

key=flow 含义是 VNI 由 flow 来决定。

拿一个 OVN 里的 Geneve 包来举例

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/geneve-package.png)

OVN 使用了 VNI 和 Options 来携带了 Metadata，其中

##### Logical Datapath as VNI: datapath?

VNI 使用了 Logical Datapath，也就是 0xb1, 这个和 southbound database 里 datapath_binding 表里的 tunnel key 一致

```
_uuid               : 8fc46e14-1c0e-4129-a123-a69bf093c04e
external_ids        : {logical-switch="182eaadd-2cc3-4ff3-9bef-3793bb2463ec", name="neutron-f3dc2e30-f3e8-472b-abf8-ed455fc928f4"}
tunnel_key          : 177
```



##### Options

Options 里携带了一个 OVN 的 TLV，其中 Option Data 为 0001002，其中第一个 0 是保留位。后面的 001 和 002 是 Logical Inpurt Port 和 Logical Output Port，和 southbound database 里的 port_biding 表里的 tunnel key 一致。

```
_uuid              : e40c929d-1997-4fac-bad3-867996eebd03
chassis            : 869e09ab-d47e-4f18-8562-e28692dc0b39
datapath           : 8fc46e14-1c0e-4129-a123-a69bf093c04e
logical_port       : "dedf0130-50eb-480d-9030-13b826093c4f"
mac                : ["fa:16:3e:ae:9a:b6 192.168.7.13"]
options            : {}
parent_port        : []
tag                : []
tunnel_key         : 1
type               : ""

_uuid              : b410ed4b-de0f-4d66-9815-1ea56b0a833c
chassis            : be5e84f9-3d01-431b-bdfa-208411c102c9
datapath           : 8fc46e14-1c0e-4129-a123-a69bf093c04e
logical_port       : "a3347aa1-a8fb-4e30-820c-04c7e1459dd3"
mac                : ["fa:16:3e:01:73:be 192.168.7.14"]
options            : {}
parent_port        : []
tag                : []
tunnel_key         : 2
type               : ""
```

##### Show Me The Code

在 ovn/controller/physical.h 中，定义 Class 为 0x0102 和 type 0x80，可以看到和上图一致。

```c
#define OVN_GENEVE_CLASS 0x0102  /* Assigned Geneve class for OVN. */
#define OVN_GENEVE_TYPE 0x80     /* Critical option. */
#define OVN_GENEVE_LEN 4
```

在 ovn/controller/physical.c 中，可以看到 ovn-controller 在 encapsulation 的时候，如果是 Geneve，会把 datapath 的 tunnel key 放到 MFF_TUN_ID 里，outport 和 inport 放到 mff_ovn_geneve 里。

```c
static void
put_encapsulation(enum mf_field_id mff_ovn_geneve,
                  const struct chassis_tunnel *tun,
                  const struct sbrec_datapath_binding *datapath,
                  uint16_t outport, struct ofpbuf *ofpacts)
{
    if (tun->type == GENEVE) {
        put_load(datapath->tunnel_key, MFF_TUN_ID, 0, 24, ofpacts);
        put_load(outport, mff_ovn_geneve, 0, 32, ofpacts);
        put_move(MFF_LOG_INPORT, 0, mff_ovn_geneve, 16, 15, ofpacts);
    } else if (tun->type == STT) {
        put_load(datapath->tunnel_key | (outport << 24), MFF_TUN_ID, 0, 64,
                 ofpacts);
        put_move(MFF_LOG_INPORT, 0, MFF_TUN_ID, 40, 15, ofpacts);
    } else if (tun->type == VXLAN) {
        put_load(datapath->tunnel_key, MFF_TUN_ID, 0, 24, ofpacts);
    } else {
        OVS_NOT_REACHED();
    }
}
```

在头文件定义里，可以看到 MFF_TUN_ID 就是 VNI

```c
/* "tun_id" (aka "tunnel_id").
     *
     * The "key" or "tunnel ID" or "VNI" in a packet received via a keyed
     * tunnel.  For protocols in which the key is shorter than 64 bits, the key
     * is stored in the low bits and the high bits are zeroed.  For non-keyed
     * tunnels and packets not received via a tunnel, the value is 0.
     *
     * Type: be64.
     * Maskable: bitwise.
     * Formatting: hexadecimal.
     * Prerequisites: none.
     * Access: read/write.
     * NXM: NXM_NX_TUN_ID(16) since v1.1.
     * OXM: OXM_OF_TUNNEL_ID(38) since OF1.3 and v1.10.
     * Prefix lookup member: tunnel.tun_id.
     */

    MFF_TUN_ID,
```

end

#### OVN实践

##### 常用命令

```bash
## 查看路由策略
ovn-nbctl lr-route-list  XXX_NAME

ovn-nbctl lr-pre-route-list
```



##### 搭建ovn环境

1. 部署ovs

   https://hub.docker.com/r/openvswitch/ovs

    ovsdb-server

2. 部署ovn组件

   https://hub.docker.com/r/openvswitch/ovn

   ovn-nb、ovn-sb、ovn-northd和ovn-controller

如下，容器部署不行，因为root filesystem隔离？

```bash
# ovs
# need privileged to create namespace
docker run -itd --privileged --net=host --name=ovsdb-server openvswitch/ovs:2.11.2_debian ovsdb-server
docker run -itd --net=host --name=ovs-vswitchd --volumes-from=ovsdb-server --privileged openvswitch/ovs:2.11.2_debian ovs-vswitchd


# ovn
docker run -itd --net=host --name=ovn-nb -v /var/run/ovn/:/var/run/ovn/  openvswitch/ovn:2.12_e60f2f2_debian_master ovn-nb
docker run -itd --net=host --name=ovn-sb -v /var/run/ovn/:/var/run/ovn/ openvswitch/ovn:2.12_e60f2f2_debian_master ovn-sb
docker run -itd --net=host --name=ovn-northd -v /var/run/ovn/:/var/run/ovn/ openvswitch/ovn:2.12_e60f2f2_debian_master ovn-northd


docker run -itd --net=host --name=ovn-nb openvswitch/ovn:2.12_e60f2f2_debian_master ovn-nb-tcp
docker run -itd --net=host --name=ovn-sb openvswitch/ovn:2.12_e60f2f2_debian_master ovn-sb-tcp
docker run -itd --net=host --name=ovn-northd openvswitch/ovn:2.12_e60f2f2_debian_master ovn-northd-tcp
```

Ubuntu安装ovn环境

```bash
apt-get -y install build-essential fakeroot
apt-get install python-six openssl -y
apt-get install openvswitch-switch openvswitch-common -y
apt-get install ovn-central ovn-common ovn-host -y
apt-get install ovn-host ovn-common -y


```



##### 创建logical vswitch:fire:

进入ovn-nb容器创建ovn logical switch

```bash
# create logical switch
# ls-add [SWITCH]           create a logical switch named SWITCH
$ ovn-nbctl ls-add ls1
# ls-list                   print the names of all logical switches
$ ovn-nbctl ls-list
59d643df-a15b-4423-98e0-0e9ff0c0a999 (ls1)

# lsp-add SWITCH PORT       add logical port PORT on SWITCH
# lsp-set-addresses PORT [ADDRESS]...  set MAC or MAC+IP addresses for PORT.
#  lsp-set-port-security PORT [ADDRS]... set port security addresses for PORT.
$ ovn-nbctl lsp-add ls1 ls1-vm1
$ ovn-nbctl lsp-add ls1 ls1-vm2
$ ovn-nbctl lsp-set-addresses ls1-vm1 00:00:00:00:00:01
$ ovn-nbctl lsp-set-addresses ls1-vm2 00:00:00:00:00:02
$ ovn-nbctl lsp-set-port-security ls1-vm1 00:00:00:00:00:01
$ ovn-nbctl lsp-set-port-security ls1-vm2 00:00:00:00:00:02                     
                            
# show                      print overview of database contents
$ ovn-nbctl show
switch 59d643df-a15b-4423-98e0-0e9ff0c0a999 (ls1)
    port ls1-vm2
        addresses: ["00:00:00:00:00:02"]
    port ls1-vm1
        addresses: ["00:00:00:00:00:01"]
        
```

end

##### reinstall ovs

 del-dp DP                delete local datapath DP

datapath？？？

```bash
> > I want to remove openvswitch from my platform and reinstall it with
> correct
> > configuration.
> >
> > Have removed all the packages. but i could still see
> >
> > ovs-system: flags=4098<BROADCAST,MULTICAST>  mtu 1500
> >         ether ea:a6:8d:26:10:7e  txqueuelen 0  (Ethernet)
> >         RX packets 0  bytes 0 (0.0 B)
> >         RX errors 0  dropped 0  overruns 0  frame 0
> >         TX packets 0  bytes 0 (0.0 B)
> >         TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
>
> The OVS kernel module most likely comes with your kernel, so you
> cannot completely remove it. To remove a device like this, you can use
> ovs-dpctl to delete it then remove the OVS module:
>
> # ovs-dpctl del-dp ovs-system
> # rmmod openvswi
```

end

##### 连通性测试

```bash
# add-br BRIDGE               create a new bridge named BRIDGE


# create and config vm
ip netns add vm1

```

end

#### OVN  K8s

https://feisky.gitbooks.io/sdn/content/ovs/ovn-kubernetes.html

#### OVN Openstack

https://feisky.gitbooks.io/sdn/content/ovs/ovn-openstack.html

OpenStack [networking-ovn](https://github.com/openstack/networking-ovn) 项目为Neutron提供了一个基于ML2的OVN插件，它使用OVN组件代替了各种Neutron的Python agent，也不再使用 RabbitMQ，而是基于OVN数据库进行通信：使用 OVSDB 协议来把用户的配置写在 Northbound DB 里面，ovn-northd 监听到 Northbound DB 配置发生改变，然后把配置翻译到 Southbound DB 里面，ovn-controller 注意到 Southbound DB 数据的变化，然后更新本地的流表。

OVN 里面报文的处理都是通过 OVS OpenFlow 流表来实现的，而在 Neutron 里面二层报文处理是通过 OVS OpenFlow 流表来实现，三层报文处理是通过 Linux TCP/IP 协议栈来实现

### go-ovn

https://github.com/eBay/go-ovn



### 引用

1. https://www.cnblogs.com/laolieren/p/learn_ovn.html
2. https://feisky.gitbooks.io/sdn/content/ovs/ovn.html
3. https://blog.csdn.net/qq_31331027/article/details/81448313
4. https://www.cnblogs.com/laolieren/p/ovn-architecture.html
5. https://feisky.gitbooks.io/sdn/content/ovs/ovn-kubernetes.html
6. https://www.sdnlab.com/20693.html
7. https://arthurchiao.art/blog/ovs-deep-dive-4-patch-port/
8. https://www.cnblogs.com/laolieren/p/ovn-architecture.htm