## [LGTM] 网卡bond和交换机配置

### Summary

网卡bond mode 1、5、6不需要交换机设置

网关bond mode 0、2、3、4需要交换机设置

### bond mode

* mode=0(balance-rr)(平衡抡循环策略)

* mode=1(active-backup)(主-备份策略)

* mode=2(balance-xor)(平衡策略)

* mode=3(broadcast)(广播策略)

* mode=4(802.3ad)(IEEE 802.3ad 动态链接聚合)

  表示支持802.3ad协议，和交换机的聚合LACP方式配合（需要xmit_hash_policy）.标准要求所有设备在聚合操作时，要在同样的速率和双工模式，而且，和除了balance-rr模式外的其它bonding负载均衡模式一样，任何连接都不能使用多于一个接口的带宽。
  特点：创建一个聚合组，它们共享同样的速率和双工设定。根据802.3ad规范将多个slave工作在同一个激活的聚合体下。
  外出流量的slave选举是基于传输hash策略，该策略可以通过xmit_hash_policy选项从缺省的XOR策略改变到其他策略。需要注意的 是，并不是所有的传输策略都是802.3ad适应的，

* mode=5(balance-tlb)(适配器传输负载均衡)

* mode=6(balance-alb)(适配器适应性负载均衡)

(1) 轮询策略(Round-robin policy)，模式代号是0。该策略是按照设备顺序依次传输数据包，直到最后一个设备。这种模式提供负载均衡和容错能力。
(2) 活动备份策略(Active-backup policy)，模式代号是1。该策略只有一个设备处理数据，当它宕机的时候就会由备份代替，仅提供容错能力。
(3) 异或策略(XOR policy)，模式代号是2。该策略是根据MAC地址异或运算的结果来选择传输设备，提供负载均衡和容错能力。
(4) 广播策略(Broadcast policy)，模式代号是3。该策略通过全部设备来传输所有数据，提供容错能力。
(5) IEEE 802.3ad 动态链接聚合(IEEE 802.3ad Dynamic link aggregation)，模式代号是4。该策略通过创建聚合组来共享相同的传输速度，需要交换机也支持 802.3ad 模式，提供容错能力。该模式下，网卡带宽最高可以翻倍（如从1Gbps翻到2Gbps）。
(6) 适配器传输负载均衡(Adaptive transmit load balancing)，模式代号是5。该策略是根据当前的负载把发出的数据分给每一个设备，由当前使用的设备处理收到的数据，如果当前正用于接收数据的网卡发生故障，则由其它网卡接管，要求所用的网卡及网卡驱动可通过ethtool命令得到speed信息。本策略的通道联合不需要专用的交换机支持，提供负载均衡和容错能力。
(7) 适配器负载均衡(Adaptive load balancing)，模式代号是6。该策略在IPV4情况下包含适配器传输负载均衡策略，由ARP协商完成接收的负载，通道联合驱动程序截获ARP在本地系统发送出的请求，用其中一个设备的硬件地址覆盖从属设备的原地址。即在策略6的基础之上，在接收数据的同时实现负载均衡，除要求ethtool命令可得到speed信息外，还要求支持对网卡MAC地址的动态修改功能。

注意：
Mode参数中的0、2、3、4模式要求交换机支持“ports group”功能并能进行相应的设置，例如在cisco中要将所连接的端口设为“port-channel”。
如果系统流量不超过单个网卡的带宽，请不要选择使用mode1之外的模式，因为负载均衡需要对流量进行计算，这对系统性能会有所损耗。
如果交换机及网卡都确认支持802.3ab，则实现负载均衡时尽量使用模式4以提高系统性能。
由于我们项目上的交换机型号不可控，且可能会更换，所以我们项目中设备绑定一般都使用mode5。

mode0（平衡负载模式）：平时两块网卡均工作，且自动备援，但需要在与服务器本地网卡相连的交换机设备上进行端口聚合来支持绑定技术。
 mode1（自动备援模式）：平时只有一块网卡工作，在它故障后自动替换为另外的网卡。
 mode6（平衡负载模式）：平时两块网卡均工作，且自动备援，无须交换机设备提供辅助支持。



### 问题分析

#### 网卡链路检测

