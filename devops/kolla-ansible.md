### kolla-ansible

#### ansible实战

用ansible的滚动更新Nginx：

思路是，nginx的健康检查模块检测后端节点上的静态文件，ansible更新后端节点时先摘除这个静态文件，然后由nginx自动移除rs是吧

稳健--

#### initialization

关selinux、防火墙、ssh免密和内核参数优化了等等

配置pip指定仓库: `~/.pip/pip.conf`

`kolla-ansibl bootstrap-servers`

https://s0docs0openstack0org.icopy.site/kolla-ansible/train/reference/deployment-and-bootstrapping/bootstrap-servers.html

##### ansible config

```bash
$ cat 
[defaults]
# uncomment this to disable SSH key host checking
# 默认就是False
host_key_checking=False
# Enabling pipelining reduces the number of SSH operations required to
# execute a module on the remote server. This can result in a significant
# performance improvement when enabled, however when using "sudo:" you must
# first disable 'requiretty' in /etc/sudoers
#
# By default, this option is disabled to preserve compatibility with
# sudoers configurations that have requiretty (the default on many distros).
#
#在不通过实际文件传输的情况下执行ansible模块来使用管道特性,从而减少执行远程模块SSH操作次数.如果开启这个#设置,将显著提高性能. 然而当使用”sudo:”操作的时候, 你必须在所有管理的主机的/etc/sudoers中禁用’requiretty’.
pipelining=True
## 默认值是5
forks=100


```



#### quick-start

安装kolla时，可以指定豆瓣的源`pip install kolla-ansible -i https://pypi.doubanio.com/simple`

https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html

emmm，官方文档确实写得不错。

Project stable branches will be in one of the following states:

| State                | Time frame                                        | Summary                                                      |
| :------------------- | :------------------------------------------------ | :----------------------------------------------------------- |
| Maintained           | Approximately 18 months                           | All bugfixes (that meet the criteria described below) are appropriate. Releases produced. |
| Extended Maintenance | While there are community members maintaining it. | All bugfixes (that meet the criteria described below) are appropriate. No Releases produced, reduced CI commitment. |
| Unmaintained         | 0 - 6 months                                      | The branch is under Extended Maintenance rules, but there are no maintainers. |
| End of Life (EOL)    | N/A                                               | Branch no longer accepting changes.                          |



问题：

所有虚拟机都依赖一个rbd image，同时N个（eg：500）虚拟机去读取这个rbd block会不会有瓶颈压力。

每个instance独享一个image（rbd block）是个解决办法，但是这样每次创建instances，需要copy 镜像给新的instance使用，这样无法做到秒级创建虚拟机、

即先做的，创建虚拟机勾选问题

测试云上面创建虚拟机要点掉，“是否创建新卷”吗？

如果创建新卷的方式创建新虚拟机，就会把image完全dd到一个新创建的卷里面，这个卷在volume池里面，被cinder管理。

如果为否，在vms池子里面，通过类似于cow的方式，创建一个卷

因为dd  image到卷，花费的时间太多了，超出了默认的等待时间，所以创建的时候才会报错。为否的话，就相当于把一个卷的父亲指向image.

所以在给nova用户授权的时候要给上volumes池子的权限。

ceph要预先设置三个pools：

* images： 给Glance用
* volumes ： 给Cinder用
* vms： 给instance root disk用。（如果不是COW方式，则不需要这个pool，但是往往都在是cow）

nova还有一种启动方式：“从镜像启动（创建一个新卷）”

这个流程中，nova会在_prep_block_device中的attach_block_device去调用cinder的create创建一个卷

然后会在_prep_block_device中的attach_block_device去调用cinder的attach挂载这个卷

然后才去spawn，孵化虚拟机，在spawn中会调用\_create_image来创建镜像，这里创建镜像不是创建卷，卷是在_prep_block_device创建好的，除非是“从镜像启动”才会在

_create_image创建卷。



基于Guest root disk实现两种不同方式

* row： 引用Glance中的image，将变动部分保存着nova自己的pool中。有点快速，确定glance pool GG了，整个vm全部挂掉哈哈哈

* copy：将image copy一份到新建cinder volume中，然后启动虚拟机，这样每个vm完全真正的独享images。

  纵然glance pool挂了，也不会影响存在的vm，因为已经解耦了。但是缺点是，第一次创建instance时太慢了。

  太慢了，而且对nova-compute的disk和整个网络带宽都有影响。





##### 为什么能够实现秒级创建虚拟机??

