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



### helm and Kustomize 

overlay  dir 可以叠加，等价于 helm values.yaml

base dir 等价于 helm template dir

helm支持go template 内容更丰富但是耗内存（每次渲染要加载template所有变量和占位符），效率不高。

kustomize 使用overlay engine 内存消耗少，执行效率高，和kubectl 命令集成了，但是没有helm支持功能丰富

Kustomize 并非设计为 `templating engine` ，它被设计为一种纯粹的配置管理声明性方法

同一个应用组件在不同的k8s集群部署时，可以使用如下参数，挺方便的：

- `namePrefix` 为所有资源和引用的名称添加前缀
- `nameSuffix` 为所有资源和引用的名称添加后缀



The best way to describe the differences is to refer to them as different types of deployment engines. One is a ***Templating Engine*** and one is an ***Overlay Engine***.

So what are these? Well when you use a templating engine you create a boilerplate example of your file. From there you abstract away contents with known filters and within these abstractions you provide references to variables. These variables are normally abstracted to another file where you insert information specific to your environment Then, on runtime, when you execute the templating engine, the templates are loaded into memory and all of the variables are exchanged with their placeholders.

> 使用template Engine每次部署应用都需要加载所有的variables和placeholders到内存中，比较耗资源，但是它支持的功能丰富。

This is different from an overlay engine in a few nuanced ways. Normally about how information gets into configuration examples. Noticed how I used the word *examples* there instead of *templates*. This was intentional as Kustomize doesn't use templates. Instead, you create a *Kustomization.yml* file. This file then points to two different things. Your *Base* and your *Overlays*. At runtime your Base is loaded into memory and if any Overlays exist that match they are merged over top of your Base configuration.

> kustomize的overlay engine 只会将base dir加载到内存中 overlay dir的内容只patch，性能和效率更高。

The latter method(kustomize) allows you to scale your configurations to large numbers of variants more easily. Imagine maintaining 10,000 different sets of variables files for 10,000 different configurations. Now imagine maintaining a hierarchy of modular and small configurations that can be inherited in any combination or permutation? It would greatly reduce redundancy and greatly improve manageability.

Another nuance to make note of is ownership of the projects. Helm is operated by a third party. Kustomize is developed directly by the Kubernetes team. In fact, Kustomize functionality is directly supported in Kubectl. You can build and perform a Kustomize project like so: `kubectl apply -k DIR`. However, the version of kustomize embedded in the kubectl binary is out of date and missing some new features.

There are a few other improvements in Kustomize too that are somewhat more minor but still worth mentioning. It can reference bases from the internet or other non-standard paths. It supports generators to build configuration files for you automatically based on files and string literals. It supports robust & granular JSON patching. It supports injecting metadata across configuration files.