## Overlay and Underlay

### Underlay

**Underlay网络可以是由多个类型设备互联而成的物理网络，负责网络之间的数据包传输。**

在Underlay网络中，互联的设备可以是各类型交换机、路由器、负载均衡设备、防火墙等，但网络的各个设备之间必须通过路由协议来确保之间IP的连通性。

Underlay网络可以是二层也可以是三层网络。其中二层网络通常应用于以太网，通过VLAN进行划分。三层网络的典型应用就是互联网，其在同一个自治域使用OSPF、IS-IS等协议进行路由控制，在各个自治域之间则采用BGP等协议进行路由传递与互联。随着技术的进步，也出现了使用[MPLS](https://info.support.huawei.com/info-finder/encyclopedia/zh/MPLS.html)这种介于二三层的WAN技术搭建的Underlay网络。

然而传统的网络设备对数据包的转发都基于硬件，其构建而成的Underlay网络也产生了如下的问题：

- 由于硬件根据目的IP地址进行数据包的转发，所以传输的路径依赖十分严重。
- 新增或变更业务需要对现有底层网络连接进行修改，重新配置耗时严重。
- 互联网不能保证私密通信的安全要求。
- 网络切片和网络分段实现复杂，无法做到网络资源的按需分配。
- 多路径转发繁琐，无法融合多个底层网络来实现负载均衡。
- end

#### Underlay网络模型

**Underlay网络就是传统IT基础设施网络，由交换机和路由器等设备组成，借助以太网协议、路由协议和VLAN协议等驱动，它还是Overlay网络的底层网络，为Overlay网络提供数据通信服务**。





### Overlay

https://www.modb.pro/db/242497

为了摆脱Underlay网络的种种限制，现在多采用网络虚拟化技术在Underlay网络之上创建虚拟的Overlay网络。

在Overlay网络中，设备之间可以通过逻辑链路，按照需求完成互联形成Overlay拓扑，只要VTEP建立的vxlan tunnel延时满足要求即可。。

相互连接的Overlay设备之间建立隧道，数据包准备传输出去时，设备为数据包添加新的IP头部和隧道头部，并且被屏蔽掉内层的IP头部，数据包根据新的IP头部进行转发。当数据包传递到另一个设备后，外部的IP报头和隧道头将被丢弃，得到原始的数据包，在这个过程中Overlay网络并不感知Underlay网络。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/overlay-arch.png)

Overlay网络有着各种网络协议和标准，包括VXLAN、NVGRE、SST、GRE、NVO3、EVPN等。

VXLAN和NVGRE是实现隧道化的高级网络虚拟化技术，它们将虚拟网络的规模从4094扩展到了1600万，并且允许第2层的数据包在第3层的网络上进行传输，因此大型数据中心通常会添加支持NVGRE和VXLAN的网络设备来扩展网络，如，使用支持NVGRE和VXLAN的交换机克服虚拟局域网在大型数据中心中的限制，且提供更为敏捷的虚拟机网络环境。

 基于IP网络构建Fabric。无特殊拓扑限制，IP可达即可；承载网络和业务网络分离；对现有网络改动较小，保护用户现有投资

#### Overlay解决的问题

在不改变原先架构的基础之上新建一个Overlay的网络，来为网络多活、数据中心互联及大数据等云计算业务提供支撑。

- Overlay网络是指建立在另一个网络上的网络。该网络中的结点可以看作通过虚拟或逻辑链路而连接起来的。
- Overlay网络具有独立的控制和转发平面，对于连接在overlay边缘设备之外的终端系统来说，物理网络是透明的。
- Overlay网络是物理网络向云和虚拟化的深度延伸，使资源池化能力可以摆脱物理网络的重重限制，是实现云网融合的关键。

部署Overlay网络后，针对VLAN技术下广播风暴问题，Overlay对广播流量转化为组播流量，可以避免网络本身的无效流量带宽浪费。

#### Overlay类型

##### Network Overlay

网络Overlay指的是隧道封装在物理交换机上完成，通过控制协议对边缘的网络设备完成网络构建和扩展。

