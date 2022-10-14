## [LGTM] k8s service IP (LVS)问题排查

## k8s service ipvs 是LVS的哪种模式

### Port Mapping[ ](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/#port-mapping)

There are three proxy modes in IPVS: NAT (masq), IPIP and DR. **Only NAT mode supports port mapping. Kube-proxy leverages NAT mode for port mapping.** The following example shows IPVS mapping Service port 3080 to Pod port 8080.

```
TCP  10.102.128.4:3080 rr
  -> 10.244.0.235:8080            Masq    1      0          0         
  -> 10.244.1.237:8080            Masq    1      0       
```

因为LVS packet-forwarding methods

* NAT：Masquerade，支持端口映射，需要关闭RS的ICMP重定向。

  用户请求和响应报文都必须经过Director Server地址重写，所以NAT模式效率最差。

* DR：二层直接修改mac完成转发，效率最高。

  调度器与真实服务器都有一块网卡连在同一物理网段上，且真实服务器不作ARP响应。

* Tunnel：DS和RS可以处在任意的网络

IP数据包的IP如果不在网卡NIC上，直接就被丢弃了（非混杂模式），进不了Linux NetworkStack。

所以，DR和Tunnel模式都需要在real server上配置VIP的，这才能保证数据包进来。

NAT模式，由于Director Server在iptables上完成了NAT转换，所以real sever 不需要配置vip，但是real server需要将网关指向Director Server。

## 问题阶段(一):ipvsadm挺好

用户反应某个redis使用卡顿，连接该redis服务使用的是svc代理，即ipvs snat的方式，ipvsadm -L发现，VIP收到的6379端口的数据包，会以rr的方式分别转发到pod的80 6379端口上，相当于会有50%的丢包，不卡才怪：

```bash
$ ipvsadm | grep -2 10.108.152.210
TCP  10.108.152.210:6379 rr
  -> 172.26.6.185:http            Masq    1      0          0         
  -> 172.26.6.185:6379            Masq    1      3          3
```

2.检查svc：

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2018-12-06T02:45:49Z
  labels:
    app: skuapi-rds-prod
    run: skuapi-rds-prod
  name: skuapi-rds-prod
  namespace: default
  resourceVersion: "41888273"
  selfLink: /api/v1/namespaces/default/services/skuapi-rds-prod
  uid: 09e1a61f-f901-11e8-878f-141877468256
spec:
  clusterIP: 10.108.152.210
  ports:
  - name: redis
    port: 6379
    protocol: TCP
    targetPort: 6379
  selector:
    app: skuapi-rds-prod
  sessionAffinity: None
  type: ClusterIP
```

可以看出svc配置没有问题，回忆起此前配置此svc时，一开始失误将port和targetPort设置为了默认值80忘记修改了，后面手动kubectl edit svc修改端口为6379。应该是修改保存之后，原本的targetPort 80
依然在ipvs中保留为RIP之一，这应该是一个bug。

这里猜测应该不是bug，而是k8s patchStrategy的问题，默认是Strategic Merge Patch，很多时候是合并而不是替换。

这里使用json patch是不是好点，直接指明`op=replace`

```bash
$ cat <<EOF >$DEMO_HOME/ingress_patch.json
[
  {"op": "replace",
   "path": "/spec/rules/0/host",
   "value": "foo.bar.io"},

  {"op": "replace",
   "path": "/spec/rules/0/http/paths/0/backend/servicePort",
   "value": 80},

  {"op": "add",
   "path": "/spec/rules/0/http/paths/1",
   "value": { "path": "/healthz", "backend": {"servicePort":7700} }}
]
EOF
$ kubectl patch deployment patch-demo --patch-file patch-file.json
```

3.解决办法：
使用ipvsadm工具删除多余的RIP，或者删除此svc然后重建。

```bash
$ ipvsadm | grep -2 10.108.152.210
TCP  10.108.152.210:6379 rr
  -> 172.26.6.185:6379            Masq    1      4          0  
```

#### 正常情况lvs and svc

```bash
# kubectl describe svc nginx-service
Name:			nginx-service
...
Type:			ClusterIP
IP:			    10.102.128.4
Port:			http	3080/TCP
Endpoints:		10.244.0.235:8080,10.244.1.237:8080
Session Affinity:	None

