## Linux NIC and VLAN and Bridge

### 问题起源

UIS的管理网和业务网是利用的两个万兆口，上联交换机是trunk口，现在装成容器工作节点后不能做管理网和业务网复用了，现有的两个万兆口设置成管理网后就没有业务网接口了，而且后续也无法在一个主机支持多VLAN容器网络。UIS主机根本没有多的接口给容器网络使用、

> why trunk?因为从bond0出去的流量可能vlan100或者vlan200而且不确定从哪个nic出去，所以nic接的交换机口必须是trunk，只有都是trunk口，当一个nic接口挂掉后才会从另一个nic出去，实现冗余。

那么可不可以一个网卡能几个子接口或者vlan，来实现一个网卡支持多个VLAN（子接口）作为容器的不同业务网出口呢。

### IP and NIC

IP地址池和网卡有关系么？

IP定位到网络设备，网卡定位到具体终端。

NIC package能不能出去关键得看NIC所连的交换机和路由器怎么配置、交换机和路由器定义的内部网段路由是什么。so，IP地址其实也是一个声明的资源配置信息，只要你的路由表写的是对的，任意正常的IP地址都能被路由出去。

IP只是将数据包传到对应的网络设备，VLAN+MAC才能将frame送到对应host。

### Linux-if网络复用：网卡桥化

前提需要交换机配合，接入Trunk口。

超融合管理平台支持**同一个物理端口或聚合端口的网络复用**，集群规模小的时候，网络复用也可以避免网络带宽浪费，在同样的物理环境下，提高链路的冗余性。

例如一台服务器配置了两张2端口的万兆网卡，配置eth0和eth1链路聚合用于管理网络和业务网络，管理网络带宽占用需求较小，此时可以提供给业务网较高的带宽占比，两端口链路聚合，同时提高了两个网络的冗余性。同样的，eth2和eth3链路聚合用于存储外网和存储内网的聚合。网络复用使超融合网络配置更加灵活，且大大提高了用户的投资价值。

配置网络复用时需要对网络进行带宽权重占比划分，初始化完成后，系统默认对复用的逻辑网络进行权重占比划分，下表是常见网络复用时的默认带宽权重占比。

> 配置比例可以修改。

| 复用场景               | 管理网络 | 业务网络 | 存储外网 | 存储内网 |
| ---------------------- | -------- | -------- | -------- | -------- |
| 管理网络和业务网络复用 | 49%      | 50%      | /        | /        |
| 存储外网和存储内网复用 | /        | /        | 18%      | 80%      |

*  网络带宽默认保留1%~2%占比用于系统内部转发，不推荐3个及以上逻辑网络复用。
* 网络带宽占比并非按照设置的比例进行网络限速，比如存储内网只占了10%的带宽，存储外网有较高带宽需求，此时允许存储外网占用更高的带宽。
* 网络带宽占比设置的意义在发生带宽抢占时，优先保证拥有更高带宽占比的逻辑网络转发，比如存储外网已经占据了70%的网络带宽，存储内网有需求占据50%，按照默认80/18的比例，存储内网会慢慢抢占网络带宽，优先保证存储内网转发。

这个是在NIC或者bond下加子接口实现的吗？但是流量分配怎么实现的... TC？

vlan ID呢？



### RedHat YYDS

#### Multiple Bridge, Multiple VLAN, and NIC Configuration

当需要启用VLAN虚拟网卡工作的时候，关联的物理网卡网络接口上必须没有IP地址的配置信息。

> 相当于将网卡当作二层交换机？

把一个网卡连接到两个 VLAN（这同时需要网络交换机的支持）。主机使用两个独立的 VNIC 来分别处理两个 VLAN 网络数据，而每个 VLAN 会连接到不同的网桥中。同时，每个网桥可以被多个虚拟机使用。

connects a single NIC to two VLANs. This presumes that the network switch has been configured to pass network traffic that has been tagged into one of the two VLANs to one NIC on the host. The host uses two VNICs to separate VLAN traffic, one for each VLAN. Traffic tagged into either VLAN then connects to a separate bridge by having the appropriate VNIC as a bridge member. Each bridge is in turn connected to by multiple virtual machines.



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/single-nice-multi-vlan.png)



#### Multiple Bridge, Multiple VLAN, and Bond Configuration

配置使用多个网卡组成一个绑定设备来连接到多个 VLAN。

bonds multiple NICs to facilitate a connection with multiple VLANs.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/multi-nic-single-bond-multi-vlan.png)

### demo：一个网卡配置两个vlan

一台服务器上一块网卡想同时划到两个不同vlan，并且让他们之间可以互相通信。

1、首先确认Linux系统内核是否已经支持VLAN功能，加载了8021q模块，lsmod |grep 8021q。
2、关于网卡的解释，好多人不知道网卡接口上的冒号和点号的区别，以下是一些解释（我也是从网上查的，仅供参考）
a、物理网卡：物理网卡指的是服务器上实际的网络接口设备，如在系统中看到的2个物理网卡分别对应是eth0和eth1这两个网络接口。
b、子网卡：子网卡并不是实际上的网络接口设备，但是可以作为网络接口在系统中出现，如eth0:1、eth1:2这种网络接口。它们必须要依赖 于物理网卡，虽然可以与物理网卡的网络接口同时在系统中存在并使用不同的IP地址，而且也拥有它们自己的网络接口配置文件。但是当所依赖的物理网卡不启用 时（Down状态）这些子网卡也将一同不能工作。
c、虚拟VLAN网卡：这些虚拟VLAN网卡也不是实际上的网络接口设备，也可以作为网络接口在系统中出现，但是与子网卡不同的是，他们没有自己 的配置文件。他们只是通过将物理网加入不同的VLAN而生成的VLAN虚拟网卡。如果将一个物理网卡添加到多个VLAN当中去的话，就会有多个VLAN虚 拟网卡出现，他们的信息以及相关的VLAN信息都是保存在/proc/net /vlan/config这个临时文件中的，而没有独自的配置文件。它们的网络接口名是eth0.1、eth1.2这种名字。