网络Overlay依仗的主要技术就是VXLAN。在网络Overlay中，要求所有的物理接入交换机都能支持VXLAN，物理服务器支持SR-IOV功能，使虚拟机通过SR-IOV技术直接与物理交换机相连，虚拟机的流量在接入交换机上进行VXLAN报文的封装和卸载，对于非虚拟化服务器，直接连接支持VXLAN接入交换机，服务器流量在接入交换机上进行VXLAN报文封装和卸载。

**在VXLAN网络和VLAN网络之间要部署VXLAN GW交换机**，实现VXLAN网络主机与VLAN网络主机之间的通信。目前，网络设备的主流芯片已经实现了VXLAN技术，对于在数据中心里部署网络Overlay提供了可能。网络Overlay的优势在于物理网络设备性能转发性能比较高，可以支持非虚拟化的物理服务器之间的组网互通。

总结：Overlay隧道封装在物理交换机完成。优势在于物理网络设备性能转发性能比较高，可以支持非虚拟化的物理服务器之间的组网互通。

* 物理设备为网络边缘设备
* 服务器无需支持Overlay
* 可能进行Overlay标签转换
* 支持多种形态的服务器

###### SR-IOV

SR-IOV是Single Root I/O Virtualization的缩写。

SR-IOV是虚拟化的一个重要功能。启用SR-IOV的这个功能，将大大减轻宿主机的CPU负荷，提高网络性能，降低网络时延等。

Intel提出来SR-IOV这个东西。SR-IOV最初应用在网卡上。简单的说，就是一个物理网卡可以虚拟出来多个轻量化的PCI-e物理设备，从而可以分配给虚拟机使用。

![so-iov](D:\temp_files\download\so-iov.jpg)

###### 使用场景

网络Overlay组网里的服务器可以是多形态，也无需支持Overlay功能，所以**网络Overlay的定位主要是网络高性能、与Hypervisor平台无关的Overlay方案。**

网络Overlay主要面向对性能敏感而又对虚拟化平台无特别倾向的客户群。该类客户群的网络管理团队和服务器管理团队的界限一般比较明显

主机Overlay将虚拟设备作为Overlay网络的边缘设备和网关设备，Overlay功能纯粹由服务器来实现。主机Overlay方案适用于服务器虚拟化的场景，支持VMware、KVM、CAS等主流Hypervisor平台。主机Overlay的网关和服务节点都可以由服务器承担，成本较低。



##### Host Overlay

这个SDN控制器的引流太6了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/主机Overlay-6.jpg)



主机Overlay指的是隧道封装在服务器的vSwitch上完成，不用增加新的网络设备即可完成Overlay部署，可以支持虚拟化的服务器之间组网互通。主机Overlay使用服务器上的vSwitch软件实现VXLAN网络功能，VXLAN技术中的VTEP、VXLAN GW、VXLAN IP GW 均通过安装在服务器上的vSwitch软件实现，只需要物理网络设备支持IP转发即可，所有IP可达的主机即可构建一个大范围二层网络。

这种主机Overlay技术实现，屏蔽了物理网络的模型与拓扑差异，将物理网络的技术实现与计算虚拟化的关键要求分离开来，几乎可以支持以太网在任意网络上的透传，使得云的计算资源调度范围空前扩大。为了使得VXLAN Overlay网络更加简化运行管理，便于云的服务提供，各厂家使用集中控制的模型，将分散在多个物理服务器上的vSwitch构成一个大型的、虚拟化的分布式Overlay vSwitch，只要在分布式vSwitch范围内，**虚拟机在不同物理服务器上的迁移，便被视为在一个虚拟的设备上迁移，如此大大降低了云中资源的调度难度和复杂度。**

总结：overlay隧道封装在vSwitch完成，不用增加新的网络设备即可完成Overlay部署，可以支持虚拟化的服务器之间的组网互通。

* 虚拟设备为网络边缘设备
* 适用服务器全虚拟化的场景
* 不能接入非虚拟化服务器

###### 使用场景

主机Overlay不能接入非虚拟化服务器，所以主机Overlay主要定位是配合VMware、KVM等主流Hypervisor平台的Overlay方案。

主机Overlay主要面向已经选择了虚拟化平台并且希望对物理网络资源进行利旧的客户。

##### Hybird Overlay