# ip addr
...
73: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
    inet 10.102.128.4/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever

# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn     
TCP  10.102.128.4:3080 rr
  -> 10.244.0.235:8080            Masq    1      0          0         
  -> 10.244.1.237:8080            Masq    1      0          0   
```



## 问题阶段(二):

### 现象：

有用户反馈svc vip连接经常失败，不限于redis的多个ip有这种情况，于是开始排查

### 排查:

通过对ipvs和系统的连接状态排查，发现了两个问题:
1.发现有大量TIME_WAIT的tcp连接,证明系统连接大多是短连接，查看ipvs tcpfin的等待时间为2分钟，两分钟还是过长，有必要修改短一些，30秒是一个比较合理的值，如果比较繁忙的服务，这个值可以改到更低。
2.有不少丢包

```bash
$ ipvsadm -lnc |wc -l
23776
$ ipvsadm -lnc |grep TIME_WAIT | wc -l
10003
$ ipvsadm -L --timeout 
Timeout (tcp tcpfin udp): 900 120 300

$ netstat -s | grep timestamp
    17855 packets rejects in established connections because of timestamp
```

备注：
根据tcp 四次挥手协议，主动发起断开连接请求的的一端(简单理解为客户端)，在其发送断开连接请求开始，其连接的生命周期会经历4个阶段分别是FIN-WAIT1 --> FIN_WAIT2 --> TIME_WAIT --> CLOSE，其中2个FIN-WAIT阶段等待的时间就是内核配置的net.ipv4.tcp_fin_timeout 的值，为了快速跳过前两个FIN-WAIT阶段从而进入TIME_WAIT状态，net.ipv4.tcp_fin_timeout值建议缩短。在进入TIME_WAIT状态后，默认等待2个MSL(Max Segment Lifetime)时间，到达最后一步CLOSE状态，关闭tcp连接释放资源。注意：MSL时间在不同平台一般是30s-2min不等，并且基本都是不可修改的(linux将这一时间值写死在了内核中)。

那么为什么要等待2*MSL呢？在stackoverflow中找到了一个较为易懂的解释：

**So the TIME_WAIT time is generally set to double the packets maximum age. This value is the maximum age your packets will be allowed to get to before the network discards them.
That guarantees that, before you’re allowed to create a connection with the same tuple, all the packets belonging to previous incarnations of that tuple will be dead.**

**翻译一下：**
time_wait时间设计为tcp分片的最大存活时间的两倍，这么设计的原因是，网络是存在延迟的，同时tcp分片在网络传输中可能出现意外，发送端在确认意外(例如到达MSL时间后)后发出数据分片的重传。假如socket连接不经等待直接关闭了，然后再重新打开了一个端口号一致的连接，可能导致新启动的socket连接，接收到了此前销毁关闭的socket连接的数据。因此，设计TIME_WAIT等待时间为2*MSL，是为了保证在等待2*MSL之后，此前旧socket的数据分片即使还没有到达接收端，也已经在网络传输中过期消逝了，新启动的socket不会接收到此前的旧数据分片。

### 优化方式

优化的思路
1.断开连接时加速进入TIME_WAIT状态，以快速提供可用的连接端口
2.解决timestamps丢包问题

#### lvs优化

\#查看ipvs设置的各类连接的超时时间，修改默认的tcpfin 2分钟为30秒

```bash
$ ipvsadm -L --timeout 
Timeout (tcp tcpfin udp): 900 120 300
$ ipvsadm --set 900 30 300