[https://int32bit.me/2017/11/23/OpenStack%E4%BD%BF%E7%94%A8Ceph%E5%AD%98%E5%82%A8-Ceph%E5%88%B0%E5%BA%95%E5%81%9A%E4%BA%86%E4%BB%80%E4%B9%88/](https://int32bit.me/2017/11/23/OpenStack使用Ceph存储-Ceph到底做了什么/)

https://specs.openstack.org/openstack/nova-specs/specs/juno/implemented/rbd-clone-image-handler.html

https://www.bladewan.com/2017/08/05/ceph_clone/

1. 创建虚拟机时并没有拷贝镜像，也不需要下载镜像，而是一个简单clone操作，因此创建虚拟机基本可以在秒级完成。
2. 如果镜像中还有虚拟机依赖，则不能删除该镜像，换句话说，删除镜像之前，必须删除基于该镜像创建的所有虚拟机。

如果没有使用ceph做后端存储，openstack创建虚机的流程是，先探测本地是否已经有镜像存在了，如果没有，则需要从glance仓库拷贝到对应的计算节点启动，如果有则直接启动，因此网络IO开销是非常大的如果使用Qcow2镜像格式，创建快照时需要commit当前镜像与base镜像合并并且上传到Glance中，这个过程也通常需要花费数分钟的时间。

当使用ceph做后端存储时，由于ceph是分布式存储，虚拟机镜像和根磁盘都是ceph的rbd image，所以就不需要copy到对应的计算节点，直接从原来的镜像中clone一个新镜像RBD image clone使用了COW技术，即写时拷贝，克隆操作并不会立即复制所有的对象，而只有当需要写入对象时才从parent image中拷贝对象到当前image中。因此，创建虚拟机几乎能够在秒级完成。

注意Glance使用Ceph RBD做存储后端时，镜像必须为raw格式，否则启动虚拟机时需要先在计算节点下载镜像到本地，并转为为raw格式，这开销非常大。

步骤如下
1，先基于镜像 做个snapshot、并添加protect、因为clone操作只能针对snapshot、

2， 创建虚拟机根磁盘
创建虚拟机时，直接从glance镜像的快照中clone一个新的RBD image作为虚拟机根磁盘:

`rbd clone 1b364055-e323-4785-8e94-ebef1553a33b@snap fe4c108a-7ba0-4238-9953-15a7b389e43a_disk`

3，启动虚拟机
启动虚拟机时指定刚刚创建的根磁盘，由于libvirt支持直接读写rbd镜像，因此不需要任何下载、导出工作。

为了秒级快照，这导致的后果就是，
1，做了快照的云主机在控制台删除后在ceph存储的pool中仍然还会存在，因为你的快照根主机的images还是存在父子关系，数据还是共享的，
2，云主机 xxxx_disk 存在snapshot，因为做clone是基于protect的snapshot，没flatten的话，snapshot自然没删除。
3，残留云主机和snapshot没清理的话会导致上传在glance的镜像也无法删除，因为你的云主机的xxx_disk是基于你glance的镜像clone的存在父子关系不能删除。。

解决办法，后台定期flatten然后rm那些客户已经删了云主机的残留数据。

thin-provisioned

当分配一个100G的空间时，并不会立刻占满这100，只是占用了一些文件的元数据，当写入数据时会根据实际的大小动态的分配,类似linux中的稀疏文件。

cow （copy on write)

写时复制，也就是做快照时，先圈好个位置但这里面是空的，只有当父镜
像有数据有变化时，这时会先将变化前的数据cp到快照的空间，然后继续修改父镜象，这样做的好处是可以节省大量做快照的时间和减少存储空间，因为snapshot存储的都是发生改变前的区域，其它区域都是与父镜像共享的。

优点：cow快照，拷贝只是拷贝一些元数据，所以拷贝速度特别快，同时相比全量快照，占用的空间也要少很多

缺点：cow快照后的第一次数据更新时父镜像每次要写数据，要先将原始数据读出来然后在拷贝到快照卷中，然后在写父镜像，这样进行一次更新操作就需要一次读+两次写，会降低父镜像的写性能，如果父镜像链接更多的快照，那性能会更低。



ceph快照是基于cow（copy on write）

ceph 使用 COW （copy on write）方式实现 snapshot：在写入object 之前，将其拷贝出来，作为 snapshot 的 data object，然后继续修改原始数据。
rbd在创建快照时，并不会向pool中创建对象，也就是说并不会占用实际的存储空间,只是增加了一些对象信息

ceph clone

clone：clone是将一个snapshot变成一个image，它是基于snapshot 创建 Clone 是将 image 的某一个 Snapshot 的状态复制变成一个 image。如 imageA 有一个 Snapshot-1，clone 是根据 ImageA 的 Snapshot-1 克隆得到 imageB。imageB 此时的状态与Snapshot-1完全一致，并且拥有 image 的相应能力，其区别在于 ImageB 此时可写。