主机Overlay是一个全部靠软件实现的技术，主要应用在全部计算虚拟化的场景中。这种场景没有考虑到虚拟化计算资源和非虚拟化计算资源的互通，主机Overlay方案与传统网络互通时，连接也比较复杂。虚拟网络完全由服务器vSwitch软件构建，对于VXLAN和非VXLAN互通的转换全靠软件技术，转换性能存在瓶颈，**通过软件实现 VXLAN IP GW很可能会成为整个网络的瓶颈**。网络Overlay是一种由物理接入交换机组成Overlay边缘设备的技术，这种场景下接入交换机需要支持VXLAN，而不需要软件支持。但**网络Overlay方案与虚拟机相连，需要通过一些特殊的要求或技术实现虚拟机与VTEP的对接，组网上不够灵活**。如此一来，单纯部署每一种技术时都存在一定限制。最理想的组网方案应该是一个结合了网络Overlay与主机Overlay两种方案优势的混合Overlay方案。  混合Overlay指的是采用网络Overlay和主机Overlay的混合组网，可以支持物理服务器和虚拟服务器之间的组网互通。混合Overlay采用软硬件结合的方式，使得软硬件都能发挥自己的优势，也保障了Overlay网络的整体性能。

总结：是网络Overlay和主机 Overlay的混合组网，可以支持物理服务器和虚拟服务器之间的组网互通。

* 物理设备和虚拟设备都可以作为网络边缘设备，灵活组网
* 可接入各种形态服务器

###### 使用场景

混合Overlay组网灵活，即可以支持虚拟化的服务器,也可以支持利旧的未虚拟化物理服务器，以及必须使用物理服务器提升性能的数据库等业务，所以混合Overlay的主要定位是Overlay整体解决方案，它可以为客户提供自主化、多样化的选择。

混合Overlay主要面向愿意即要保持虚拟化的灵活性,又需要兼顾对于高性能业务的需求, 或者充分利旧服务器的要求,满足客户从传统数据中心向基于SDN的数据中心平滑演进的需求。

###### vSwitch and Switch路由同步

##### 混合Overlay路由同步：frr

通常，混合Overlay则需要主机Overlay angent对网络leaf设备的做相应的优化，提升vxlan的识别和信息传输。

假设基于OVS实现的vSwitch，作为Leaf，与物理Spine组网。

N个运行OVS的宿主机都是请求彼此ovs vswtich上的vms路由信息的，但是有些基于物理Leaf设备的路由，天然和ovs vswitch信息不同的。

所以，需要ovs 宿主机上要借助一个agent，通过bgp协议从RR(Spine设备)同步物理Leaf上的信息,并翻译成ovs vswitch可以理解的路由。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/混合overlay路由.jpg)

#### Overlay组网流量转发:star:

H3C提供的网络Overlay组网方式，支持以下转发模式：

http://www.h3c.com/cn/d_201603/920927_30003_0.htm#_Toc444502050

1. 控制器流转发模式(强控？)： 控制器负责Overlay网络部署、主机信息维护和转发表项下发，即VXLAN L2 GW上的MAC表项由主机上线时控制器下发，VXLAN IP GW上的ARP表项也由控制器在主机上线是自动下发，并由控制器负责代答和广播ARP信息。这种模式下，如果设备和控制器失，设备会临时切换到自转发状态进行逃生

2. 数据平面EVPN转发模式（弱控？）： 控制器负责Overlay网络的灵活部署，转发表项由Overlay网络交换机通过MP-BGP EVPN学习，即VXLAN L2 GW上自学习主机MAC和网关MAC信息，VXLAN IP GW上可以通过EVPN路由学习主机ARP信息并在网关组成员内同步。

3. 混合转发模式： 同时控制器也可以基于主机上线向VXLAN IP GW上下发虚机流表，如果VXLAN IP GW上自学习ARP和控制器下发的虚机流表信息不一样，则以VXLAN IP GW上自学习ARP表项为主，交换机此时触发一次arp请求，保证控制器和交换机自学习主机信息的正确性和一致性；数据平面自转发模式下ARP广播请求报文在VXLAN网络内广播的同时也会上送控制器，控制器可以做代答，这种模式是华三的一种创新,实现了Overlay网络转发的双保险模型

#### Overlay的技术实现

Overlay技术是一个网络叠加技术。可以灵活的在已有的三层网络上，通过隧道技术，叠加一个二层网络。

