## podman and skopeo

Kubernetes can work with any container that meets the [Open Container Initiative](https://opencontainers.org/) (OCI) image specification, which Podman's containers do.



### k8s OCI: image and runtime

**Open Container Initiative**

* Container runtime & Image specification
* Runtime specs define input to create a container
* Multiple platform supported(Linux, Windows, Solars & VM)
* runc is default implementation of OCI Runtime Specs

**It contains two specifications**: 
\- runtime-spec: The runtime specification 
\- image-spec: The image specification

OCI, 也就是前文提到的”开放容器标准”其实就是一坨文档, 其中主要规定了两点:

1. 容器镜像要长啥样, 即 [ImageSpec](https://github.com/opencontainers/image-spec). 里面的大致规定就是你这个东西需要是一个压缩了的文件夹, 文件夹里以 xxx 结构放 xxx 文件;
2. 容器要需要能接收哪些指令, 这些指令的行为是什么, 即 [RuntimeSpec](https://github.com/opencontainers/runtime-spec). 这里面的大致内容就是”容器”要能够执行 “create”, “start”, “stop”, “delete” 这些命令, 并且行为要规范

####  bundle 

OCI 镜像可以通过工具转换成 bundle，然后 OCI 容器引擎能够识别这个 bundle 来运行容器。

当我们拿到一个镜像之后，如果用它来启动一个容器呢？这里就涉及到了 OCI 规范中的另一个规范即运行时规范 [runtime-spec](https://github.com/opencontainers/runtime-spec) 。容器运行时通过一个叫 [OCI runtime filesytem bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md) 的标准格式将 OCI 镜像通过工具转换为 bundle ，然后 OCI 容器引擎能够识别这个 bundle 来运行容器。

> filesystem bundle 是个目录，用于给 runtime 提供启动容器必备的配置文件和文件系统。标准的容器 bundle 包含以下内容：
>
> - config.json: 该文件包含了容器运行的配置信息，该文件必须存在 bundle 的根目录，且名字必须为 config.json
> - 容器的根目录，可以由 config.json 中的 root.path 指定



### k8s CRI

CRI(container-runtime-interface)是kubernetes的kubelet sig-node 小组提出来的。目的是减少对不同container runtime的适配支持工作（eg：docker，rkt，kata等），于是kubelet sig-node规范了container runtime接口，让其他容器运行时开发者适配kubelet.

Container Runtime Interface(a.ka. CRI) is a standard way to integrate Container Runtime with Kubernetes. It is new plugin interface for container runtimes. It is a plugin interface which enables kubelet to use a wide variety of container runtimes, without the need to recompile.Prior to the existence of CRI, container runtimes (e.g., docker, rkt ) were integrated with kubelet through implementing an internal, high-level interface in kubelet. Formerly known as OCID, CRI-O is strictly focused on OCI-compliant runtimes and container images. 

```bash
KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock

## kubelet
--container-runtime string
The container runtime to use. Possible values: 'docker', 'remote', 'rkt(deprecated)'. (default "docker")
--container-runtime-endpoint string
[Experimental] The endpoint of remote runtime service. Currently unix socket endpoint is supported on Linux, while npipe and tcp endpoints are supported on windows. Examples:'unix:///var/run/dockershim.sock', 'npipe:////./pipe/dockershim' (default "unix:///var/run/dockershim.sock")

```

end



#### docker-shim：弃用

https://www.modb.pro/db/446613

弃用 dockershim 的话，这意味着原本使用 docker 作为容器引擎的用户需要为此计划实施迁移到 containerd 或者其他支持 CRI 的容器运行时，这会是一个不小的时间和人力成本。

The `dockershim` component of Kubernetes allows to use Docker as a Kubernetes's container runtime. Kubernetes' built-in `dockershim` component was removed in release v1.24.

docker-shim作用：

* Embedded into kubelet
* Dockershim talks to docker， which manage pods
* Default CRI implementation & enjoy majority in current kubernets deployment

可以看下kubernetes中kubelet包下的dockershim源代码commit history就可以知道什么时间kubelet集成了dockershim

dockershim: *https://github.com/kubernetes/kubernetes/tree/v1.22.0/pkg/kubelet/dockershim*

the Kubernetes project created an adapter component, `dockershim`. The dockershim adapter allows the kubelet to interact with Docker as if Docker were a CRI compatible runtime.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/kubelet-docker-shim.png)

#### containerd

containerd是从docker守护进程中独立出来的容器运行时，最终也要通过runC运行容器。

在CRI标准被提出后，为了兼容CRI，减少调用开销，containerd开发了一个守护进程，叫CRI-containerd。原先调用链kubelet -> dockershim -> dockerd -> containerd 被简化成为 kubelet -> CRI-containerd -> containerd。后来，containerd干脆将CRI-containerd以CRI插件形式内建在项目中，直接通过方法调用，进一步将调用链简化为 kubelet -> containerd。

对应的是如下版本变迁：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/cri-containerd-1.png)

In containerd 1.1, the cri-containerd daemon is now refactored to be a containerd CRI plugin. The CRI plugin is built into containerd 1.1, and enabled by default. 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/containerd.png)



#### cir-o

CRI标准被提出后，红帽按照CRI开发的一个轻量级容器运行时，是CRI标准的最小实现。此模式下kubelet直接调用cri-o，再由cri-o调用runC完成容器创建和管理，调用链比较简洁。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/k8s-cri-o.png)

