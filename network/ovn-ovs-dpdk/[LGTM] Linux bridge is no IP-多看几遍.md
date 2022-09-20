## [LGTM] Linux bridge

## 1. Bridge

A bridge is a self-learning L2 forwarding device. [IEEE 802.1D](https://en.wikipedia.org/wiki/IEEE_802.1D) describes the bridge definition.

Bridge maintains a forwarding table, which stores `{src_mac, in_port}` pairs, and forwards packets (more accurately, frames) based on `dst_mac`.

For example, when a packet with `src_mac=ff:00:00:00:01` enters the bridge through port 1 (`in_port=1`), the bridge learns that host `ff:00:00:00:01` connected to it via port 1. Then it will add (if the entry is not cached yet) an entry `src_mac=ff:00:00:00:01, in_port=1` into its forwarding table. After that, if a packet with `dst_mac=ff:00:00:00:01` enters bridge, it decides that this packet is intended for host `ff:00:00:00:01`, and that host is connected to it via port 1, so it should be forwarded to port 1.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/connect_via_bridge.jpg)

Fig.1.1 Hosts connected by a bridge

From the forwarding process we could see, **a hypothetical bridge works entirely in L2**. But in real environments, a bridge is always configured with an IP address. This seems a paradox: **why we configure a L2 device with an IP address?**

The reason is that for real a bridge, it must provide some remote management abilities to be practically useful. So there must be an access port that we could control the bridge (e.g. restart) remotely.

Access ports are IP based, so it is **L3 ports**. This is is different from other ports which just work in L2 for traffic forwarding - for the latter no IPs are configured on them, they are **L2 ports**.

**L2 ports works in dataplane (DP), for traffic forwarding; L3 ports works in control plane (CP), for management.** They are different physical ports.

## 2. Linux Bridge

Linux bridge is a software bridge, it implements a subset of the ANSI/IEEE 802.1d standard. It manages both physical NICs on the host, as well as virutal devices, e.g. tap devices.

Physical port managed by Linux bridge are all dataplane ports (**L2 ports**), they just forward packets inside the bridge. We’ve mentioned that L2 ports do not have IPs configured on them.

So a problem occurs when all the physical ports are added to the linux bridge: **the host loses connection!**

To keep the host reachable, there are two solutions:

- leave at least one physical port as accessing port
- use virtual accessing port

**as long as they are managed by the OVS (or linux bridge), these physical ports will never be seen from the outside.**

### 2.1 Solution 1: Physical Access Port

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/phy_access_port.png)

Fig.2.1 Physical Access Port

In this solution, a physical port is reserved for host accessing, and not connected to Linux bridge. It will be configured with an IP (thus L3 port), and all CP traffic will be transmited through it. Other ports are connected to Linux bridge (L2 ports), for DP forwarding.

**pros:**

- CP/DP traffic isolation

- robustness

  even linux bridge misbehaves (e.g. crash), the host is still accessible

**cons:**

- resource under-utilized

  the access port is dedicated for accessing, which is wasteful

### 2.2 Solution 2: Virtual Accessing Port

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/virtual_access_port.png)

Fig.2.2 Virtual Access Port

In this solution, a virtual port is created on the host, and configured with an IP address, used as accessing port. Since all physical ports are connected to Linux host, to make this accessing port reachable from outside, it has to connected to Linux bridge, too!

Then, some triky things come.

First, CP traffic, will also be sent/received through DP ports, as physical ports are the only places that could interact with outside, and all physical ports are DP ports.

Secondly, all egress packets of this host, are with source MAC that is none of the physical port MACs. For example, if you ping this host, the ICPM reply packet will be sent out from one of the physical ports, **but**, **the source MAC of this packet is not the MAC of the physical port via which it is sent out.**

> 通过br0出去的数据包，source mac是br0的而不是NIC MAC

We will verify this later. Now let’s continue to OVS - a more powerful software bridge.

## OVS `internal port`

OVS is more powerful bridge than linux bridge, but since it is still a L2 bridge, some general bridge conventions it has to conform to.

Among those basic rules, one is that it should provide the ability to hold an IP for an OVS bridge: to be more clear, it should provide a similar functionality as Linux bridge’s virtual accessing port does. **With this functionality, even if all physical port are added to OVS bridge, the host could still be accessible from outside (as we discussed in secion 2, without this, the host will lose connection).**