##### STT

StatelessTransportTunneling是一个术语

Protocol是Nicira提交的一种通道协议，与VXLAN和VGGRE相似，它将两个帧封装在一个ip消息包的payload中，并且在前面加上tcp头和STT头。请注意STT的tcp头是为了充分利用TSO,LRO,GRO等网卡特性而精心构造的。 

作者：云V小编 https://www.bilibili.com/read/cv11414019 出处：bilibili

##### NVGRE：硬件有要求

NVGRE使用了GRE扩展字段flow ID进行流量负载分担，这就要求物理网络能够识别GRE隧道的扩展信息，这就对硬件有要求了。

NVGRE的主要支持者是Microsoft。不像VXLAN,NVGRE没有采用标准的传输协议(TCP/UDP)，而是借助了通用路由封装协议(GRE)。NVGRE使用GRE头的低24位(TNI)作为租户网络标识符，它能像VXLAN一样支持1600个虚拟网络。传输网络需要使用GRE头来提供描述带宽利用率粒度的流，但这会导致NVGRE不能兼容传统的负载平衡，这是NVGRE与VXLAN最大的差异和缺点。为增强负载平衡能力，建议每台NVGRE主机使用多个IP地址，以确保负载平衡的能力

 作者：云V小编 https://www.bilibili.com/read/cv11414019 出处：bilibili

##### Geneve

Generic Network Virtualization Encapsulation的简称，对应中文是通用网络虚拟化封装，由IETF草案定义。

在实现上，GENEVE与VXLAN类似，仍然是Ethernet over UDP，也就是用UDP封装Ethernet。VXLAN header是固定长度的（8个字节，其中包含24bit VNI），与VXLAN不同的是，GENEVE header中增加了TLV（Type-Length-Value），由8个字节的固定长度和0~252个字节变长的TLV组成。GENEVE header中的TLV代表了可扩展的元数据。

##### VxLAN:star2:

VXLAN技术已经成为目前Overlay技术事实上的标准，得到了非常广泛的应用。

在这三大技术中，VXLAN 的表现最为出众。首先，业务可在任意位置灵活 部署，更具灵活性；因其部署方便，有更好的扩展性，在配合了高可靠 SDN Controller 完成控制面的配置和管理后，则更具备部署的简易性。同时，L2-4 层链路 HASH 能力强，不需要改造现有网络。NVGRE 技术需要网络设备支持， 对传输层无修改，使用标准的 UDP 传输流量。STT 则需要修改 TCP。相对而言， VXLAN 在业内获得的支持度最高。

VXLAN（Virtual eXtensible LAN，可扩展虚拟局域网络）是基于IP网络、采用“MAC in UDP”封装形式的二层VPN技术。VXLAN可以基于已有的服务提供商或企业IP网络，为分散的物理站点提供二层互联功能，主要应用于数据中心网络。

####　混合Overlay实现demo

主机与主机之间使用：geneve 

主机和网络设备之间使用：VxLAN

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/混合overlay-demo.jpg)

主机overlay架构

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/主机Overlay架构.jpg)



#### SDN/NFV and Overlay: 666

设备要求 --> NFV --> H3C：666

Underlay网络正如其名，是Overlay网络的底层物理基础。

为了实现Overlay的组网需求，Underlay对设备也有要求，比如要支持VxLAN或者NVGRE协议，但是当交换机设备不支持时怎么办，那就使用sdn/NFV哇，用软件模拟一个支持VxLAN或者NVGRE协议的网络设备。这么多虚拟设备怎么管理呢，就需要一个SDN/NFV 控制器了。

再联系到新华三的产品：

* NFV：为SDN Overlay网络提供虚拟网络功能设备，然后还得使用交换或路由协议组网，实现高可用或者聚合等等

* VCF：VCFC-DC 接管整个 SDN Overlay 网络。H3C的SDN控制器称为VCF控制器

* CAS虚拟平台，配置CVM和VCFC资源联动，即实现cas平台创建出来的虚拟机可以使用定义的SDN Overlay网络--

  > cvk则是cas中的计算节点 cvm是cas管理节点 cvm+cvm=cas

* end

#### 数据中心Overlay

