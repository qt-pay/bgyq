## GRE 隧道技术

### GRE协议

GRE（Generic Routing Encapsulation，通用路由封装），用于将使用一个路由协议的数据包封装在另一协议的数据包中。“封装”是指将一个数据包包装在另一个数据包中，就像将一个盒子放在另一个盒子中一样。GRE 是在网络上建立直接点对点连接的一种方法，目的是简化单独网络之间的连接。

> 可以将任何协议的数据包封装到另一个协议中，没有限制。
>
> 还可以将IPV6数据包封装到IPV4中传输

利用GRE连接IPV6网络孤岛（公网设备只识别IPV4）。

核心：建立隧道，打通私网

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-protoc.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-tunnel.png)

GRE最大缺点：没有任何安全性

点到点配置也比较麻烦，有新设备加入就需要更新配置

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-pro-and-con.png)

#### 协议号

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/IP协议号.jpg)

#### 封装

一个封装在IP Tunnel中的X协议报文的格式如下

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-封装.gif)

需要封装和传输的数据报文，称之为净荷（Payload），净荷的协议类型为乘客协议（Passenger Protocol）。系统收到一个净荷后，首先使用封装协议（Encapsulation Protocol）对这个净荷进行GRE封装，即把乘客协议报文进行了“包装”，加上了一个GRE头部成为GRE报文；然后再把封装好的原始报文和GRE头部封装在IP报文中，这样就可完全由IP层负责此报文的前向转发（Forwarding）。通常把这个负责前向转发的IP协议称为传输协议（Delivery Protocol或者Transport Protocol）。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-封装-2.2.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-封装-3.png)

Tunnel0接口的地址也是私网地址不会出现在公网路由中，所以封装的公网IP头是Tunnel创建时关联的公网接口IP

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-封装-5.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre封装-6.jpg)

GRE header 很简单就只有 `flags and Version`和`0x0800` Protocol Type来标识这个数据流字节从这里开始就是GRE协议的内容了（图中13.0.0.1就是私网地址了）。对于支持GRE的设备来说，从这里开始解析数据，对于不支持GRE的设备这就相当于无效的数据流，它只负责传输就行了。

类似k8s Annotation，只要被自己的CRD对象对应的进程解析和执行相应逻辑就行了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-封装-4.png)

#### 效率

GRE收发双方的加封装、解封装处理，以及由于封装造成的数据量增加，会导致使用GRE后设备的数据转发效率有一定程度的下降。

相当于一段程序一直在遍历数据流，然后发现GRE header就处理下数据，效率必须下降。

#### 配置

创建Tunnel接口时关键公网路由接口。Tunnel接口负责引流，它作为私网通讯的下一跳（如果走路由器公网接口出去，就相当于私有地址进入运营商网络了，私网数据包会直接被丢弃无法进行转发），Tunnel关联的公网接口实现数据包的公网传输。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-tunnel-转发.png)

1. 互通的路由设备上配置tunnel，建立虚拟隧道
2. 配置隧道路由，即发网对端私网地址的下一跳地址为`Tunnle0`
3. 设备进行GRE封装，将Tunnel关联的公网IP放到GRE外层IP数据包中
4. 根据路由器公网IP进行路由转发直到到达Tunnel0另一端。

```bash
# 创建隧道接口并配置隧道src和des 地址
interface tunnel 0/0/1
tunnel-protocol gre
source 12.0.0.2
destination 13.0.0.3

## 配置路由


```

### GRE注意事项

### 不能宣告公网接口:face_with_head_bandage:

使用动态路由协议宣告接口时，千万不能宣告Tunnel关联的公网接口。

GRE隧道的逻辑：

1. 私网IP下一跳是Tunnel0
2. 在Tunnel0完成GRE封装，外出IP协议src和des变成公网地址
3. 根据路由表将公网IP的src和des传输到指定网络设备

如果动态路由协议将公网IP宣告出去了，`s1/0`公网IP的下一跳变成了Tunnel0。当私网数据包发送时，私网数据包包进入Tunnel0完成封装，再进入路由表，一查看公网ip下一跳还是Tunnel0就变成无限递归的路由了。

因为OSPF协议宣告的路由优先级是大于static或default的。

> 通常，路由器的公网出口都是default route，优先级很低

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-宣告公网路由.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-禁止宣告公网IP.jpg)



### Tunnel and Nat

Tunnel数据包的传输依赖公网路由，抓包的数据看起来和NAT方式一样。但是基于Nat方式有很多弊端

* 映射IP或端口管理

* 无法直接实现Site-to-Site或LAN-to-LAN的通讯

  即隧道可以直接方式对方的私网地址

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/vpn-and-nat.png)

### GRE-IPSec

GRE可以封装组播数据，并可以和IPSec结合使用从而保证语音、视频等组播业务的安全。

GRE提供隧道，IPSec负责安全

组播可以减少上游带宽！

> 一个交换机连接50台电脑（用户），如果这50个用户都在看网络直播，假设直播流速为2Mbps, 请问**这个交换机从上游下拉的流速是多少？是2Mbps**吧？**向下游（用户）推送的总流速是多少？ 50\*2= 100 Mbps!**
>
> 意味着虽然有50个用户在看直播，但对上游网络的带宽压力只有 2Mbps。
>
> 但如果是单播呢？对上游网络的带宽压力为 50*2 = 100Mbps。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gre-ipsec.gif)



### GRE VPN and VXLAN



### 引用

1. http://www.h3c.com/cn/d_200805/605933_30003_0.htm
2. https://www.zhihu.com/question/65869123/answer/235679423