CRI-O stands for "Container Runtime Interface - OpenShift". It is just an interface, that is, an API anyone can use from a program to communicate with a CRI-O compatible server. It is actually an extension of the "CRI" interface defined by Kubernetes. IIRC, Podman implements CRI-O, while Docker implements "CRI".

Kubernetes uses cri-o as runtime for containers, while developers use Podman or Docker to build container images, run containers, pull/push images, etc...

Kubernetes' support of the Docker runtime is expected to end with the removal of the dockershim from Kubernetes in release v1.24. As a result other runtimes such as cri-o or containerd will be used with Kubernetes to replace Docker, and the labs are reflecting that by introducing cri-o as runtime.



#### high-level runtimes

dockershim, containerd 和cri-o都是遵循CRI的容器运行时，它们都是**高层级运行时（High-level Runtime）**。

#### low-level runtimes

直接基于namespaces 和 cgroup 层面的操作

 https://www.ianlewis.org/en/container-runtimes-part-2-anatomy-low-level-contai 

```bash
## image spec
$ CID=$(docker create busybox)
$ ROOTFS=$(mktemp -d)
$ docker export $CID | tar -xf - -C $ROOTFS

## cgroup
$ UUID=$(uuidgen)
$ cgcreate -g cpu,memory:$UUID
$ cgset -r memory.limit_in_bytes=100000000 $UUID
$ cgset -r cpu.shares=512 $UUID
$ cgset -r cpu.cfs_period_us=1000000 $UUID
$ cgset -r cpu.cfs_quota_us=2000000 $UUID

# Next we can execute a command in the container. This will execute the command within the cgroup we created, unshare the specified namespaces, set the hostname, and chroot to our filesystem
$ cgexec -g cpu,memory:$UUID \
>     unshare -uinpUrf --mount-proc \
>     sh -c "/bin/hostname $UUID && chroot $ROOTFS /bin/sh"
/ # echo "Hello from in a container"
Hello from in a container
/ # exit

```

end

### podman

Podman 原来是 CRI-O 项目的一部分，后来被分离成一个单独的项目叫 libpod。Podman 的使用体验和 Docker 类似，不同的是 Podman 没有 daemon。

Podman 比较简单粗暴，它不使用 Daemon，而是直接通过 OCI runtime（默认也是 runc）来启动容器，所以容器的进程是 podman 的子进程。这比较像 Linux 的 fork/exec 模型，而 Docker 采用的是 C/S（客户端/服务器）模型。


Container Runtime  is a low-level runtime？

Podman is an open-source, Linux-native tool designed to develop, manage, and run containers and pods under the Open Container Initiative (OCI) standards.

Podman shares many of the same underlying components with other container engines, including CRI-O, providing a proving ground for new and different container runtimes, and other experiments in underlying technology like CRIU, Udica, Kata Containers, gVisor, etc.

**Podman is that it is daemon-less**. 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/podman-arch.png)

