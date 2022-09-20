## OVS 

### 概述

OVS 只是一个单机软件，它并没有集群的信息，自己无法了解整个集群的虚拟网络状况，也就无法只通过自己来构建集群规模的虚拟网络。这就好比是单机的 Docker，而 OVN 就相当于是 OVS 的k8s，它提供了一个集中式的 OVS 控制器。这样可以从集群角度对整个网络设施进行编排。同时 OVN 也是新版 OpenStack 中 Neutron 的后端实现，基本可以认为未来的 OpenStack 网络都是通过OVN 来进行控制的。

#### 应用？

如果host_1上有一个vm1，vm1所属vxlan_01是switch_01，host_2上有一个vm2，vm2也所属vxlan_01，则host_1和host_2都有switch_01且有vxlan_01下所有vms信息。

### ovs 问题排查

vlan是交换机上的元数据，云平台下发虚拟机时给vm指定vlan就是给vm指定网络了，同vlan在一个网段就能通信
**ovs 命令列出所有port 然后抓vm在cvk(虚拟化物理机)上vswitch上的包来分析**

### vlan and ovs and libvirt

https://blog.scottlowe.org/2012/11/07/using-vlans-with-ovs-and-libvirt/

https://docs.openvswitch.org/en/latest/howto/vlan/

### ovs port mode

在默认模式下（VLAN_mode没被设置），如果指定了端口的tag属性，那么这个端口就工作在access模式，并且其trunk属性的值应该保持为空。否则，这个port就工作在trunk模式下，如果trunk被指定，则使用指定的trunk值。

#### trunk

trunk模式的端口允许传输所有在其trunk属性中指定的那些VLAN对应的数据包。其他VLAN的数据包就会被丢弃。从trunk模式的端口中进入的数据包其VLAN ID不会发生变化。如果进入的数据包不含有VLAN ID，则该数据包进入交换机后的VLAN为0。从trunk模式的端口出去的数据包，如果VLAN ID不为空，则依然保持该VLAN ID，如果VLAN ID为空，则出去后不再包含802.1Q头部。

#### **access**

access模式的端口只允许不带VLAN的数据包进入，不管数据包的VLAN ID是否与其tag相同，只要含有VLAN ID，这个数据包都会被端口drop。数据包进入access端口后会被打上和端口tag相同的VLAN，而再从access端口出去时，数据包的VLAN会被删除，也就是说从access的端口出去的数据包和进来时一样是不带VLAN的。

#### **native-tagged**

native-tagged端口类似于trunk端口，它们之间的区别是如果进入native-tagged端口的数据包不含有802.1Q头部，即没有指定VLAN，那么该数据包会被当作native VLAN，其VLAN ID由端口的tag指定。

#### native-untagged

native-untagged端口类似于native-tagged端口，不同点是native VLAN中的数据包从native-untagged端口出去时，会被去掉802.1Q头部。

### L2vni and L3vni

L3 VNI与二层VNI是完全不同的。L2 VNI映射的是一个VLAN，或者一个子网；L3 VNI映射的是一个VRF。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vxlan-L2-and-L3-traffic-flow.jpg)

### OVS：as Docker only is L2 drive

https://arthurchiao.art/blog/ovs-deep-dive-0-overview/

OVS可以说是网络虚拟化里最重要的工业级开源产品，OVS模仿物理交换机设备的工作流程，实现了很多物理交换机当时才支持的许多网络功能。

vSwitch负责连接vNIC与物理网卡，同时也桥接同一物理服务器内的各个VM的vNIC。

ovs-vswitchd - Open vSwitch daemon

A daemon that manages and controls any number of Open vSwitch switches on the local machine.

ovs-vswitchd switches may be configured with any of the following features:

•      L2 switching with MAC learning.

•      NIC bonding with automatic fail-over and source MAC-based
       TX load balancing ("SLB").

•      802.1Q VLAN support.

•      Port mirroring, with optional VLAN tagging.

•      NetFlow v5 flow logging.

•      sFlow(R) monitoring.