$ ipvsadm -L --timeout 
Timeout (tcp tcpfin udp): 900 30 300
```

#### 内核参数优化

\#内核参数优化
添加入/etc/sysctl.conf文件中

```
# 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
  net.ipv4.tcp_tw_reuse = 1

  # 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。注意，net.ipv4.tcp_timestamps默认为开启，tcp_tw_recycle此选项也开启后，tcp timestamp机制就会激活，在部分场景下，需要关闭tcp_timestamps功能，见下方此选项的说明。
  net.ipv4.tcp_tw_recycle = 1

  # 修改tcp会话进去fin状态后的等待时间，超时则关闭会话
  net.ipv4.tcp_fin_timeout = 30

  # 处于TIME_WAIT最大的socket数量，默认为180 000，超过这个数目的socket立即被清除
  net.ipv4.tcp_max_tw_buckets=180000

  # tcp缓存时间戳,RFC1323中描述了，系统缓存每个ip最新的时间戳，后续请求中该ip的tcp包中的时间戳如果小于缓存的时间戳(即非最新的数据包
)，即视为无效，相应的数据包会被丢弃，而且是无声的丢弃。在默认情况下，此机制不会有问题，但在nat模式时,不同的ip被转换成同一个ip再去请
求真正的server端，在server端看来，源ip都是相同的，且此时包的时间戳顺序可能不一定保持递增，由此会出现丢包的现象，因此，如果是nat模式工作，建议关闭此选项。
  net.ipv4.tcp_timestamps = 0
```

这几个参数之间还有一些关联关系，参考此篇文章,写得非常详细:
http://www.freeoa.net/osuport/cluster/lvs-inuse-problem-sets_3111.html

## 问题阶段(三):

### 现象

完成上面的操作后，TIME_WAIT数量下降到了4位数，丢包数量没有再增加。但是过了一些天之后，再一次出现了偶尔个别VIP无法建立连接的情况。挑选了其中一个VIP 10.111.99.131开始排查
client: 192.168.58.36
DIP: 10.111.99.131
RIP: 172.26.8.17

### 开始排查

三层连接没有问题:

```bash
$ traceroute 10.111.99.131
traceroute to 10.111.99.131 (10.111.99.131), 30 hops max, 60 byte packets
 1  192.168.58.254 (192.168.58.254)  7.952 ms  8.510 ms  9.131 ms
 2  10.111.99.131 (10.111.99.131)  0.253 ms  0.243 ms  0.226 ms

$ ping 10.111.99.131
PING 10.111.99.131 (10.111.99.131) 56(84) bytes of data.
64 bytes from 10.111.99.131: icmp_seq=1 ttl=63 time=0.296 ms
64 bytes from 10.111.99.131: icmp_seq=2 ttl=63 time=0.318 ms
^C
--- 10.111.99.131 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1020ms
rtt min/avg/max/mdev = 0.296/0.307/0.318/0.011 ms
```

tcp连接无法建立

```bash
$ telnet 10.111.99.131 80
Trying 10.111.99.131...
telnet: Unable to connect to remote host: Connection timeout
```

在lvs diretor server上查看连接状态：
```bash
$ ipvsadm -lnc | grep 58.36
TCP 00:59 TIME_WAIT 192.168.9.13:58236 10.97.85.43:80 172.26.5.209:80
TCP 00:48 SYN_RECV 192.168.58.36:57964 10.111.99.131:80 172.26.8.17:80
TCP 00:48 SYN_RECV 192.168.58.36:57963 10.111.99.131:80 172.26.8.17:80
TCP 00:48 SYN_RECV 192.168.58.36:57965 10.111.99.131:80 172.26.8.17:80
```

发现tcp连接状态为SYN_RECV状态，那么根据三次握手协定，再结合lvs的工作流程，说明LVS server接收到了clien的syn包,向client回复了syn+ack然后进入了SYN_RECV状态，同时direct server会向后端的real server发起建立连接的请求。既然现在direct server能与client端交互，那么当前的问题应该在于：
**direct Server和real Server之间没有正常地进行数据包交互或者出现了丢包，查阅了很多资料，rp_filter这一内核参数可能导致这一问题。**

来看一下官方关于这一参数的解释:

rp_filter 默认为1 ，需要改为0，关闭此功能。Linux的rp_filter用于实现反向过滤技术，也即uRPF，它验证反向数据包的流向，以避免伪装IP攻击。然而，在**LVS TUN 模式**中，我们的数据包是有问题的，因为从realserver eth0 出去的IP数据包的源IP地址应该为192.168.1.62，而不是VIP地址。所以必须关闭这一项功能。DR和TUN在网络层实际上使用了一个伪装IP数据包的功能。让client收到数据包后，返回的请求再次转给分发器。

```
rp_filter - INTEGER
	0 - No source validation.
	1 - Strict mode as defined in RFC3704 Strict Reverse Path
	    Each incoming packet is tested against the FIB and if the interface
	    is not the best reverse path the packet check will fail.
	    By default failed packets are discarded.
	2 - Loose mode as defined in RFC3704 Loose Reverse Path
	    Each incoming packet's source address is also tested against the FIB
	    and if the source address is not reachable via any interface
	    the packet check will fail.

	Current recommended practice in RFC3704 is to enable strict mode
	to prevent IP spoofing from DDos attacks. If using asymmetric routing
	or other complicated routing, then loose mode is recommended.

	The max value from conf/{all,interface}/rp_filter is used
	when doing source validation on the {interface}.

	Default value is 0. Note that some distributions enable it
	in startup scripts.