OVS `internal port` is just for this purpose.

### Usage

When creating an `internal port` on OVS bridge, an IP could be configured on it, and the host is accessible by this IP address. Ordinary OVS users should not worry about the implementation details, they just need to know that `internal ports` act similar as linux tap devices.

Create an internal port `vlan1000` on bridge `br0`, and configure and IP on it:

```bash
$ ovs-vsctl add-port br0 vlan1000 -- set Interface vlan1000 type=internal
$ ifconfig vlan1000
$ ifconfig vlan1000 <ip> netmask <mask> up
```

### Some Experiments

We have **hostA**, and the OVS bridge on **hostA** looks like this:

```bash
root@hostA # ovs-vsctl show
ce8cf3e9-6c97-4c83-9560-1082f1ae94e7
    Bridge br-bond
        Port br-bond
            ## 创建bridge时默认自动添
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

root@hostA # ifconfig vlan1000
vlan1000  Link encap:Ethernet  HWaddr a6:f2:f7:d0:1d:e6  
          inet addr:10.18.138.168  Bcast:10.18.138.255  Mask:255.255.255.0
```

ping **hostA** from another host **hostB** (with IP 10.32.4.123), capture the packets on **hostA** and show the MAC address of L2 frames:

```bash
root@hostA # tcpdump -e -i vlan1000 'icmp'
10:28:24.176777 64:f6:9d:5a:bd:13 > a6:f2:f7:d0:1d:e6, 10.32.4.123   > 10.18.138.168: ICMP echo request
10:28:24.176833 a6:f2:f7:d0:1d:e6 > aa:bb:cc:dd:ee:ff, 10.18.138.168 > 10.32.4.123:   ICMP echo reply
10:28:25.177262 64:f6:9d:5a:bd:13 > a6:f2:f7:d0:1d:e6, 10.32.4.123   > 10.18.138.168: ICMP echo request
10:28:25.177294 a6:f2:f7:d0:1d:e6 > aa:bb:cc:dd:ee:ff, 10.18.138.168 > 10.32.4.123:   ICMP echo reply
```

We could see that the **source MAC (`a6:f2:f7:d0:1d:e6`) of ICMP echo reply packets** is just the **vlan1000’s address, not eth0 or eth1’s - although the packets will be sent out from either eth0, or eth1**. What this implies is that, from the outside view, ***hostA\*** is seen to have only one interface with MAC address `a6:f2:f7:d0:1d:e6`, and no matter how many physical ports are on ***hostA\***, as long as they are managed by the OVS (or linux bridge), these physical ports will never be seen from the outside.

## ovs port

端口Port与物理交换机的端口概念类似，Port是OVS Bridge上创建的一个虚拟端口，每个Port都隶属于一个Bridge。Port有以下几种类型

### 1）Normal

可以把操作系统中已有的网卡(物理网卡em1/eth0,或虚拟机的虚拟网卡tapxxx)挂载到ovs上，ovs会生成一个同名Port处理这块网卡进出的数据包。此时端口类型为Normal。

如下，主机中有一块物理网卡eth1，把其挂载到OVS网桥br-ext上，OVS会自动创建同名Port eth1。

```bash
## Bridge br-ext中出现port “eth1”
ovs-vsctl add-port br-ext eth1 
```

有一点要注意的是，挂载到OVS上的网卡设备不支持分配IP地址，**因此若之前eth1配置有IP地址，挂载到OVS之后IP地址将不可访问。**这里的网卡设备不只包括物理网卡，也包括主机上创建的虚拟网卡。

因为 L2 bridge是没有IP的



### 2）Internal：可以配置IP

Internal类型是OVS内部创建的虚拟网卡接口，每创建一个Port，OVS会自动创建一个同名接口(Interface)挂载到新创建的Port上。接口的概念下面会提到。

下面创建一个网桥br0，并创建一个internal类型的Port p0

```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 p0 -- set Interface p0 type=internal
```

查看结果

