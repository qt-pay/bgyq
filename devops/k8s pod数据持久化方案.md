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

### emptydir

根据官方给出的最佳使用实践的建议，emptyDir可以在以下几种场景下使用：

- 临时空间，例如基于磁盘的合并排序
- 设置检查点以从崩溃事件中恢复未执行完毕的长计算
- 保存内容管理器容器从Web服务器容器提供数据时所获取的文件

默认情况下，emptyDir可以使用任何类型的由node节点提供的后端存储。如果你有特殊的场景，需要使用tmpfs作为emptyDir的可用存储资源也是可以的，只需要在创建emptyDir卷时增加一个emptyDir.medium字段的定义，并赋值为”Memory”即可。

HostPath 不能挂载tmpfs。

### HostPath

hostPath类型则是映射node文件系统中的文件或者目录到pod里。在使用hostPath类型的存储卷时，也可以设置type字段，支持的类型有文件、目录、File、Socket、CharDevice和BlockDevice。

使用场景：

- 当运行的容器需要访问Docker内部结构时，如使用hostPath映射/var/lib/docker到容器；
- 当在容器中运行cAdvisor时，可以使用hostPath映射/dev/cgroups到容器中；
- 允许 Pod 指定给定的 hostPath 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在。

**emptyDir和hostPath在功能上的异同分析**

- 二者都是node节点的本地存储卷方式；
- emptyDir可以选择把数据存到tmpfs类型的本地文件系统中去，hostPath并不支持这一点；
- hostPath除了支持挂载目录外，还支持File、Socket、CharDevice和BlockDevice，既支持把已有的文件和目录挂载到容器中，也提供了“如果文件或目录不存在，就创建一个”的功能；
- emptyDir是临时存储空间，完全不提供持久化支持；
- hostPath的卷数据是持久化在node节点的文件系统中的，即便pod已经被删除了，volume卷中的数据还会留存在node节点上；

### LocalVolume:bind_mount

The Local Persistent Volumes feature has been promoted to GA in Kubernetes 1.14. 

The biggest difference is that the Kubernetes scheduler understands which node a Local Persistent Volume belongs to. With HostPath volumes, a pod referencing a HostPath volume may be moved by the scheduler to a different node resulting in data loss. But with Local Persistent Volumes, the Kubernetes scheduler ensures that a pod using a Local Persistent Volume is always scheduled to the same node.

While HostPath volumes may be referenced via a Persistent Volume Claim (PVC) or directly inline in a pod definition, Local Persistent Volumes can only be referenced via a PVC. This provides additional security benefits since Persistent Volume objects are managed by the administrator, preventing Pods from being able to access any path on the host.

最重要的区别，就是Local PV和具体节点是有关联的，这意味着使用了Local PV的pod，重启多次都会被Kubernetes scheduler调度到同一节点，而如果用的是HostPath Volume，每次重启都可能被Kubernetes scheduler调度到新的节点，然后使用同样的本地路径、

HostPath Volume的时候，既可以在PVC声明，又可以直接写到Pod的配置中，但是Local PV只能在PVC声明，对于PV资源，通常都有专人管理，这样就避免了Pod开发者擅自使用本地磁盘带来的冲突和风险、

静态provisioner配置程序仅支持发现和管理挂载点（对于Filesystem模式存储卷）或符号链接（对于块设备模式存储卷）。 对于基于本地目录的存储卷，必须将它们通过bind-mounted的方式绑定到发现目录中。

使用 local 卷时，需要使用 PersistentVolume 对象的 nodeAffinity 字段。 它使 Kubernetes 调度器能够将使用 local 卷的 Pod 正确地调度到合适的节点。可以将 PersistentVolume 对象的 volumeMode 字段设置为 “Block”（而不是默认值 “Filesystem”），以将 local 卷作为原始块设备暴露出来。 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  # volumeMode field requires BlockVolume Alpha feature gate to be enabled.
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```



#### mode point??

如下

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/local-pv-spec.png)

下面的挂载显示就很离谱

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/local-pv-with-blockmount.png)

### NFS/NAS

#### 不同deployment共享数据目录



场景：

应用模块A和应用模块B都需要同一个数据目录，A模块多副本写入，B模块多副本读

**You cannot bind 2 pvc to the same pv**. **But** you can bind 2 pv on the same nfs path。

解决：
在NFS filesystem下创建一个sub dir，让后创建两个不同的PV但是都映射到同提个sub dir目录，然后通过pvc挂载给不同的deployment，实现不同组件共享同一个数据目录。

### 引用

1. https://www.lixueduan.com/post/kubernetes/09-volume/
1. https://blog.z0ukun.com/?p=2996