```

**简单解释一下:**
0:表示不开启源检测
1:严格模式，根据数据包的源，通过查FIB表(Forward Information Table,可以理解为路由表)，检查数据包进入端口是同时也是出端口，以视为最佳路径，如果不符合最佳路径，则丢弃数据包
2:松散模式,检查数据包的来源，查FIB表，如果通过任意端口都无法到达此源，则丢包
**结合使用场景来说:**
在LVS (nat)+k8s的工作场景下，LVS Server送往Real Server的包可能走的tunnel接口，而Real Server通过tunnel接口收到包后，查路由表发现回包要走物理eth/bond之类接口，如果rp_filter开启了严格模式，会导致网络异常状况

**检查每一台kube node的网卡配置参数,发现centos7.4的几台node默认确实开启了rp_filter，ubuntu大部分则没有:**

```bash
# 容器内的veth网卡可忽略，因为容器本身只有一块对外的网卡
$ sysctl -a | grep rp_filter | grep -v 'veth'
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.bond0.rp_filter = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.docker0.rp_filter = 2
net.ipv4.conf.dummy0.rp_filter = 0
net.ipv4.conf.em1.rp_filter = 1
net.ipv4.conf.em2.rp_filter = 1
net.ipv4.conf.em3.rp_filter = 1
net.ipv4.conf.em4.rp_filter = 1
net.ipv4.conf.kube-bridge.rp_filter = 0
net.ipv4.conf.kube-dummy-if.rp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.tun-192168926.rp_filter = 1
net.ipv4.conf.tun-192168927.rp_filter = 1
net.ipv4.conf.tun-192168928.rp_filter = 1
net.ipv4.conf.tun-192168929.rp_filter = 1
net.ipv4.conf.tunl0.rp_filter = 0
```

\#关闭此功能

```bash
echo "
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.bond0.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.docker0.rp_filter = 2
net.ipv4.conf.dummy0.rp_filter = 0
net.ipv4.conf.em1.rp_filter = 0
net.ipv4.conf.em2.rp_filter = 0
net.ipv4.conf.em3.rp_filter = 0
net.ipv4.conf.em4.rp_filter = 0
net.ipv4.conf.kube-bridge.rp_filter = 0
net.ipv4.conf.kube-dummy-if.rp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.tun-192168926.rp_filter = 0
net.ipv4.conf.tun-192168927.rp_filter = 0
net.ipv4.conf.tun-192168928.rp_filter = 0
net.ipv4.conf.tun-192168929.rp_filter = 0
net.ipv4.conf.tunl0.rp_filter = 0
" >> /etc/sysctl.conf
```

\#加载生效

```bash
sysctl -p
```

### 总结

VIP偶尔无法建立TCP连接的问题已解决，一个星期过去了没有再复现，继续观察ing.

## 基于LVS DR模式的Kubernetes Service External-IP实现:???

k8s `Service`的[官方文档](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips)中介绍了一种辅助方式, 叫`External-IP`, 可以在`worker`节点上会通过该`IP`来暴露服务，而且可以使用在任意类型的`service`上。集群外的用户就可以通过该`IP`来访问服务。但如果这个`IP`只存在于一个`worker`节点上，那么就不具备高可用的能力了，我们需要在多个`worker`节点上配置这个`VIP:Virtual IP`。我们可以使用`LVS`(也叫做`IPVS`)的`DR(Director Routing)`模式作为外部负载均衡器将流量分发到多个`worker`节点上，同时保持数据包的目的地址为该`VIP`。

`DR`模式只会修改数据包的目的`MAC`地址为后端`RealServer`的`MAC`地址，因而要求负载均衡器`Director`和`RealServer`在同一个二层网络，而且响应包不会经过`Director`。

下面我们来实验如何使用`LVS`的`DR`模式实现`service`负载均衡。

我们在之前的实验集群中创建一个类型为`ClusterIP`(默认类型)的`service`, 指定一个外部`IP`:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: whoami
  name: whoami
spec:
  ports:
  - port: 80
    name: web
    protocol: TCP
  selector:
    app: whoami
  externalIPs:
  - 10.240.0.201
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: containous/whoami
        ports:
        - containerPort: 80
          name: web
```

