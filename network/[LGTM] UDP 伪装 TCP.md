## [LGTM] UDP 伪装 TCP

### 运营商 UDP QoS

运营商处于什么动机对 UDP 进行 QoS 呢，为啥非要故意把网络质量变差呢

在国内网络环境下会遇到一个致命的问题：**UDP 封锁/限速**

国内运营商根本没有能力和精力根据 TCP 和 UDP 的不同去深度定制不同的 QoS 策略，几乎全部采取一刀切的手段：**对 UDP 进行限速甚至封锁**。

广域网上是没法广播的。主要原因就是 UDP （或者说非 TCP 、SCTP 协议）不带流控，容易被暴力发包滥用。

但现在 GitHub 上已经有很多把 UDP 伪装成 TCP 来绕过限速的工具了。

#### 基于UDP的协议

| 协议 | 全称                                                  | 默认端口                                         |
| :--- | :---------------------------------------------------- | :----------------------------------------------- |
| DNS  | Domain Name Service (域名服务)                        | 53                                               |
| TFTP | Trivial File Transfer Protocol (简单文件传输协议)     | 69                                               |
| SNMP | Simple Network Management Protocol (简单网络管理协议) | 通过UDP端口161接收，只有Trap信息采用UDP端口162。 |
| NTP  | Network Time Protocol (网络时间协议)                  | 123                                              |
| QUIC | quick udp internet connection 快速 UDP 互联网连接     |                                                  |

### UDP 伪装 TCP

目前支持将 UDP 流量伪装成 TCP 流量的主流工具是 **udp2raw**和**Phantun**。

Phantun 整个项目**完全使用 Rust 实现**，性能吊打 udp2raw。它的初衷和 udp2raw 类似，都是为了实现一种简单的用户态 TCP 状态机来对 UDP 流量做伪装。主要的目的是希望能让 UDP 流量看起来像是 TCP，又不希望受到 TCP retransmission 或者 congestion control 的影响。

需要申明的是，**Phantun 的目标不是为了替代 udp2raw**，从一开始 Phantun 就希望设计足够的简单高效，所以 udp2raw 支持的 **ICMP 隧道，加密，防止重放**等等功能 Phantun 都选择不实现。

Phantun 假设 UDP 协议本身已经解决了这些问题，所以整个转发过程就是简单的明文换头加上一些必要的 TCP  状态控制信息。对于我日常使用的 WireGuard 来说，Ph

#### 应用 WireGuard VPN

[WireGuard](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU1MzY4NzQ1OA==&action=getalbum&album_id=1612086810350829568&scene=173&from_msgid=2247513357&from_itemidx=1&count=3&nolastread=1#wechat_redirect) 作为一个更先进、更现代的 VPN 协议，比起传统的 IPSec、OpenVPN 等实现，效率更高，配置更简单，并且已经合并入 Linux 内核，使用起来更加方便。