###  云网网络组网和流量分析demo

#### 整体架构

广域网网络架构没有Spine-leaf架构，Spine-leaf是数据中心网络架构。

下面互联网区，一个Internet Border和一个互联网出口有什么区别？互联网出口支持NAT？？？

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/net-arch-vxlan-demo.jpg)

董哥的意思是，网络架构一共就三层核心层、汇聚层、接入层，这个Internet Border IRF是汇聚层负责策略。

解封装数据包也算策略的一部分的样子--

核心层管路由，接入层管终端。

##### Intranet  and Internet 

互联网区（公网区）和公共服务区都用一组FW + Border，它们border解封装和FW作nat的逻辑是一样的。区别是

公共服务区是一个预留的固定网段，在Spine上配置路由优先级，DMZ区网段单独路由条目，其余流量走默认路由到互联网出口区。即通过静态路由区别流量去DMZ还是默认出口。

内网核心业务区Spine(RR)



#### 互联网出口区

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/互联网出口区.jpg)

**互联网出口区：**

* 互联网区Border节点设备在网络架构中中承担边界网关的功能，对于云主机访问Internet的业务流量进行VXLAN解封装，将IP报文发送给FW进行处理。

  核心作用：是租户vxlan报文的解封装。 因为核心区Spine只负责VxLAN数据转发，没有解封装操作。

* 互联网区FW将云主机访问Internet的业务进行安全过滤和NAT转换，然后发送到Inetnet。 

* 业务区核心交换机**上连**一组交换机(IRF)作为Internet border设备，border旁挂一组防火墙（IRF）作为租户南北向防火墙，为云内VPC业务配置安全策略；

* Internet-Border-IRF上连一组FW互联网边界防火墙作为云数据中心边界防火墙设备，为整个云数据中心提供安全防护及公网NAT映射；

* FW互联网边界防火墙上连一组互联网业务接入交换机，运营商互联网链路直接连至互联网出口交换机，互联网业务接入交换机上；

  **两台互联网接入交换机，根据运营商分配多个公网地址，实现多路径备份。**

租户云主机访问互联网区（默认地址网段）时，流量到达Leaf后转发到Internet Border，Internet Border解封装后转发到Internet FW，在Internet FW上使用公网弹性网络地址池（Internet EIPpool）中的弹性IP进行NAT后转发到互联网。

#### 专线区

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/fw-as-border.jpg)

* 运营商专线直接连线至一组广域网边界防火墙，对所有进出专线区的流量做安全防护；

* 广域网边界防火墙下连至一组广域网接入交换机，作为业务接入交换机，上面承载所有VPC的分布式网关；

  接入leaf交换机负载数据中心vxlan数据包解封装。

从虚机发出的流量，源IP是VPC内虚机的IP，目的IP为用户数据中心内部的网络IP，流量从VM出发，在OVS经过VLAN封装，到达接入交换机Leaf经过VXLAN封装后，经过核心交换机Spine，转到专线交换机专线Leaf，流量在专线Leaf上解封装VXLAN之后，根据不同的VRF封装指定的VLAN到达专线FW，在专线FW上经过一定的策略或者NAT规则之后，根据跨VRF的路由转换为该用户的专线VRF到达专线接入交换机ACC，最后经过专线VLAN到达用户数据中心。

#### 核心区

网络Overlay方案整体采用Spine-Leaf架构，Spine节点设备在网络架构中配置为RR（路由反射器），RR与leaf 及Border建立iBGP邻居，用于反射EVPN发布的路由。各Leaf及各Border之间通过RR同步EVPN路由信息。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/net-vxlan-core.jpg)

* 两台业务核心交换机作为VXLAN网络的Spine设备，不同服务器就近通过业务Leaf设备接入VXLAN网络，Spine作为RR设备，与leaf之间多条链路通过ECMP实现。
* 在业务接入leaf上进行业务报文VXLAN封装，然后在border上解封装；

#### 流量分析

前提：数据中心内流量都是vxlan流量，进出数据中心需要vxlan网关设备实现数据的解封装。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/internet-datacenter-traffic.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/acc-traffic.jpg)

**出方向流量模型**

从虚机发出的流量，源IP是VPC内虚机的IP，目的IP为用户数据中心内部的网络IP，流量从VM出发，在OVS经过VLAN封装，到达接入交换机Leaf经过VXLAN封装后，经过核心交换机Spine，转到专线交换机专线Leaf，流量在专线Leaf上解封装VXLAN之后，根据不同的VRF封装指定的VLAN到达专线FW，在专线FW上经过一定的策略或者NAT规则之后，根据跨VRF的路由转换为该用户的专线VRF到达专线接入交换机ACC，最后经过专线VLAN到达用户数据中心。

**入方向流量模型**

用户数据中心的流量经过专线VLAN到达专线接入ACC，在专线ACC上根据VRF区分不同的专线，通过指定的VLAN到达专线FW，在专线FW经过指定的策略或者NAT后，根据跨VRF的路由转换为VPC内子网的流量，达到专线Leaf，封装为指定的VXLAN，到达业务网的Leaf，解封装VXLAN,流量到达虚机。



#### 内网核心switch

内网核心spine交换机，在整个网络架构中是处于什么角色呢？

大二层网络架构，下面业务网关都在核心上，现在把一条外网线，直接插在核心上，这个该怎么上网，需要怎么配置，下面的业务能和互联网通，核心能直接当出口吗？

答：不能，核心交换机不支持NAT，不能当出口。

用户需要访问internet，需要在防火墙上做**NAT映射**，将内网地址映射成地址池中的公网IP地址。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/医院拓扑.png)

end