从用户角度来看，一个 clone 和别的 RBD image 完全一样。你可以对它做 snapshot、读/写、改变大小 等等，总之从用户角度来说没什么限制。同时，创建速度很快，这是因为 Ceph 只允许从 snapshot 创建 clone，而 snapshot 需要是只读（protect）的。

向 clone 的 instance 的object 写数据
ceph的克隆也是采用cow技术， 从本质上是 clone 的 RBD image 中读数据，对于不是它自己的 data objects，ceph 会从它的 parent snapshot 上读，如果它也没有，继续找它的parent image，直到一个 data object 存在。从这个过程也看得出来，该过程是缺乏效率的。

向 clone 的 instance object 写数据

Ceph 会首先检查该 clone image 上的 data object 是否存在。如果不存在，则从 parent snapshot 或者 image 上拷贝该 data object，然后执行数据写入操作。这时候，clone 就有自己的 data object 了



```bash
# 这么多inventory分组

$ cat deploy/multinode |grep '^\['|wc -l
230

# 大多数都是引用其他分组要修改的就
$ cat multinode |grep -B 1 '01'
# These hostname must be resolvable from your deployment host
control01
--
# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla
--
[network]
network01
--
[compute]
compute01
--
[monitoring]
monitoring01
--
# and specify like below:
#compute01 neutron_external_interface=eth0 api_interface=em1 storage_interface=em1 tunnel_interface=em1
--
[storage]
storage01


### kolla 要修改的地方 ： globals.yml
### 官网写的真细
### globals.yml is the main configuration file for Kolla-Ansible. There are a few options that are required to deploy Kolla-Ansible:
## Source builds are proven to be slightly more reliable than binary. proven
kolla_install_type: "source"
## 指定OpenStack 安装版本
## Valid option is Docker repository tag 小写即可
openstack_release: "train"

# Docker options
################
# Below is an example of a private repository with authentication. Note the
# Docker registry password can also be set in the passwords.yml file.

docker_registry: "172.16.0.10:4000"
docker_namespace: "companyname"
docker_registry_insecure: "yes"
docker_registry_username: "sam"
docker_registry_password: "correcthorsebatterystaple"
Docker client timeout in seconds.
docker_client_timeout: 120
#docker_configure_for_zun: "no"

## network
## https://docs.openstack.org/kolla-ansible/latest/admin/production-architecture-guide.html#network-configuration
## 这么多接口，对比来看下吧
$ cat globals.yml |grep interface|grep '"'
network_interface: "eth0"
kolla_external_vip_interface: "{{ network_interface }}"
api_interface: "{{ network_interface }}"
storage_interface: "{{ network_interface }}"
cluster_interface: "{{ network_interface }}"
swift_storage_interface: "{{ storage_interface }}"
swift_replication_interface: "{{ swift_storage_interface }}"
tunnel_interface: "{{ network_interface }}"
dns_interface: "{{ network_interface }}"
octavia_network_interface: "{{ api_interface }}"
neutron_external_interface: "eth1"
ironic_dnsmasq_interface: "{{ network_interface }}"

#Enable additional services
## 默认是no
enable_cinder: "yes"

##开启本地lvm做存储后端
enable_cinder_backend_lvm: "yes"

# 有这么多组件可以开启的嘛
$ cat globals.yml |grep enable_|wc -l
178


```

####kolla-ansible

83个roles可还行，也就是83个组件。麻痹

