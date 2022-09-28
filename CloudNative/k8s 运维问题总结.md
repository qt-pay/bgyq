## k8s 运维问题总结

### NodePort port被占用

现象：kube-proxy监听的nodeport有可能被主动请求（即本地listen port）占用导致监听失败

两个问题：

1. 那么为什么kube-proxy监听的端口是`30000-32767`呢？
2. 为什么本地监听端口会占用kube-proxy默认的nodeport范围呢？

答案：

默认local_port_range内核参数就是从32768开始的，所以才有了k8s nodeport的30000~32767范围。

The /proc/sys/net/ipv4/ip_local_port_range defines the local port range that is used by TCP and UDP traffic to choose the local port. You will see in the parameters of this file two numbers: The first number is the first local port allowed for TCP and UDP traffic on the server, the second is the last local port number**. For high-usage systems you may change its default parameters to 32768-61000 -first-last.**

到这里就清楚为什么本地监听会抢占k8s nodeport的范围了，肯定是系统的`net.ipv4.ip_local_port_range`被修改了。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/linux-local-port-range.png)

果然是这个问题，董哥可以哇

### k8s 安全

cncf的cks内容

https://segmentfault.com/a/1190000041175935

### 重启pod

三种方法：

* kubectl rollout restart deployment deployment_name: 要求k8s 1.15 版本以上，支持滚动重启部署。
* kubectl scale --replicas=0 deployment deployment_name and kubectl scale --replicas=N deployment deployment_name 会业务终端吧，瞬间数量调整到0
* 修改label selector，将pod label修改，使deployment controller重新创建新的pod，然后手动删除掉旧pod