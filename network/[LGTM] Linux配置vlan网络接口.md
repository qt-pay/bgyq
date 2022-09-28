## [LGTM] Linux配置vlan网络接口

### vlan if应用场景

Linux的使用中很少会涉及VLAN和网络的配置，因为多数的服务部署集中在应用层面而非底层网络（switch/route/vlan/interface），系统安装时默认生成的网络配置已经足够应对实际需求。

但是当一个服务器的网卡以Trunk口的形式接入交换机时，就会有不同vlan traffic的需求。

在某些场景中，我们希望在 服务器上的同一网卡分配来自不同VLAN的多个ip。这可以通过启用VLAN标记接口来实现，但要实现这一点，首先必须确保交换机上添加多个vlan。

假设我们有一个Linux服务器，其中有两个以太网卡(ens33和ens38)，第一个网卡(ens33)用于数据流量，第二个网卡(ens38)用于控制/管理流量。**对于数据流，将使用多个vlan(将在数据流网卡上分配来自不同vlan的多个ip)。**

假设从交换机连接到服务器数据流量网卡的端口被配置为Trunk，通过映射多个vlan到它。下面是映射到数据流量网卡的vlan:

- VLAN ID (200)，172.168.10.0/24
- VLAN ID (300)，172.168.20.0/24

类似交换机的trunk port + vlan if吧

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/trunk-and-vlan-if.png)

在CentOS 7 /RHEL 7 / CentOS 8 /RHEL 8系统上使用VLAN标记接口，必须加载内核模块8021q。

Linus中这些VLAN网卡接口不是实际上的网络接口设备，但也可以作为网络接口在系统中出现，但是与子网卡不同的是，他们没有自己的配置文件。它们只是通过将物理网加入不同的VLAN而生成的VLAN虚拟网卡。

如果将一个物理网卡添加到多个VLAN当中去的话，就会有多个VLAN虚拟网卡出现，它们的信息以及相关的VLAN信息都是保存在/proc/net/vlan/config这个临时文件中的，而没有独自的配置文件。它们的网络接口名是eth0.1、eth1.2这种名字。



### vlan if demo:fire:

在物理设备上， 都是先有lan， 然后才划分vlan， 而在linux上则刚好相反， 因为构建lan的所需的bridge是一个逻辑设备， 因此， 在linux上需要先构建一个vlan设备， 然后将其绑定到一个bridge上去， 所有绑定到该bridge上的设备构成一个vlan， 而构建该vlan设备时指定的网络设备则作为trunk端口.

**简介**

VLAN是网络栈的一个附加功能，且位于下两层。首先来学习Linux中网络栈下两层的实现，再去看如何把VLAN这个功能附加上去。下两层涉及到具体的硬件设备，日趋完善的Linux内核已经做到了很好的代码隔离，对网络设备驱动也是如此，如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-1844341/b93307pu8v.png?imageView2/2/w/1620)

这里要注意的是，Linux下的网络设备net_dev并不一定都对应实际的硬件设备，只要注册一个struct net_device{}结构体（netdevice.h）到内核中，那么这个网络设备就存在了。该结构体很庞大，其中包含设备的协议地址（对于IP即IP地址），这样它就能被网络层识别，并参与路由系统，最有名的当数loopback设备。不同的设备（包括硬件和非硬件）的ops操作方法各不相同，由驱动自己实现。一些通用性的、与设备无关的操作流程（如设备锁定等）则被Linux提炼出来，我们称为驱动框架。

**linux虚拟网络设备之vlan配置** 

我们通过一个网桥两个设备对，来连接两个网络名字空间，每个名字空间中创建两个vlan