```bash
# 这个操作 ???
$ cat /etc/modprobe.d/bond1.conf

alias bond0 bonding
options bond0 miimon=100 mode=1    # 模式1

$ /etc/rc.d/rc.local   # eth0 eth1的工作顺序(仅在主备模式下需要做这个设置，其他的模式不需要做这个设置)
ifenslave bond0 eth0 eth1

注：在高可用的环境下，网卡配置bonding后，vip_nic要为bond0
$ ethtool eth0
$ ethtool eth1

$ ethtool ens1f0
Settings for ens1f0:
        Supported ports: [ FIBRE ]
        Supported link modes:   10000baseT/Full
        Supported pause frame use: Symmetric
        Supports auto-negotiation: No
        Supported FEC modes: Not reported
        Advertised link modes:  10000baseT/Full
        Advertised pause frame use: Symmetric
        Advertised auto-negotiation: No
        Advertised FEC modes: Not reported
        Speed: 10000Mb/s
        Duplex: Full
        Port: FIBRE
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: off
        Supports Wake-on: d
        Wake-on: d
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes

```

执行结果：**Link detected：yes（说明网线链路连接正常)**

#### bond状况查看

```bash
$ cat /proc/net/bonding/bond1
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer2 (0)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

802.3ad info
LACP rate: slow
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: 74:85:c4:17:a5:48
Active Aggregator Info:
        Aggregator ID: 2
        Number of ports: 2
        Actor Key: 15
        Partner Key: 8
        Partner Mac Address: 58:6a:b1:c7:02:fa
# 网卡信息
Slave Interface: ens1f0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 74:85:c4:17:a5:48
Slave queue ID: 0
Aggregator ID: 2
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 74:85:c4:17:a5:48
    port key: 15
    port priority: 255
    port number: 1
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 58:6a:b1:c7:02:fa
    oper key: 8
    port priority: 32768
    port number: 8
    port state: 61
## 网卡信息
Slave Interface: ens1f1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 74:85:c4:17:a5:4a
Slave queue ID: 0
Aggregator ID: 2
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 74:85:c4:17:a5:48
    port key: 15
    port priority: 255
    port number: 2
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 58:6a:b1:c7:02:fa
    oper key: 8
    port priority: 32768
    port number: 73
    port state: 61

```

end

#### bond对应端口数

```bash
#（说明 这里参数正常为2 ，1为异常）,因为有两个网卡
$ cat /sys/class/net/bond1/bonding/ad_num_ports
2

```

end

#### 查看系统内部网卡设备的mac地址

```bash
$ cat /etc/udev/rules.d/70-persistent-ipoib.rules
# This is a sample udev rules file that demonstrates how to get udev to
# set the name of IPoIB interfaces to whatever you wish.  There is a
# 16 character limit on network device names.
#
# Important items to note: ATTR{type}=="32" is IPoIB interfaces, and the
# ATTR{address} match must start with ?* and only reference the last 8
# bytes of the address or else the address might not match the variable QPN
# portion.
#
# Modern udev is case sensitive and all addresses need to be in lower case.
#
# ACTION=="add", SUBSYSTEM=="net", DRIVERS=="?*", ATTR{type}=="32", ATTR{address}=="?*00:02:c9:03:00:31:78:f2", NAME="mlx4_ib3"

```

将这里查到的网卡mac地址 与 带外上的网卡mac地址对比，看是否一致

#### 检查对应bond1、eth0和eth1的配置文件

```bash
$ cat /etc/sysconfig/network-scripts/ifcfg-bond-bond1
BONDING_OPTS=mode=802.3ad
TYPE=Bond
BONDING_MASTER=yes
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=bond-bond1
UUID=0c494011-de68-4ad0-9e23-d66407cc0457
DEVICE=bond1
ONBOOT=yes
IPADDR=x.x.x.x
PREFIX=24
$ cat /etc/sysconfig/network-scripts/ifcfg-ens1f0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens1f0
UUID=3872c9a0-e31f-4599-a2ee-2eb22686c15b
DEVICE=ens1f0
ONBOOT=no

$ cat /etc/sysconfig/network-scripts/ifcfg-ens1f1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens1f1
UUID=4d2f9df6-ee50-4887-a07a-3fd3f9a81bef
DEVICE=ens1f1
ONBOOT=no

```

end

#### 登录带外进入操作系统，重新激发网卡，测试网络

```bash
ifdown eth0

ifup eth0

ethtool eth0

ifdown eth1

ifup eth1

ethtool eth1

ethtool bond1
```

测试结果：手动测试down掉eth0网络可以ping通，down掉eth1网络ping不通

#### 检查链路状态和系统日志内网卡的设置

```bash
ip link show
ip addr show

dmesg |egrep -i 'eth[0,1]'
```

