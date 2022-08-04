## k8s 应用记录client源IP

### http x-forward headers

clinet发送请求时，自带x-forward headers，后端解析数据获取源IP即可。

X-Forwarded-For 是一个 HTTP 扩展头部。

X-Forwarded-For 请求头格式非常简单，就这样：

```shell
X-Forwarded-For: client, proxy1, proxy2
```

可以看到，XFF 的内容由「英文逗号 + 空格」隔开的多个部分组成，最开始的是离服务端最远的设备 IP，然后是每一级代理设备的 IP。

如果一个 HTTP 请求到达服务器之前，经过了三个代理 Proxy1、Proxy2、Proxy3，IP 分别为 IP1、IP2、IP3，用户真实 IP 为 IP0，那么按照 XFF 标准，服务端最终会收到以下信息：

```shell
X-Forwarded-For: IP0, IP1, IP2
```

Server端只要获取到`X-Forwarded-For`对应的IP list，然后读取第一个元素，就可以拿到客户端真实IP了。

### NodePort

You can set the `spec.externalTrafficPolicy` field to control how traffic from external sources is routed. Valid values are `Cluster` and `Local`. Set the field to `Cluster` to route external traffic to all ready endpoints and `Local` to only route to ready node-local endpoints. If the traffic policy is `Local` and there are no node-local endpoints, the kube-proxy does not forward any traffic for the relevant Service.

API Server 要求只有使用 `LoadBalancer` 或者 `NodePort` 类型的 Service 才能够使用这种策略。这是因为 `Local` 策略只跟外部访问相关。

如果使用了 `Local` 策略，`kube-proxy` 只会代理到本地 `endpoint` 的流量，不会向其它节点转发。如果本地没有相应端点，发送到该节点的流量就会被丢弃，所以数据包中会保留正确的源 IP，可以放心的在数据包处理规则中使用。

如下，记录了四种访问情况：

* externalTrafficPolicy为Cluster，通过非pod所在node访问，Nginx记录的client IP为`10.249.76.0`

  通过`ip r`命令查看`10.249.76.0`是pod坐着node的网关信息，流量路径挺合理的。

  anti-pod-node-IP --> pod-in-node-calico-gw-IP --> svc --> pod

* 

```bash
$ tail nginx.log
## externalTrafficPolicy: Cluster
10.249.76.0 - - [02/Aug/2022:17:00:54 +0800] "POST /apache/tbsp/00000000000000/03010103A0051 HTTP/1.1" 200 8307 "http://172.168.165.159:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
10.249.76.0 - - [02/Aug/2022:17:00:54 +0800] "GET /module/basicPlatform/basicPlatform.js?bust=1659431122136 HTTP/1.1" 200 94839 "http://172.168.165.159:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
...
## externalTrafficPolicy: Cluster
172.168.165.151 - - [02/Aug/2022:17:02:52 +0800] "GET / HTTP/1.1" 200 1346 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
172.168.165.151 - - [02/Aug/2022:17:02:52 +0800] "GET /sysconfig.js HTTP/1.1" 200 57456 "http://172.168.165.151:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
172.168.165.151 - - [02/Aug/2022:17:02:52 +0800] "GET /static/byte/byteSize.js HTTP/1.1" 200 6992 "http://172.168.165.151:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
...
172.168.165.151 - - [02/Aug/2022:17:02:56 +0800] "GET /frame/img/login-bg.jpg HTTP/1.1" 200 191059 "http://172.168.165.151:31042/frame/frame.031edd6b.css" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
172.168.165.151 - - [02/Aug/2022:17:02:56 +0800] "GET /frame/img/login-headerJYYH.png HTTP/1.1" 200 22690 "http://172.168.165.151:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
172.168.197.239 - - [02/Aug/2022:17:05:45 +0800] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"

## externalTrafficPolicy: Local
172.168.197.135 - - [02/Aug/2022:17:15:28 +0800] "POST /apache/tbsp/logout HTTP/1.1" 200 67 "http://172.168.165.151:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"
172.168.197.135 - - [02/Aug/2022:17:15:28 +0800] "GET / HTTP/1.1" 304 0 "http://172.168.165.151:31042/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.162 Safari/537.36" "-"


$ ip r
default via 172.168.164.1 dev enp1s0 
...
10.249.76.0/24 via 172.168.165.159 dev tunl0 proto bird onlink 
10.252.150.0/24 via 172.168.165.210 dev tunl0 proto bird onlink 
10.255.123.0/24 via 172.168.165.150 dev tunl0 proto bird onlink 
10.255.219.0/24 via 172.168.165.151 dev tunl0 proto bird onlink 
169.254.0.0/16 dev enp1s0 scope link metric 1002 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
172.168.164.0/23 dev enp1s0 proto kernel scope link src 172.168.165.158 

```



