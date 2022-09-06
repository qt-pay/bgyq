## Centos 修改系统默认网卡显示名称

### 问题

从习惯的eth0变成了ensp或者enp了，觉得很别扭，如何可以修改Centos下NIC Name。

#### Linux网卡接口命名规则

Linux系统的命名原来是eth0,eth1这样的形式，但是这个编号往往不一定准确对应网卡接口的物理顺序。

Centos 7 命名规则：

**net.ifnames命名规范为:设备类型+设备位置+数字**

设备类型：
en 表示Ethernet
wl 表示WLAN
ww 表示无线广域网WWAN

**实际的例子:**
eno1 板载网卡
enp0s2 pci网卡
ens33  pci网卡
wlp3s0 PCI无线网卡
wwp0s29f7u2i2  4G modem
wlp0s2f1u4u1  连接在USB Hub上的无线网卡
enx78e7d1ea46da pci网卡



By default, systemd will name interfaces using the following policy to apply the supported naming schemes: 

* Scheme 1:Names incorporating Firmware or BIOS provided index numbers for on-board devices (example: eno1), are applied if that information from the firmware or BIOS is applicable and available, else falling back to scheme 2. 
* Scheme 2: Names incorporating Firmware or BIOS provided PCI Express hotplug slot index numbers (example: ens1) are applied if that information from the firmware or BIOS is applicable and available, else falling back to scheme 3.
*  Scheme 3:Names incorporating physical location of the connector of the hardware (example: enp2s0), are applied if applicable, else falling directly back to scheme 5 in all other cases.
*  Scheme 4: Names incorporating interface's MAC address (example: enx78e7d1ea46da), is not used by default, but is available if the user chooses. 
* Scheme 5:The traditional unpredictable kernel naming scheme, is used if all other methods fail (example: enp1s0).

### Linux NIC_NAME

```bash
# 显示网卡名称
$ ip a
## link :  network device
$ ip link 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp3: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP mode DEFAULT group default qlen 1000
    link/ether 34:6b:5b:ea:26:00 brd ff:ff:ff:ff:ff:ff
3: enp4: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP mode DEFAULT group default qlen 1000
    link/ether 34:6b:5b:ea:26:00 brd ff:ff:ff:ff:ff:ff
4: enp1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master ovs-system state DOWN mode DEFAULT group default qlen 1000
    link/ether 34:6b:5b:ea:26:02 brd ff:ff:ff:ff:ff:ff
...
```

Centos下网卡`enps3`去`/etc/sysconfig/network-scripts/`匹配`ifcfg-<device nama>`文件，然后根据配置信息给NIC配置IP和Bond等东西。

### ifcfg-[device_name]

接口配置(`ifcfg)`文件可控制各个网络设备的软件接口。些文件通常被命名为 `ifcfg-name`，其中后缀 *name* 指的是配置文件控制的设备的名称。按照惯例，`ifcfg` 文件的后缀与配置文件中 `DEVICE` 指令提供的字符串相同。

网卡A，只会去匹配`ifcfg-A`的配置。纵然，`ifcfg-B`记录的device和hwaddr和网卡A一致也不会去读取。

疑惑起因：

一个物理网卡，对应了两个ifcfg-xxx文件到底哪个生效，我都不知道...即为什么显示的enp3而不是eth0

答案：根据网卡名匹配/etc/sysconfig/network-scripts/ifcfg-xxx.conf配置文件，根据ifcfg配置文件里的device参数的名字。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ifcfg-name.png)

#### device

ifcfg-xxx配置文件中，device必须填入系统中网卡名称，如果随便填一个不存在的网卡名，重启网络服务会报错，提示：Device XXX does not seem to be present.

### nmcli

 nmcli - command-line tool for controlling NetworkManager

使用nmcli 把网卡名称ens34也改成Gi0

```bash
$ nmcli connection show
NAME   UUID                                  TYPE      DEVICE
ens34  b85c1bc2-d3d2-3abb-a1e9-41eab250e4b3  ethernet  Gi0
$ nmcli connection modify ens34 connection.id Gi0
$ nmcli connection reload
$ nmcli connection show
NAME  UUID                                  TYPE      DEVICE
Gi0   b85c1bc2-d3d2-3abb-a1e9-41eab250e4b3  ethernet  Gi0   
```

nmcli命令依赖NetworkManager服务，如果NetworkManager没启动会提示下面错误

Error：NetworkManager is not running.

### grub：修改网卡名称

希望继续使用 eth0 这样的传统名称，在GRUB_CMDLINE_LINUX="" 引号中添加参数net.ifnames=0 biosdevname=0

```bash
$ cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="biosdevname=0 rhgb elevator=deadline transparent_hugepage=always net.ifnames=0 crashkernel=256M quiet"
GRUB_DISABLE_RECOVERY="true"

# 重新加载到启动中
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg > /dev/null 2>&1
grub2-mkconfig -o /boot/grub2/grub.cfg > /dev/null 2>&1

# 重启后生效
$ reboot
```

end

### grub 内存巨大页

```bash
#设置大页内存
#可重复执行
grub(){
    rpm -qa | grep unic-tools > /dev/null 2>&1
    if [ $? -ne 0 ];then
        rpm -ivh /mnt/unic-tools-1.0.0-1.unic.el7.x86_64.rpm > /dev/null 2>&1
        sed -i /GRUB_CMDLINE_LINUX/c"GRUB_CMDLINE_LINUX=\"crashkernel=512M intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G isolcpus=2,3,4,5 rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet\"" /etc/default/grub
        grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg > /dev/null 2>&1
        grub2-mkconfig -o /boot/grub2/grub.cfg > /dev/null 2>&1
        PSUCCESS "grub success"
    else
        PBLUE "grub skip"
    fi
}
```