•      Connectivity to an external OpenFlow controller, such as NOX.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovs-arch-1.webp)

As depicted in Fig.2.1, OVS is composed of three components:

- vswitchd
  - user space program, ovs deamon
  - tools: `ovs-appctl`
- ovsdb-server
  - user space program, database server of OVS
  - tools: `ovs-vsctl`, `ovs-ofctl`

- kernel module (datapath)
  - kernel space module, OVS packet forwarder
  - tools: `ovs-dpctl`

#### OVS Daemon

`ovs-vswitchd` is **the main Open vSwitch userspace program**. It reads the desired Open vSwitch configuration from ovsdb-server over an IPC channel and passes this configuration down to the ovs bridges (implemented as a library called `ofproto`). It also passes certain status and statistical information from ovs bridges back into the database.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vswitchd_ovsdb_ofproto.png)

#### OVSDB

Some transient configurations, e.g. flows, are stored in datapaths and vswitchd. Persistent configurations are stored in ovsdb, which survives reboot.

`ovsdb-server` provides RPC itnerfaces to OVSDB. It supports JSON-RPC client connections over active or passive TCP/IP or Unix domain sockets.

`ovsdb-server` runs either as a backup server, or as an active server. Only the active server handles transactions that will change the OVSDB.

#### Datapath

Datapath is the main packet forwarding module of OVS, implemented in kernel space for high performance. It caches OpenFlow flows, and execute actions on received packets which match specific flow(s). If no flow is matched for one packet, the packet will be delivered to userspace program `ovs-vswitchd`. Usually, `ovs-vswitchd` will issue an new flow to datapath which will be used to handle subsequent packets of this type. The high performance comes from the fact that most packets will match flows successfully in datapath, thus will be processed directly in kernel space.

#### ovs流表处理流程

`ovs`处理流表的过程是：
1.ovs的datapath接收到从ovs连接的某个网络设备发来的数据包，从数据包中提取源/目的IP、源/目的MAC、端口等信息。
2.ovs在**内核状态**下查看流表结构（通过Hash），观察是否有缓存的信息可用于转发这个数据包。
3.内核不知道如何处置这个数据包会将其发送给用户态的ovs-vswitchd。
4.ovs-vswitchd进程接收到upcall后，将检查数据库以查询数据包的目的端口是哪里，然后告诉内核应该将数据包转发到哪个端口，例如eth0。
5.内核执行用户此前设置的动作。即内核将数据包转发给端口eth0，进而数据被发送出去。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovs-arch-2.png)

The main components that an OVS distribution provides are:

- **ovs-vswitchd**, a daemon that implements and controls the switch on the local machine, along with a companion Linux kernel module for flow-based switching. This daemon performs a lookup for the configuration data from the database server to set up the data paths.
- **ovsdb-server**, a lightweight database server that ovs-vswitchd queries to obtain its configuration, which includes the interface, the flow content, and the Vlans. It provides RPC interfaces to the vswitch databases.
- **ovs-dpctl**, a tool for configuring the switch kernel module and controlling the forwarding rules.
- **ovs-vsctl**, a utility for querying and updating the configuration of ovs-vswitchd. It updates the index in ovsdb-server.
- **Ovs-appctl**, is mainly a utility that sends commands to running Open vSwitch daemons (usually not used).
- **Scripts and specs** for building RPMs for Citrix XenServer and Red Hat Enterprise Linux. The XenServer RPMs allow Open vSwitch to be installed on a Citrix XenServer host as a drop-in replacement for its switch, with additional functionality.



#### ovs netdev

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/netdev_rx_tx.png)