随着数据中心架构演进，现在数据中心多采用Spine-Leaf架构构建Underlay网络，通过VXLAN技术构建互联的Overlay网络，业务报文运行在VXLAN Overlay网络上，与物理承载网络解耦。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/data-center-overlay.png)

Leaf与Spine全连接，等价多路径(ECMP)提高了网络的可用性。

Leaf节点作为网络功能接入节点，提供Underlay网络中各种网络设备接入VXLAN网络功能，同时也作为Overlay网络的边缘设备承担VTEP（VXLAN Tunnel EndPoint）的角色。

Spine节点即骨干节点，是数据中心网络的核心节点，提供高速IP转发功能，通过高速接口连接各个功能Leaf节点。

#### SD-WAN中的Overlay网络

SD-WAN的Underlay网络基于广域网，通过混合链路的方式达成总部站点、分支站点、云网站点之间的互联。通过搭建Overlay网络的逻辑拓扑，完成不同场景下的互联需求。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/SD-wan-overlay.png)

SD-WAN的网络主要由CPE设备构成，其中CPE又分为Edge和GW两种类型。

- Edge：是SD-WAN站点的出口设备。
- GW：是联接SD-WAN站点和其他网络（如传统VPN）的网关设备。

根据企业网络规模、中心站点数量、站点间互访需求可以搭建出多个不同类型的Overlay网络。

- Hub-spoke：适用于企业拥有1~2个数据中心，业务主要在总部和数据中心，分支通过WAN集中访问部署在总部或者数据中心的业务。分支之间无或者有少量的互访需求，分支之间通过总部或者数据中心绕行。
- Full-mesh：适用于站点规模不多的小企业，或者在分支之间需要进行协同工作的大企业中部署。大企业的协同业务，如VoIP和视频会议等高价值的应用，对于网络丢包、时延和抖动等网络性能具有很高的要求，因此这类业务更适用于分支站点之间直接进行互访。
- 分层组网：适应于网络站点规模庞大或者站点分散分布在多个国家或地区的大型跨国企业和大企业，网络结构清晰，网络可扩展性好。
- 多Hub组网：适用于有多个数据中心，每个数据中心均部署业务服务器为分支提供业务服务的企业。
- POP组网：当运营商/MSP面向企业提供SD-WAN网络接入服务时，企业一时间不能将全部站点改造为SD-WAN站点，网络中同时存在传统分支站点和SD-WAN站点这两类站点，且这些站点间有流量互通的诉求。一套IWG（Interworking Gateway，互通网关）组网能同时为多个企业租户提供SD-WAN站点和已有的传统MPLS VPN网络的站点连通服务。

end



### Underlay vs Overlay

Underlay网络 VS Overlay网络

| 对比项       | Underlay网络                                                 | Overlay网络                                                  |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据传输     | 通过网络设备例如路由器、交换机进行传输                       | 沿着节点间的虚拟链路进行传输                                 |
| 包封装和开销 | 发生在网络的二层和三层                                       | 需要跨源和目的封装数据包，产生额外的开销                     |
| 报文控制     | 面向硬件                                                     | 面向软件                                                     |
| 部署时间     | 上线新服务涉及大量配置，耗时多                               | 只需更改虚拟网络中的拓扑结构，可快速部署                     |
| 多路径转发   | 因为可扩展性低，所以需要使用多路径转发，而这会产生更多的开销和网络复杂度 | 支持虚拟网络内的多路径转发                                   |
| 扩展性       | 底层网络一旦搭建好，新增设备较为困难，可扩展性差             | 扩展性强，例如VLAN最多可支持4096个标识符，而VXLAN则提供多达1600万个标识符 |
| 协议         | 以太网交换、VLAN、路由协议（OSPF、IS-IS、BGP等）             | VXLAN、NVGRE、SST、GRE、NVO3、EVPN                           |
| 多租户管理   | 需要使用基于NAT]或者VRF的隔离，这在大型网络中是个巨大的挑战  | 能够管理多个租户之间的重叠IP地址                             |

end

### Overlay on VPN：好好看

MPLS/ L3 VPN中介绍的Overlay是怎样的。

VPN的两大类别：

Peer-to-Peer VPN
Overlay VPN

### 引用

1. https://www.cnblogs.com/fengdejiyixx/p/15567609.html
2. 