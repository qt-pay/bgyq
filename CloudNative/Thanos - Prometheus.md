### Thanos - Prometheus

他山之石：http://www.xuyasong.com/?p=1921

### 实际应用

首先，数据 = label + value，同一个/一组endpoint，在不同的Prometheus instance采集是数据是不一样的，因为Label有区别。当多个副本采集数据时，要对“相同”的endpoints数据进行去重。

这就是HA Prometheus cluster要做的事情。

目前大多数的 Prometheus 的集群方案是在存储、查询两个角度上保证数据的一致:

- 存储角度：如果使用 Remote Write 远程存储， A 和 B 后面可以都加一个 Adapter，Adapter 做选主逻辑，只有一份数据能推送到 TSDB，这样可以保证一个异常，另一个也能推送成功，数据不丢，同时远程存储只有一份，是共享数据。方案可以参考[这篇文章](https://blog.timescale.com/blog/Prometheus-ha-postgresql-8de68d19b6f5)
- 存储角度：仍然使用 Remote Write 远程存储，但是 A 和 B 分别写入 TSDB1 和 TSDB2 两个时序数据库，利用 Sync 的方式在 TSDB1 和 TSDB2 之间做数据同步，保证数据是全量的。
- 查询角度：上边的方案需要自己实现，有侵入性且有一定风险，因此大多数开源方案是在查询层面做文章，比如 Thanos 或者 Victoriametrics，仍然是两份数据，但是查询时做数据去重和 join。只是 Thanos是通过 Sidecar 把数据放在对象存储，Victoriametrics是把数据 Remote Write 到自己的 Server 实例，但查询层 Thanos Query 和 Victor的 Promxy的逻辑基本一致，都是为全局视图服务

#### Thanos 去重

query 组件启动时，默认会根据 query.replica-label 字段做重复数据的去重，你也可以在页面上勾选**deduplication** 来决定。query 的结果会根据你的 query.replica-label的 label 选择副本中的一个进行展示。可如果 0，1，2 三个副本都返回了数据，且值不同，query 会选择哪一个展示呢？

> `query.replica-label` 对应的就是 Prometheus的`external_labels`

Thanos 会基于打分机制，选择更为稳定的 replica 数据, 具体逻辑在：https://github.com/thanos-io/thanos/blob/55cb8ca38b3539381dc6a781e637df15c694e50a/pkg/query/iter.go

#### 最佳实践

N个机器实例，有M个Prometheus Instance实例分开采集，避免当个Prometheus 扛不住压力。

然后，在给每个Prometheus Instance 作一份数据冗余/高可用即再启动M个Prome_Instances。

实践，利有region作为key，A，B，C，D作为value，将N个机器分成4份采集，再通过replication key，0和1作为value没每个Prometheus实现HA和数据冗余。

汇总数据时，通过去重显示一份数据即可，Thanos配置根据`region` 和`replication`label进行去重。

Grafana 通过变量在一个大盘界面展示不同region数据即可，类似mihoy。

https://www.modb.pro/db/50590



### 部署Prometheus

Golang开发，下载二进制tgz，解压启动即可。

```bash
./prometheus \
        --config.file /root/test/prometheus-2.31.1.linux-amd64/prometheus.yml \
        # --storage.tsdb.path="data/"
        # Base path for metrics storage.
        --storage.tsdb.path /root/test/prometheus-2.31.1.linux-amd64/data \
        --web.console.templates=/root/test/prometheus-2.31.1.linux-amd64/consoles \
        --web.console.libraries=/root/test/prometheus-2.31.1.linux-amd64/console_libraries \
        --web.listen-address="0.0.0.0:31000"
== 使用下面的
/root/test/prometheus-2.31.1.linux-amd64/prometheus \
        --config.file /root/test/prometheus-2.31.1.linux-amd64/prometheus.yml \
        --storage.tsdb.path /root/test/prometheus-2.31.1.linux-amd64/data \
        --web.console.templates=/root/test/prometheus-2.31.1.linux-amd64/consoles \
        --web.console.libraries=/root/test/prometheus-2.31.1.linux-amd64/console_libraries \
        --storage.tsdb.max-block-duration=2h \
        --storage.tsdb.min-block-duration=2h \
        --storage.tsdb.retention.time=2h \
        --web.listen-address="0.0.0.0:31000"
```

end

##### prometheus.yaml external_labels

`external_labels`用来标识Prometheus instance，当这些Prome_instance采集相同的endpoint，数据会有冗余，然后通过`deduplication`，选择一份数据显示。

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  external_labels:
     region: 'A'
     replica: 0

....
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["0.0.0.0:31000"]

```

end

### 部署Minio

MinIO offers high-performance, S3 compatible object storage.
Native to Kubernetes, MinIO is the only object storage suite available on
every public cloud, every Kubernetes distribution, the private cloud and the
edge. MinIO is software-defined and is 100% open source under GNU AGPL v3.

使用Minio提供对象存储，部署Thanos的前置工作。

若不设置账号密码则 Access Key和Secret默认都是minioadmin 

若初始化账号密码则Access Key长度必须大于3，Secret长度大于8。

```bash
$ docker pull minio/minio
# MINIO_ACCESS_KEY and MINIO_SECRET_KEY
minio server - start object storage server
USAGE:
  minio server [FLAGS] DIR1 [DIR2..]
  minio server [FLAGS] DIR{1...64}
  minio server [FLAGS] DIR{1...64} DIR{65...128}

FLAGS:
  --address value              bind to a specific ADDRESS:PORT, ADDRESS can be an IP or hostname (default: ":9000")
  --listeners value            bind N number of listeners per ADDRESS:PORT (default: 1)
  --console-address value      bind to a specific ADDRESS:PORT for embedded Console UI, ADDRESS can be an IP or hostname
  --certs-dir value, -S value  path to certs directory (default: "/root/.minio/certs")
  --quiet                      disable startup information
  --anonymous                  hide sensitive information from logging
  --json                       output server logs and startup information in json format
  --help, -h                   show help


#  -p, --publish ip:[hostPort]:containerPort | [hostPort:]containerPort
$ docker run -p 31003:8999 --name minio  -v /usr/local/docker/minio/data:/data -v /usr/local/docker/minio/config:/root/.minio -d minio/minio server --console-address="0.0.0.0:8999" /data 

$ docker logs minio
API: http://172.17.0.2:9000  http://127.0.0.1:9000

Console: http://0.0.0.0:8999

Documentation: https://docs.min.io
WARNING: Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables

```

参考配置：https://cxyzjd.com/article/qq_41291945/108831013

其他应用/SDK使用Minio配置文件

```yaml
// bucket_config.yaml

type: s3

    config:

      bucket: bucket_name

      endpoint: IP:PORT

      access_key: XXX

      secret_key: XXX  #minio的密码8位以上

      insecure: true

```

end

### 部署Thanos

thanos 是无侵入的，只是Prometheus的上层套件。

而在thanos中与对象存储直接交互的组件有：

- sidecar - 将prometheus采集的指标写入对象存储；
- receive - 将从prometheus上报的数据写入对象存储；
- store - query通过store在对象存储中查询指标数据；
- compact - 将对象存储里的指标数据压缩处理；
- rule - 将新生成的指标数据存储至对象存储。

#### Sidecar

 The sidecar implementation allows you to query the metric data stored in Prometheus.

如果想上传prometheus本地的metrics数据，prometheus必须添加两个参数

```bash
--storage.tsdb.min-block-duration=2h 
--storage.tsdb.max-block-duration=2h
$ ./prometheus-2.31.1.linux-amd64/prometheus --help|grep min-block-duration|wc -l
0
$ ./prometheus-2.31.1.linux-amd64/prometheus --version
prometheus, version 2.31.1 (branch: HEAD, revision: 411021ada9ab41095923b8d2df9365b632fd40c3)

```

保证每个block里有相同时间跨度的数据。除此之外还需要禁用prometheus的压缩功能(默认未开启)。

```bash
# thanos help sidecar
./thanos sidecar \
# Be sure that the sidecar can use this url!
--prometheus.url="http://Prometheus_IP:8090" \
#  Path to YAML file that contains object store configuration. 指定使用哪个厂商的对象存储
# Storage configuration for uploading data
--objstore.config-file=<YAML_Path> \
# TSDB data directory of Prometheus
--tsdb.path=<Dir_Path>

Thanos uses object storage as primary storage for metrics and metadata related to them

## 配置文件
  objectstorage.yaml: 
    type: s3
    config:
      bucket: BUCKET_NAME
      endpoint: IP:PORT
      access_key: XXX
      secret_key: XXX
      insecure: true
## 启动命令
$ cat start_thanos.sh
/root/test/thanos-0.23.1.linux-amd64/thanos sidecar \
        --prometheus.url="http://172.21.170.84:31000" \
        --objstore.config-file=/root/test/thanos-0.23.1.linux-amd64/objectstore.yaml \
        --tsdb.path=/root/test/prometheus-2.31.1.linux-amd64/data
## 报错了...
level=error ts=2021-11-24T08:59:50.093866791Z caller=main.go:157 err="found that TSDB Max time is 1d12h and Min time is 2h. Compaction needs to be disabled (storage.tsdb.min-block-duration = storage.tsdb.max-block-duration)\nmain.validatePrometheus\n\t/app/cmd/thanos/sidecar.go:354\nmain.runSidecar.func3\n\t/app/cmd/thanos/sidecar.go:151\ngithub.com/oklog/run.(*Group).Run.func1\n\t/go/pkg/mod/github.com/oklog/run@v1.1.0/group.go:38\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371\nvalidate Prometheus flags\nmain.runSidecar.func3\n\t/app/cmd/thanos/sidecar.go:152\ngithub.com/oklog/run.(*Group).Run.func1\n\t/go/pkg/mod/github.com/oklog/run@v1.1.0/group.go:38\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371\nsidecar command failed\nmain.main\n\t/app/cmd/thanos/main.go:157\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:225\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371"

## prometheus 需要修改
## https://thanos.io/tip/components/sidecar.md/ 解决
prometheus \
  --storage.tsdb.max-block-duration=2h \
  --storage.tsdb.min-block-duration=2h 
  
## 又报错
level=error ts=2021-11-24T09:32:56.473795403Z caller=main.go:157 err="no external labels configured on Prometheus server, uniquely identifying external labels must be configured; see https://thanos.io/tip/thanos/storage.md#external-labels for details.\nmain.runSidecar.func3\n\t/app/cmd/thanos/sidecar.go:202\ngithub.com/oklog/run.(*Group).Run.func1\n\t/go/pkg/mod/github.com/oklog/run@v1.1.0/group.go:38\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371\nsidecar command failed\nmain.main\n\t/app/cmd/thanos/main.go:157\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:225\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371

## 解决：
Prometheus的配置文件yaml必须指定external-labels字段配置。
```

end

#### Store API

默认http监听10902、grpc监听10901

The Sidecar component implements and exposes a gRPC *[Store API](https://github.com/thanos-io/thanos/blob/main/pkg/store/storepb/rpc.proto#L27)*. The sidecar implementation allows you to query the metric data stored in Prometheus.

Let’s extend the Sidecar in the previous section to connect to a Prometheus server, and expose the Store API.

```bash
## thanos help sidecar
## --http-address="0.0.0.0:10902" Listen host:port for HTTP endpoints.
## --grpc-address="0.0.0.0:10901"
## Listen ip:port address for gRPC endpoints (StoreAPI). Make sure this address is routable from other components.
thanos sidecar \
    --tsdb.path                 /var/prometheus \
    --objstore.config-file      bucket_config.yaml \       # Bucket config file to send data to
    --prometheus.url            http://localhost:9090 \    # Location of the Prometheus HTTP server
    --http-address              0.0.0.0:10902 \            # HTTP endpoint for collecting metrics on the Sidecar
    --grpc-address              0.0.0.0:10901              # GRPC endpoint for StoreAPI
```

end

#### Query/Querier

简而言之，sidecar暴露了StoreAPI，Query从多个StoreAPI中收集数据，查询并返回结果。Query是完全无状态的，可以水平扩展。

The Query component is stateless and horizontally scalable and can be deployed with any number of replicas. Once connected to the Sidecars, it automatically detects which Prometheus servers need to be contacted for a given PromQL query.

Thanos Querier also implements Prometheus’s official HTTP API and can thus be used with external tools such as Grafana. It also serves a derivative of Prometheus’s UI for ad-hoc querying and stores status.

Below, we will set up a Thanos Querier to connect to our Sidecars, and expose its HTTP UI.

```bash
# --store=<store> ...        Addresses of statically configured store API servers (repeatable). The scheme may be prefixed with 'dns+' or 'dnssrv+' to detect store API servers through respective DNS lookups.

thanos query \
    --http-address 0.0.0.0:19192 \   # HTTP Endpoint for Thanos Querier UI
    --grpc-address=0.0.0.0:1001 \    # 默认的10901端口被thanos sidecar 占用了
    --store        1.2.3.4:10902 \   # Static gRPC Store API Address for the query node to query
    --store        1.2.3.5:19090 \   # Also repeatable
    --store        dnssrv+_grpc._tcp.thanos-store.monitoring.svc  # Supports DNS A & SRV records
    /root/test/thanos-0.23.1.linux-amd64/thanos query  --http-address=0.0.0.0:31001  --store=0.0.0.0:10901

```

Go to the configured HTTP address that should now show a UI similar to that of Prometheus. If the cluster formed correctly you can now query across all Prometheus instances within the cluster. You can also check the Stores page to check up on your stores.

##### 去重参数：`replica-label`

使用Query从Sidecar和Store进行数据的聚合、去重。如从`10和11`两个sidecar进行数据的检索。

去重使用`--query.replica-label`（这label就是Prometheus Instance的`external_labels`）选项指定标识副本的标签。

通过这个label过滤数据--

```bash
$ cat /usr/lib/systemd/system/thanos-query.service 
[Unit]
Description=thanos-query service
After=network.target

[Service]
User=productprd
Restart=on-failure
WorkingDirectory=/opt/thanos/
ExecStart=/opt/thanos/thanos query \
          --query.max-concurrent=100 \
          --query.replica-label "region" \
          --query.replica-label "replica" \
          --store=10.0.0.10:10901 \
          --store=10.0.0.11:10901
ExecReload=/bin/kill -s SIGHUP $MAINPID

[Install]
WantedBy=multi-user.target
```

end

#### 另一个监控： Pixie

https://github.com/pixie-io/pixie

#### 其他HA

https://github.com/tkestack/kvass