创建服务:

```bash
kubectl apply -f whoami.yaml
```

查看服务，可以看到`whoami`的`EXTERNAL-IP`为`10.240.0.201`:

```bash
[root@master1 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP    PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.32.0.1    <none>         443/TCP   24d   <none>
whoami       ClusterIP   10.32.0.60   10.240.0.201   80/TCP    30m   app=whoami
```

在`worker`节点上检查`iptables`规则，可以看到在`KUBE-SERVICES`链中添加了`EXTERNAL-IP`相关的规则:

```bash
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.60/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.60/32 -p tcp -m comment --comment "default/whoami:web cluster IP" -m tcp --dport 80 -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES -d 10.240.0.201/32 -p tcp -m comment --comment "default/whoami:web external IP" -m tcp --dport 80 -m addrtype --dst-type LOCAL -j KUBE-SVC-225DYIB7Z2N6SCOU
-A KUBE-SERVICES ! -s 10.230.0.0/16 -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.32.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

当数据包目的地址为`10.240.0.201:80`时，跳转到`KUBE-SVC-*`链，从而分发到相应的`pod`中。

我们在节点上添加上这个`VIP`:

```bash
[root@node1 ~]# ip addr add 10.240.0.201/32 dev lo
[root@node1 ~]# ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.240.0.201/32 scope global lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```

因为这个`VIP`需要在多个`worker`节点上存在，因而把它配置在`lo`上，并抑制相应网卡上对该`VIP`的`ARP`响应:

```bash
sysctl -w net.ipv4.conf.eth1.arp_ignore = 1
sysctl -w net.ipv4.conf.eth1.arp_announce = 2
```

在节点上尝试访问`VIP`, 可以成功访问:

```bash
[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-kbv68
IP: 127.0.0.1
IP: ::1
IP: 10.230.95.10
IP: fe80::d43a:9eff:fe3e:4425
RemoteAddr: 10.230.74.0:60086
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*

[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-n6jmj
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.25
IP: fe80::9889:dff:fedf:f376
RemoteAddr: 10.230.74.1:60088
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*

[root@node1 ~]# curl http://10.240.0.201
Hostname: whoami-5df4df6ff5-2h6qf
IP: 127.0.0.1
IP: ::1
IP: 10.230.74.24
IP: fe80::2493:9aff:fe7b:5dbd
RemoteAddr: 10.230.74.1:60090
GET / HTTP/1.1
Host: 10.240.0.201
User-Agent: curl/7.29.0
Accept: */*
```

接着我们在`worker`节点所在二层网络再启动一台虚拟机作为`LVS`的`Director`。在该机器上给与`worker`节点二层互通的网卡添加`VIP`:

```
ip addr add 10.240.0.201/32 dev eth1
```

使用`ipvsadm`创建负载均衡服务, 并使用`DR`模式添加两个`worker`节点做为后端的`RealServer`:

```bash
ipvsadm -A -t 10.240.0.201:80 -s rr
ipvsadm -a -t 10.240.0.201:80 -r 10.240.0.101 -g
ipvsadm -a -t 10.240.0.201:80 -r 10.240.0.102 -g
```

查看负载均衡服务:

```
[root@lb1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.240.0.201:80 rr
  -> 10.240.0.101:80              Route   1      0          0
  -> 10.240.0.102:80              Route   1      0          0
```

环境配置完成。我们找一台客户端访问`VIP:10.240.0.201`, 同时在`Director`机器上抓包，可以看到:

```
[root@lb1 ~]# tcpdump -ieth1 -nn -e tcp port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
11:50:01.024615 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: 10.240.0.10.38482 > 10.240.0.201.80: Flags [S], seq 1959573689, win 29200, options [mss 1460,sackOK,TS val 304318064 ecr 0,nop,wscale 6], length 0
11:50:01.024640 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 74: 10.240.0.10.38482 > 10.240.0.201.80: Flags [S], seq 1959573689, win 29200, options [mss 1460,sackOK,TS val 304318064 ecr 0,nop,wscale 6], length 0
11:50:01.026358 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 3346334225, win 457, options [nop,nop,TS val 304318066 ecr 304104626], length 0
11:50:01.026406 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 1, win 457, options [nop,nop,TS val 304318066 ecr 304104626], length 0
11:50:01.027197 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 142: 10.240.0.10.38482 > 10.240.0.201.80: Flags [P.], seq 0:76, ack 1, win 457, options [nop,nop,TS val 304318067 ecr 304104626], length 76: HTTP: GET / HTTP/1.1
11:50:01.027210 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 142: 10.240.0.10.38482 > 10.240.0.201.80: Flags [P.], seq 0:76, ack 1, win 457, options [nop,nop,TS val 304318067 ecr 304104626], length 76: HTTP: GET / HTTP/1.1
11:50:01.032443 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 327, win 473, options [nop,nop,TS val 304318070 ecr 304104630], length 0
11:50:01.032468 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 327, win 473, options [nop,nop,TS val 304318070 ecr 304104630], length 0
11:50:01.036452 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [F.], seq 76, ack 327, win 473, options [nop,nop,TS val 304318072 ecr 304104630], length 0
11:50:01.037159 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [F.], seq 76, ack 327, win 473, options [nop,nop,TS val 304318072 ecr 304104630], length 0
11:50:01.047556 08:00:27:2d:af:18 > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 328, win 473, options [nop,nop,TS val 304318087 ecr 304104647], length 0
11:50:01.047583 08:00:27:48:90:6c > 08:00:27:23:1b:95, ethertype IPv4 (0x0800), length 66: 10.240.0.10.38482 > 10.240.0.201.80: Flags [.], ack 328, win 473, options [nop,nop,TS val 304318087 ecr 304104647], length 0
```

数据包的目的`MAC`地址被修改为`node2`上`eth1`的`MAC`地址, 而且响应包并不经过`Director`:

```
[root@node2 ~]# ip link show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:23:1b:95 brd ff:ff:ff:ff:ff:ff
```

根据之前网上的[这篇文章](https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/), `worker`节点可以不设置`VIP`，因为`VIP`并不需要由用户态程序来接收流量，而是直接由`iptables`来进行数据包转换。

在大多数场景下这是正确的。但是如果需要直接从`worker`节点上通过`VIP`访问该服务时就需要在`worker`节点上配置`VIP`了。数据包在从`worker`节点发送出去时，会经由`nat:OUTPUT`和`nat:POSTROUTING`来处理。`iptables`的`NAT`实现是基于`conntrack`来实现的，发出时系统会建立`conntrack`条目。`iptables`的`nat:PREROUTING`和`nat:POSTROUTING`的处理都会调用`nf_nat_ipv4_fn`函数。当数据包由`LVS Director`把数据包返回到自身这台`RealServer`时, `nat:PREROUTING`阶段调用`nf_nat_ipv4_fn`:

```
switch (ctinfo) {
case IP_CT_RELATED:
case IP_CT_RELATED_REPLY:
    if (ip_hdr(skb)->protocol == IPPROTO_ICMP) {
        if (!nf_nat_icmp_reply_translation(skb, ct, ctinfo,
                           ops->hooknum))
            return NF_DROP;
        else
            return NF_ACCEPT;
    }
    /* Fall thru... (Only ICMPs can be IP_CT_IS_REPLY) */
case IP_CT_NEW:
    /* Seen it before?  This can happen for loopback, retrans,
     * or local packets.
     */
    if (!nf_nat_initialized(ct, maniptype)) {
        unsigned int ret;

        ret = do_chain(ops, skb, state, ct);
        if (ret != NF_ACCEPT)
            return ret;

        if (nf_nat_initialized(ct, HOOK2MANIP(ops->hooknum)))
            break;

        ret = nf_nat_alloc_null_binding(ct, ops->hooknum);
        if (ret != NF_ACCEPT)
            return ret;
    } else {
        pr_debug("Already setup manip %s for ct %p\n",
             maniptype == NF_NAT_MANIP_SRC ? "SRC" : "DST",
             ct);
        if (nf_nat_oif_changed(ops->hooknum, ctinfo, nat,
                       state->out))
            goto oif_changed;
    }
    break;

default:
    /* ESTABLISHED */
    NF_CT_ASSERT(ctinfo == IP_CT_ESTABLISHED ||
             ctinfo == IP_CT_ESTABLISHED_REPLY);
    if (nf_nat_oif_changed(ops->hooknum, ctinfo, nat, state->out))
        goto oif_changed;
}
```