```bash
$ tree  -L  2 ./
./
├── action_plugins
│   ├── merge_configs.py
│   └── merge_yaml.py
├── bifrost.yml
├── certificates.yml
├── destroy.yml
├── filter_plugins
│   ├── address.py
│   └── services.py
├── gather-facts.yml
├── group_vars
│   └── all.yml
├── inventory
│   ├── all-in-one
│   └── multinode
├── kolla-host.yml
├── library
│   ├── bslurp.py
│   ├── kolla_ceph_keyring.py
│   ├── kolla_container_facts.py
│   ├── kolla_docker.py
│   └── kolla_toolbox.py
├── mariadb_backup.yml
├── mariadb_recovery.yml
├── nova.yml
├── post-deploy.yml
├── roles
│   ├── aodh
│   ├── barbican
│   ├── baremetal
│   ├── bifrost
│   ├── blazar
│   ├── ceilometer
│   ├── ceph
│   ├── ceph_pools.yml
│   ├── certificates
│   ├── chrony
│   ├── cinder
│   ├── cloudkitty
│   ├── collectd
│   ├── common
│   ├── congress
│   ├── cyborg
│   ├── designate
│   ├── destroy
│   ├── elasticsearch
│   ├── etcd
│   ├── freezer
│   ├── glance
│   ├── gnocchi
│   ├── grafana
│   ├── haproxy
│   ├── haproxy-config
│   ├── heat
│   ├── horizon
│   ├── influxdb
│   ├── ironic
│   ├── iscsi
│   ├── kafka
│   ├── karbor
│   ├── keystone
│   ├── kibana
│   ├── kuryr
│   ├── magnum
│   ├── manila
│   ├── mariadb
│   ├── masakari
│   ├── memcached
│   ├── mistral
│   ├── module-load
│   ├── monasca
│   ├── mongodb
│   ├── multipathd
│   ├── murano
│   ├── neutron
│   ├── nova
│   ├── nova-cell
│   ├── nova-hyperv
│   ├── octavia
│   ├── opendaylight
│   ├── openvswitch
│   ├── ovs-dpdk
│   ├── panko
│   ├── placement
│   ├── prechecks
│   ├── prometheus
│   ├── qdrouterd
│   ├── qinling
│   ├── rabbitmq
│   ├── rally
│   ├── redis
│   ├── sahara
│   ├── searchlight
│   ├── senlin
│   ├── service-ks-register
│   ├── service-rabbitmq
│   ├── service-stop
│   ├── skydive
│   ├── solum
│   ├── storm
│   ├── swift
│   ├── tacker
│   ├── telegraf
│   ├── tempest
│   ├── trove
│   ├── vitrage
│   ├── vmtp
│   ├── watcher
│   ├── zookeeper
│   └── zun
└── site.yml

88 directories, 23 files

```



#### multi regions

https://docs.openstack.org/kolla-ansible/latest/user/multi-regions.html

#### 涉及到的其他组件详细介绍

https://docs.openstack.org/kolla-ansible/latest/reference/index.html

#### Neutron HA

https://www.bladewan.com/2017/07/23/dvr/



#### Openstack 科普

![openstack-arch-kilo-logical-v1](D:\data_files\MarkDown\Images\openstack-arch-kilo-logical-v1.png)

可以放大细细品，核心组件：

OpenStack Bare Metal Service： ironic

OpenStack Dashboard：Horizon

OpenStack Identity Service： Keystone

Openstack Telemetry: ceilometer ？？？

OpenStack Object Storage： Swift

OpenStack Compute： Nova

OpenStack Block Storage： cinder

OpenStack Image service： glance

OpenStack Networking： Neutron

OpenStack Data Processing： Sahara

至于和ceph关系密切的ceph嘛

openstack的后端存储需求包括块、对象、文件，而ceph是同时提供这三类存储接口的统一存储，通过整合Openstack和ceph，可以大大降低云环境的部署和运维复杂度。

https://zhuanlan.zhihu.com/p/31581145

Ceph是当前非常流行的开源分布式存储系统，具有高扩展性、高性能、高可靠性等优点，同时提供块存储服务(rbd)、对象存储服务(rgw)以及文件系统存储服务(cephfs)。

使用Ceph作为OpenStack后端存储，具有如下优点：

- 所有的计算节点共享存储，迁移时不需要拷贝根磁盘，即使计算节点挂了，也能立即在另一个计算节点启动虚拟机（evacuate）。

- 利用COW（Copy On Write)特性，创建虚拟机时，只需要基于镜像clone即可，不需要下载整个镜像，而clone操作基本是0开销，从而实现了秒级创建虚拟机。

- Ceph RBD支持thin provisioning，即按需分配空间，有点类似Linux文件系统的sparse稀疏文件。创建一个20GB的虚拟硬盘时，最开始并不占用物理存储空间，只有当写入数据时，才按需分配存储空间。

  RBD管理的核心对象为块设备(block device)，通常我们称为volume，不过Ceph中习惯称之为image（注意和OpenStack image的区别）。

Ceph中还有一个pool的概念，类似于namespace，不同的pool可以定义不同的副本数、pg数、放置策略等。每个image都必须指定pool。image的命名规范为`pool_name/image_name@snapshot`，比如`openstack/test-volume@test-snap`，表示在`openstack`pool中`test-volume`image的快照`test-snap`



**Glance管理的核心实体是image**，它是OpenStack的核心组件之一，为OpenStack提供镜像服务(Image as Service)，主要负责OpenStack镜像以及镜像元数据的生命周期管理、检索、下载等功能。