A network device (e.g. physical NIC) has two ends/parts, one end works in kernel, which is responsible for sending/receiving, and the other end in userspace, which manages the kernel parts, such as changing device MTU size, disabling/enabling queues, etc. The communication between kernel and userspace space is usually through [netlink](https://arthurchiao.art/blog/ovs-deep-dive-4-patch-port/) or [ioctl](https://arthurchiao.art/blog/ovs-deep-dive-4-patch-port/) (deprecated).

For virtual network devices, such as TUN/TAP, the working process is similar, execpt that the packets a TAP device receives is not from outside, but from the userspace; and the packets a TAP device sends does not go to outside, but goes to userspace.

In OVS, A `struct netdev` instance represents a network device in OVS userspace, it is used for controlling the kernel end of this device, it maybe a physical NIC, a TAP device, or other types.

#### OVS internet port

Access ports are IP based, so it is **L3 ports**. This is is different from other ports which just work in L2 for traffic forwarding - for the latter no IPs are configured on them, they are **L2 ports**.

L2 ports works in dataplane (DP), for traffic forwarding; L3 ports works in control plane (CP)

OVS is more powerful bridge than linux bridge, but since it is still a L2 bridge, some general bridge conventions it has to conform to.

> OVS创建L2 bridge， bridge是没有IP的，所以将physical NIC加入OVS L2 bridge，会导致网卡IP消失

Among those basic rules, one is that it should provide the ability to hold an IP for an OVS bridge: to be more clear, it should provide a similar functionality as Linux bridge’s virtual accessing port does. With this functionality, even if all physical port are added to OVS bridge, the host could still be accessible from outside (as we discussed in secion 2, without this, the host will lose connection).

OVS `internal port` is just for this purpose.

#####  Usage

When creating an `internal port` on OVS bridge, an IP could be configured on it, and the host is accessible by this IP address. Ordinary OVS users should not worry about the implementation details, they just need to know that `internal ports` act similar as linux tap devices.

Create an internal port `vlan1000` on bridge `br0`, and configure and IP on it:

```bash
$ ovs-vsctl add-port br0 vlan1000 -- set Interface vlan1000 type=internal

$ ifconfig vlan1000

$ ifconfig vlan1000 <ip> netmask <mask> up
```

##### Some Experiments

We have ***hostA\***, and the OVS bridge on ***hostA\*** looks like this:

```bash
root@hostA # ovs-vsctl show
ce8cf3e9-6c97-4c83-9560-1082f1ae94e7
    Bridge br-bond
        Port br-bond
            Interface br-bond
                type: internal
        Port "vlan1000"
            tag: 1000
            Interface "vlan1000"
                type: internal
        Port "bond1"
            Interface "eth1"
            Interface "eth0"
    ovs_version: "2.3.1"
```

Two physical ports *eth0* and *eth1* is added to the bridge (bond), two internal ports *br-bond* (the default one of this bridge, not used) and *vlan1000* (we created it). We make *vlan1000* as the accessing port of this host by configuring an IP address on it:

```bash
root@hostA # ifconfig vlan1000 10.18.138.168 netmask 255.255.255.0 up

## virtual L3 Port MAC and IP
root@hostA # ifconfig vlan1000
vlan1000  Link encap:Ethernet  HWaddr a6:f2:f7:d0:1d:e6  
          inet addr:10.18.138.168  Bcast:10.18.138.255  Mask:255.255.255.0
```

ping ***hostA\*** from another host ***hostB\*** (with IP 10.32.4.123), capture the packets on ***hostA\*** and show the MAC address of L2 frames:

```bash
root@hostA # tcpdump -e -i vlan1000 'icmp'
10:28:24.176777 64:f6:9d:5a:bd:13 > a6:f2:f7:d0:1d:e6, 10.32.4.123   > 10.18.138.168: ICMP echo request
10:28:24.176833 a6:f2:f7:d0:1d:e6 > aa:bb:cc:dd:ee:ff, 10.18.138.168 > 10.32.4.123:   ICMP echo reply
10:28:25.177262 64:f6:9d:5a:bd:13 > a6:f2:f7:d0:1d:e6, 10.32.4.123   > 10.18.138.168: ICMP echo request
10:28:25.177294 a6:f2:f7:d0:1d:e6 > aa:bb:cc:dd:ee:ff, 10.18.138.168 > 10.32.4.123:   ICMP echo reply
```

We could see that the **source MAC (`a6:f2:f7:d0:1d:e6`) of ICMP echo reply packets** is just the **vlan1000’s address, not eth0 or eth1’s - although the packets will be sent out from either eth0, or eth1**. What this implies is that, from the outside view, ***hostA\*** is seen to have only one interface with MAC address `a6:f2:f7:d0:1d:e6`, and no matter how many physical ports are on ***hostA\***, as long as they are managed by the OVS (or linux bridge), these physical ports will never be seen from the outside.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/bridge_managed_host_outside_view.png)

