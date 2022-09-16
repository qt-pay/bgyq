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

## 原文

https://arthurchiao.art/blog/ovs-deep-dive-6-internal-port/