下面为实际操作，我的环境是一台H3C S5120三层交换机，服务器有4个网卡。

#### 交换机trunk配置

配置trunk，并且配置允许vlan id，我这里端口是g1/0/11 到g1/0/15

设置该端口为trunk模式

```bash
# vlan端口默认为access模式
[H3C-Git1/0/11]port link-type trunk 

# 替换后面的数字为all则是允许所有vlan tag通过该端口
[H3c-Git1/0/11]port trunk permit vlan 40 50 

# 给vlan配置网关
[H3C]interface Vlan-interface 40
[H3C-Vlan-interface1]ip address 192.168.4.254 255.255.255.0

[H3C]interface Vlan-interface 50
[H3C-Vlan-interface1]ip address 192.168.5.254 255.255.255.0
```



#### Linux配置

Linux要配置多vlan口。注：当需要启用VLAN虚拟网卡工作的时候，关联的物理网卡网络接口上必须没有IP地址的配置信息。
首先确认是否安装vconfig，然后查看核心是否提供VLAN 功能，执行

```bash
$ dmesg | grep -i 802
802.1Q VLAN Support v1.8 Ben Greeargreearb@candelatech.com

$ modprobe 8021q
# 查看系统内核是否支持802.1q协议
$ lsmod |grep 8021q 
8021q 18633 0

## 创建vlan接口
# eno3为物理网络接口名称，300为 802.1q tag id 也即 vlan ID
vconfig add eno3 300 
vconfig add eno3 500

ifconfig eno3.300 192.168.4.5/24 up
ifconfig eno3.500 172.31.5.5/24 up

## 验证vlan是否创建成功
$ ls /proc/net/vlan/

### 移除vlan
$ vconfig rem eno3.300 
Removed VLAN -:eno3.300 :-
$ vconfig rem eno3.500 
Removed VLAN -:eno3.500 :-
```

基于bond做的配置也是一样的

```bash
# vconfig命令增加子接口：
$ vconfig add bond0 100
Added VLAN with VID == 100 to IF -:bond0:-

$ vconfig add bond0 200
Added VLAN with VID == 200 to IF -:bond0:-

$ cat ifcfg-vlan100

DEVICE='vlan100'

ETHERDEVICE='bond0'

STARTMODE='auto'

IPADDR='172.16.1.21'

NETMASK='255.255.255.0'

$ cat ifcfg-bond0.200

DEVICE='vlan200'

ETHERDEVICE='bond0'

STARTMODE='auto'

IPADDR='172.16.1.21'

NETMASK='255.255.255.0'
```



#### 测试连通性

测试在服务器上分别ping 192.168.4.254和172.31.5.254可以ping通就行。
不能通的话， router -n 查看路由，是不是还是以前的老路由，是的话要删除

### bond双IP双网卡，不同VLAN

交换机接口都是Trunk，这样才能实现冗余即要保障每个接口都能出去vlan 30和 vlan 51的frame

```bash
$ /etc/sysconfig/network-scripts/ifcfg-bond0 
DEVICE=bond0
BOOTPROTO=none
IPV6INIT=no
NM_CONTROLLED=no
ONBOOT=yes
TYPE=Ethernet
BONDING_OPTS="mode=4 miimon=100"    # 绑定模式选择

## 配置sub-ifterface
$ vim /etc/sysconfig/network-scripts/ifcfg-bond0.30
DEVICE=bond0.30
BOOTPROTO=none
NM_CONTROLLED=no
ONBOOT=yes
IPADDR=10.0.0.102
NETMASK=255.255.255.0
GATEWAY=10.0.0.1
VLAN=yes
DNS1=223.5.5.5
DNS2=114.114.114.114

$ vim /etc/sysconfig/network-scripts/ifcfg-bond0.51
DEVICE=bond0.51
BOOTPROTO=none
NM_CONTROLLED=no
ONBOOT=yes
IPADDR=157.0.0.202
NETMASK=255.255.255.0
GATEWAY=157.0.0.1
VLAN=yes
DNS1=223.5.5.5
DNS2=114.114.114.114
```

如果是，双网卡作bond且bond只有一个IP，直接将IP写到bond的ifcfg文件中就行了

```bash
$ cat /etc/sysconfig/network-scripts/ifcfg-bond0 
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
DEVICE=bond0
IPV6INIT=no   # 关闭IPv6
PEERDNS=yes   # 运行网卡在启动时向DHCP服务器查询DNS信息，并自动覆盖/etc/resolv.conf配置文件
IPADDR=192.168.1.210
NETMASK=255.255.255.0
GATEWAY=192.168.1.2
DNS1=114.114.114.114
DNS2=223.5.5.5
```

end

### 引用

1. https://www.jianshu.com/p/2d5773b8e981
2. https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.0/html/technical_reference/vlan_configuration
3. https://www.bladewan.com/2017/05/23/linux_vconfig/
4. https://bbs.huaweicloud.com/blogs/detail/104512