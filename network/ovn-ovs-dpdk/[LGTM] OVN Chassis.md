## [LGTM] OVN Chassis

## LGTM

https://www.mankier.com/7/ovn-architecture



## What is Chassis:+1:

Hypervisors and gateways are together called *transport node* or ***chassis***.

Each chassis has a bridge mapping for one of the **localnet** physical networks only.

> A **localnet** logical switch port bridges a logical switch to a physical VLAN. A logical switch may have one or more **localnet** ports. 
>
> chassis肯定是接入一个物理网络的

An OVN *logical network* is a network implemented in software that is insulated from physical (and thus virtual) networks by tunnels or other encapsulations. 

```bash
$ ovn-sbctl show
Chassis local_chassis
    hostname: TJ-UNISTACK-YFB-SEC01
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
$ ovn-sbctl chassis-add fakechassis geneve 127.0.0.1
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}


$ ovn-nbctl ls-add sw0
$ ovn-nbctl lsp-add sw0 sp0
$ ovn-nbctl show
    switch d02ad16a-2b74-434e-a26f-aa45ad698b4e (sw0)
        port sp0
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
        
#  bind logical port PORT to CHASSIS
$ ovn-sbctl lsp-bind sp0 fakechassis
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
    # new binding
    Port_Binding "sp0"
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
```



### localnet

  Introduce a new logical port type called "localnet".  A logical port with this type also has an option called "network_name".  A "localnet" logical port represents a connection to a network that is locally accessible from each chassis running ovn-controller.  ovn-controller will use the ovn-bridge-mappings configuration to figure out which patch port on br-int should be used for this port.

https://patchwork.ozlabs.org/project/openvswitch/patch/1441298701-32163-3-git-send-email-rbryant@redhat.com/

### gateway

Gateways provide limited connectivity between logical networks and physical ones.

#### vtep gateway

A `VTEP gateway` connects an OVN logical network to a physical (or virtual) switch that implements the OVSDB VTEP schema that accompanies Open vSwitch. (The `VTEP gateway` term is a misnomer, since a VTEP is just a VXLAN Tunnel Endpoint, but it is a well established name.) 

> misnomer: a name that does not suit what it refers to, or the use of such a name.
>
> vtep gateway，名字起的多少有点问题，它就是一个vxlan tunnel endpoint，用来连接ovn和physical switch.

The main intended use case for VTEP gateways is to attach physical servers to an OVN logical network using a physical top-of-rack switch that supports the OVSDB VTEP schema

#### L2 gateway: ls

A L2 gateway simply attaches a designated physical L2 segment available on some chassis to a logical network. The physical network effectively becomes part of the logical network.

To set up a L2 gateway, the CMS adds an **L2 gateway** LSP to an appropriate **logical switch**， setting LSP options to name the chassis on which it should be bound. **ovn-northd** copies this configuration into a southbound **Port_Binding** record. On the designated chassis, **ovn-controller** forwards packets appropriately to and from the physical segment.

L2 gateway ports have features in common with **localnet** ports. However, with a **localnet** port, the physical network becomes the transport between hypervisors. With an L2 gateway, packets are still transported between hypervisors over tunnels and the **L2 gateway** port is only used for the packets that are on the physical network. The application for L2 gateways is similar to that for VTEP gateways, e.g. to add non-virtualized machines to a logical network, but **L2 gateways do not require special support from top-of-rack hardware switches.**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/l2gw-usecase5.png)



#### L3 gateway router：lr

Gateway router和logical router的区别就是在创建时绑定了chassis

```bash
# ovn-nbctl create Logical_Router name=edge1 options:chassis={chassis_uuid}
# options:chassis={chassis_uuid}指定了虚拟路由器的物理位置，表示和外部通信的实现是在此chassis上面。
# options:chassis 也表示这样是一个 Gateway Routre

# 查看chassis
$ ovn-sbctl show
Chassis fakechassis
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
    Port_Binding "sp0"
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
# Gateway router
$ ovn-nbctl create Logical_Router name=edge1 options:chassis=br-test
df4e9b2f-c30c-4925-a256-ac8b7b16b44d
$ ovn-nbctl show

#  logical router 
$ ovn-nbctl lr-add test-r
$ ovn-nbctl show
    switch d02ad16a-2b74-434e-a26f-aa45ad698b4e (sw0)
        port sp0
    router df4e9b2f-c30c-4925-a256-ac8b7b16b44d (edge1)
    router 376614cb-bcda-4be8-9385-dc68603a24fe (test-r)
```



A gateway router is a logical router that is **bound** to a physical location.

> gateway router 和 logical router都是ovn-nbctl lr-add添加的

Gateway routers are typically used in between distributed  logical  routers  and  physical networks.

an OVN gateway router is centralized on a single host (chassis) 

vn的逻辑路由器是分布式的，这意味着它没有绑定到某个节点上，而是存在于所有节点上的，同时它是通过每个节点的openflow流表来实现的，所有vm之间的东西向流量可以在本节点就能找到目的节点，不用再发送的网络节点处理。

对于一些有状态的服务是有问题的，比如SNAT和DNAT，这些服务需要在同一个节点上实现。为了解决这个问题，引入了网关路由器，其和逻辑路由器的区别是，网关路由器会通过Logical_Router表的选项options:chassis绑定到指定的节点上。