![img](https://ask.qcloudimg.com/http-save/yehe-1844341/kohtatwa6a.jpeg?imageView2/2/w/1620)

借助vconfig来配置vlan：

```bash
#创建网桥
brctl addbr br-test-vlan 
#创建veth对儿
ip link add veth01 type veth peer name veth10
ip link add veth02 type veth peer name veth20 
#将veth对的一端添加到网桥
brctl addif br-test-vlan veth01
brctl addif br-test-vlan veth02 
#启动设备
ip link set dev br-test-vlan up
ip link set dev veth01 up
ip link set dev veth02 up
ip link set dev veth10 up
ip link set dev veth20 up 
#创建网络名字空间，模拟vm
ip netns add test-vlan-vm01
ip netns add test-vlan-vm02 
#将设备对儿的另一端添加到另个名字空间（其实在一个名字空间也能玩，只是两个名字空间更加形象）
ip link set veth10 netns test-vlan-vm01
ip link set veth20 netns test-vlan-vm02 
#分别进入两个名字空间创建vlan和配置ip
#配置名字空间test-vlan-vm01
ip netns exec test-vlan-vm01 bash
#配置vlan 3001 和 vlan 3002
vconfig add veth10 3001
vconfig add veth10 3002
#启动两个vlan的设备
ip link set veth10.3001 up
ip link set veth10.3002 up 
#分别在两个vlan上配置ip （这里简单起见，使用了同一个网段了IP，缺点是，需要了解一点儿路由的知识）
ip a add 172.16.30.1/24 dev veth10.3001
ip a add 172.16.30.2/24 dev veth10.3002 
#添加路由
route add 172.16.30.21 dev veth10.3001
route add 172.16.30.22 dev veth10.3002 
#配置名字空间test-vlan-vm02
ip netns exec test-vlan-vm02 bash
#配置vlan 3001 和 vlan 3002
vconfig add veth20 3001
vconfig add veth20 3002
#启动两个vlan的设备
ip link set veth20.3001 up
ip link set veth20.3002 up
#分别在两个vlan上配置ip （这里简单起见，使用了同一个网段了IP，缺点是，需要了解一点儿路由的知识）
ip a add 172.16.30.21/24 dev veth20.3001
ip a add 172.16.30.22/24 dev veth20.3002 
#添加路由
route add 172.16.30.1 dev veth20.3001
route add 172.16.30.2 dev veth20.3002
```

查看一下vlan配置：

```bash
$ cat /proc/net/vlan/config 
VLAN Dev name | VLAN ID
Name-Type: VLAN_NAME_TYPE_RAW_PLUS_VID_NO_PAD
veth10.3001 | 3001 | veth10
veth10.3002 | 3002 | veth10
```

现在，我们可以分别在两个名字空间来ping另外一个名字空间的两个IP，虽然两个IP都能ping通，但是使用的源IP是不同的，走的vlan也是不同的，我们可以在veth01/veth10/veth02/veth20/br-test-vlan 任意一个上抓包，会看到vlan信息：

```bash
$ tcpdump -i veth10 -nn -e
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth10, link-type EN10MB (Ethernet), capture size 262144 bytes
15:38:18.381010 82:f7:0e:2d:3f:62 > 9e:58:72:fa:11:15, ethertype 802.1Q (0x8100), length 102: vlan <span style="color: #ff0000;">3001</span>, p 0, ethertype IPv4, <strong><span style="color: #ff0000;">172.16.30.1 > 172.16.30.21</span></strong>: ICMP echo request, id 19466, seq 1, length 64
15:38:18.381183 9e:58:72:fa:11:15 > 82:f7:0e:2d:3f:62, ethertype 802.1Q (0x8100), length 102: vlan <span style="color: #ff0000;"><strong>3001</strong></span>, p 0, ethertype IPv4, 172.16.30.21 > 172.16.30.1: ICMP echo reply, id 19466, seq 1, length 64
15:38:19.396796 82:f7:0e:2d:3f:62 > 9e:58:72:fa:11:15, ethertype 802.1Q (0x8100), length 102: vlan 3001, p 0, ethertype IPv4, 172.16.30.1 > 172.16.30.21: ICMP echo request, id 19466, seq 2, length 64
15:38:19.396859 9e:58:72:fa:11:15 > 82:f7:0e:2d:3f:62, ethertype 802.1Q (0x8100), length 102: vlan 3001, p 0, ethertype IPv4, 172.16.30.21 > 172.16.30.1: ICMP echo reply, id 19466, seq 2, length 64
15:38:23.162052 82:f7:0e:2d:3f:62 > 9e:58:72:fa:11:15, ethertype 802.1Q (0x8100), length 102: vlan 3002, p 0, ethertype IPv4, 172.16.30.2 > <strong><span style="color: #ff0000;">172.16.30.22</span></strong>: ICMP echo request, id 19473, seq 1, length 64
15:38:23.162107 9e:58:72:fa:11:15 > 82:f7:0e:2d:3f:62, ethertype 802.1Q (0x8100), length 102: vlan 3002, p 0, ethertype IPv4, <strong><span style="color: #ff0000;">172.16.30.22 > 172.16.30.2</span></strong>: ICMP echo reply, id 19473, seq 1, length 64
```

如果试图从veth10.3001 去ping 172.16.30.22 是不能通的，因为是不同的vlan：

```bash
$ ping -I veth10.3001 172.16.30.22
PING 172.16.30.22 (172.16.30.22) from 172.16.30.1 veth10.3001: 56(84) bytes of data.
^C
--- 172.16.30.22 ping statistics ---
9 packets transmitted, 0 received, 100% packet loss, time 8231ms
```

不适用vconfig的解法：

```bash
$ ip link add link veth10 name veth10.3001 type vlan id 3001
```

另： vlan 一般以 设备名.vlanid 来命名，不过并非强制，如下命名为 vlan3003也是没问题的

```bash
$ ip link add link veth10 name vlan3003 type vlan id 3003
```

**注意：**一个主设备上相同vlan好的子设备最多只能有一个

```bash
$ ip link add link veth10 name vlan3001 type vlan id 3001
 RTNETLINK answers: File exists
```

所以，正常来讲，一般是这样的：

![img](https://ask.qcloudimg.com/http-save/yehe-1844341/7a53bqjtfc.jpeg?imageView2/2/w/1620)

参考： http://network.51cto.com/art/201504/473419.htm



### Redhat配置vlan if

在 Red Hat Enterprise Linux 7 中，默认载入 `8021q` 模块。如果需要，您可以以 `root` 用户身份运行以下命令来确保载入该模块：

```bash
$ modprobe --first-time 8021q
modprobe: ERROR: could not insert '8021q': Module already in kernel
```

要显示模块信息，请运行以下命令：

```bash
$ modinfo 8021q
```

有关更多命令选项，请参阅 `modprobe(8)手册页`。

使用 ifcfg 文件设置 802.1Q VLAN 标记

1. 在 `/etc/sysconfig/network-scripts/ifcfg-device_name` 中配置父接口，其中 *device_name* 是接口的名称：

   ```bash
   DEVICE=interface_name
   TYPE=Ethernet
   BOOTPROTO=none
   ONBOOT=yes
   ```

2. 在 `/etc/sysconfig/network-scripts/` 目录中配置 VLAN 接口配置。配置文件名称应当是父接口加上 a `.` 字符加上 VLAN ID 号。例如，如果 VLAN ID 为 192，并且父接口为 *enp1s0*，则配置文件名称应为 `ifcfg-enp1s0.192` ：

   ```bash
   DEVICE=enp1s0.192
   BOOTPROTO=none
   ONBOOT=yes
   IPADDR=192.168.1.1
   PREFIX=24
   NETWORK=192.168.1.0
   VLAN=yes
   ```

   如果需要在同一接口 *enp1s0 上配置第二个 VLAN（如 VLAN ID 193），请添加一个名称为 `enp1s0.193`* 的新文件，并提供 VLAN 配置详细信息。

3. 重新启动网络服务，以使更改生效。作为 `root` 发出以下命令：

   ```bash
   $ systemctl restart network
   ```

#### Configuring VLAN tagging using nmcli commands

This section describes how to configure Virtual Local Area Network (VLAN) tagging using the `nmcli` utility.

 nmcli - command-line tool for controlling NetworkManager

**Prerequisites**

- The interface you plan to use as a parent to the virtual VLAN interface supports VLAN tags.
- If you configure the VLAN on top of a bond interface:
  - The ports of the bond are up.
  - The bond is not configured with the `fail_over_mac=follow` option. A VLAN virtual device cannot change its MAC address to match the parent’s new MAC address. In such a case, the traffic would still be sent with the incorrect source MAC address.
  - The bond is usually not expected to get IP addresses from a DHCP server or IPv6 auto-configuration. Ensure it by setting the `ipv4.method=disable` and `ipv6.method=ignore` options while creating the bond. Otherwise, if DHCP or IPv6 auto-configuration fails after some time, the interface might be brought down.
- The switch, the host is connected to, is configured to support VLAN tags. For details, see the documentation of your switch.

**Procedure**

1. Display the network interfaces:

   ```bash
   $ nmcli device status
   DEVICE   TYPE      STATE         CONNECTION
   enp1s0   ethernet  disconnected  enp1s0
   bridge0  bridge    connected     bridge0
   bond0    bond      connected     bond0
   ...
   ```

2. Create the VLAN interface. For example, to create a VLAN interface named `vlan10` that uses `enp1s0` as its parent interface and that tags packets with VLAN ID `10`, enter:

   ```bash
   $ nmcli connection add type vlan con-name vlan10 ifname vlan10 vlan.parent enp1s0 vlan.id 10
   ```

   Note that the VLAN must be within the range from `0` to `4094`.

3. By default, the VLAN connection inherits the maximum transmission unit (MTU) from the parent interface. Optionally, set a different MTU value:

   ```bash
   $ nmcli connection modify vlan10 ethernet.mtu 2000
   ```

4. Configure the IP settings of the VLAN device. Skip this step if you want to use this VLAN device as a port of other devices.

   1. Configure the IPv4 settings. For example, to set a static IPv4 address, network mask, default gateway, and DNS server to the `vlan10` connection, enter:

      ```bash
      $ nmcli connection modify vlan10 ipv4.addresses '192.0.2.1/24'
      $ nmcli connection modify vlan10 ipv4.gateway '192.0.2.254'
      $ nmcli connection modify vlan10 ipv4.dns '192.0.2.253'
      $ nmcli connection modify vlan10 ipv4.method manual
      ```

   2. Configure the IPv6 settings. For example, to set a static IPv6 address, network mask, default gateway, and DNS server to the `vlan10` connection, enter:

      ```bash
      $ nmcli connection modify vlan10 ipv6.addresses '2001:db8:1::1/32'
      $ nmcli connection modify vlan10 ipv6.gateway '2001:db8:1::fffe'
      $ nmcli connection modify vlan10 ipv6.dns '2001:db8:1::fffd'
      $ nmcli connection modify vlan10 ipv6.method manual
      ```

5. Activate the connection:

   ```bash
   $ nmcli connection up vlan10
   ```

**Verification steps**

- Verify the settings:

  ```bash
  $ ip -d addr show vlan10
  4: vlan10@enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
      link/ether 52:54:00:72:2f:6e brd ff:ff:ff:ff:ff:ff promiscuity 0
      vlan protocol 802.1Q id 10 <REORDER_HDR> numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
      inet 192.0.2.1/24 brd 192.0.2.255 scope global noprefixroute vlan10
         valid_lft forever preferred_lft forever
      inet6 2001:db8:1::1/32 scope global noprefixroute
         valid_lft forever preferred_lft forever
      inet6 fe80::8dd7:9030:6f8e:89e6/64 scope link noprefixroute
         valid_lft forever preferred_lft forever
  ```

### Ubuntu vcinfig命令配置 vlan if

要让Linux 系统支持VLAN需要安装vlan软件包`apt-get install vlan（Ubuntu）`，然后modprobe 8021q加载vlan module，可以使用lsmod | grep -i 8021q检查内核是否加载模块，如果遇见不会自动加载模块的系统使用`echo "modprobe 8021q" > /etc/rc.local`使其开机加载。

vconfig命令是vlan软件包的主命令，用来添加和移除VLAN interface，VLAN interface建立后才能对其配置IP和子网掩码。而所谓的VLAN interface则需要建立在物理接口的基础上，一般情况下划分VLAN的物理接口使用不分配IP的static method。
Ubuntu接口/etc/network/interfaces配置如下:

```bash
auto second
iface second inet static
```

voconfig命令格式如下：

```bash
vconfig add <interface> <vid>
vconfig rem <vlan interface>
```


默认的VLAN interface名字也因系统的不同而略有差异，Ubuntu系统中默认的VLAN interface名为`<interface>.<vid>`，如下所示：

```bash
vconfig add second 32
vconfig rem second.32
```


到此阶段，VLAN interface就可以使用ifconfig命令配置IP使用，但并没有相应interface配置文件的定义，因此重启会失效。系统当前的VLAN配置可以在`/proc/net/vlan/config`文件中查看，VLAN接口文件则可在`/proc/net/vlan/`目录下找到。

```bash
cat /proc/net/vlan/config
ls -al /proc/net/vlan/
## ubuntu demo
$ cat /proc/net/vlan/config 
VLAN Dev name	 | VLAN ID
Name-Type: VLAN_NAME_TYPE_RAW_PLUS_VID_NO_PAD
second.32      | 32  | second
```

如之前提及，没有配置文件定义的VLAN interface会在系统重启后失效，因此一般都是通过配置文件直接配置VLAN,Ubuntu系统下支持的VLAN interface命名方式只有`<interface>.<vid>`一种，可以在/etc/network/interfaces文件中定义VLAN interface：

```bash
# Debain - Ubuntu VLAN interface配置
auto second
iface second inet static

auto second.32
iface second:32 inet static
address 192.168.23.25/24
```


end

### 原文：

1. https://blog.csdn.net/melancholy123/article/details/70232889
2. https://www.jianshu.com/p/8a05a5ba3124
3. https://lishiwen4.github.io/network/virtual-network-interface-vlan