## k8s pod数据持久化方案

### NFS/NAS

#### 不同deployment共享数据目录

场景：

应用模块A和应用模块B都需要同一个数据目录，A模块多副本写入，B模块多副本读

解决：
在NFS filesystem下创建一个sub dir，让后创建两个不同的PV但是都映射到同提个sub dir目录，然后通过pvc挂载给不同的deployment，实现不同组件共享同一个数据目录。