## etcd运维总结

### V2/V3 API区别



### 添加新的节点/恢复一个节点

新加入node的`ETCD_INITIAL_CLUSTER_STATE`配置值不能为`new`，要设置为`existing`

一个节点宕机，比如磁盘故障，并不会影响整个集群的正常工作。此时可通过以下几步恢复集群。

1. 在正常节点上查看集群状态并摘除异常节点 `etcdctl endpoint status`

2. 摘除异常节点 `etcdctl member remove $ID`

3. 重新部署服务后，将节点重新加入集群

   - 由于ETCD集群证书依赖于服务器IP和主机名，为避免重新制作证书，需要保持节点IP和主机名不变 

   - 将节点重新加入集群`etcdctl member add $name --peer-urls=https://x.x.x.x:2380`此时查看集群状态，新加入的节点状态为unstarted

   - 删除新增成员的旧数据目录，更改相关配置,需将原etcd服务的旧数据目录删除，否则etcd会无法正常启动。新增节点是加入已有集群，所以需要修改配置`ETCD_INITIAL_CLUSTER_STATE="existing"`

     ```bash
     ## etcd.conf
     ETCD_DATA_DIR=""
     ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
     ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
     ETCD_NAME=""
     ETCD_INITIAL_ADVERTISE_PEER_URLS=""
     ETCD_ADVERTISE_CLIENT_URLS=""
     ETCD_INITIAL_CLUSTER=""
     ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
     ETCD_INITIAL_CLUSTER_STATE="existong"
     ```

   - 启动恢复节点etcd服务 检测集群是否正常`etcdctl endpoint status`

### I/O性能不足问题

* failed to send out heartbeat on time 

  通常情况下这个issue是disk运行过慢导致的，写磁盘过程可能要与其他应用竞争，或者因为磁盘是一个虚拟的或者是SATA类型的导致运行过慢，此时只有更好更快磁盘硬件才能解决问题。etcd暴露给Prometheus的metrics指标wal_fsync_duration_seconds就显示了wal日志的平均花费时间，通常这个指标应低于10ms。

* etcdserver：read-only range request "key：xxx" took too long to execute 

  常见原因多为磁盘性能不达标或者性能不稳定，IO抖动大，且集群任何一个节点的磁盘问题都可能导致该问题，日常可以通过iostat，iotop等工具，监测etcd大量出现上述报错时的IO情况，iowait是否明显增大；监控可以通过etcd两个metrics观测etcd_disk_wal_fsync_duration_seconds；



### 定时备份

```bash
export ETCDCTL_API=3;$ETCDCTL_PATH --endpoints="$ENDPOINTS" snapshot save ...
```



### 引用

1. http://book.wangzonghang.cn/etcd/etcd%E5%A4%87%E4%BB%BD%E4%B8%8E%E6%81%A2%E5%A4%8D.html