#### neutron-openvswitch-agent：ML2 plugin

neutron-openvswitch-agent作为adaptor根据来自neutron-server的消息（neutron-DB）对ovs进行管理，增加删除虚拟端口、创建vlan、决定转发策略等等。

neutron-server –> neutron-openvswitch –> ovs

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/openstack-openvswitch-agent.png)

其他暂且不用深入那么细节，直接理解思路7、9——初始化之后agent会向server请求已有的网络设备（device就可以简单理解为配置的虚拟端口啦）的详细信息来对ovs进行管理，之后对ovs上的device信息提交给server进行更新。在ovs-agent启动初始化之后，在neutron-server跟ovs-agent频繁交互message周期检测ovs上的端口状态（可以在ovs-agent端tail -f /var/log/neutron/openvswitch-agent.log看到周期交互的相关细节）。在周期检查的背后，实际上是ovs-agen t代码进入主函数之后持续进行rpc_loop()循环状态检查、更新


```bash
# 安装网络组件
$ yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch

# 编辑[ml2]小节，启用flat和generic routing encapsulation (GRE)网络类型驱动，配置GRE租户网络和OVS驱动机制。
[ml2]
...
type_drivers= flat,gre
tenant_network_types= gre
mechanism_drivers= openvswitch

# 
$ systemctl enable openvswitch.service
ln -s '/usr/lib/systemd/system/openvswitch.service' '/etc/systemd/system/multi-user.target.wants/openvswitch.service'
$ systemctl start openvswitch.service
```

ML2 plugin被社区提出来的目的是用于取代所有的Core Plugin，它采用了更加灵活的结构进行实现。作为一个Core plugin，ML2 自然会实现network/subnet/port这三种核心资源，同时它也实现了包括Port Binding等在内的部分扩展资源。

ML2实现了网络拓扑类型（Flat、VLAN、VXLAN、GRE）和底层虚拟网络（linux bridge、OVS）分离的机制，并分别通过Driver的形式进行扩展。其中，不同的网络拓扑类型对应着type driver，由type manager管理，不同的网络实现机制对应着Mechanism Driver（比如Linux bridge、OVS、NSX等），由Mechanism Manager管理。



没有OVN的模式：

每台机器装了一个Openvswitch，然后每台机器装了neutron-openvswitch-agent。

neutron-openvswitch-agent从neutron获取网络配置，然后同步到ovs。

#### OVS实践

##### Docker容器动手实践-单机

默认使用linux bridge + veth pair 实现多个容器直接的通信,可以将linux Bridge 换成OVS的vswitch + patch port（openvswitch internal ports）类实现多个容器的通信。

- 使用 OVS patch ports：性能更好
- 不要使用 Linux veth pairs：它会带来明显的性能下降

理论知识了解后就能动手来实践了，由于环境资源的限制，只能用更轻量级的docker容器来做实验。

环境拓扑

使用docker创建四个不带网卡的容器

厉害了！！

```bash
docker run -t -i -d --name vm01 --net=none --privileged sshd-centos7 /bin/bash
docker run -t -i -d --name vm02 --net=none --privileged sshd-centos7 /bin/bash
docker run -t -i -d --name vm03 --net=none --privileged sshd-centos7 /bin/bash
docker run -t -i -d --name vm04 --net=none --privileged sshd-centos7 /bin/bash
```

创建OVS网桥

```bash
ovs-vsctl add-br vswitch0
ovs-vsctl add-br vswitch1
```

