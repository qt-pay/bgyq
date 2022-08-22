## [LGTM] 网卡bond和交换机配置

### Summary

网卡bond mode 1、5、6不需要交换机设置

网关bond mode 0、2、3、4需要交换机设置



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