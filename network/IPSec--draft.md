## IPSec--draft

### 定义

IPSec是IETF（Internet Engineering Task Force）制定的**一组**开放的网络安全协议(或者说是安全框架)。IPSec并不是一个单独的协议（抓包看不到IPSec协议），而是一系列为IP网络提供安全性的协议和服务的集合，包括认证头AH（Authentication Header）和封装安全载荷ESP（Encapsulating Security Payload）两个安全协议、密钥交换和用于验证及加密的一些算法等。

IPSec，作为一种开放标准的安全架构结构，可以用来保证IP数据报文在网络上传输的机密性、完整性和防重放。

IPSec不支持组播，所以OSPF的路由组播就用不了...

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec.jpg)

### 核心

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-core.jpg)

* 机密性：对数据进行加密，确保数据在传输过程中不被其他人员查看
* 完整性：对接收到数据包进行完整性验证，以确保数据在传输过程中没有被篡改。
* 真实性：验证数据源，以保证数据来自真实的发送者（IP报文内的源地址）。身份验证，比如wifi password
* 防重放：防止恶意用户通过重复发送捕获的数据包所进行的攻击即接收方会拒绝旧的或重复的数据包。

#### 防重放

**防重放**（**Anti-replay**）亦称**反重放**、**抗重放**，是IETF之IPsec的一个子协议。它的主要目的是避免黑客注入或篡改从源到目的地的网路封包。防重放协议使用单向安全关联（英语：Unidirectional security association）为网络中的两个节点之间建立安全连接。在安全连接建立后，防重放协议使用封包序列号抗衡重放攻击，具体如下：在源发送消息时，它向其封包添加一个序列号；序列号从0开始，后续每个封包递增1。目的地则以“滑动窗口”维护有效接收分组的序列号记录，它将拒绝所有所含序列号低于滑动窗口中最小值（即太旧）或者序列号已于滑动窗口中出现（即重复/重放）的封包。通过验证的已接收封包将使滑动窗口更新，如果滑动窗口已满则将最小的序列号移出窗口。



API重放攻击（Replay Attacks）又称重播攻击、回放攻击，这种攻击会不断恶意或欺诈性地重复一个有效的API请求。攻击者利用网络监听或者其他方式盗取API请求，进行一定的处理后，再把它重新发给认证服务器，是黑客常用的攻击方式之一。

HTTPS加密可以有效防止明文数据被监听，但是却防止不了重放攻击。

原理

1. 请求所有的内容都被加入签名计算，所以请求的任何修改，都会造成签名失败。
2. 不修改内容
   - X-Ca-Timestamp：发起请求的时间，可以取自机器的本地实现。当API网关收到请求时，会校验这个参数的有效性，误差不超过15分钟。
   - X-Ca-Nonce：这个是请求的唯一标识，一般使用UUID来标识。API网关收到这个参数后会校验这个参数的有效性，同样的值，15分内只能被使用一次。

### 架构

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-arch.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-arch-2.jpg)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-arch-3.jpg)

#### AH/ESP

可以两个协议同时使用，通常单独使用ESP协议就满足需求了 。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-core-protocol.jpg)



#### IKE密钥协商

IKE要配置的参数挺多的...

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-ike-args.jpg)

#### 封装方式:lollipop:

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-encapsulation.jpg)

隧道模式，IPSec头的位置换成GRE头就是GRE VPN的封装包了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-encapsulation-diff.png)

demo

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-encapsulation-demo.jpg)

##### 传输模式

在传输模式（Transport Mode）下，IPSec头被插入到IP头之后但在所有传输协议之前，或所有其他IPSec协议之前。

单独的AH是没有加密部分的--

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-transport-mode.jpg)



##### 隧道模式

在隧道模式（Tunnel Mode）下，IPSec头插在原始IP头之前，另外生成一个新的报文头放到AH或ESP之前。

其实，本质上还是和Transport Mode一样的，AH/ESP报文要在负责路由的IP Header之后。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ipsec-tunnel-mode.jpg)

### 配置IPSec