这时`nf_nat_initialized`会返回`0`, 因而跳过`do_chain`的调用，也就不再会执行`nat:PREROUTING`所设置的链和规则，放行通过进入到路由决策阶段。但由于数据包的源`IP`是本机地址，默认情况下Linux路由实现不允许非`loopback`设备之外的设备所进入的数据包源地址为本机地址, 因而该数据包会被丢弃。

但内核提供了参数`accept_local`可以允许这种包通过:

```
accept_local - BOOLEAN
    Accept packets with local source addresses. In combination
    with suitable routing, this can be used to direct packets
    between two local interfaces over the wire and have them
    accepted properly.

    rp_filter must be set to a non-zero value in order for
    accept_local to have an effect.


rp_filter - INTEGER
    0 - No source validation.
    1 - Strict mode as defined in RFC3704 Strict Reverse Path
        Each incoming packet is tested against the FIB and if the interface
        is not the best reverse path the packet check will fail.
        By default failed packets are discarded.
    2 - Loose mode as defined in RFC3704 Loose Reverse Path
        Each incoming packet's source address is also tested against the FIB
        and if the source address is not reachable via any interface
        the packet check will fail.

    Current recommended practice in RFC3704 is to enable strict mode
    to prevent IP spoofing from DDos attacks. If using asymmetric routing
    or other complicated routing, then loose mode is recommended.

    The max value from conf/{all,interface}/rp_filter is used
    when doing source validation on the {interface}.

    Default value is 0. Note that some distributions enable it
    in startup scripts.
```

修改相应参数放行数据包:

```
sysctl -w net.ipv4.conf.eth1.rp_filter=1
sysctl -w net.ipv4.conf.eth1.accept_local=1
```

再次从`worker`节点访问`VIP`, 同时开启`tcpdump`抓包分析:

```
[root@node1 ~]# curl http://10.240.0.201
curl: (7) Failed connect to 10.240.0.201:80; No route to host
```

可以看到返回路由错误，而从抓包结果看，我们放行数据包后，根据目的地址，数据包会再被发送出去，从而形成环路。直到`IP`包的`ttl`减为`0`返回了路由错误。

```
[root@node1 ~]# tcpdump -ieth1 tcp port 80 -nn -e -v
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
02:57:18.801428 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 64, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0x173c (incorrect -> 0xc00b), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.801803 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 64, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.814275 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 63, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:18.814579 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 63, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0

...

02:57:19.054672 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 2, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.054982 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 2, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.057681 08:00:27:3a:25:df > 08:00:27:48:90:6c, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 1, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
02:57:19.057978 08:00:27:48:90:6c > 08:00:27:3a:25:df, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 1, id 48372, offset 0, flags [DF], proto TCP (6), length 60)
    10.240.0.101.39700 > 10.240.0.201.80: Flags [S], cksum 0xc00b (correct), seq 542364503, win 29200, options [mss 1460,sackOK,TS val 2387875 ecr 0,nop,wscale 7], length 0
```

本文只是简单实验可行性。如果用于生产环境，需要额外的方案考虑，比如:

- `LVS`本身可以配合`keepalived`使用主备模式保证`Director`的HA
- 使用`OSPF`的`ECMP`来配置多主的`Director`集群(可以参考之前的文章[<<基于Cumulus VX实验ECMP+OSPF负载均衡>>](http://just4coding.com/2020/05/05/emcp-ospf-cumulus/))
- 省略`LVS`的`Director`层，直接使用`OSPF`的`ECMP`将流量分发到`worker`节点的`VIP`

### 原文

1. http://www.4k8k.xyz/article/ywq935/84952724
2. http://just4coding.com/2021/11/14/external-ip/