### Ingress ：nginx

#### Nginx Ingress作为服务对外直接入口

##### 内网环境

这种很简单：

nginx-ingress pod需要使用`externalTrafficPolicy: Local` 或者 直接使用hostnetwork，这样才能避免从其他k8s node转发过来请求导致client source IP丢失。

##### 公网出口

公网访问 --> 公网IP映射Nginx IP --> nginx pod --> app pod

两个Nginx Ingress pod都是用hotsnetwork方式，在使用keepalive 搞个VIP，把这个VIP作为LoadBalancerIP手动加到nginx-ingress service中,再将一个公网映射到这个VIP上，实现公网可以直接访问Nginx Ingress代理的pod

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    ...
  managedFields:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  clusterIP: 10.96.122.221
  clusterIPs:
  - 10.96.122.221
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  loadBalancerIP: 10.0.1.190
  ports:
  - appProtocol: http
    name: http
    port: 8080
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 8443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  sessionAffinity: None
  type: ClusterIP
```



#### Nginx Ingress前面还有其他代理

```bash
kubectl -n kube-system edit cm nginx-configuration
data:
    compute-full-forwarded-for: "true"
    forwarded-for-header: "X-Forwarded-For"
    use-forwarded-headers: "true"
```

1. **use-forwarded-headers**

   查看**NGINX Ingress Controller的ConfigMaps配置文档**，可以找到以下配置项**use-forwarded-headers**如果为true，NGINX会将传入的 **X-Forwarded-\*** 头传递给**upstreams**。当NGINX位于另一个正在设置这些标头的 L7 proxy / load balancer 之后时，请使用此选项。
   如果为false，NGINX会忽略传入的 X-Forwarded-* 头，用它看到的请求信息填充它们。如果NGINX直接暴露在互联网上，或者它在基于 L3/packet-based load balancer 后面，并且不改变数据包中的源IP，请使用此选项。

   ps： NGINX Ingress Controller直接暴露互联网也就是Edge模式不能开启为true，否则会有**伪造ip**的安全问题。也就是k8s有公网ip，直接让客户端访问，本配置不要设为true！

2. **forwarded-for-header**

   设置标头字段以标识客户端的原始IP地址。 默认: X-Forwarded-For

   ps：如果 NGINX Ingress Controller 在CDN，WAF，LB等后面，设置从头的哪个字段获取IP，默认是X-Forwarded-For

   这个配置应该和use-forwarded-headers配合使用

3. **compute-full-forwarded-for**

   将远程地址附加到 X-Forwarded-For 标头，而不是替换它。 启用此选项后，upstreams应用程序将根据其自己的受信任代理列表提取客户端IP

#### 单纯Nginx配置

Nginx作反向代理时，获取client真实IP

```nginx
location /api/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://IP:Port;
}
```

x-real-ip获取到的是上一级代理的IP，而不是真正的客户端IP，需要proxy_add_x_forwarded_for进行合作判断。
x-forwarded-for 存储从客户端到各级代理再到服务端链路上经过的所有IP。

proxy_add_x_forwarded_for 的意思是向x-forwared-for上添加一次记录。

`X-Forwarded-For` 最后一节是 Nginx 追加上去的，但之前部分都来自于 Nginx 收到的请求头，这部分用户输入内容完全不可信。使用时需要格外小心，符合 IP 格式才能使用，不然容易引发 SQL 注入或 XSS 等安全漏洞。

安全策略：

1. 直接对外提供服务的 Web 应用，在进行与安全有关的操作时，只能通过 Remote Address 获取 IP，不能相信任何请求头；
2. 使用 Nginx 等 Web Server 进行反向代理的 Web 应用，在配置正确的前提下，要用 `X-Forwarded-For` 最后一节 或 `X-Real-IP` 来获取 IP（因为 Remote Address 得到的是 Nginx 所在服务器的内网 IP）；同时还应该禁止 Web 应用直接对外提供服务

如下，nginx配置是无法获取的client真实IP的

```bash
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $remote_addr;
```

请求到达 Nginx 之前的所有代理信息都被抹掉，无法为真正使用代理的用户提供更好的服务。

### Gateway： Kong

### Aliyun SLB

这个场景很简单了，将

### 引用

1. https://juejin.cn/post/6976058706756632606
2. https://imququ.com/post/x-forwarded-for-header-in-http.html
3. https://blog.fleeto.us/post/life-of-a-packet-in-k8s-3/
