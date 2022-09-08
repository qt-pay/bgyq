## Centos grub修改网卡名称和HugePages

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

通过使用hugepage分配可以提高性能，因为需要更少的页，因此需要更少Translation Lookaside Buffers (TLB，高速传送缓存)，使用TLB可以减少将虚拟页地址转换成物理页地址的时间。

如果没有hugepage，使用标准4K页大小的话，可能产生大量TLB miss，影响性能。

4KB 的内存页作为内存映射的单位，但有些场景我们希望使用更大的内存页作为映射单位（如 2MB）。使用更大的内存页作为映射单位有如下好处：

- 减少 `TLB（Translation Lookaside Buffer）` 的失效情况。
- 减少 `页表` 的内存消耗。
- 减少 PageFault（缺页中断）的次数。

> Tips：`TLB` 是一块高速缓存，TLB 缓存虚拟内存地址与其映射的物理内存地址。MMU 首先从 TLB 查找内存映射的关系，如果找到就不用回溯查找页表。否则，只能根据虚拟内存地址，去页表中查找其映射的物理内存地址。

因为映射的内存页越大，所需要的 `页表` 就越小（很容易理解）；`页表` 越小，TLB 失效的情况就越少。

使用大于 4KB 的内存页作为内存映射单位的机制叫 `HugePages`，目前 Linux 常用的 HugePages 大小为 2MB 和 1GB。

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

#### 验证是否修改成功

Defines the number of persistent huge pages configured in the kernel at boot time. **The default value is 0.** It is only possible to allocate huge pages if there are sufficient physically contiguous free pages in the system. Pages reserved by this parameter cannot be used for other purposes.

This value can be adjusted after boot by changing the value of the `/proc/sys/vm/nr_hugepages` file.

**In a NUMA system,** huge pages assigned with this parameter are divided equally between nodes. You can assign huge pages to specific nodes at runtime by changing the value of the node's `/sys/devices/system/node/*node_id*/hugepages/hugepages-1048576kB/nr_hugepages` file.

```bash
##从 node1 分配 171 个 1GB 的大页面
$ cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
171
## 从 node3 分配 1,024 个 2MB 的大页面：
$ echo 1024 > /sys/devices/system/node/node3/hugepages/hugepages-2048kB/nr_hugepages
```



For more information, read the relevant kernel documentation, which is installed in `/usr/share/doc/kernel-doc-kernel_version/Documentation/vm/hugetlbpage.txt` by default.

#### NUMA

在NUMA架构出现前，CPU欢快的朝着频率越来越高的方向发展。受到物理极限的挑战，又转为核数越来越多的方向发展。如果每个core的工作性质都是share-nothing（类似于map-reduce的node节点的作业属性），那么也许就不会有NUMA。由于所有CPU Core都是通过共享一个北桥来读取内存，随着核数如何的发展，北桥在响应时间上的性能瓶颈越来越明显。于是，聪明的硬件设计师们，先到了把内存控制器（原本北桥中读取内存的部分）也做个拆分，平分到了每个die上。于是NUMA就出现了！

NUMA是什么?

NUMA中，虽然内存直接attach在CPU上，但是由于内存被平均分配在了各个die上。只有当CPU访问自身直接attach内存对应的物理地址时，才会有较短的响应时间（后称Local Access）。而如果需要访问其他CPU attach的内存的数据时，就需要通过inter-connect通道访问，响应时间就相比之前变慢了（后称Remote Access）。所以NUMA（Non-Uniform Memory Access）就此得名。



NUMA(Non-Uniform Memory Access) 非均匀内存访问架构是指多处理器系统中，内存的访问时间是依赖于处理器和内存之间的相对位置的。 这种设计里存在和处理器相对近的内存，通常被称作本地内存；还有和处理器相对远的内存， 通常被称为非本地内存。

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-memory-configuring-huge-pages

### kvm and hugepage

https://qkxu.github.io/2020/04/15/KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E4%BD%BF%E7%94%A8%E5%A4%A7%E9%A1%B5%E5%86%85%E5%AD%98.html

#### Linux挂载hugetlbfs

大页分配以后要挂载hugetlbfs，应用程序才可以使用

默认情况下，系统会挂载hugetlbfs文件系统

```bash
[root@localhost ~]# mount |grep hugetlbfs
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime)
```

这个挂载的是1G还是2M大页，取决于系统默认大页值。

默认值为2M大页场景下，如果要使用1G大页，则需要再次为1G大页挂载hugetlbfs

```
mkdir /dev/hugepages-1048576kB
mount -t hugetlbfs -o pagesize=2MB test_1G /dev/hugepages-1048576kB
```

如果开机启动挂载，则需要添加到：

```
vim /etc/fstab
test_1G /dev/hugepages-1048576kB hugetlbfs pagesize=1GB 0 0
```

默认值为1G大页场景下，如果要使用2M大页则与上述类似，pagesize=2M即可。

#### libvirt配置

libvirt在启动时会从mount中读取挂载的hugetlbfs，如果mount动作在libvirt服务启动之后，则必须重启libvirtd服务才能生效。

在xml中配置如下

```
<memoryBacking>
  <hugepages/>
</memoryBacking>
```

上述配置系统会采用默认的大页配置，也可以指定相应的大页内存。

1G大页配置

```
<memoryBacking>
    <hugepages>
      <page size="1" unit="G"/>
    </hugepages>
</memoryBacking>
```

2M大页配置

```
<memoryBacking>
    <hugepages>
      <page size="2" unit="M"/>
    </hugepages>
</memoryBacking>
```

官网中还page选项中附带nodeset选项，像是绑定numa，但是这个并不是强制，numa绑定还是要靠numatune属性来完成。

创建后可以查看大页相关的使用情况，前文已经讲述，不再贴图。

#### qemu支持

创建大页虚拟机后，可以看到qemu命令中相关参数：

```
-m 4096 -mem-prealloc -mem-path /dev/hugepages/libvirt/qemu/test_vm 
```

手动用qemu启动时，加上上述命令即可。
