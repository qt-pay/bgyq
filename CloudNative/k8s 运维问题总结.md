## k8s 运维问题总结

### 大佬博客

https://qingwave.github.io/

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

### `cfs_period_us`：k8s cpu limit node

有人说 `cgroupfs` driver is not affected of this particular problem. If we switch to use `cgroupfs` as driver everything works as expected.只有设置`cgroupdriver=systemd`才会出现，但是在我遇到的场景中，cgroup driver已经是`cgroupfs`了还是出现了这个错误。

现象：所有启动的pod都会提示这个错误

```bash
# pod error info
Error: failed to start container "sdn": Error response from daemon: oci runtime error: container_linux.go:235: starting container process caused "process_linux.go:327: setting cgroup config for procHooks process caused \"failed to write 100000 to cpu.cfs_period_us: write /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod033663cd-4848-11ed-8086-0cda411d9448/sdn/cpu.cfs_period_us: invalid argument\""

```

解决： reboot node 或者升级内核

分析：cfs的bug

CPU requests 对应于 Docker 容器的 HostConfig.CpuShares 属性。而 CPU limits 就不太明显了，它由两个属性控制：HostConfig.CpuPeriod 和 HostConfig.CpuQuota。Docker 容器中的这两个属性又会映射到进程的 cpu,couacct cgroup 的另外两个属性：`cpu.cfs_period_us` 和 `cpu.cfs_quota_us`。我们来看一下：

```bash
$ sudo cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod2f1b50b6-db13-11e8-b1e1-42010a800070/f0845c65c3073e0b7b0b95ce0c1eb27f69d12b1fe2382b50096c4b59e78cdf71/cpu.cfs_period_us
100000

$ sudo cat /sys/fs/cgroup/cpu,cpuacct/kubepods/burstable/pod2f1b50b6-db13-11e8-b1e1-42010a800070/f0845c65c3073e0b7b0b95ce0c1eb27f69d12b1fe2382b50096c4b59e78cdf71/cpu.cfs_quota_us
10000
```

如我所说，这些值与容器配置中指定的值相同。但是这两个属性的值是如何从我们在 Pod 中设置的 100m cpu limits 得出的呢，他们是如何实现该 limits 的呢？这是因为 cpu requests 和 cpu limits 是使用两个独立的控制系统来实现的。Requests 使用的是 cpu shares 系统，cpu shares 将每个 CPU 核心划分为 1024 个时间片，并保证每个进程将获得固定比例份额的时间片。如果总共有 1024 个时间片，并且两个进程中的每一个都将 cpu.shares 设置为 512，那么它们将分别获得大约一半的 CPU 可用时间。但 cpu shares 系统无法精确控制 CPU 使用率的上限，如果一个进程没有设置 shares，则另一个进程可用自由使用 CPU 资源。

大约在 2010 年左右，谷歌团队和其他一部分人注意到了这个问题。为了解决这个问题，后来在 linux 内核中增加了第二个功能更强大的控制系统：CPU 带宽控制组。带宽控制组定义了一个 周期，通常为 1/10 秒（即 100000 微秒）。还定义了一个 配额，表示允许进程在设置的周期长度内所能使用的 CPU 时间数，两个文件配合起来设置CPU的使用上限。两个文件的单位都是微秒（us），`cfs_period_us` 的取值范围为 1 毫秒（ms）到 1 秒（s），cfs_quota_us 的取值大于 1ms 即可，如果 cfs_quota_us 的值为 -1（默认值），表示不受 CPU 时间的限制。

下面是几个例子：

```bash
# 1.限制只能使用1个CPU（每250ms能使用250ms的CPU时间）
$ echo 250000 > cpu.cfs_quota_us /* quota = 250ms */
$ echo 250000 > cpu.cfs_period_us /* period = 250ms */

# 2.限制使用2个CPU（内核）（每500ms能使用1000ms的CPU时间，即使用两个内核）
$ echo 1000000 > cpu.cfs_quota_us /* quota = 1000ms */
$ echo 500000 > cpu.cfs_period_us /* period = 500ms */

# 3.限制使用1个CPU的20%（每50ms能使用10ms的CPU时间，即使用一个CPU核心的20%）
$ echo 10000 > cpu.cfs_quota_us /* quota = 10ms */
$ echo 50000 > cpu.cfs_period_us /* period = 50ms */
```

在本例中我们将 Pod 的 cpu limits 设置为 100m，这表示 100/1000 个 CPU 核心，即 100000 微秒的 CPU 时间周期中的 10000。所以该 limits 翻译到 cpu,cpuacct cgroup 中被设置为 cpu.cfs_period_us=100000 和 cpu.cfs_quota_us=10000。顺便说一下，其中的 cfs 代表 Completely Fair Scheduler（绝对公平调度），这是 Linux 系统中默认的 CPU 调度算法。还有一个实时调度算法，它也有自己相应的配额值。

现在让我们来总结一下：

- 在 Kubernetes 中设置的 cpu requests 最终会被 cgroup 设置为 cpu.shares 属性的值， cpu limits 会被带宽控制组设置为 cpu.cfs_period_us 和 cpu.cfs_quota_us 属性的值。与内存一样，cpu requests 主要用于在调度时通知调度器节点上至少需要多少个 cpu shares 才可以被调度。
- 与 内存 requests 不同，设置了 cpu requests 会在 cgroup 中设置一个属性，以确保内核会将该数量的 shares 分配给进程。
- cpu limits 与 内存 limits 也有所不同。如果容器进程使用的内存资源超过了内存使用限制，那么该进程将会成为 oom-killing 的候选者。但是容器进程基本上永远不能超过设置的 CPU 配额，所以容器永远不会因为尝试使用比分配的更多的 CPU 时间而被驱逐。系统会在调度程序中强制进行 CPU 资源限制，以确保进程不会超过这个限制。

如果你没有在容器中设置这些属性，或将他们设置为不准确的值，会发生什么呢？与内存一样，如果只设置了 limits 而没有设置 requests，Kubernetes 会将 CPU 的 requests 设置为 与 limits 的值一样。如果你对你的工作负载所需要的 CPU 时间了如指掌，那再好不过了。如果只设置了 CPU requests 却没有设置 CPU limits 会怎么样呢？这种情况下，Kubernetes 会确保该 Pod 被调度到合适的节点，并且该节点的内核会确保节点上的可用 cpu shares 大于 Pod 请求的 cpu shares，但是你的进程不会被阻止使用超过所请求的 CPU 数量。既不设置 requests 也不设置 limits 是最糟糕的情况：调度程序不知道容器需要什么，并且进程对 cpu shares 的使用是无限制的，这可能会对 node 产生一些负面影响。

#### kernel patch

Yeah reboot would for sure mitigate the problem. Until you hit it again after the reboot :)

https://github.com/xueweiz/scripts/blob/master/cos/fix-cfs/fix-cfs.py

https://github.com/kubernetes/kubernetes/issues/72878

### 引用

1. https://www.yisu.com/zixun/596259.html
2. https://qingwave.github.io/understanding-resource-limits-in-kubernetes/#cpu%E9%99%90%E5%88%B6