**Gateway routers are typically used in between distributed  logical  routers  and  physical networks.**  The distributed logical router and the logical switches behind it, to which VMs and containers attach, effectively reside on each hypervisor. The **distributed  router**  and the  **gateway  router**  are  connected by another logical switch, sometimes referred to as **a join logical switch.** On the other side, the gateway router  connects  to  another  logical switch that has a localnet port connecting to the physical network.

> Localnet  ports  represent the points of connectivity between logical switches and the physical network. They are implemented as OVS  patch ports  between  the  integration bridge and the separate Open vSwitch bridge that underlay physical ports attach to.

 When  using  gateway  routers, DNAT and SNAT rules are associated with the gateway router, which provides a central location that can handle one-to-many SNAT (aka IP masquerading).

网关路由器，其需要通过单独的switch LSjoin连接到逻辑路由器(多个逻辑路由器可以直接相连，但是网关路由器得通过LSjoin连接)。

环境:

- node1 172.17.42.160/16 – will serve as OVN Central and Gateway Node
- node2 172.17.42.161/16 – will serve as an OVN Host
- node3 172.17.42.162/16 – will serve as an OVN Host
- client 172.18.1.10/16 - physical network node

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

The following diagram shows a typical situation. One or more logical switches LS1, ..., LSn connect to distributed logical router LR1, which in turn connects through LSjoin to gateway logical router GLR, which also connects to logical switch LSlocal, which includes a **localnet** port to attach to the physical network.

```
                                LSlocal
                                   |
                                  GLR
                                   |
                                LSjoin
                                   |
                                  LR1
                                   |
                              +----+----+
                              |    |    |
                             LS1  ...  LSn
```

To configure an L3 gateway router, the CMS sets **options:chassis** in the router’s northbound **Logical_Router** to the chassis’s name. In response, **ovn-northd** uses a special **l3gateway** port binding (instead of a **patch** binding) in the southbound database to connect the logical router to its neighbors. In turn, **ovn-controller** tunnels packets to this port binding to the designated L3 gateway chassis, instead of processing them locally.

DNAT and SNAT rules may be associated with a gateway router, which provides a central location that can handle one-to-many SNAT (aka IP masquerading). Distributed gateway ports, described below, also support NAT.

##### L-Switch-join的好处

1. L-Router可以通过一个接口对接N个GW-Router。物理场景下节省路由器端口

2. 可以做到配置分离，L-Router只负责vm of different L-Switch的转发

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovn-L3-GW-demo.jpg)

When  using  gateway  routers, DNAT and SNAT rules are associated with the gateway router,
which provides a central location that can handle one-to-many SNAT (aka IP masquerading).





### Hypervisors 

A hypervisor, also known as a virtual machine monitor or VMM, is software that creates and runs virtual machines (VMs). 

## Chassis作用

OVN网络与外界流量通讯肯定是要过chassis。

`Chassis` 可以是 `HV`，也可以是 `VTEP` 网关（vxlan tunnel endpoint for Hybird Overlay）。 `Chassis` 的信息保存在 `Southbound DB` 里面，由 `ovn-controller/ovn-controller-vtep` 来维护。

* ovn-controller:  ovn-controller is the local controller daemon for OVN, the Open Virtual Network. It connects up to the OVN Southbound database vSwitch database  over the OVSDB  protocol and to ovs-vswitchd **via OpenFlow.**  

  Each hypervisor and  software gateway in an OVN deployment runs its own independent copy of ovn-controller;

* ovn-controller-vtep:   ovn-controller-vtep is the local controller daemon in OVN, the Open Virtual Network, for VTEP enabled physical switches. It connects up to the OVN Southbound database over the OVSDB protocol, and down to the VTEP database.
  over the OVSDB protocol.

### demo： 业务bond作chassis

一个HV(OVN主机)有一个chassis绑定了physical NIC，然后将OVN的L2 gateway 或者 L3 gateway都绑定到这个chassis，就可以实现OVN网络和外界网络的通讯了。

当然，这个chassis还可以用于OVN网络中，vm跨node的通讯。

这就是物理机上 业务网的作用了。



## Chassis ：br-int

Each chassis has a bridge mapping for one of the **localnet** physical networks only.

Each chassis in an OVN deployment  must  be  configured  with  an  Open vSwitch  bridge dedicated for OVN’s use, called the **integration bridge**.System startup  scripts  may  create  this  bridge  prior  to  starting
**ovn-controller**  if desired. If this bridge does not exist when ovn-controller starts, it will be created automatically with the default  configuration  suggested  below.  The  ports  on  the  integration  bridge
include:

> The  ovn-controller  process  on  each  chassis receives the updated southbound database, with the updated  nb_cfg.

* On any chassis, tunnel ports that OVN  uses  to  maintain logical   network   connectivity.   ovn-controller  adds, updates, and removes these tunnel ports.

* On a hypervisor, any VIFs that are to be attached to logical  networks.  For instances connected through software emulated ports such as TUN/TAP or VETH pairs, the  hypervisor  itself  will  normally  create ports and plug them into the  integration  bridge.  For  instances  connected through  representor  ports, typically used with hardware offload, the ovn-controller may on CMS direction  consult a  VIF plug provider for representor port lookup and plug them into the integration bridge (please refer  to  Docu‐
  mentation/top‐ics/vif-plug-providers/vif-plug-providers.rst for more information). In  both  cases  the  conventions described  in Documentation/topics/integration.rst in the Open vSwitch source tree is followed  to  ensure  mapping between  OVN  logical port and VIF. (This is pre-existing integration work that has already been done  on  hypervisors that support OVS.)