end

#### （showtime）登录交换机查看配置，这里的9003假设是网卡eth0对应的mac地址后4位

```bash
$ display arp | in 9003

执行结果：x.x.x.x  BAGG7

# 对应的是接在交换机的BAGG7的端口

$ display current-configuration interface XXX_BAGG7
执行结果：Link-aggregation mode dynamic  （动态链路聚合）
```

这里交换机上的动态链路聚合配置是没有问题的

#### 尝试从服务器上down/up网卡来判断服务器-交换机链路和端口接线

**A、**备份原有的网卡配置文件，方便后期还原

```bash
$ mkdir /root/network_config_backup

$ cp ifcfg-\*  /root/network_config_backup

$ rm ifcfg-bond1
```

end

**B、**修改对应的网卡配置文件

```bash
$ sed -i  '/MASTER=bond1/d' ifcfg-eth0
$ sed -i  '/SLAVE/d' ifcfg-eth0
$ sed -i  '/MASTER=bond1/d' ifcfg-eth1
$ sed -i  '/SLAVE/d' ifcfg-eth1
```

end

**C、**配置IP到ifcfg-eth0或者 eth1

```bash
$ IPADDR='<IP>'

$ NETMASK='<MASK>'

$ GATEWAY='<GATEWAY>'
```

end

**D、**reboot重启服务器后检查网卡链路与IP

```bash
$ ip link show
$ ip addr show
```

1）先把 bond 接口去掉，查到没有说明已经去掉了

2）如果配了ip ，就 ping 一下网关，看能不能通

若是不能 ping ，重启网络服务器，若是报错查看相关报错日志

```bash
$ systemctl  restart network

$ systemctl  status network

$ journalctl --catalog --unit=network --no-pager
```

end

**E、**登录上联交换上，搜索mac 地址

```bash
# \*** ***eth0对应的mac地址后4位
$ display mac-address | in 9002
# \*** **e****th1对应的mac地址后4位
$ display mac-address | in 9003
```

end

**F、**若是没搜索到对应的mac地址，搜索对应的端口

```bash
# 搜索对应交换机上的7端口
$ display mac-address  | in 0/7 
```

end

**G、**若是对应的端口也没搜索到，则批量查到

```bash
$ display interface brief
```

执行结果：找到第7个接口，应该会有三个，0/7 和 BAGG7

**H、**根据批量找到的接口，分别针对性的查询具体接口信息

```bash
$ display current-configuration interface vlan 2

$ display current-configuration interface XGE1/0/7

$ display current-configuration interface XGE2/0/7
```

end

**L、**在服务器上抓包，看是否能抓到

```bash
$ tcpdump ether proto 0x88cc -ni eth1 -vvvv
```

执行结果：未抓到包，因为网络本身不通

将服务器网卡down掉，查看对应交换机上的哪个端口down了

服务器：

```bash
$ ip link set eth1 down
$ ip link set eth0 down
$ ip link show
```

交换机：

```bash
$ display interface brief
```

操作系统这边的网络配置，看起来没有问题；

结合现场机房查一下这台服务器，接到了哪台交换机的哪个接口，来进一步排查，看是不是交换机上的2台服务器对应的端口接反了；

如果接口没有问题，只能动交换机，在交换机上 Down 接口或者解除 bond重新配置对应端口的动态链路聚合；

**详见：H3C交换机配置动态链路聚合示例**

**N、**服务器上对应的eth0端口网线断开，查看系统内部对应的哪张网卡链路断开

服务器硬件的网卡标注,网线接口处左下角会有0/1的标识，来区别eth0和eth1

```bash
# 拔掉交换机上服务器上eth0网线，但是实际上eth1却异常了
$ ethtool  eth1
...
Link detected : no
```

根据在系统查询，系统eth0是服务器的eth1口;这里有些异常；

最后，找到问题的根因了，服务器eth0对应的网线，交换机上的这2个口接反了，对调网线后，对bond1和网络信息进行核实

1）检查网卡信息，在此处三块网卡的mac地址是一样的

2）检查模式及网卡信息。实际mac地址是不一样的

`cat /proc/net/bonding/bond0`

检查bond1真实信息

`cat /proc/net/bonding/bond1`

**3）检查bond对应的端口数**

`cat /sys/class/net/bond1/bonding/ad_num_ports`

*2* *(说明这里参数值为：2是正常)*



### 原文

https://www.modb.pro/db/247977

https://blog.51cto.com/linuxnote/1680315