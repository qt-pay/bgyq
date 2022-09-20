## [LGTM] Using Open vSwitch with DPDK

This document describes how to use Open vSwitch with DPDK.

Important

Using DPDK with OVS requires configuring OVS at build time to use the DPDK library. The version of DPDK that OVS supports varies from one OVS release to another, as described in the [releases FAQ](https://docs.openvswitch.org/en/latest/faq/releases/). For build instructions refer to [Open vSwitch with DPDK](https://docs.openvswitch.org/en/latest/intro/install/dpdk/).

## Ports and Bridges

ovs-vsctl can be used to set up bridges and other Open vSwitch features. Bridges should be created with a `datapath_type=netdev`:

```bash
$ ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev
```

ovs-vsctl can also be used to add DPDK devices. ovs-vswitchd should print the number of dpdk devices found in the log file:

```bash
$ ovs-vsctl add-port br0 dpdk-p0 -- set Interface dpdk-p0 type=dpdk \
    options:dpdk-devargs=0000:01:00.0
$ ovs-vsctl add-port br0 dpdk-p1 -- set Interface dpdk-p1 type=dpdk \
    options:dpdk-devargs=0000:01:00.1
```

Some NICs (i.e. Mellanox ConnectX-3) have only one PCI address associated with multiple ports. Using a PCI device like above won’t work. Instead, below usage is suggested:

```bash
$ ovs-vsctl add-port br0 dpdk-p0 -- set Interface dpdk-p0 type=dpdk \
    options:dpdk-devargs="class=eth,mac=00:11:22:33:44:55"
$ ovs-vsctl add-port br0 dpdk-p1 -- set Interface dpdk-p1 type=dpdk \
    options:dpdk-devargs="class=eth,mac=00:11:22:33:44:56"
```

Hotplugging physical interfaces is not supported using the above syntax. This is expected to change with the release of DPDK v18.05. For information on hotplugging physical interfaces, you should instead refer to [Hotplugging](https://docs.openvswitch.org/en/latest/topics/dpdk/phy/#port-hotplug).

After the DPDK ports get added to switch, a polling thread continuously polls DPDK devices and consumes 100% of the core, as can be checked from `top` and `ps` commands:

```
$ top -H
$ ps -eLo pid,psr,comm | grep pmd
```

Creating bonds of DPDK interfaces is slightly different to creating bonds of system interfaces. For DPDK, the interface type and devargs must be explicitly set. For example:

```bash
$ ovs-vsctl add-bond br0 dpdkbond p0 p1 \
    -- set Interface p0 type=dpdk options:dpdk-devargs=0000:01:00.0 \
    -- set Interface p1 type=dpdk options:dpdk-devargs=0000:01:00.1
```

To stop ovs-vswitchd & delete bridge, run:

```bash
$ ovs-appctl -t ovs-vswitchd exit
$ ovs-appctl -t ovsdb-server exit
$ ovs-vsctl del-br br0
```

## demo？？？

```bash
## dpdk初始化配置（需要确定业务网卡name，如下示例 eth0 和 eth1 ，请按顺序执行）


## 加载VFIO内核模块

modprobe vfio
modprobe vfio-pci
ifconfig eth0 down 
ifconfig eth1 down

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



#安装hostoverlay的UCS相关RPM包

# 配置网桥

ovs-vsctl add-br br-tun -- set bridge br-tun datapath_type=netdev
ovs-vsctl set interface br-tun mtu-request=2000




# 物理网卡需要和原网络结构兼容，需要两个port支持bond协议，但现在dpdk接管网卡。一种方案是用ovs 本身的bond, 但是这样对于ovs，看到的还是两个port, 如果加流表规则, 需要加两份，这样不便于控制管理。另一种方案，使用dpdk 的bond，为ovs添加支持dpdk-bond类型port的代码,  类似dpdk, dpdk-vhost-user, tap类型。

# 两个网卡配置是bond
# p0 和 p1 是什么东西
# p0 和 p1随便写的吧，关键是set Interface时设置p0和p1指向的dpdk-devargs
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
ovs-vsctl set interface p0 lldp:enable=true
ovs-vsctl set interface p1 lldp:enable=true

cvkname=`hostname` ; ovs-vsctl -- --id=@auto create AutoAttach system_name=$cvkname -- set bridge br-tun auto_attach=@auto



# 业务网络和存储网路配置

ovs-vsctl add-br vswitch-stor
# add-bond BRIDGE PORT IFACE...  add bonded port PORT in BRIDGE from IFACES
ovs-vsctl add-bond  vswitch-stor vswitch-stor_bond eth3 eth4 lacp=active bond_mode=balance-tcp
ip link set dev vswitch-stor up



cat >/etc/sysconfig/network-scripts/ifcfg-br-tun  <<EOF

TYPE=Ethernet

BOOTPROTO=static

NAME=br-tun

DEVICE=br-tun

MTU=2000

ONBOOT=yes

IPADDR=br-tun-ip

NETMASK=255.x.x.x

EOF



cat >/etc/sysconfig/network-scripts/ifcfg-vswitch-stor  <<EOF

TYPE=Ethernet

BOOTPROTO=static

NAME=vswitch-stor

DEVICE=vswitch-stor

ONBOOT=yes

IPADDR=storage_ip

NETMASK=255.x.x.x

EOF


for i in `ls /etc/sysconfig/network-scripts/ | grep ifcfg-e.*`; do sed -i 's/ONBOOT=no/ONBOOT=yes/g' /etc/sysconfig/network-scripts/$i ; sed -i 's/BOOTPROTO=dhcp/BOOTPROTO=none/g' /etc/sysconfig/network-scripts/$i; done

for i in `ls /etc/sysconfig/network-scripts/ | grep ifcfg-e.* | cut -d "-" -f 2,3`; do lldptool -L -i $i adminstatus=rxtx; lldptool -T -i $i -V sysName enableTx=yes;done

systemctl  restart lldpad
systemctl  restart network



# ovs流表预配置

ovs-ofctl add-flows vswitch-stor /opt/ovs_default_flows.txt

chmod +x /etc/rc.d/rc.local && echo  "/usr/bin/ovs-ofctl add-flows vswitch-stor /opt/ovs_default_flows.txt" >>/etc/rc.local



# 修改libvirt配置文件
#更新cvk上的/etc/libvirt/qemu.conf文件并重启libvirtd服务

sed -i  '/auto_dump_path = /,+d' /etc/libvirt/qemu.conf 
#注：ansible执行时引号需要字符 echo  'auto_dump_path = \"/vms/libvirt/qemu/dump\"' >> /etc/libvirt/qemu.conf 
echo  'auto_dump_path = "/vms/libvirt/qemu/dump"' >> /etc/libvirt/qemu.conf     

mkdir -p /vms/libvirt/qemu/dump 

systemctl restart libvirtd


```



## 原文

https://docs.openvswitch.org/en/latest/howto/dpdk/