Glance支持将镜像保存到多种存储系统中，后端存储系统称为store，访问镜像的地址称为location，location可以是一个http地址，也可以是一个rbd协议地址。只要实现store的driver就可以作为Glance的存储后端，其中driver的主要接口如下:

- get: 获取镜像的location。
- get_size: 获取镜像的大小。
- get_schemes: 获取访问镜像的URL前缀(协议部分)，比如rbd、swift+https、http等。
- add: 上传镜像到后端存储中。
- delete: 删除镜像。
- set_acls: 设置后端存储的读写访问权限。



镜像的snapshot、diff、cow等特性都是由ceph rbd提供的特性。





**Nova管理的核心实体为server**，为OpenStack提供计算服务，它是OpenStack最核心的组件。

其实Nova主要作用可以对标k8s了

启动虚拟机之前首先需要准备根磁盘(root disk)，Nova称为image，和Glance一样，Nova的image也支持存储到本地磁盘、Ceph以及Cinder(boot from volume)中。需要注意的是，image保存到哪里是通过image type决定的，存储到本地磁盘可以是raw、qcow2、ploop等，如果image type为rbd，则image存储到Ceph中。不同的image type由不同的image backend负责，其中rbd的backend为`nova/virt/libvirt/imageackend`中的`Rbd`类模块实现。

> 这里强哥提到一个问题，所有虚拟机都依赖一个rbd image，同时N个（eg：500）虚拟机去读取这个rbd block会不会有瓶颈压力。
>
> 每个instance独享一个image（rbd block）是个解决办法，但是这样每次创建instances，需要copy 镜像给新的instance使用，这样无法做到秒级创建虚拟机、

When you create and instance, there are 2 kinds of disks: Root and Ephemeral. Default flavors use only root disks. Root disks are stored in `/var/lib/nova/instances/<instanceid>` on the host where that instance was spinned up (nova compute node), by default. That parameter is controlled by 'instances_path' variable in nova.conf.



首先说点题外话，我感觉Nova把create image和create snapshot弄混乱了，我理解的这二者的区别:

- create image：把虚拟机的根磁盘上传到Glance中。
- create snapshot: 根据image格式对虚拟机做快照，qcow2和rbd格式显然都支持快照。快照不应该保存到Glance中，由Nova或者Cinder(boot from Cinder)管理。

可事实上，Nova创建快照的子命令为`image-create`，API方法也叫`_action_create_image()`，之后调用的方法叫`snapshot()`。而实际上，对于大多数image type，如果不是从云硬盘启动(boot from volume)，其实就是create image，即上传镜像到Glance中，而非真正的snapshot。

当然只是命名的区别而已，这里对create image和create snapshot不做任何区别。



Cinder是OpenStack的块存储服务，类似AWS的EBS，管理的实体为volume。Cinder并没有实现volume provide功能，而是负责管理各种存储系统的volume，比如Ceph、fujitsu、netapp等，支持volume的创建、快照、备份等功能，对接的存储系统我们称为backend。只要实现了`cinder/volume/driver.py`中`VolumeDriver`类定义的接口，Cinder就可以对接该存储系统。





OpenStack 里有三个地方可以和 Ceph 块设备结合：

- Images： OpenStack 的 Glance 管理着 VM 的 image 。Image 相对恒定， OpenStack 把它们当作二进制文件、并以此格式下载。
- Volumes： Volume 是块设备， OpenStack 用它们引导虚拟机、或挂载到运行中的虚拟机上。 OpenStack 用 Cinder 服务管理 Volumes 。
- Guest Root Disks: Guest  Root disks 是装有客户操作系统的磁盘。默认情况下，启动一台虚拟机时，它的系统盘表现为 hypervisor 文件系统的一个文件（通常位于 /var/lib/nova/instances//）。

ceph要预先设置三个pools，共享一个`ceph.conf`的ceph配置信息，但是各种组件中要指定pool_name参数：

* images： 给Glance用
* volumes ： 给Cinder用
* vms： 给instance root disk用。（如果不是COW方式，则不需要这个pool，但是往往都在是cow）



#### dhcp-agent

https://yq.aliyun.com/articles/70786

核心： 每个计算几点都启动一个neutron-dhcp-agent，然后添加任意数量到Network中提供DHCP服务。

neutron-dhcp-agent，用于创建和管理虚拟DHCP server，每个虚拟网络都会有一个DHCP server，这个DHCP server为这个虚拟网络里面的虚拟机提供IP。

为了保障neutron-dhcp-agent的高可用，我们应该在每个网络中只是存在两个neutron-dhcp-agent。

Kolla-ansible默认每个网络就一个neutron-dhcp-agent。可通过参数修改设置：