#### podman vs docker

If you are wondering how Podman is different from docker, the following table helps you with some key differences.

| Podman                                                       | Docker                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Podman is Daemonless                                         | Docker has a daemon (containerd ). The docker CLI interacts with the daemon to manage containers. |
| Podman interacts with the Linux kernel directly through runc | Docker daemon owns all the child processes of running containers |
| Podman can deploy pods with multiple containers. The same pod manifest can be used in Kubernetes. Also, you can deploy a Kubernetes pod manifest as a Podman pod. | There is no concept of a pod in Docker                       |
| Can run rootless containers without any additional configurations. You can run containers with root or non-privileged users. | Docker rootless mode requires additional configurations.     |

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/docker-podman-differences.png.webp)

#### podman and k8s

Kubernetes can work with any container that meets the [Open Container Initiative](https://opencontainers.org/) (OCI) image specification, which Podman's containers do.

The podman generate kube command allows you to export your existing containers into Kubernetes Pod YAML. This YAML can then be imported into OpenShift or a Kubernetes cluster. The podman play kube does the opposite, it allows you to take a Kubernetes YAML and run it in Podman.

podman可以将容器运行命令转换成k8s yaml或者使用k8s yaml运行一个容器，比较方便验证

```bash
## 创建容器
$ podman run -dt --pod new:webserver -p 8080:80 nginx
## 生产k8s可以部署的yaml
$ podman generate kube webserver
$ podman generate kube webserver >> webserver.yaml

## 根据k8s yaml创建一个pod
$ podman play kube webserver.yaml

## kubelet指定下就好了
## kubelet
$ kubelet --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock'

--container-runtime string
The container runtime to use. Possible values: 'docker', 'remote', 'rkt(deprecated)'. (default "docker")
--container-runtime-endpoint string
[Experimental] The endpoint of remote runtime service. Currently unix socket endpoint is supported on Linux, while npipe and tcp endpoints are supported on windows. Examples:'unix:///var/run/dockershim.sock', 'npipe:////./pipe/dockershim' (default "unix:///var/run/dockershim.sock")

```

##### 关系梳理

k8s 和 podman是两个东西，k8s 通过cri调用containerd或者cir-o来创建pod。

podman只是k8s node上管理pod和container的工具，podman要取代的是docker。

Docker和Podman的存在其实对k8s应该是无感的，k8s要做的只是通过cri接口下发pod。docker和pod只是node上的容器管理工具。

#### podman的好处

很重要的一点，命令和docker相近，比`crictl`、`runc`和`ctr`这些好用多了。

性能和安全性比docker好，而且支持container 和pod yaml的转换。

### `--cgroup-driver`

Driver that the kubelet uses to manipulate cgroups on the host.  Possible values: 'cgroupfs', 'systemd' (default "cgroupfs") 

When systemd is chosen as the init system for a Linux distribution, the init process generates and consumes a root control group (`cgroup`) and acts as a cgroup manager. Systemd has a tight integration with cgroups and will allocate cgroups per process. It’s possible to configure your container runtime and the kubelet to use `cgroupfs`. Using `cgroupfs` alongside systemd means that there will then be two different cgroup managers.

> systemd是系统自带的cgroup管理器, 系统初始化就存在的, 和cgroups联系紧密,为每一个进程分配cgroups,  用它管理就行了. 如果设置成cgroupfs就存在2个cgroup控制管理器, 实验证明在资源有压力的情况下,会存在不稳定的情况.

Control groups are used to constrain resources that are allocated to processes. A single cgroup manager will simplify the view of what resources are being allocated and will by default have a more consistent view of the available and in-use resources. When we have two managers we end up with two views of those resources. We have seen cases in the field where nodes that are configured to use `cgroupfs` for the kubelet and Docker, and `systemd` for the rest of the processes running on the node becomes unstable under resource pressure.

Changing the settings such that your container runtime and kubelet use `systemd` as the cgroup driver stabilized the system. 



### docker迁移到containerd

ebay 从 docker 切换到 containerd

https://colstuwjx.github.io/2021/08/%E5%8E%9F%E5%88%9B-containerd-%E8%BF%81%E7%A7%BB%E4%BA%8C%E4%B8%89%E4%BA%8B/

https://www.huaqiang.art/2021/01/04/containerd/

k8s 和 docker 两个开发阵营之间的 gap 还是很多的，这里列举几个比较典型的吧：

- cadvisor 在开发的时候压根就没考虑过调用 docker 接口之类的，直接读 cgroup 和 /proc，这个做法也一直延续至今；
- CRI 约定的只是 kubelet 管理容器相关需要的一些接口，至于 events、volume 这些都是没有包含在内的。其实个人认为至少应该把 image volume（即在 Dockerfile 里面通过 `VOLUME` 指定的匿名卷，它的定义会体现在 OCI 镜像里 ）和 container volume 这些给明确定义出来，而不是放任不管，然后在实现的时候一股脑塞到原生 grpc 接口的 extra info 里…
- 如果想用 crictl 或者类似的工具去调用 containerd 裸起一个容器，相信我，这真不是一件简单的事儿。其实也能理解，这并不是 k8s 社区关注的重点；
- containerd 原生暴露的 event 接口里能够监听到 sandbox 容器 start 的事件，然而在查询容器这层面的接口又无法查到 Sandbox 容器的信息，那么 Sandbox 到底是不是一个 container 呢？似乎和 dockershim 时代的 infra container 相比，Sandbox 这个概念在 containerd 具体实现时变得更加模糊了；
- containerd 自身是有区分 namespace 的，比如 docker 容器默认都在 `moby` 这个 namespace 下面，k8s 的则会是 `k8s.io`。此 namespace 非 k8s 的那个彼 namespace。这个 namespace 概念感觉有点类似于 jenkins 的 workspace 的意思，然而它是没有体现到 CRI 标准里的，这也就造成了我们实际在用 containerd 的时候，只有当找不到容器了，才会意识到有这个 namespace 的区分；
- 早期的云原生开发者多是站在 docker 用户的角度思考问题，这也是为什么 log-pilot 会直接选择把宿主机目录以只读形式挂载到它的容器里，这样来采集日志。然而，时代变了，今天回头再看 log-pilot 的这个做法，恐怕已经是有悖于云原生理念了，至少在安全性方面是很难达标的；
- docker 的 event 机制乍一看好像和 k8s 的 informer 机制差别不大，但是仔细一想其实出入挺大的：event 传递的是变化的增量信息，informer 除了传递这个变化以外，还鼓励和提倡遵循 “reconcile” 这套理念。



### buildah

使用 Buildah 能带来什么好处呢？我理解是我们可以通过更多的手段去构建镜像，不局限于 Dockerfile 中有限的关键字，我们有了更多的可能性，这就足够了。无论是 Buildah 原生命令还是通过 `buildah mount` 挂载到本地文件系统，都让我们可以更舒服的构建镜像，我们从维护一个 Dockerfile 转变为维护一个 Shell 脚本。

buildah 提供了 `buildah mount` 命令，可以将运行中容器挂载到 Host 的文件系统上，我们可以直接在 Host 上对容器内文件进行操作：

```bash
#!/usr/bin/env bash

set -o errexit

# Create a container
container=$(buildah from fedora:28)
mountpoint=$(buildah mount $container)

buildah config --label maintainer="yiran <zdyxry@gmail.com>" $container

curl -sSL http://ftpmirror.gnu.org/hello/hello-2.10.tar.gz \
     -o /tmp/hello-2.10.tar.gz
tar xvzf src/hello-2.10.tar.gz -C ${mountpoint}/opt

pushd ${mountpoint}/opt/hello-2.10
./configure
make
make install DESTDIR=${mountpoint}
popd

chroot $mountpoint bash -c "/usr/local/bin/hello -v"

buildah config --entrypoint "/usr/local/bin/hello" $container
buildah commit --format docker $container hello
buildah unmount $container
```

end

#### 应用

将敏感信息放到host上，buildah挂载敏感信息执行完后续操作后生成镜像，则镜像中不会包含敏感信息。

> eg, gitlab token、ssh-key

### skopeo：镜像搬运

Skopeo 的功能很简单，一句话描述就是：提供远程仓库的镜像管理能力。

`skopeo` is a command line utility that performs various operations on container images and image repositories.

`skopeo` does not require the user to be running as root to do most of its operations.

`skopeo` does not require a daemon to be running to perform its operations.

`skopeo` can work with [OCI images](https://github.com/opencontainers/image-spec) as well as the original Docker v2 images.

#### docker overlay and overlay2 FS

本质区别是镜像层之间共享数据的方法不同

overlay共享数据方式是通过硬连接
而overlay2是通过每层的 lower文件

镜像引用关系图

```
centos:7.5
   ↑
base-os (创建了一系列目录, 安装了部分命令工具)
   ↑
open-jdk (下载并部署 jdk 至 /opt/soft 下, 335MB)
   ↑
application (应用部署, 且执行了 chown user:user /opt)
```

引发镜像体积增大的原因:

*  在 open-jdk dockerfile 中, 将 jdk 文件拷贝至 `/opt/soft` 下, 该文件夹体积为335MB;
*  在 application dockerfile 中, 执行了一行 shell 命令 `chown -R /opt`

由于 overlayfs 的硬链接引用的原理, chown 只更改 metadata 数据, 而 metadata 是作用在 inode 节点上的, 一个硬链接文件的属性被修改, 同步的, 所有指向这个 inode 节点的文件的属性都会变化. 也就是说, overlayfs 通过硬链接将文件挂载到新的镜像层之后, 对里面已存在的文件做 chown 操作, 意味着底层(只读层)的文件属性也会被修改, docker 就会认为这个操作没有引发任何文件的变化, 所以就不会再拷贝那335MB 的文件. 而 overlayfs2 的每层都是独立的, 即使文件属性的变化, 也会导致整个文件被拷贝, 所以在 overlayfs2 下, 会产生335MB 的空间浪费。

> 硬连接文件有相同的 inode 及 data block

`chown` 是个很神奇的命令, 它只修改文件的 metadata 属性, 这完美的切中了 overlayfs 与 overlayfs2 最本质的区别.

`OverlayFS`的机制是只支持两层目录映射成一个目录。所以，当Image 有多层时，则需要借助`hard link` 方式。

While the `overlay` driver only works with a single lower OverlayFS layer and hence requires hard links for implementation of multi-layered images, the `overlay2` driver natively supports up to 128 lower OverlayFS layers

`overlay2` 本身就支持128个 lower OverlayFS layers,所以不需要`hard link`方式。



#### OCI image and Docker V2 images

OCI 的建立推动了容器技术的工业标准化，但是否此标准就是唯一呢？其实不然。在成立 OCI 并制定 `image-spec` 标准的时候 Docker 已经空前繁荣，并得到了广泛的应用。

由于标准只定义了最基本的内容，想要将 Docker 的实现全部按照标准进行改造的话，会对 Docker 造成破坏性变更，也不利于 Docker 功能的迭代。

所以，Docker 为了支持 OCI 标准的普及，已经推进了 registry 对 OCI 镜像的支持，现在也正在给 Docker 自身增加适配中，目标是让 Docker 支持两种镜像格式，分别是符合 Docker 标准的镜像和符合 OCI 标准的镜像。

每个 Docker 镜像都是由一系列的配置清单和相应的层进行组织的。每个层一般都是 tar 格式的归档，配置清单中描述了对应的层应该按何种顺序进行组织，以及镜像的一些元属性。比如镜像所支持的架构，例如 `amd64` 之类的，还有 ENV 等提前配置好的一些参数等。

> 这也是为什么docker hub显示镜像大小50M，docker pull下来后就变成200M了，应该是tar解压了。

在 Docker Image 中也包含着构建镜像时候所用的 Docker 版本 `docker_version` 以及构建镜像的历史记录 `history` 等信息。所以你在 `DockerHub` 或者其他的镜像仓库上可以看到构建镜像所用的 Docker 版本， 或者可通过 `docker history <IMAGE>` 的方式来查看构建历史。

OCI镜像详解

#### why skopeo？

 Skopeo 为了解决什么问题？

在 Quora 上搜到了这个问题，RedHat 容器运行时团队的 Leader Daniel Walsh 回答了这个问题，大概是以下几点原因：

- 最初向 Docker 提 PR 想增加 `docker inspect -remote` 功能，即不用拉取镜像就可以获取镜像信息，但是被拒绝了，官方建议自己实现该功能

  ```bash
  # 直接从 Docker 官方仓库中查看远程镜像的信息，默认显示所有 RepoTags，也可以增加 tag 参数来显示具体的某个 Tag 镜像的信息。
  $ skopeo inspect docker://docker.io/fedora       
  ```

- Skopeo 在希腊语中的意思是 **远程查看**，最初实现的功能就是远程查看镜像信息

- 后续扩展功能增加了镜像的拉取，推送，复制等功能

skopeo 这个镜像搬运工具简直是个神器，尤其是在 CI/CD 流水线中搬运两个镜像仓库里的镜像简直爽的不得了。

#### skopeo login

在使用 skopeo 前如果 src 或 dest 镜像是在 registry 中的，如果非 public 的镜像需要相应的 auth 认证，可以使用 docker login 或者 skopeo login 的方式登录到 registry，生成如下格式的 registry 登录配置文件。

```bash
$ jq "." ~/.docker/config.json
```

#### skopeo image format

* **containers-storage**: docker-reference An image located in a local containers/storage image store. Both the location and image store are specified in /etc/containers/storage.conf. (Backend for Podman, CRI-O, Buildah and friends)
* **dir:** path An existing local directory *path* storing the manifest, layer tarballs and signatures as individual files. This is a non-standardized format, primarily useful for debugging or noninvasive container inspection.
* **docker://**: docker-reference An image in a registry implementing the “Docker Registry HTTP API V2”. By default, uses the authorization state in either `$XDG_RUNTIME_DIR/containers/auth.json`, which is set using `(skopeo login)`. If the authorization state is not found there, `$HOME/.docker/config.json` is checked, which is set using `(docker login)`.
* **docker-archive:** path[:docker-reference] An image is stored in the `docker save` formatted file. *docker-reference* is only used when creating such a file, and it must not contain a digest.
* **docker-daemon:**docker-reference An image *docker-reference* stored in the docker daemon internal storage. *docker-reference* must contain either a tag or a digest. Alternatively, when reading images, the format can be docker-daemon:algo:digest (an image ID).
* **oci: **path:tag  An image *tag* in a directory compliant with “Open Container Image Layout Specification” at *path*.

这几种镜像的名字，对应着镜像存在的方式，不同存在的方式对镜像的 layer 处理的方式也不一样，比如 `docker://` 这种方式是存在 registry 上的，`docker-daemon:` 是存在本地 docker pull 下来的，再比如 `docker-archive` 是通过 docker save 出来的镜像



#### skopeo copy

使用 skopeo copy 两个 registry 中的镜像时，skopeo 请求两个 registry API 直接 copy `original blob` 到另一个 registry ，这样免去了像 docker pull –> docker tag –> docker push 那样 pull 镜像对镜像进行解压缩，push 镜像进行压缩。尤其是在搬运一些较大的镜像（几 GB 或者几十 GB 的镜像），使用 skopeo copy 的加速效果十分明显。

#### skopeo inspect 

用 skopeo inspect 命令可以很方方便地通过 registry 的 API 来查看镜像的 manifest 文件，以前需要用 curl 命令的，要 token 还要加一堆参数，所以比较麻烦，所以后来就用上了 skopeo inspect

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ori runtime filesystem bundle.webp)

#### skopeo sync

Skopeo sync 的功能基本上等同于阿里云的 [image-syncer](https://github.com/AliyunContainerService/image-syncer) 工具，不过个人觉着 skopeo 要比 image-syncer 更强大，灵活性更强一些。

### 引用

1. https://devopscube.com/podman-tutorial-beginners/
2. https://www.huaqiang.art/2021/01/04/containerd/
3. https://zdyxry.github.io/2019/10/19/Buildah-%E5%88%9D%E6%AC%A1%E4%BD%93%E9%AA%8C/
4. https://segmentfault.com/a/1190000040909285
5. https://zdyxry.github.io/2019/10/26/Skopeo-%E5%88%9D%E6%AC%A1%E4%BD%93%E9%AA%8C/
6. https://blog.k8s.li/Exploring-container-image.html