* On  a gateway, the physical port used for logical network connectivity. System startup scripts add this port to the bridge  prior  to  starting ovn-controller. This can be a patch port to another bridge, instead of a physical port,in more sophisticated setups.

Other  ports  should not be attached to the integration bridge. In particular, physical ports attached to the underlay network (as opposed to gateway  ports,  which are physical ports attached to logical networks)
must not be attached to the integration bridge. Underlay physical ports should instead be attached to a separate Open vSwitch bridge (they need not be attached to any bridge at all, in fact).

The integration bridge should be configured as described below. The effect of each of these settings is documented in [ovs-vswitchd.conf.db(5)](https://www.mankier.com/5/ovs-vswitchd.conf.db):

- **fail-mode=secure**

  Avoids switching packets between isolated logical networks before **ovn-controller** starts up. See **Controller Failure Settings** in [ovs-vsctl(8)](https://www.mankier.com/8/ovs-vsctl) for more information.

- **other-config:disable-in-band=true**

  Suppresses in-band control flows for the integration bridge. It would be unusual for such flows to show up anyway, because OVN uses a local controller (over a Unix domain socket) instead of a remote controller. It’s possible, however, for some other bridge in the same system to have an in-band remote controller, and in that case this suppresses the flows that in-band control would ordinarily set up. Refer to the documentation for more information.

The customary name for the integration bridge is  br-int,  but  another name may be used.



## *Logical Switch Port Types*

OVN supports a number of kinds of logical switch ports. VIF ports that connect to VMs or containers, described above, are the most ordinary kind of LSP. In the OVN northbound database, VIF ports have an empty string for their **type**. This section describes some of the additional port types.

ports, so properly speaking  one  should  usually  talk  about  logical switch ports or logical router ports. However, an unqualified "logical port" usually refers to a logical switch port.

### VIF

A **router** logical switch port connects a logical switch to a logical router, designating a particular LRP as its peer.

### localnet

A **localnet** logical switch port bridges a logical switch to a physical VLAN. A logical switch may have one or more **localnet** ports. Such a logical switch is used in two scenarios:

- With one or more **router** logical switch ports, to attach L3 gateway routers and distributed gateways to a physical network.
- With one or more VIF logical switch ports, to attach VMs or containers directly to a physical network. In this case, the logical switch is not really logical, since it is bridged to the physical network rather than insulated from it, and therefore cannot have independent but overlapping IP address namespaces, etc. A deployment might nevertheless choose such a configuration to take advantage of the OVN control plane and features such as port security and ACLs.

When a logical switch contains multiple localnet ports, the following is assumed.

- Each chassis has a bridge mapping for one of the **localnet** physical networks only.
- To facilitate interconnectivity between VIF ports of the switch that are located on different chassis with different physical network connectivity, the fabric implements L3 routing between these adjacent physical network segments.

Note: nothing said above implies that a chassis cannot be plugged to multiple physical networks as long as they belong to different switches.

### localport

A **localport** logical switch port is a special kind of VIF logical switch port. These ports are present in every chassis, not bound to any particular one. Traffic to such a port will never be forwarded through a tunnel, and traffic from such a port is expected to be destined only to the same chassis, typically in response to a request it received. OpenStack Neutron uses a **localport** port to serve metadata to VMs. A metadata proxy process is attached to this port on every host and all VMs within the same network will reach it at the same IP/MAC address without any traffic being sent over a tunnel. For further details, see the OpenStack documentation for networking-ovn.

### vtep and L2 gateway

LSP types **vtep** and **l2gateway** are used for gateways. See **Gateways**, below, for more information



## Distributed Gateway Ports：？？？

可以设置多个chassis，并指定chassis的优先级，只有优先级最高的chassis工作，其他chassis作为备份，chassis之间使用bfd检测是否存活。

A *distributed gateway port* is a logical router port that is specially configured to designate one distinguished chassis, called the *gateway chassis*, for centralized processing. A distributed gateway port should connect to a logical switch that has an LSP that connects externally, that is, either a **localnet** LSP or a connection to another OVN deployment (see **OVN Deployments Interconnection**). Packets that traverse the distributed gateway port are processed without involving the gateway chassis when they can be, but when needed they do take an extra hop through it.

The following diagram illustrates the use of a distributed gateway port. A number of logical switches LS1, ..., LSn connect to distributed logical router LR1, which in turn connects through the distributed gateway port to logical switch LSlocal that includes a **localnet** port to attach to the physical network.

```
                                LSlocal
                                   |
                                  LR1
                                   |
                              +----+----+
                              |    |    |
                             LS1  ...  LSn
```

**ovn-northd** creates two southbound **Port_Binding** records to represent a distributed gateway port, instead of the usual one. One of these is a **patch** port binding named for the LRP, which is used for as much traffic as it can. The other one is a port binding with type **chassisredirect**, named **cr-***port*. The **chassisredirect** port binding has one specialized job: when a packet is output to it, the flow table causes it to be tunneled to the gateway chassis, at which point it is automatically output to the **patch** port binding. Thus, the flow table can output to this port binding in cases where a particular task has to happen on the gateway chassis. The **chassisredirect** port binding is not otherwise used (for example, it never receives packets).

The CMS may configure distributed gateway ports three different ways. See **Distributed Gateway Ports** in the documentation for **Logical_Router_Port** in [ovn-nb(5)](https://www.mankier.com/5/ovn-nb) for details.

Distributed gateway ports support high availability. When more than one chassis is specified, OVN only uses one at a time as the gateway chassis. OVN uses BFD to monitor gateway connectivity, preferring the highest-priority gateway that is online.

A logical router can have multiple distributed gateway ports, each connecting different external networks. Load balancing is not yet supported for logical routers with more than one distributed gateway port configured.

```bash
# 设置 Distributed Gateway Ports 的方式之一：
ovn-nbctl ha-chassis-group-add ha1
ovn-nbctl ha-chassis-group-add-chassis ha1 master 1
ovn-nbctl ha-chassis-group-add-chassis ha1 node1 2
# 465efd10-c0e0-4966-be32-a20b213a2dbc 为 ha1 的uuid，可通过 ovn-nbctl ha-chassis-group-list 查看
ovn-nbctl set Logical_Router_Port  lr1-lslocal ha_chassis_group=465efd10-c0e0-4966-be32-a20b213a2dbc
 
# 设置 Distributed Gateway Ports 的方式之二：
#  Set gateway chassis for port. chassis is the name of the chassis. This creates a gateway chassis entry in Gateway_Chassis table.
ovn-nbctl lrp-set-gateway-chassis lr1-lslocal master 1
ovn-nbctl lrp-set-gateway-chassis lr1-lslocal node1 2
```

end

## 连接Overlay and Underlay：chassis:fire:

### lsp-set-address : unknow 

`ovn-nbctl lsp-set-addresses <port_name> unknown`

OVN delivers unicast Ethernet packets whose destination MAC address  is  not in any logical port’s addresses column to ports with address unknown.

### lsp-set-type

Set the type for the logical port. The type must be one of the following:

* (empty string):  A VM (or VIF) interface.
* router: A connection to a logical router.
* localnet: A  connection  to  a  locally  accessible  network  from each ovn-controller instance. A logical switch can only have a single  localnet  port  attached. This is used to model direct connectivity to an existing network.
* localport: A  connection  to  a local VIF. Traffic that arrives on a localport is never forwarded over a tunnel to another chassis. These ports are present on every  chassis  and  have  the  same  address in all of them. This is used to model  connectivity to local services that run on every hypervisor.
* l2gateway: A connection to a physical network.
* vtep :  A port to a logical switch on a VTEP gateway.

### lsp-set-options

`ovn-nbctl lsp-set-options l2gateway-chassis=<chassis_name> `

The chassis on which the l2gateway logical port should be bound to. ovn-controller running on the defined chassis will connect this logical port to the physical network.

```xml
        <column name="options" key="network_name">
          Required.  The name of the network to which the <code>l2gateway</code>
          port is connected.  The L2 gateway, via <code>ovn-controller</code>,
          uses its local configuration to determine exactly how to connect to
          this network.
        </column>

        <column name="options" key="l2gateway-chassis">
          Required. The chassis on which the <code>l2gateway</code> logical
          port should be bound to. <code>ovn-controller</code> running on the
          defined chassis will connect this logical port to the physical network.
        </column>

```



### ovs mapping network

```xml
      <column name="options" key="network_name">
        Required.  <code>ovn-controller</code> uses the configuration entry
        <code>ovn-bridge-mappings</code> to determine how to connect to this
        network.  <code>ovn-bridge-mappings</code> is a list of network names
        mapped to a local OVS bridge that provides access to that network.  An
        example of configuring <code>ovn-bridge-mappings</code> would be:

        <pre>$ ovs-vsctl set open . external-ids:ovn-bridge-mappings=physnet1:br-eth0,physnet2:br-eth1</pre>

        <p>
          When a logical switch has a <code>l2gateway</code> port attached,
          the chassis that the <code>l2gateway</code> port is bound to
          must have a bridge mapping configured to reach the network
          identified by <code>network_name</code>.
        </p>
      </column>
```



官方的nb.xml有说明

https://github.com/ovn-org/ovn/blob/b6e6ef7c470304955f5ded89f24a1f7e49d942de/ovn-nb.xml

### L2 gateway 连接外部网络

L2 gateway就是将一个lsp声明成一个三层接口，类似vlan if，然后将这个vif映射到一个物理网卡。

* ovn-nbctl lsp-set-options {port_name} network_name= {Net_Name}
* ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings={Net_Name}:{Port_name}

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/l2-gw-with-external-net.webp)

ens8所在的网络为外部网络。
 因为l2gateway端口绑定到了master chassis，所以只在master的br-int通过patch端口和br-ens8互连，用来连接外部网络。
 其他chassis上的vm如果想访问外部网络，必须先经过tunnel network发送到master上的br-int，再发给br-ens8来实现。

```bash
# 创建 logical switch ls1
ovn-nbctl ls-add ls1

# 添加第一个 logical port ls1-vm1
ovn-nbctl lsp-add ls1 ls1-vm1
ovn-nbctl lsp-set-addresses ls1-vm1 00:00:00:00:00:03
ovn-nbctl lsp-set-port-security ls1-vm1 00:00:00:00:00:03

# 添加第二个 logical port ls1-vm2
ovn-nbctl lsp-add ls1 ls1-vm2
ovn-nbctl lsp-set-addresses ls1-vm2 00:00:00:00:00:04
ovn-nbctl lsp-set-port-security ls1-vm2 00:00:00:00:00:04

# 添加第三个 logical port ls1-l2gateway，类型为 l2gateway，用来连接外部网络
ovn-nbctl lsp-add ls1 ls1-l2gateway
ovn-nbctl lsp-set-type ls1-l2gateway l2gateway
ovn-nbctl lsp-set-addresses ls1-l2gateway unknown
ovn-nbctl lsp-set-options ls1-l2gateway network_name=externalnet l2gateway-chassis=master

# 因为选项 l2gateway-chassis 指定了 chassis 为 master，所以只在master节点上执行如下命令
ovs-vsctl add-br br-ens8
ovs-vsctl add-port br-ens8 ens8
##  set TBL REC COL[:KEY]=VALUE set COLumn values in RECord in TB
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=externalnet:br-ens8
ip link set dev br-ens8 up
ip addr add 10.10.10.4/24 dev br-ens8

# 在master上创建vm1 namespace
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 netns vm1
ip netns exec vm1 ip link set vm1 address 00:00:00:00:00:03
ip netns exec vm1 ip addr add 10.10.10.2/24 dev vm1
ip netns exec vm1 ip link set vm1 up
# 通过iface-id=ls1-vm1和逻辑端口ls1-vm1绑定
ovs-vsctl set Interface vm1 external_ids:iface-id=ls1-vm1

# 在node1上创建vm2 namespace
ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 netns vm2
ip netns exec vm2 ip link set vm2 address 00:00:00:00:00:04
ip netns exec vm2 ip addr add 10.10.10.3/24 dev vm2
ip netns exec vm2 ip link set vm2 up
# 通过iface-id=ls1-vm2和逻辑端口ls1-vm2绑定
ovs-vsctl set Interface vm2 external_ids:iface-id=ls1-vm2
```



### L3 gateway连接外部网络

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/L3-gw-connect-external-net.webp)

创建两个交换机(ls1和ls2)和一个路由器(lr1)

```bash
# 创建两个虚拟交换机 ls1 和 ls2
ovn-nbctl ls-add ls1
ovn-nbctl ls-add ls2
# 创建一个虚拟路由器 lr1
ovn-nbctl lr-add lr1

# 在虚拟路由器 lr1 上添加端口，用来连接虚拟交换机 ls1
ovn-nbctl lrp-add lr1 lr1-ls1 00:00:00:00:00:01 10.10.10.1/24

# 在虚拟交换机 ls1 上添加端口，用来连接虚拟路由器 lr1
ovn-nbctl lsp-add ls1 ls1-lr1
# 端口类型必须为 router
ovn-nbctl lsp-set-type ls1-lr1 router
# 设置地址，必须和 lr1-ls1 的一致
ovn-nbctl lsp-set-addresses ls1-lr1 00:00:00:00:00:01
# 指定 router-port
ovn-nbctl lsp-set-options ls1-lr1 router-port=lr1-ls1

# 在虚拟路由器 lr1 上添加端口，用来连接虚拟交换机 ls2
ovn-nbctl lrp-add lr1 lr1-ls2 00:00:00:00:00:02 10.10.20.1/24

# 在虚拟交换机 ls2 上添加端口，用来连接虚拟路由器 lr1
ovn-nbctl lsp-add ls2 ls2-lr1
# 端口类型必须为 router
ovn-nbctl lsp-set-type ls2-lr1 router
# 设置地址，必须和 lr1-ls2 的一致
ovn-nbctl lsp-set-addresses ls2-lr1 00:00:00:00:00:02
# 指定 router-port
ovn-nbctl lsp-set-options ls2-lr1 router-port=lr1-ls2
```

在交换机上ls1和ls2上添加vm接口

```bash
# 在虚拟交换机 ls1 上添加两个端口，指定 mac 和 ip(10.10.10.0/24网段)，用来连接vm
ovn-nbctl lsp-add ls1 ls1-vm1
ovn-nbctl lsp-set-addresses ls1-vm1 "00:00:00:00:00:03 10.10.10.2"
ovn-nbctl lsp-set-port-security ls1-vm1 "00:00:00:00:00:03 10.10.10.2"

ovn-nbctl lsp-add ls1 ls1-vm2
ovn-nbctl lsp-set-addresses ls1-vm2 "00:00:00:00:00:04 10.10.10.3"
ovn-nbctl lsp-set-port-security ls1-vm2 "00:00:00:00:00:04 10.10.10.3"

# 在虚拟交换机 ls2 上添加两个端口，指定 mac 和 ip(10.10.20.0/24网段)，用来连接vm
ovn-nbctl lsp-add ls2 ls2-vm1
ovn-nbctl lsp-set-addresses ls2-vm1 "00:00:00:00:00:03 10.10.20.2"
ovn-nbctl lsp-set-port-security ls2-vm1 "00:00:00:00:00:03 10.10.20.2"

ovn-nbctl lsp-add ls2 ls2-vm2
ovn-nbctl lsp-set-addresses ls2-vm2 "00:00:00:00:00:04 10.10.20.3"
ovn-nbctl lsp-set-port-security ls2-vm2 "00:00:00:00:00:04 10.10.20.3"
```

创建四个namespace，模拟四个vm

```bash
# 在 master 节点上，创建两个namespace，用来模拟两个vm，使用 "iface-id" 指定
# 这两个vm属于 ls1
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 netns vm1
ip netns exec vm1 ip link set vm1 address 00:00:00:00:00:03
ip netns exec vm1 ip addr add 10.10.10.2/24 dev vm1
ip netns exec vm1 ip link set vm1 up
ip netns exec vm1 ip route add default via 10.10.10.1 dev vm1
ovs-vsctl set Interface vm1 external_ids:iface-id=ls1-vm1


ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 netns vm2
ip netns exec vm2 ip link set vm2 address 00:00:00:00:00:04
ip netns exec vm2 ip addr add 10.10.10.3/24 dev vm2
ip netns exec vm2 ip link set vm2 up
ip netns exec vm2 ip route add default via 10.10.10.1 dev vm2
ovs-vsctl set Interface vm2 external_ids:iface-id=ls1-vm2


# 在 node1 节点上，创建两个namespace，用来模拟两个vm，使用 "iface-id" 指定这两个vm属于 ls2
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 netns vm1
ip netns exec vm1 ip link set vm1 address 00:00:00:00:00:03
ip netns exec vm1 ip addr add 10.10.20.2/24 dev vm1
ip netns exec vm1 ip link set vm1 up
ip netns exec vm1 ip route add default via 10.10.20.1 dev vm1
ovs-vsctl set Interface vm1 external_ids:iface-id=ls2-vm1


ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 netns vm2
ip netns exec vm2 ip link set vm2 address 00:00:00:00:00:04
ip netns exec vm2 ip addr add 10.10.20.3/24 dev vm2
ip netns exec vm2 ip link set vm2 up
ip netns exec vm2 ip route add default via 10.10.20.1 dev vm2
ovs-vsctl set Interface vm2 external_ids:iface-id=ls2-vm2
```

开始创建网关路由器, 用于连接逻辑路由器的lsjoin和用于连接外部网络的lslocal

```bash
# 在master节点执行，创建第二个虚拟路由器 lr2,并添加两个虚拟路由器端口
# ovn-nbctl create Logical_Router name=edge1 options:chassis={chassis_uuid}
# 其中 options:chassis={chassis_uuid}指定了虚拟路由器的物理位置，表示和外部通信的实现是在此chassis上面。
# options:chassis 也表示这样是一个 Gateway Routre
ovn-nbctl create Logical_Router name=lr2 options:chassis=master
ovn-nbctl lrp-add lr2 lr2-lsjoin 00:00:00:00:00:06 10.10.30.2/24
ovn-nbctl lrp-add lr2 lr2-lslocal 00:00:00:00:00:07 10.10.40.1/24

# 在master节点执行，创建虚拟交换机 lsjoin，用来连接两个路由器 lr1 和 lr2
ovn-nbctl ls-add lsjoin
ovn-nbctl lsp-add lsjoin lsjoin-lr2
ovn-nbctl lsp-set-type lsjoin-lr2 router
ovn-nbctl lsp-set-addresses lsjoin-lr2 00:00:00:00:00:06
ovn-nbctl lsp-set-options lsjoin-lr2 router-port=lr2-lsjoin

# 在master节点执行，在虚拟路由器 lr1 上添加虚拟路由器端口，用来连接 lsjoin
ovn-nbctl lrp-add lr1 lr1-lsjoin 00:00:00:00:00:05 10.10.30.1/24
# 在master节点执行，在虚拟交换机 lsjoin 上添加虚拟交换机端口，用来连接 lr1
ovn-nbctl lsp-add lsjoin lsjoin-lr1
ovn-nbctl lsp-set-type lsjoin-lr1 router
ovn-nbctl lsp-set-addresses lsjoin-lr1 00:00:00:00:00:05
ovn-nbctl lsp-set-options lsjoin-lr1 router-port=lr1-lsjoin

# 在master节点执行，在虚拟路由器 lr1 和 lr2 上添加静态路由
ovn-nbctl lr-route-add lr2 "10.10.10.0/24" 10.10.30.1
ovn-nbctl lr-route-add lr1 "0.0.0.0/0" 10.10.30.2

# 在master节点执行，创建虚拟交换机 lslocal，用来连接到外部网络
ovn-nbctl ls-add lslocal
ovn-nbctl lsp-add lslocal lslocal-lr2
ovn-nbctl lsp-set-type lslocal-lr2 router
ovn-nbctl lsp-set-addresses lslocal-lr2 00:00:00:00:00:07
ovn-nbctl lsp-set-options lslocal-lr2 router-port=lr2-lslocal

# 创建连接外部网络的switch br-ens8，其中 ovn-bridge-mappings 指定了网络名称和实际网桥的映射关系
# 必须在网关路由器的选项 options:chassis=master 指定的chassis上执行。本实验指定的chassis为master，
# 所以下面命令在master上执行。
ovs-vsctl add-br br-ens8
ovs-vsctl add-port br-ens8 ens8
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=externalnet:br-ens8
ip link set dev br-ens8 up
ip addr add 10.10.40.2/24 dev br-ens8

# 在master节点执行，在虚拟交换机 lslocal上添加 localnet 类型的端口，并设置 network_name 为 externalnet，
# externalnet 为 ovn-bridge-mappings 指定的，对应实际网桥 br-ens8
ovn-nbctl lsp-add lslocal lslocal-localnet
ovn-nbctl lsp-set-addresses lslocal-localnet unknown
ovn-nbctl lsp-set-type lslocal-localnet localnet
ovn-nbctl lsp-set-options lslocal-localnet network_name=externalnet
```

执行完上面命令后，从ovn网络 lr1上的 vm1 ping 外部网络是不通的，这是因为从外部网络返回的响应报文查不到回程路由，
 最终走默认路由，发给其他接口了。解决办法有两个：
 a. 在外部网络上配置返程的静态路由
 b. 在网关路由器 lr2 上添加 snat 表项，使lr1上的 vm1 ping报文的源ip修改为外部网络的网段ip

```bash
    # 在master节点执行
    ovn-nbctl -- --id=@nat create nat type="snat" logical_ip=10.10.10.0/24 \
    external_ip=10.10.40.1 -- add logical_router lr2 nat @nat
```





### port_binding

ovn-controller will then set up the flows necessary for these ports to be able to communicate each other as defined by the OVN logical topology.

```bash
# lsp-bind logical-port chassis Binds the logical port named logical-port to chassis
$ ovn-sbctl [--may-exist] lsp-bind logical-port chassis
$ ovn-sbctl [--if-exists] lsp-unbind logical-port
```

除了vm/container port， attached underlay network physical port, tunnel port ，

Other  ports  should not be attached to the integration bridge.

### east-west traffic 

The  following happens when a VM sends an east-west traffic which needs to be routed:

1.  The packet first  enters  the  ingress  pipeline,  and  then egress  pipeline of the source localnet logical switch datapath and is sent out via  a  localnet  port  of  the  source localnet  logical  switch  (instead  of sending it to router pipeline).
    
2.  The gateway chassis receives the packet via a localnet  port of  the  source  localnet logical switch and sends it to the integration bridge. The packet then enters the ingress pipeline,  and then egress pipeline of the source localnet logical switch datapath and enters the ingress pipeline  of  the logical router datapath.
    
3.  Routing decision is taken.

4.  From the router datapath, packet enters the ingress pipeline and then egress pipeline of the destination localnet logical switch  datapath. It then goes out of the integration bridge to the provider bridge ( belonging to the destination  logical switch) via a localnet port.
    
5.  The  destination  chassis receives the packet via a localnet port and sends it to  the  integration  bridge.  The  packet enters  the ingress pipeline and then egress pipeline of the destination localnet logical switch and finally delivered to the destination VM port.

### 配置demo

https://www.openvswitch.org/support/dist-docs-2.5/tutorial/OVN-Tutorial.md.html

在github上将 ovs的分支切换到2.5就能看到shell script

## OVN Southbound DB 

OVN Southbound DB 包括如下表：

The  following  list  summarizes  the  purpose of each of the tables in the OVN_Southbound database.  Each table is described in more detail on a later page.

| Table            | Purpose                                             |
| ---------------- | --------------------------------------------------- |
| SB_Global        | Southbound configuration                            |
| Chassis          | Physical Network Hypervisor and Gateway Information |
| Encap            | Encapsulation Types                                 |
| Address_Set      | Address Sets                                        |
| Logical_Flow     | Logical Network Flows                               |
| Multicast_Group  | Logical Port Multicast Groups                       |
| Datapath_Binding | Physical-Logical Datapath Bindings                  |
| Port_Binding     | Physical-Logical Port Bindings                      |
| MAC_Binding      | IP to MAC bindings                                  |
| DHCP_Options     | DHCP Options supported by native OVN DHCP           |
| DHCPv6_Options   | DHCPv6 Options supported by native OVN DHCPv6       |
| Connection       | OVSDB client connections.                           |
| SSL              | SSL configuration.                                  |
| DNS              | Native DNS resolution                               |
| RBAC_Role        | RBAC_Role configuration.                            |
| RBAC_Permission  | RBAC_Permission configuration.                      |
| Gateway_Chassis  | Gateway_Chassis configuration.                      |

### chassis table

在物理网络中每一行表示一个 hypervisor 或者网关，每一个 chassis 通过 ovn-controler/ovn-controller-vtep 添加和更新自己的行，并保留剩余行的拷贝来确定如何访问其他的 hppervisor。

```bash
$ ovn-sbctl list Chassis
_uuid               : f20ae8d7-800a-46dd-adb2-c2f279cb4456
encaps              : [e94ba8c3-7648-41eb-94cf-5615d83c8182]
external_ids        : {}
hostname            : ""
name                : br-test
nb_cfg              : 0
vtep_logical_switches: []

```

### ovn-sbctl chassis-add:fire:

The `ovn-sbctl` utility can be used to see into the state stored in the `OVN_Southbound` database. 

```bash
# chassis-add chassis encap-type encap-ip  Creates  a  new  chassis  named  chassis. 
$ ovn-sbctl chassis-add br-test geneve 172.21.170.84
$ ovn-sbctl show
Chassis br-test
    Encap geneve
        ip: "172.21.170.84"
        options: {csum="true"}
```

end



## ovs add chassis?:confused:

Let us create an *Integration Bridge* with the Name OVN on kvmhost01 ad kvmhost03:

```bash
## OVN is bridge name. It always is br-int
$ ovs-vsctl add-br OVN -- set Bridge OVN fail-mode=secure
```

We now tell the Hosts, how the *ovn-central* system can be reached and which *Integration Bridge* should be used:

```bash
id_file=/etc/openvswitch/system-id.conf
test -e $id_file || uuidgen > $id_file

ovs-vsctl set open . external_ids:system-id=$(cat $id_file)
ovs-vsctl set open . external_ids:ovn-nb="tcp:172.18.31.1:6641"
ovs-vsctl set open . external-ids:ovn-remote="tcp:172.18.31.1:6642"
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.31.10
ovs-vsctl set open . external_ids:ovn-bridge=OVN
```

On Arch Linux the Package doesn’t generate an system-id.conf file with an system wide uuid, so the first lines will create them if it isn’t already there. The above settings are made on kvmhost01, be sure to change the *ovn-encap-ip* to your Overlay IP of the Host. And start the *ovn-controller* after configure the settings above:

```
/usr/share/openvswitch/scripts/ovn-ctl start_controller
```

If all went well, you can see on your *ovn-central* system the new Chassis with `ovn-sbctl show`:

```
Chassis "bffe235d-b222-49ce-9723-180ad5ba8b90"
    hostname: "kvmhost01"
    Encap geneve
	ip: "172.18.31.10"
	options: {csum="true"}
```

Just repeat the settings on all of your participating hosts and `ovn-sbctl show` and with our lab setup, it should look like:

```
Chassis "bffe235d-b222-49ce-9723-180ad5ba8b90"
    hostname: "kvmhost01"
    Encap geneve
	ip: "172.18.31.10"
	options: {csum="true"}
Chassis "804c7da4-04c8-416e-9420-0345f7335284"
    hostname: "kvmhost03"
    Encap geneve
	ip: "172.18.31.30"
	options: {csum="true"}
```

end

## [Life Cycle of a VTEP gateway](https://www.mankier.com/7/ovn-architecture#Description-Life_Cycle_of_a_VTEP_gateway)

A gateway is a chassis that forwards traffic between the OVN-managed part of a logical network and a physical VLAN, extending a tunnel-based logical network into a physical network.

The steps below refer often to details of the OVN and VTEP database schemas. Please see [ovn-sb(5)](https://www.mankier.com/5/ovn-sb), [ovn-nb(5)](https://www.mankier.com/5/ovn-nb) and [vtep(5)](https://www.mankier.com/5/vtep), respectively, for the full story on these databases.

1. A VTEP gateway’s life cycle begins with the administrator registering the VTEP gateway as a **Physical_Switch** table entry in the **VTEP** database. The **ovn-controller-vtep** connected to this VTEP database, will recognize the new VTEP gateway and create a new **Chassis** table entry for it in the **OVN_Southbound** database.
2. The administrator can then create a new **Logical_Switch** table entry, and bind a particular vlan on a VTEP gateway’s port to any VTEP logical switch. Once a VTEP logical switch is bound to a VTEP gateway, the **ovn-controller-vtep** will detect it and add its name to the *vtep_logical_switches* column of the **Chassis** table in the **OVN_Southbound** database. Note, the *tunnel_key* column of VTEP logical switch is not filled at creation. The **ovn-controller-vtep** will set the column when the correponding vtep logical switch is bound to an OVN logical network.
3. Now, the administrator can use the CMS to add a VTEP logical switch to the OVN logical network. To do that, the CMS must first create a new **Logical_Switch_Port** table entry in the **OVN_Northbound** database. Then, the *type* column of this entry must be set to "vtep". Next, the *vtep-logical-switch* and *vtep-physical-switch* keys in the *options* column must also be specified, since multiple VTEP gateways can attach to the same VTEP logical switch. Next, the *addresses* column of this logical port must be set to "unknown", it will add a priority 0 entry in "ls_in_l2_lkup" stage of logical switch ingress pipeline. So, traffic with unrecorded mac by OVN would go through the **Logical_Switch_Port** to physical network.
4. The newly created logical port in the **OVN_Northbound** database and its configuration will be passed down to the **OVN_Southbound** database as a new **Port_Binding** table entry. The **ovn-controller-vtep** will recognize the change and bind the logical port to the corresponding VTEP gateway chassis. Configuration of binding the same VTEP logical switch to a different OVN logical networks is not allowed and a warning will be generated in the log.
5. Beside binding to the VTEP gateway chassis, the **ovn-controller-vtep** will update the *tunnel_key* column of the VTEP logical switch to the corresponding **Datapath_Binding** table entry’s *tunnel_key* for the bound OVN logical network.
6. Next, the **ovn-controller-vtep** will keep reacting to the configuration change in the **Port_Binding** in the **OVN_Northbound** database, and updating the **Ucast_Macs_Remote** table in the **VTEP** database. This allows the VTEP gateway to understand where to forward the unicast traffic coming from the extended external network.
7. Eventually, the VTEP gateway’s life cycle ends when the administrator unregisters the VTEP gateway from the **VTEP** database. The **ovn-controller-vtep** will recognize the event and remove all related configurations (**Chassis** table entry and port bindings) in the **OVN_Southbound** database.
8. When the **ovn-controller-vtep** is terminated, all related configurations in the **OVN_Southbound** database and the **VTEP** database will be cleaned, including **Chassis** table entries for all registered VTEP gateways and their port bindings, and all **Ucast_Macs_Remote** table entries and the **Logical_Switch** tunnel keys.



## 其他引用

https://www.jianshu.com/p/3b1d7b2ddd88