```bash
## 列出kolla-ansible中配置每个网络中的dhcp-agent的个数配置
## emm，这里已经是2了但是好像没生效
$ grep -inr 'per_network' ./
./ansible/roles/neutron/defaults/main.yml:271:dhcp_agents_per_network: 2
./ansible/roles/neutron/templates/neutron.conf.j2:53:dhcp_agents_per_network = {{ dhcp_agents_per_network }}

$ neutron  agent-list|grep dhcp
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
| 05ce16cf-baff-4dab-b266-44dfe831a090 | DHCP agent         | cs1-opc-14.zpepc.com.cn | nova              | :-)   | True           | neutron-dhcp-agent        |
| 0b644014-891e-4c39-8ca0-c1de3a642624 | DHCP agent         | cs1-opc-17.zpepc.com.cn | nova              | :-)   | True           | neutron-dhcp-agent        |
| 2f397530-f141-4ecd-a083-e0e761700bfe | DHCP agent         | cs1-opc-2               | nova              | :-)   | True           | neutron-dhcp-agent        |
| 34654b51-dd3c-46f7-ba24-51a44d9762e9 | DHCP agent         | cs1-opc-3               | nova              | :-)   | True           | neutron-dhcp-agent        |
| 353db729-3466-4087-bd6b-908c293404c3 | DHCP agent         | cs1-opc-5               | nova              | :-)   | True           | neutron-dhcp-agent        |
| 3fc8a01f-4fe2-408a-ba53-1d607632a8e9 | DHCP agent         | cs1-opc-15.zpepc.com.cn | nova              | :-)   | True           | neutron-dhcp-agent        |
| 51489a5c-c440-4a4c-979f-4694cf4f4b90 | DHCP agent         | cs1-opc-4               | nova              | :-)   | True           | neutron-dhcp-agent        |
| 664dc784-fdfe-4dad-90e1-c675710dec0c | DHCP agent         | cs1-opc-16.zpepc.com.cn | nova              | :-)   | True           | neutron-dhcp-agent        |
| 69f22986-bfeb-4c79-ac45-6ad0c3ef9d53 | DHCP agent         | cs1-opc-11              | nova              | :-)   | True           | neutron-dhcp-agent        |
| 79ae4821-65f0-4f36-8684-f9572ce1ace4 | DHCP agent         | cs1-opc-12              | nova              | :-)   | True           | neutron-dhcp-agent        |
| 8d1aacd1-5981-47ee-9242-37639a0792ba | DHCP agent         | cs1-opc-10              | nova              | :-)   | True           | neutron-dhcp-agent        |
| 8db76277-6611-42df-a96e-a348415df370 | DHCP agent         | cs1-opc-18.zpepc.com.cn | nova              | :-)   | True           | neutron-dhcp-agent        |
| 95db5ef9-8248-490b-8a41-c4be003f7307 | DHCP agent         | cs1-opc-7               | nova              | :-)   | True           | neutron-dhcp-agent        |
| 996dcd07-2984-46c2-937c-cc3e6a3e7de8 | DHCP agent         | cs1-opc-8               | nova              | :-)   | True           | neutron-dhcp-agent        |
| afcb7026-567a-4fb4-a5eb-b8e20d4d2c93 | DHCP agent         | cs1-opc-1               | nova              | :-)   | True           | neutron-dhcp-agent        |
| b5f3296b-c682-4047-a9aa-d512e0717053 | DHCP agent         | cs1-opc-6               | nova              | :-)   | True           | neutron-dhcp-agent        |
| ca942631-a0d7-43a2-ae1f-559352851999 | DHCP agent         | cs1-opc-13              | nova              | :-)   | True           | neutron-dhcp-agent        |
| dc83abde-949f-41e7-9021-0ff4478f827f | DHCP agent         | cs1-opc-9               | nova              | :-)   | True           | neutron-dhcp-agent        |

## 列出所有网络
$ neutron net-list

neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+-------------+----------------------------------+----------------------------------------------------+
| id                                   | name        | tenant_id                        | subnets                                            |
+--------------------------------------+-------------+----------------------------------+----------------------------------------------------+
| 18dadce9-f262-4b89-ac33-6854c28e7b24 | cs1-yw-3    | ae39f633e6f14a578ab383b673f31f95 | 59ac56f5-8375-4a21-9869-5e73b9ba17eb 21.49.18.0/24 |
| 226b260c-a4bd-468a-879e-cd0a4d780869 | cs1-yw-1    | ae39f633e6f14a578ab383b673f31f95 | 561f60f1-f76f-4d8b-ac89-bda2f0078d49 21.49.16.0/24 |
| 487f7b67-4e42-4fe4-9a20-e17aba8a5005 | cs1-fzjh-24 | ae39f633e6f14a578ab383b673f31f95 | ae5f60e0-f13a-4840-bdd8-37fdbb0ef16a 21.49.24.0/24 |
| bf85cee8-7814-4bd2-8b5d-c423734c177d | cs1-yw-2    | ae39f633e6f14a578ab383b673f31f95 | 32921f9b-afab-4479-b68b-02d1691e9938 21.49.17.0/24 |
+--------------------------------------+-------------+----------------------------------+----------------------------------------------------+

## 查看网络中指定的dhcp-agent
$ neutron  dhcp-agent-list-hosting-net cs1-yw-1

neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+------------+----------------+-------+
| id                                   | host       | admin_state_up | alive |
+--------------------------------------+------------+----------------+-------+
| 69f22986-bfeb-4c79-ac45-6ad0c3ef9d53 | cs1-opc-11 | True           | :-)   |
+--------------------------------------+------------+----------------+-------+
$ neutron  dhcp-agent-list-hosting-net cs1-yw-3

neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+------------+----------------+-------+
| id                                   | host       | admin_state_up | alive |
+--------------------------------------+------------+----------------+-------+
| 79ae4821-65f0-4f36-8684-f9572ce1ace4 | cs1-opc-12 | True           | :-)   |
+--------------------------------------+------------+----------------+-------+

## 给网络新加一个dhcp-agent提供ha
$ neutron dhcp-agent-network-add 8d1aacd1-5981-47ee-9242-37639a0792ba cs1-yw-1
## 再次查看
$ neutron  dhcp-agent-list-hosting-net cs1-yw-1

+--------------------------------------+------------+----------------+-------+
| id                                   | host       | admin_state_up | alive |
+--------------------------------------+------------+----------------+-------+
| 69f22986-bfeb-4c79-ac45-6ad0c3ef9d53 | cs1-opc-11 | True           | :-)   |
+--------------------------------------+------------+----------------+-------+
| 8d1aacd1-5981-47ee-9242-37639a0792ba | cs1-opc-10 | True           | :-)   |
+--------------------------------------+------------+----------------+-------+
```