```bash
## 查看网桥br0
ovs-vsctl show br0
	Bridge "br0"
		fail_mode:secure
		Port "p0"
			Interface "p0"
				type:internal
		Port "br0"
			Interface "br0"
				type:internal
```

可以看到有两个Port。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port。在OVS中，只有”internal”类型的设备才支持配置IP地址信息，因此我们可以为br0接口配置一个IP地址，当然p0也可以配置IP地址

```bash
ip addr add 192.168.10.11/24 dev br0
ip link set br0 up
## 添加默认路由
ip route add default via 192.168.10.1 dev br0
```

上面两种Port类型区别在于，Internal类型会自动创建接口(Interface)，而Normal类型是把主机中已有的网卡接口添加到OVS中

### 3）Patch

当主机中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，从一个Patch Port收到的数据包会被转发到另一个Patch Port，类似于Linux系统中的veth。使用Patch连接的两个网桥跟一个网桥没什么区别，OpenStack Neutron中使用到了Patch Port。网桥br-ext中的Port phy-br-ext与br-int中的Port int-br-ext是一对Patch Port

可以使用ovs-vsctl创建patch设备，如下创建两个网桥br0,br1，然后使用一对Patch Port连接它们

```bash
ovs-vsctl add-br br0
ovs-vsctl add-br br1
ovs-vsctl \
-- add-port br0 patch0 --set interface patch0 type=patch options:peer=patch1 \
-- add-port br1 patch1 --set interface patch1 type=patch options:peer=patch0

```

结果如下

```bash
ovs-vsctl show
	Bridge "br0"
		Port "br0"
			Interface "br0"
				type:internal
		Port "patch0"
			Interface "patch0"
				type:patch
				options:{peer="patch1"}
	Bridge "br1"
		Port "br1"
			Interface "br1"
				type:internal
		Port "patch1"
			Interface "patch1"
				type:patch
				options:{peer="patch0"}
```

### 4）Tunnel

OVS中支持添加隧道(Tunnel)端口，常见隧道技术有两种gre或vxlan。隧道技术是在现有的物理网络之上构建一层虚拟网络，上层应用只与虚拟网络相关，以此实现的虚拟网络比物理网络配置更加灵活，并能够实现跨主机的L2通信以及必要的租户隔离。不同隧道技术其大体思路均是将以太网报文使用隧道协议封装，然后使用底层IP网络转发封装后的数据包，其差异性在于选择和构造隧道的协议不同。Tunnel在OpenStack中用作实现大二层网络以及租户隔离，以应对公有云大规模，多租户的复杂网络环境。

OpenStack是多节点结构，同一子网的虚拟机可能被调度到不同计算节点上，因此需要有隧道技术来保证这些同子网不同节点上的虚拟机能够二层互通，就像他们连接在同一个交换机上，同时也要保证能与其它子网隔离。

OVS在计算和网络节点上建立隧道Port来连接各节点上的网桥br-int，这样所有网络和计算节点上的br-int互联形成了一个大的虚拟的跨所有节点的逻辑网桥(内部靠tunnel id或VNI隔离不同子网)，这个逻辑网桥对虚拟机和qrouter是透明的，它们觉得自己连接到了一个大的br-int上。从某个计算节点虚拟机发出的数据包会被封装进隧道通过底层网络传输到目的主机然后解封装。

网桥br-tun中Port "vxlan-080058ca"就是一个vxlan类型tunnel端口。下面使用两台主机测试创建vxlan隧道

```bash
# 主机192.168.7.21上
ovs-vsctl add-br br-vxlan
# 主机192.168.7.23上
ovs-vsctl add-br br-vxlan
# 主机192.168.7.21上添加连接到7.23的Tunnel Port
ovs-vsctl add-port br-vxlan tun0 --set Interface tun0 type=vxlan options:remote_ip=192.168.7.23
# 主机192.168.7.23上添加连接到7.21的Tunnel Port
ovs-vsctl add-port br-vxlan tun0 --set Interface tun0 type=vxlan options:remote_ip=192.168.7.21
```

然后，两个主机上桥接到br-vxlan的虚拟机就像连接到同一个交换机一样，可以实现跨主机的L2连接，同时又完全与物理网络隔离。

## 原文

https://arthurchiao.art/blog/ovs-deep-dive-6-internal-port/