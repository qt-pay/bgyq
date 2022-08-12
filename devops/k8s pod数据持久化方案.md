## k8s pod数据持久化方案

### 挂载流程

挂载流程需要 kube-controller-manager、 kubelet 以及 cri 共同完成，一个完整的流程大致可分为以下 两个步骤：

- 1）远程存储挂载至宿主机
  - 1）Attach
    - 由 controller 执行，将远程块存储作为一个远程磁盘挂载到宿主机
  - 2）Mount
    - 由 kubelet 执行，将远程磁盘格式化并挂载到 Volume 对应的宿主机目录
    - 一般是在 /var/lib/kubelet 目录下
- 2）将宿主机目录挂载至 Pod 中
  - 由 CRI 执行，将宿主机上的目录挂载到 Pod 中

所以，挂载失败要查看kubelet和CRI组件

#### 准备volume

根据 volume 类型的不同，准备工作也不完全一样，比如 NFS 本身就是文件系统就可以省略 Attach，而 hostpath 本身就是挂载到宿主机目录，甚至连 Mount 这一步都不需要，完全不需要准备工作，CRI 直接挂载即可。



该步骤分为两个阶段：

- 阶段一：Attach，挂载磁盘
- 阶段二：Mount，格式化磁盘并 mount 至宿主机目录

##### Attach to host

Attach 阶段由 AttachDetachController + external-attacher 完成。

一般缩写为 ADController，是 VolumeController 的一部分，所以也是 kube-controller-manager 的一部分。

AttachDetachController 不断地检查每一个 Pod 对应的 PV 和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作，为虚拟机挂载远程磁盘。Kubernetes 提供的可用参数是 nodeName，即宿主机的名字。

对于 Google Cloud 的 Persistent Disk（GCE 提供的远程磁盘服务），那么该阶段就是调用 Goolge Cloud 的 API，将它所提供的 Persistent Disk 挂载到 Pod 所在的宿主机上。

对于Cinder，则现在Openstack集群创建volume，然后挂载到node上

##### Mount to host

Mount 阶段由 VolumeManagerReconciler 完成。

由于需要在具体的 node 上执行，因此这是 kubelet 组件的一部分。

kubelet 通过 VolumeManager 启动 reconcile loop，当观察到有新的使用 PersistentVolumeSource 为 CSI 的 PV 的 Pod 调度到本节点上，于是调用 reconcile 函数进行 Attach/Detach/Mount/Unmount 相关逻辑处理。

将磁盘设备格式化并挂载到 Volume 宿主机目录。Kubernetes 提供的可用参数是 dir，即 Volume 的宿主机目录

对于块存储相当于执行了下面命令

```bash
# 通过 lsblk 命令获取磁盘设备 ID
$ lsblk
# 格式化成 ext4 格式
$ mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/<磁盘设备ID>
# 挂载到挂载点
$ mkdir -p /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>
$ mount /dev/<磁盘设备ID> /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>
```

#### CSI将volume挂载到Pod

调用 CRI 接口创建 Pod，将 volume 通过 Mounts 参数传递过去，最终由 CRI 将 volume 对应的宿主机目录挂载到 Pod 中。

类似docker的如下操作
```bash
$ docker run -v /home:/test ...
```

底层实现原理

1）创建容器进程并开启 Mount namespace

```bash
int pid = clone(main_function, stack_size, CLONE_NEWNS | SIGCHLD, NULL); 
```

2）将宿主机目录挂载到容器进程的目录中来

```bash
mount("/home", "/test", "", MS_BIND, NULL)
```

此时虽然开启了 mount namespace，只代表主机和容器之间 mount 点隔离开了，容器仍然可以看到主机的文件系统目录。

通过 **bind mount** 将宿主机目录挂载到容器目录。

3）调用 `pivot_root` 或 `chroot`，改变容器进程的根目录。至此，容器再也看不到宿主机的文件系统目录了。

#### Cinder volume

https://stackoverflow.com/questions/40162641/kubernetes-cinder-volumes-do-not-mount-with-cloud-provider-openstack







### NFS/NAS

#### 不同deployment共享数据目录

场景：

应用模块A和应用模块B都需要同一个数据目录，A模块多副本写入，B模块多副本读

解决：
在NFS filesystem下创建一个sub dir，让后创建两个不同的PV但是都映射到同提个sub dir目录，然后通过pvc挂载给不同的deployment，实现不同组件共享同一个数据目录。

### 引用

1. https://www.lixueduan.com/post/kubernetes/09-volume/