###### vm挂载boot失败

直观现象：ip丢失，网卡down status，无法连接主机

且提示无法重启network服务，一层层排查，发现`polkit`没启动，然后`busctl`发现`D-bus`全部不见，

最后reboot。看到了错误信息：` Failed to mount /boot` ，然后查看`systemctl status boot.mount`.

最后，发现`/etc/fstab`被修改了导致`/boot`挂载失败--

```bash
Restarting network (via systemctl):  Authorization not available. Check if polkit service is running or see debug message for more information.
Welcome to emergency mode! After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or ^D to
try again to boot into default mode.
Give root password for maintenance
(or press Control-D to continue): 



[root@qdxt-tst2 ~]# systemctl cat polkit
# /usr/lib/systemd/system/polkit.service
[Unit]
Description=Authorization Manager
Documentation=man:polkit(8)

[Service]
Type=dbus
BusName=org.freedesktop.PolicyKit1
ExecStart=/usr/lib/polkit-1/polkitd --no-debug
[root@qdxt-tst2 ~]# systemctl status  polkit
● polkit.service - Authorization Manager
   Loaded: loaded (/usr/lib/systemd/system/polkit.service; static; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:polkit(8)
[root@qdxt-tst2 ~]# systemctl start  polkit
Authorization not available. Check if polkit service is running or see debug message for more information.
Welcome to emergency mode! After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or ^D to
try again to boot into default mode.
Give root password for maintenance
(or press Control-D to continue): 
Login incorrect

Give root password for maintenance
(or press Control-D to continue): 
[root@qdxt-tst2 ~]# systemctl status  polkit
● polkit.service - Authorization Manager
   Loaded: loaded (/usr/lib/systemd/system/polkit.service; static; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:polkit(8)


[root@qdxt-tst2 ~]# /usr/lib/polkit-1/polkitd
Successfully changed to user polkitd
Error getting system bus: Could not connect: No such file or directory
** (polkitd:3512): WARNING **: 18:53:26.782: Error getting system bus: Could not connect: No such file or directory
18:53:26.856: Loading rules from directory /etc/polkit-1/rules.d
18:53:26.859: Loading rules from directory /usr/share/polkit-1/rules.d
18:53:26.876: Finished loading, compiling and executing 2 rules
Entering main event loop
18:53:26.878: Lost the name org.freedesktop.PolicyKit1 - exiting
Shutting down
Exiting with code 0


## busctl???
[root@qdxt-tst2 ~]# busctl 
Failed to connect to bus: No such file or directory


## 重启

[root@qdxt-tst2 ~]# reboot
Authorization not available. Check if polkit service is running or see debug message for more information.
[ 7790.675067] systemd-shutdown[1]: Failed to finalize  DM devices, ignoring
[ 7790.689242] Restarting system.

[FAILED] Failed to mount /boot.
See 'systemctl status boot.mount' for details.


[FAILED] Failed to mount /boot.
See 'systemctl status boot.mount' for details.
[DEPEND] Dependency failed for Local File Systems.
[DEPEND] Dependency failed for Migrate local... structure to the new structure.
[DEPEND] Dependency failed for Relabel all filesystems, if necessary.



[root@qdxt-tst2 ~]# systemctl status boot.mount -l
● boot.mount - /boot
   Loaded: loaded (/etc/fstab; bad; vendor preset: disabled)
   Active: failed (Result: exit-code) since Fri 2020-04-10 19:14:30 CST; 8min ago
    Where: /boot
     What: /dev/vda
     Docs: man:fstab(5)
           man:systemd-fstab-generator(8)
  Process: 2657 ExecMount=/bin/mount /dev/vda /boot -t xfs (code=exited, status=32)

Apr 10 19:14:30 qdxt-tst2.novalocal systemd[1]: Mounting /boot...
Apr 10 19:14:30 qdxt-tst2.novalocal mount[2657]: mount: /dev/vda is already mounted or /boot busy
Apr 10 19:14:30 qdxt-tst2.novalocal systemd[1]: boot.mount mount process exited, code=exited status=32
Apr 10 19:14:30 qdxt-tst2.novalocal systemd[1]: Failed to mount /boot.
Apr 10 19:14:30 qdxt-tst2.novalocal systemd[1]: Unit boot.mount entered failed state.
```