使用ovs-docker工具给容器添加网卡到ovs网桥

```bash
$ wget https://raw.githubusercontent.com/openvswitch/ovs/master/utilities/ovs-docker
$ chmod +x ovs-docker
ovs-docker add-port vswitch0 eth0 vm01 --ipaddress=192.168.1.2/24
ovs-docker add-port vswitch1 eth0 vm02 --ipaddress=192.168.1.3/24
ovs-docker add-port vswitch0 eth0 vm03 --ipaddress=192.168.1.4/24
ovs-docker add-port vswitch1 eth0 vm04 --ipaddress=192.168.1.5/24

## vm01和vm03 在s1上, vm02和vm04在s2上。
$ ovs-vsctl add-port br0 vlan1000 -- set Interface vlan1000 type=internal
```

连接vswitch0和vswitch1

网桥与网桥的连接需要依靠**一对patch类型的端口**

0.0 openvswitch internal ports 类似 veth pairs

```bash
$ ovs-vsctl add-port vswitch0 patch_to_vswitch1
ovs-vsctl: Error detected while setting up 'patch_to_vswitch1'.  See ovs-vswitchd log for details.

$ ovs-vsctl add-port vswitch1 patch_to_vswitch0
ovs-vsctl: Error detected while setting up 'patch_to_vswitch0'.  See ovs-vswitchd log for details.


$ ovs-vsctl set interface patch_to_vswitch1 type=patch
$ ovs-vsctl set interface patch_to_vswitch0 type=patch

$ ovs-vsctl set interface patch_to_vswitch0 options:peer=patch_to_vswitch1
$ ovs-vsctl set interface patch_to_vswitch1 options:peer=patch_to_vswitch0
```

创建完这对patch类型的端口后，两个交换机就可以相互连通了。ovs端口默认是trunk模式，且可以trunk所有的VLAN tag，所以在这对patch类型的端口上可以通过打上了任意tag的流量。

Interface types

| 类型     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| Normal   | 用户可以把操作系统中的网卡绑定到ovs上，ovs会生成一个普通端口处理这块网卡进出的数据包。 <br />Normal is default value. ovs-vsctl list interface 显示 type: "" |
| Internal | 端口类型为internal时，ovs会创建一块虚拟网卡，端口收到的所有数据包都会交给该网卡，发出的包会通过该端口交给ovs。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port. |
| Patch    | 当机器中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，在两个网桥之间交换数据。 |
| Tunnel   | 隧道端口是一种虚拟端口，支持使用gre或vxlan等隧道技术与位于网络上其他位置的远程端口通讯。<br />ovs-vsctl add-port br-tun vxlan0 -- set Interface vxlan0 type=vxlan options:remote_ip=IP<br />Interface type： vxlan |

Interface是连接到Port的网络接口设备，是OVS与外部交换数据包的组件，在通常情况下，Port和Interface是一对一的关系，只有在配置Port为 bond模式后，Port和Interface是一对多的关系。这个网络接口设备可能是创建`Internal`类型Port时OVS自动生成的虚拟网卡，也可能是系统的物理网卡或虚拟网卡(TUN/TAP)挂载在ovs上。 OVS中只有”Internal”类型的网卡接口才支持配置IP地址

> `ovs-vsctl add-port br0 vlan1000 -- set Interface vlan1000 type=internal`

`Interface`是一块网络接口设备，负责接收或发送数据包，Port是OVS网桥上建立的一个虚拟端口，`Interface`挂载在Port上。

查看ovs网桥的所有端口

