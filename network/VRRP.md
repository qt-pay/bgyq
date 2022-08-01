## VRRP

### VRRP协议

虚拟路由冗余协议（IETF开发，公有）

Virtual Router Redundancy Protocol，虚拟路由器(叫网关更接地气)冗余协议

* 将多个物理网关加入到备份组中，形成一台虚拟网关，承担物理网关功能
* 只要备份组中仍有一台物理网关正常工作，虚拟网关就仍正常工作
* vrrp2针对ipv4，vrrp3针对ipv6

#### 背景

通常，一个网段内只有一个网关。因此一旦网关出现故障，该网段就被孤立。

#### 原理

原理：通过虚拟IP、虚拟MAC地址在多个网关间切换。

> 虚拟IP可以和某个实际IP相同
>
> ![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrrp-25.jpg)

虚拟mac生产规则

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrrp-virtual-mac.jpg)

核心：谁是Master谁来相应对虚拟IP地址的ARP请求，转发发往虚拟网关的数据包。

#### 解决问题

网关冗余，一个组中有一个活跃路由器，实现负载分担时，可配置多个热备组。

解决网关单点，避免用户切换网关配置

#### 应用

如下，网关采用虚拟IP地址，也会生成虚拟MAC。如果判断是否采用了vrrp，可以使用traceroute命令，这样会显示真正承担网关角色的是哪个IP

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vrrp-vip.jpg)



### VRRP 高可用VLAN gw

Imagine VLAN a virtual equivalent of a physical switch. Computers in a VLAN is like getting a physical switch (a “dumb” one, without management) and connecting all computers to it.

Does that have a default gateway? No connection whatsoever. The gateway is a concept present on a higher OSI level. **VLANs (same as switches) exist on OSI layer 2.**

So there is no connection between a VLAN and a gateway other than the fact that they are part of the same structure.

VLANs doesn’t have it’s own default gateway. For inter VLAN routing, a VLAN is connected to a Router( Actually the Switch that creates the VLANs is connected to the router). So if a host in a VLAN needs to connect another host outside(another VLAN or Internet) its VLAN, the connection is established through the router. The router checks if tha destination host is from a VLAN which connected to it or outside the main network. If it is an outsider the router uses its default gateway address to router the traffic to ISP.

跨vlan 通讯才会涉及到gateway，而是平时pc配置的网关`192.168.1.254`，其实是个VIP，它可能是两个交换机上的`192.168.1.253`和`192.168.1.252`。

**通常，同一网段内的所有主机上都存在一条相同的、以网关为下一跳的缺省路由。主机发往其他网段的报文将通过缺省路由发往网关，再由网关进行转发，从而实现主机与外部网络的通信。当网关发生故障时，本网段内所有以网关为缺省路由的主机将无法与外部网络通信。增加出口网关是提高系统可靠性的常见方法，此时如何在多个出口之间进行选路就成为需要解决的问题。**

> keepalived就是基于vrrp实现的。网络yyds...

VRRP的出现很好的解决了这个问题。VRRP能够在不改变组网的情况下，采用将多台路由设备组成一个虚拟路由器，通过配置虚拟路由器的IP地址为默认网关，实现默认网关的备份。当网关设备发生故障时，VRRP机制能够选举新的网关设备承担数据流量，从而保障网络通信的可靠性。

在配置VRRP备份组内各交换机时，建议将Backup配置为立即抢占，即不延迟（延迟时间为0），而将Master配置为延时抢占，并且配置15秒以上的延迟时间。这样配置的目的是为了在网络环境不稳定时，在上下行链路的状态恢复一致性期间等待一定时间，避免由于双方频繁抢占导致用户设备学习到错误的Master设备地址而导致流量中断问题。

- 抢占模式：在抢占模式下，如果Backup设备的优先级比当前Master设备的优先级高，则主动将自己切换成Master。
- 非抢占模式：在非抢占模式下，只要Master设备没有出现故障，Backup设备即使随后被配置了更高的优先级也不会成为Master设备。

### VRRP高可用路由网关