end



####官网： 秒级启动虚拟机

流弊流弊，官网还是吊

不用cow流程：

每个实例实现在ceph中新建一个rbd volume，然后又nova-compute node将镜像（如果本地没有还得从glance下载镜像）写入该volume，这非常耗时（ceph分布式存储，写入数据需要校验什么的，而且镜像很大）。

这导致，虚拟机创建十分缓慢。

Currently RBD-backed ephemeral disks are created by downloading an image from Glance to a local file, then uploading that file intoRBD. Even if the file is cached, uploading may take a long time, since ‘rbd import’ is synchronous and slow. If the image is already stored in RBD by Glance, there’s no need for any local copies - it can be cloned to a new image for a new disk without copying the data at all.

Problem description[¶](https://specs.openstack.org/openstack/nova-specs/specs/juno/implemented/rbd-clone-image-handler.html#problem-description)

The primary use case that benefits from this change is launching an instance from a Glance image where Ceph RBD backend is enabled for both Glance and Nova, and Glance images are stored in RBD in RAW format.

Following problems are addressed:

- Disk space on compute nodes is wasted by caching an additional copy of the image on each compute node that runs instances from that image.
- Disk space in Ceph is wasted by uploading a full copy of an image instead of creating a copy-on-write clone.
- Network capacity is wasted by downloading the image from RBD to a compute node the first time that node launches an instance from that image, and by uploading the image to RBD every time a new instance is launched from the same image.
- Increased time required to launch an instance reduces elasticity of the cloud environment and increases the number of in-flight operations that have to be maintained by Nova.

Proposed change[¶](https://specs.openstack.org/openstack/nova-specs/specs/juno/implemented/rbd-clone-image-handler.html#proposed-change)

Extract RBD specific utility code into a new file, align its structure and provided functionality in line with similar code in Cinder. This includes the volume cleanup code that should be converted from rbd CLI to using the RBD library.

Add utility functions to support cloning, including checks whether image exists and whether it can be cloned.

Add direct_fetch() method to nova.virt.libvirt.imagebackend, make its implementation in the Rbd subclass try to clone the image when possible. Following criteria are used to determine that the image can be cloned:

- Image location uses the rbd:// schema and contains a valid reference to an RBD snapshot;
- Image location references the same Ceph cluster as Nova configuration;
- Image disk format is ‘raw’;
- RBD snapshot referenced by image location is accessible by Nova.

Extend fetch_to_raw() in nova.virt.images to try direct_fetch() when a new optional backend parameter is passed. Make the libvirt driver pass the backend parameter.

Instead of calling disk.get_disk_size() directly from verify_base_size(), which assumes the disk is stored locally, add a new method that is overridden by the Rbd subclass to get the disk size.