```bash
$ ovs-vsctl show
270e167e-2f5e-4b46-be62-2c9c142cdd9f
    Bridge "vswitch1"
        Port "patch_to_vswitch0"
            Interface "patch_to_vswitch0"
                type: patch
                options: {peer="patch_to_vswitch1"}
        Port "vswitch1"
            Interface "vswitch1"
                type: internal
        Port "6eef85c4718a4_l"
            Interface "6eef85c4718a4_l"
        Port "39c0765e1c964_l"
            Interface "39c0765e1c964_l"
    Bridge "vswitch0"
        Port "patch_to_vswitch1"
            Interface "patch_to_vswitch1"
                type: patch
                options: {peer="patch_to_vswitch0"}
        Port "vswitch0"
            Interface "vswitch0"
                type: internal
        Port "2720669d9c064_l"
            Interface "2720669d9c064_l"
        Port "f4e1ae64ef704_l"
            Interface "f4e1ae64ef704_l"
    ovs_version: "2.5.0"
```

测试容器的连通性

使用vm01 ping其他几个容器

```bash
# docker attach vm01
[root@ffa59e57d97f /]# ping 192.168.1.3 -c 3  
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.230 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 192.168.1.3: icmp_seq=3 ttl=64 time=0.062 ms

--- 192.168.1.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.057/0.116/0.230/0.080 ms
[root@ffa59e57d97f /]# ping 192.168.1.4 -c 3   
PING 192.168.1.4 (192.168.1.4) 56(84) bytes of data.
64 bytes from 192.168.1.4: icmp_seq=1 ttl=64 time=0.241 ms
64 bytes from 192.168.1.4: icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from 192.168.1.4: icmp_seq=3 ttl=64 time=0.058 ms

--- 192.168.1.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.053/0.117/0.241/0.087 ms
[root@ffa59e57d97f /]# ping 192.168.1.5 -c 3   
PING 192.168.1.5 (192.168.1.5) 56(84) bytes of data.
64 bytes from 192.168.1.5: icmp_seq=1 ttl=64 time=0.209 ms
64 bytes from 192.168.1.5: icmp_seq=2 ttl=64 time=0.054 ms
64 bytes from 192.168.1.5: icmp_seq=3 ttl=64 time=0.055 ms

--- 192.168.1.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.054/0.106/0.209/0.072 ms
```

vm01可以和vm02、vm03、vm04相互通信。

为port设置VLAN tag

为vm01、vm02对应的ovs端口设置tag为100，为vm03、vm04对应的ovs端口设置tag为200

```bash
# ovs-vsctl list interface 6eef85c4718a4_l | grep container_id
external_ids        : {container_id="vm04", container_iface="eth0"}
# ovs-vsctl set port 6eef85c4718a4_l tag=200

# ovs-vsctl list interface 39c0765e1c964_l | grep container_id
external_ids        : {container_id="vm02", container_iface="eth0"}
# ovs-vsctl set port 39c0765e1c964_l tag=100

# ovs-vsctl list interface 2720669d9c064_l | grep container_id
external_ids        : {container_id="vm03", container_iface="eth0"}
# ovs-vsctl set port 2720669d9c064_l tag=200
 
# ovs-vsctl list interface f4e1ae64ef704_l | grep container_id               
external_ids        : {container_id="vm01", container_iface="eth0"}
# ovs-vsctl set port f4e1ae64ef704_l tag=100
```

测试容器连通性

设置完tag后，理论上，设置了相同tag的端口之间才能相互通信，即vm01能和vm02通信，vm03能和vm04通信，而vm01、vm02无法和vm03、vm04通信。

先测试vm01

```bash
# docker attach vm01
[root@ffa59e57d97f /]# ping 192.168.1.3 -c 1
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.211 ms
^C
--- 192.168.1.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.211/0.211/0.211/0.000 ms


[root@ffa59e57d97f /]# ping 192.168.1.4 -c 1  
PING 192.168.1.4 (192.168.1.4) 56(84) bytes of data.
From 192.168.1.2 icmp_seq=1 Destination Host Unreachable
^C
--- 192.168.1.4 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms


[root@ffa59e57d97f /]# ping 192.168.1.5 -c 1 
PING 192.168.1.5 (192.168.1.5) 56(84) bytes of data.
From 192.168.1.2 icmp_seq=1 Destination Host Unreachable
^C
--- 192.168.1.5 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

vm01只能ping通设置了相同tag的vm02，其他容器则无法ping通。

再测试vm03

```bash
# docker attach vm03
[root@c17ff55307d7 /]# ping 192.168.1.4
PING 192.168.1.4 (192.168.1.4) 56(84) bytes of data.
64 bytes from 192.168.1.4: icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from 192.168.1.4: icmp_seq=2 ttl=64 time=0.040 ms
^C
--- 192.168.1.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.033/0.036/0.040/0.007 ms


[root@c17ff55307d7 /]# ping 192.168.1.5
PING 192.168.1.5 (192.168.1.5) 56(84) bytes of data.
64 bytes from 192.168.1.5: icmp_seq=1 ttl=64 time=0.269 ms
64 bytes from 192.168.1.5: icmp_seq=2 ttl=64 time=0.057 ms
^C
--- 192.168.1.5 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.057/0.163/0.269/0.106 ms


[root@c17ff55307d7 /]# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1000ms

[root@c17ff55307d7 /]# 
```

vm03也只能ping通设置了相同tag的vm04

设置网桥之间的patch端口只trunk100

无论是vm01与vm02通信，还是vm03与vm04通信都需要经过两个网桥之间的patch类型的端口，默认是trunk所有的tag的，现在设置它们只trunk tag100。

```bash
ovs-vsctl set port patch_to_vswitch1 VLAN_mode=trunk
ovs-vsctl set port patch_to_vswitch0 VLAN_mode=trunk

ovs-vsctl set port patch_to_vswitch0 trunk=100
ovs-vsctl set port patch_to_vswitch1 trunk=100
```

测试容器之间的连通性

使用vm01 ping vm02

```bash
# docker attach vm01
[root@ffa59e57d97f /]# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.222 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=0.051 ms
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.051/0.136/0.222/0.086 ms
```

由于vm01、vm02对应端口的tag为100，在patch端口的trunk范围内，所以vm01依然可以ping通vm02

使用vm03 ping vm04

```unknown
# docker attach vm03
[root@c17ff55307d7 /]# ping 192.168.1.5
PING 192.168.1.5 (192.168.1.5) 56(84) bytes of data.
^C
--- 192.168.1.5 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

vm03和vm04对应端口的tag为200，不在patch端口的trunk范围内，所以vm03已经无法ping通vm04

查看flow table

```bash
$ ovs-ofctl dump-flows vswitch0
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=2504.456s, table=0, n_packets=292, n_bytes=36188, idle_age=410, priority=0 actions=NORMAL
 
$ ovs-ofctl dump-flows vswitch1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=46151.880s, table=0, n_packets=155, n_bytes=10998, idle_age=63, priority=0 actions=NORMAL
```

没有转发用的flow table存在，所以猜测普通VLAN tag的转发是不需要依靠flow的，但是支持open flow又是ovs比较核心的东西，所以在后面的文章中会和大家一起分享我对flow相关的学习内容。

##### docker multi-host communicate with openvswitch

 啧啧，实验都失败了，我感觉是因为我的实验环境在OpenStack上，一些包，被默认的流表规则拦下了。

类似，k8s的Calico，明明一个网段的VM却需要`ipip mode`,实现二层的连通性。

Gre：

<https://docker-k8s-lab.readthedocs.io/en/latest/docker/docker-ovs.html>

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovs-gre-docker.png)

vxlan：

<https://blog.csdn.net/shudaqi2010/article/details/76618970>

<https://cloud.tencent.com/info/013fef2b859bc306d40b8940034cf3a3.html>

```bash
   host1:10.1.86.203 
   ovs0
    |
    |-veth1 <-------> eth1 192.168.0.3  con3
    |
    |-vxlan1-------------+
    |                    |
                         |
    host2:10.1.86.204    |
    ovs0                 |
     |                   |
     |-vxlan1------------+
     |
     |-veth1 <--------> eth1 192.168.0.4 con4
     |
```

end

### OVS with dpdk:fire:

ovs-vswitchd主要包含ofproto、dpif、netdev模块。ofproto模块实现openflow的交换机；dpif模块抽象一个单转发路径；netdev模块抽象网络接口（无论物理的还是虚拟的）。 openvswitch.ko主要由数据通路模块组成，里面包含着流表。流表中的每个表项由一些匹配字段和要做的动作组成。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ovs-with-dpdk-datapath.png)

DPDK加速的OVS数据流转发的大致流程如下：

1）OVS的ovs-vswitchd接收到从OVS连接的某个网络端口发来的数据包，从数据包中提取源/目的IP、源/目的MAC、端口等信息。 

2）OVS在用户态查看精确流表和模糊流表，如果命中，则直接转发。 

3）如果还不命中，在SDN控制器接入的情况下，经过OpenFlow协议，通告给控制器，由控制器处理。

4）控制器下发新的流表，该数据包重新发起选路，匹配；报文转发，结束。

DPDK加速的OVS与原始OVS的区别在于，从OVS连接的某个网络端口接收到的报文不需要openvswitch.ko内核态的处理，报文通过DPDK PMD驱动直接到达用户态ovs-vswitchd里。

OVS使用 DPDK的好处

- 支持轻量型协议栈；
- 兼容virtio，vhost-net和基于DPDK的vhost-user加速技术；
- 支持Vxlan功能、端口Bonding功能；
- 支持OpenFlow1.0&1.3；
- 支持Meter功能、镜像功能；
- 支持VM热迁移功能；
- 性能较内核OVS可提升8~9倍。

#### demo

```bash
# 加载VFIO内核模块

modprobe vfio
modprobe vfio-pci
ifconfig eth0 down ; ifconfig eth1 down
mkdir /etc/sysconfig/network-scripts/back ; mv /etc/sysconfig/network-scripts/{ifcfg-eth0,ifcfg-eth1}   /etc/sysconfig/network-scripts/back



nic1_businfo=`ethtool -i eth0 | grep bus-info | awk '{print $2}'`
nic2_businfo=`ethtool -i eth1 | grep bus-info | awk '{print $2}'`


# 绑定dpdk网卡
dpdk-devbind -b vfio-pci "$nic1_businfo"
dpdk-devbind -b vfio-pci "$nic2_businfo"



#dpdk添加开机启动
chmod +x /etc/rc.d/rc.local && echo  "modprobe vfio" >>/etc/rc.local
echo  "modprobe vfio-pci" >>/etc/rc.local
echo  "dpdk-devbind -b vfio-pci "$nic1_businfo"" >>/etc/rc.local
echo  "dpdk-devbind -b vfio-pci "$nic2_businfo"" >>/etc/rc.local



# 两个网卡配置是bond

ovs-vsctl add-bond br-tun business_bond p0 p1  lacp=active  bond_mode=balance-tcp \

-- set Interface p0 type=dpdk options:dpdk-devargs="$nic1_businfo" \

-- set Interface p1 type=dpdk options:dpdk-devargs="$nic2_businfo" \

-- set Interface p0 mtu_request=2000 \

-- set Interface p1 mtu_request=2000



# 物理网卡根据DPDK核数开启多队列

ovs-vsctl set Interface   p0 options:n_txq=4
ovs-vsctl set Interface   p0 options:n_rxq=4
ovs-vsctl set Interface   p1 options:n_txq=4
 ovs-vsctl set Interface   p1 options:n_rxq=4

```

end



### 引用

1. https://www.cnblogs.com/laolieren/p/learn_ovn.html
2. https://feisky.gitbooks.io/sdn/content/ovs/ovn.html
3. https://blog.csdn.net/qq_31331027/article/details/81448313
4. https://www.cnblogs.com/laolieren/p/ovn-architecture.html
5. https://feisky.gitbooks.io/sdn/content/ovs/ovn-kubernetes.html
6. https://www.sdnlab.com/20693.html
7. https://arthurchiao.art/blog/ovs-deep-dive-4-patch-port/
8. https://www.cnblogs.com/laolieren/p/ovn-architecture.htm