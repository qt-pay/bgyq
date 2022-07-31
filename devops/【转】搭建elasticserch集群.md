# 【转】搭建elasticserch集群

https://www.cnblogs.com/michael-xiang/p/13715692.html

首先引用 Elasticsearch （下文简称 ES）[官网](https://www.elastic.co/cn/elasticsearch/)的一段描述：

> Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

本文主要介绍 Elasticsearch 集群的搭建。通过在一台服务器上创建 3 个 ES 实例来创建一个建议的 ES 集群。

### Java使用es

Elasticsearch是基于Lucene开发的一个分布式全文检索框架，向Elasticsearch中存储和从Elasticsearch中查询，格式是json。

　　a）、索引index，相当于数据库中的database。

　　b）、类型type相当于数据库中的table。

　　c）、主键id相当于数据库中记录的主键，是唯一的。

　　d）、向Elasticsearch中存储数据，其实就是向es中的index下面的type中存储json类型的数据。

实际场景：
一个dcdp的业务系统其中一个exchange组件给es推数据，一个dls组件只要能从es 指定index中获取到数据就行了。这就是简单的es使用场景。

## Elasticsearch/ES[#](https://www.cnblogs.com/michael-xiang/p/13715692.html#1966269308)

官方的[Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/index.html) 提供了不同版本的文档连接，真是赞！

> 如果英文的不想看，还提供了中文版的 [Elasticsearch 2.x: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)，版本不是最新的，但是了解基本概念也是有帮助的。

Elasticsearch 7.x 包里自包含了 OpenJDK 的包。如果你想要使用你自己配置好的 Java 版本，需要设置 `JAVA_HOME` 环境变量 —— [参考](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html)

官方文档 `Set up Elasticsearch` 有各个 OS 的安装指导，页面 [Installing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/install-elasticsearch.html) 中提供了多种安装包对应的指导链接！

本文选择绿色安装包的的方式（`tar.gz`）安装。

> 由于实验机器有限，可以在同一台机器上模拟出 3 个节点，安装 ES 集群。

## 安装 ES

### 准备工作

{% note warning %}

不能使用 root 用户启动 es，否则会报错：

Caused by: java.lang.RuntimeException: can not run elasticsearch as root

{% endnote %}

如果需要新建用户的话可以运行 `sudo adduser es`，修改 es 用户的密码：`sudo passwd es`。

### 下载 ES 安装包

安装包下载地址：

- [官方-Past Releases](https://www.elastic.co/cn/downloads/past-releases#elasticsearch) 官网的下载速度龟速

其他镜像站tar包可能有问题，标准的elasticsearch应该包含

* 内置jdk：注意不需要额外下载jdk
* modules

下面的步骤参考 [Set up Elasticsearch » Installing Elasticsearch » Install Elasticsearch from archive on Linux or MacOS](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html)，选择的安装包是 elasticsearch-7.3.0 版本。

```shell
Copy# 下载安装包
wget https://mirrors.huaweicloud.com/elasticsearch/7.3.0/elasticsearch-7.3.0-linux-x86_64.tar.gz
wget https://mirrors.huaweicloud.com/elasticsearch/7.3.0/elasticsearch-7.3.0-linux-x86_64.tar.gz.sha512
# 验证安装包的完整性，如果没问题，会输出 OK
shasum -a 512 -c elasticsearch-7.3.0-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.3.0-linux-x86_64.tar.gz
# 将目录复制三份，作为三个节点，后面配置 ES 集群时，对应了三个 ES 实例
cp -R elasticsearch-7.3.0 es-7.3.0-node-1
cp -R elasticsearch-7.3.0 es-7.3.0-node-2
mv elasticsearch-7.3.0 es-7.3.0-node-3
# 因为以 root 用户启动不了 ES
chown -R es es-7.3.0*
```

解压后的[目录组成](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html#targz-layout)：

```shell
Copy.
├── bin # 二进制脚本存放目录，包括 elasticsearch 来指定运行一个 node，包括 elasticsearch-plugin 来安装 plugins
├── config # 包含了 elasticsearch.yml 配置文件
├── data # 节点上分配的每个 index/分片 的数据文件
|-- jdk # 内置jdk 
├── lib
├── LICENSE.txt
├── logs
├── modules
├── NOTICE.txt
├── plugins # 插键文件存放的位置
└── README.textile
```

### 运行 Elasticsearch

我们先运行一个节点，创建 ES 单机版实例：

```shell
./bin/elasticsearch
```

如果要将 ES 作为守护程序运行，请在命令行中指定 `-d`，指定 `-p` 参数，将进程 ID 记录到 `pid` 文件：

```shell
./bin/elasticsearch -d -p pid
```

日志在 `$ES_HOME/logs` 目录中。

如果要停止 ES，运行如下的命令：

```shell
kill -F pid
```

### 检查一下运行状态

```shell
curl -X GET "localhost:9200/?pretty"
```

或者在浏览器中访问 `localhost:9200` 都可以，会返回：

```shell
Copy{
    "name": "node-1",
    "cluster_name": "appsearch-7.3.2",
    "cluster_uuid": "GlzI_v__QJ2s9ewAgomOqg",
    "version": {
        "number": "7.3.0",
        "build_flavor": "default",
        "build_type": "tar",
        "build_hash": "de777fa",
        "build_date": "2019-07-24T18:30:11.767338Z",
        "build_snapshot": false,
        "lucene_version": "8.1.0",
        "minimum_wire_compatibility_version": "6.8.0",
        "minimum_index_compatibility_version": "6.0.0-beta1"
    },
    "tagline": "You Know, for Search"
}
```

如果你是在远端服务器上部署的 ES，那么，此时在你本地的工作机上还无法调通 `<IP>:9200`，需要对 ES 进行相关配置才能访问，下文会介绍。

## ES 配置相关

官网关于配置的内容主要有两处：

- [Configuraing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)
- [Important Elasticsearch configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html)

Elasticsearch 主要有三个配置文件：

- `elasticsearch.yml` ES 的配置，[more](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html)
- `jvm.options` ES JVM 配置，[more](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html#jvm-options)
- `log4j2.properties` ES 日志配置，[more](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html#logging)

> 配置文件主要位于 `$ES_HOME/config` 目录下，也可以通过 `ES_PATH_CONF` 环境变量来修改

YAML 的配置形式参考：

```yaml
path:
    data: /var/lib/elasticsearch
    logs: /var/log/elasticsearch
```

设置也可以按如下方式展平：

```yaml
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
```

### JVM 配置

[JVM 参数设置](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html#jvm-options)可以通过 `jvm.options` 文件（推荐方式）或者 `ES_JAVA_OPTS` 环境变量来修改。

`jvm.options` 位于

- `$ES_HOME/config/jvm.options` 当通过 `tar` or `zip` 包安装
- `/etc/elasticsearch/jvm.options` 当通过 Debian or RPM packages

官网也介绍了如何[设置堆大小](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html)。

默认情况，ES 告诉 JVM 使用一个最小和最大都为 1GB 的堆。但是到了生产环境，这个配置就比较重要了，确保 ES 有足够堆空间可用。

ES 使用 `Xms(minimum heap size)` 和 `Xmx(maxmimum heap size)` 设置堆大小。你应该将这两个值设为同样的大小。

**`Xms` 和 `Xmx` 不能大于你物理机内存的 50%。**

设置的示例：

```shell
-Xms2g 
-Xmx2g
```

### elasticsearch.yml 配置

ES 默认会加载位于 `$ES_HOME/config/elasticsearch.yml` 的配置文件。

备注：任何能够通过配置文件设置的内容，都可以通过命令行使用 `-E` 的语法进行指定，例如：

```shell
Copy./bin/elasticsearch -d -Ecluster.name=my_cluster -Enode.name=node_1
```

------

#### `cluster.name`

`cluster.name` 设置[集群名称](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.name.html)。一个节点只能加入一个集群中，默认的集群名称是 `elasticsearch`。

```shell
Copycluster.name: search-7.3.2
```

{% note info %}
确保节点的集群名称要设置正确，这样才能加入到同一个集群中。上面示例就自定义了集群名称为 appsearch-7.3.2。
{% endnote %}

------

#### `node.name`

`node.name`：可以配置每个[节点的名称](https://www.elastic.co/guide/en/elasticsearch/reference/current/node.name.html)。用来提供可读性高的 ES 实例名称，它默认名称是机器的 `hostname`，可以自定义：

```shell
Copynode.name: node-1
```

> 同一集群中的节点名称不能相同

------

#### `network.host`

`network.host`：设置访问的[地址](https://www.elastic.co/guide/en/elasticsearch/reference/current/network.host.html)。默认仅绑定在回环地址 `127.0.0.1` 和 `[::1]`。如果需要从其他服务器上访问以及多态机器搭建集群，我们需要设定 ES 运行绑定的 Host，节点需要绑定非回环的地址。建议设置为主机的公网 IP 或 `0.0.0.0`：

```shell
Copynetwork.host: 0.0.0.0
```

> 更多的网络设置可以阅读 [Network Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html)

------

#### `http.port`

`http.port` 默认端口是 9200 ：

```shell
Copyhttp.port: 9200
```

{% note warning %}
注意：这是指 http 端口，如果采用 REST API 对接 ES，那么就是采用的 http 协议
{% endnote%}

------

#### transport.port

REST 客户端通过 HTTP 将请求发送到您的 Elasticsearch 集群，但是接收到客户端请求的节点不能总是单独处理它，通常必须将其传递给其他节点以进行进一步处理。它使用传输网络层（transport networking layer）执行此操作。传输层用于集群中节点之间的所有内部通信，与远程集群节点的所有通信，以及 Elasticsearch Java API 中的 TransportClient。

`transport.port` 绑定端口范围。默认为 9300-9400

```shell
Copytransport.port: 9300
```

> 因为要在一台机器上创建是三个 ES 实例，这里明确指定每个实例的端口。

------

#### `discovery.seed_hosts`

`discovery.seed_hosts`：发现设置。有两种重要的发现和集群形成配置，以便集群中的节点能够彼此发现并且选择一个主节点。[官网/Important discovery and cluster formation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/discovery-settings.html)

`discovery.seed_hosts` 是组件集群时比较重要的配置，用于启动当前节点时，发现其他节点的初始列表。

开箱即用，无需任何网络配置， ES 将绑定到可用的环回地址，并将扫描本地端口 `9300 - 9305`，以尝试连接到同一服务器上运行的其他节点。 这无需任何配置即可提供自动群集的体验。

如果要与其他主机上的节点组成集群，则必须设置 `discovery.seed_hosts`，用来提供集群中的其他主机列表（它们是符合主机资格要求的`master-eligible`并且可能处于活动状态的且可达的，以便寻址[发现过程](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-hosts-providers.html)）。此设置应该是群集中所有符合主机资格的节点的地址的列表。 每个地址可以是 IP 地址，也可以是通过 DNS 解析为一个或多个 IP 地址的主机名（`hostname`）。

> 当一个已经加入过集群的节点重启时，如果他无法与之前集群中的节点通信，很可能就会报这个错误 `master not discovered or elected yet, an election requires at least 2 nodes with ids from`。因此，我在一台服务器上模拟三个 ES 实例时，这个配置我明确指定了端口号。

配置集群的主机地址，配置之后集群的主机之间可以自动发现（可以带上端口，例如 `127.0.0.1:9300`）：

```shell
Copydiscovery.seed_hosts: ["172.168.172.54:9300","172.168.172.55:9300",'172.168.172.56:9300']
```

> the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured

必须至少配置 `[discovery.seed_hosts，discovery.seed_providers，cluster.initial_master_nodes]` 中的一个。

------

#### `cluster.initial_master_nodes`

`cluster.initial_master_nodes`: 初始的候选 master 节点列表。初始主节点应通过其 `node.name` 标识，默认为其主机名。确保 `cluster.initial_master_nodes` 中的值与 `node.name` 完全匹配。

首次启动**全新的 ES 集群**时，会出现一个[集群引导/集群选举/cluster bootstrapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-bootstrap-cluster.html)步骤，该步骤确定了在第一次选举中的符合主节点资格的节点集合。在[开发模式](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/bootstrap-checks.html#dev-vs-prod-mode)下，如果没有进行发现设置，此步骤由节点本身自动执行。由于这种自动引导从本质上讲是[不安全的](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-quorums.html)，因此当您在[生产模式](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html#dev-vs-prod-mode)下第一次启动全新的群集时，你必须显式列出符合资格的主节点。也就是说，需要使用 `cluster.initial_master_nodes` 设置来设置该主节点列表。**重新启动集群或将新节点添加到现有集群时，你不应使用此设置**

> 在新版 7.x 的 ES 中，对 ES 的集群发现系统做了调整，不再有 `discovery.zen.minimum_master_nodes` 这个控制集群脑裂的配置，转而由集群自主控制，并且新版在启动一个新的集群的时候需要有 `cluster.initial_master_nodes` 初始化集群主节点列表。如果一个集群一旦形成，你不该再设置该配置项，应该移除它。该配置项仅仅是集群第一次创建时设置的！集群形成之后，这个配置也会被忽略的！

{% note warning %}
`cluster.initial_master_nodes` 该配置项并不是需要每个节点设置保持一致，设置需谨慎，如果其中的主节点关闭了，可能会导致其他主节点也会关闭。因为一旦节点初始启动时设置了这个参数，它下次启动时还是会尝试和当初指定的主节点链接，当链接失败时，自己也会关闭！

因此，为了保证可用性，预备做主节点的节点不用每个上面都配置该配置项！保证有的主节点上就不设置该配置项，这样当有主节点故障时，还有可用的主节点不会一定要去寻找初始节点中的主节点！
{% endnote%}

关于 `cluster.initial_master_nodes` 可以查看如下资料：

- [Bootstrapping a cluster](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-discovery-bootstrap-cluster.html)
- [Discovery and cluster formation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-settings.html)

#### `node.max_local_storage_nodes`

当一个节点启动多个elasticsearch实例时，这个参数不用设置默认值为`1`即可满足需求

```bash
#这个配置限制了单节点上可以开启的ES存储实例的个数，我们需要开多个实例，因此需要把这个配置写到配置文件中，并为这个配置赋值为2或者更高
node.max_local_storage_nodes: 3
```

end

#### `cors`

```bash
# 	是否支持跨域，默认为false
http.cors.enabled: true

#当设置允许跨域，默认为*,表示支持所有域名，如果我们只是允许某些网站能访问，那么可以使用正则表达式。比如只允许本地地址。 /https?:\/\/localhost(:[0-9]+)?/
http.cors.allow-origin: *
```

end

#### 其他

集群的主要配置项上面已经介绍的差不多了，同时也给出了一些文档拓展阅读。实际的生产环境中，配置稍微会复杂点，下面补充一些配置项的介绍。需要说明的是，下面的一些配置即使不配置，ES 的集群也可以成功启动的。

- [Elasticsearch 集群中节点角色的介绍](https://michael728.github.io/2020/09/20/elk-es-node-cluster/) 对上文中的 `node.master` 等配置做了介绍。如果本地仅是简单测试使用，上文中的 `node.master/node.data/node.ingest` 不用配置也没影响。

## 创建集群

实验机器有限，我们在同一台机器上创建三个 ES 实例来创建集群，分别明确指定了这些实例的 `http.port` 和 `transport.port`。**`discovery.seed_hosts`**明确指定实例的端口对测试集群的高可用性很关键。

> 如果后期有新节点加入，新节点的 `discovery.seed_hosts` 没必要包含所有的节点，只要它里面包含集群中已有的节点信息，新节点就能发现整个集群了。

### 集群配置预览

分别进入`es-7.3.0-node-1`、`es-7.3.0-node-2` 和 `es-7.3.0-node-3` 的文件夹，`config/elasticsearch.yml` 设置如下：

或者`discovery.seed_hosts:['hostname_1','hostname_2'] `使用hostname，但是要在`/etc/hosts`配置域名解析。

```shell
Copy# es-7.3.0-node-1
cluster.name: search-7.3.2
node.name: node-1
node.master: true
node.data: false
node.ingest: false
node.max_local_storage_nodes: 3
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["172.168.172.54:9300","172.168.172.55:9300","172.168.172.56:9300"]
cluster.initial_master_nodes: ["node-1"]

# es-7.3.0-node-2
cluster.name: search-7.3.2
node.name: node-2
node.master: true
node.data: true
node.ingest: false
node.max_local_storage_nodes: 3
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["172.168.172.54:9300","172.168.172.55:9300","172.168.172.56:9300"]

# es-7.3.0-node-3
cluster.name: search-7.3.2
node.name: node-3
node.master: true
node.data: true
node.ingest: false
network.host: 0.0.0.0
node.max_local_storage_nodes: 3
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["172.168.172.54:9300","172.168.172.55:9300","172.168.172.56:9300"]
```

> node-1 节点仅仅是一个 master 节点，它不是一个数据节点。

经过上面的配置，可以通过命令 `egrep -v "^#|^$" config/elasticsearch.yml` 检查配置项。

先启动 node-1 节点，因为它设置了初始主节点的列表。这时候就可以使用 `http://<host IP>:9200/` 看到结果了。然后逐一启动 node-2 和 node-3。通过访问 `http://127.0.0.1:9200/_cat/nodes` 查看集群是否 OK：

```shell
192.168.3.112 25 87 13 4.29   dm - node-3
192.168.3.112 26 87 16 4.29   dm - node-2
192.168.3.112 35 87 16 4.29   m  * node-1
```

> `http://127.0.0.1:9200/_nodes` 将会显示节点更多的详情信息

插键显示结果：



有没有发现，我并没有给 `node-2` 和 `node-3` 明确指定端口，为什么在一台机器上也成功启动了这两个节点？

因为 Elasticsearch 会取用 9200~9299 这个范围内的端口，如果 9200 被占用，就选择 9201，依次类推。

补充：其实，还有一个简单的方法模拟创建集群（该方法我未测试，仅供参考）。

我们首先将上面运行的三个节点停止掉，然后进入 `es-7.3.0-node-1` 文件夹下：

```shell
## 一个节点启动三个elasticsearch实例
mkdir -p data/data{1,2,3}
./bin/elasticsearch -E node.name=node-1 -E cluster.name=appsearch-7.3.2 -E path.data=data/data1 -E path.logs=logs/logs1 -d -p pid1
./bin/elasticsearch -E node.name=node-2 -E cluster.name=appsearch-7.3.2 -E path.data=data/data2 -E path.logs=logs/logs2 -E http.port=9201 -d -p pid2
./bin/elasticsearch -E node.name=node-3 -E cluster.name=appsearch-7.3.2 -E path.data=data/data3 -E path.logs=logs/logs3 -E http.port=9202 -d -p pid3

## 多个节点启动单个elasticsearch实例
./bin/elasticsearch -E  node.name=node-1 
./bin/elasticsearch -E  node.name=node-2
./bin/elasticsearch -E  node.name=node-3
```

end

### 验证集群

```bash
# 查看集群信息
curl 127.0.0.1:9200

# 查看现存索引
curl 127.0.0.1:9200/_cat/indices?v

# 数据查询验证
# 多次执行，对比执行结果 hits.total.value的值是否相同
# took表示查询花了多少时间
curl 127.0.0.1:9200/assert_door/_search

```

## 配置集群x-pack认证:key:

### 生产ca和cert

如下操作在其中一个node节点执行即可，生成完证书传到集群其他节点即可。

```bash
## 默认在/usr/local/elasticsearch-7.7.0/目录下生成elastic-stack-ca.p12  
/usr/local/elasticsearch-7.7.0/bin/elasticsearch-certutil ca
...
## 不能输入密码
Please enter the desired output file [elastic-stack-ca.p12) : 

## 默认在/usr/local/elasticsearch-7.7.0/目录下生成elastic-certificates.p12
/usr/local/elasticsearch-7.7.0/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
...
Please enter the desired output file [elastic-certificates.p12]: 
```

`两条命令均一路回车即可`，不需要给秘钥再添加密码。

证书创建完成之后，默认在es的数据目录，这里统一放到`/usr/local/elasticsearch-7.7.0/conf`下：

生产ca和cert时，不能设置password不然elasticsearch启动会报错

```bash
ElasticsearchSecurityException[failed to load SSL configuration [xpack.security.transport.ssl]]; nested: ElasticsearchException[failed to initialize SSL TrustManager]; nested: IOException[keystore password was incorrect]; nested: UnrecoverableKeyException[failed to decrypt safe contents entry: javax.crypto.BadPaddingException: Given final block not properly padded. Such issues can arise if a bad key is used during decryption.];
Likely root cause: java.security.UnrecoverableKeyException: failed to decrypt safe contents entry: javax.crypto.BadPaddingException: Given final block not properly padded. Such issues can arise if a bad key is used during decryption.

```

### 修改yaml配置文件

三台机器配置文件新增如下：

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
## 这里可以写绝对路径
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

然后启动es集群，如下访问有输出，说明集群添加x-pack正常

`curl localhost:9200/_xpack/security/user?pretty'`

### 配置内置账户密码

ES中内置了几个管理其他集成组件的账号即：`apm_system`, `beats_system`, `elastic`, `kibana`, `logstash_system`, `remote_monitoring_user`，使用之前，首先需要添加一下密码。

```bash
$ elasticsearch-setup-passwords interactive
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana]:
Reenter password for [kibana]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]

## 验证es账密
curl -XGET -u elastic 'localhost:9200/_xpack/security/user?pretty'

curl 127.0.0.1:9200 -u elastic
```

### 非集群模式配置认证

不用生成证书

```bash
$ vim /opt/es/elasticsearch-7.7.0/config/elasticsearch.yml
 
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true

# 设置密码
$./elasticsearch-7.7.0/bin/elasticsearch-setup-passwords interactive
```





## 安装插键

```shell
./bin/elasticsearch-plugin install analysis-icu
```

如果插键安装慢，可以先下载下来，再安装：

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch-plugins/analysis-icu/analysis-icu-7.3.0.zip
./bin/elasticsearch-plugin install file://file path Of analysis-icu-7.1.0.zip
```

## kibana arm 部署

由于Elastic官网没有支持ARM版的Kibana，因此需要下载kibana软件包名带有x86字段，由于Kibana是Java语言编写，因此不影响Kibana在ARM上运行。

但是，kibana内置的node需要替换为arm版本的。

> elasticsearch 内置了jdk，kibana内置了node

1. 通过官网下载kibana

   ```bash
   wget kibana-7.7.0
   ```

2. 解压软件包：

   ```bash
   tar -zxvf kibana-7.7.0-linux-x86_64.tar.gz
   ```

3. 替换Kibana软件包中的Node软件包：

   ```bash
   cd kibana-7.7.0-linux-x86_64
   ## 删除内置node之前运行，node -version确定下node的版本
   rm -fr node
   mv ../node-v10.15.2-linux-arm64 ./node
   ```

4. 修改kibana配置和启动kibana，根据启动情况处理报错

   ```bash
   vim config/kibana.yml
   #放开对应字段，默认注释，并把对应字段修改成如下内容：
   #IP地址“XX.XX.XX.XX”请根据实际填写。
   server.port: 5601
   server.host: "0.0.0.0"
   elasticsearch.hosts: "http://XX.XX.XX.XX:9200"
   kibana.index: ".kibana"
   
   #kibana默认禁止root启动，需要创建账号
   useradd test
   passwd test
   usermod -G test:test
   chown -R test:test /opt/kibana-7.7.0-linux-x86_64
   
   ## 启动
   su - test
   /opt/kibana-7.7.0-linux-x86_64/bin/kibana
   ```

   end

## ES-FAQ

### Q1：`[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`

```shell
echo "vm.max_map_count=262144" > /etc/sysctl.conf
sysctl -p
```

### Q2：`max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]`

```shell
sudo vim /etc/security/limits.conf
# 加入以下内容
* soft nofile 300000
* hard nofile 300000
* soft nproc 102400
* soft memlock unlimited
* hard memlock unlimited
```

### Q3：`master_not_discovered_exception`

主节点指定的名字要保证存在，别指定了不存在的节点名。

## 总结

本文是通过 tar 包方式安装的，安装目录相对集中、配置方便。用 RPM 包安装的话，可以直接用 systemctl 的命令查看 ES 状态、对其重启等。

## 参考

- [learnku/Elasticsearch中文文档-7.3版本](https://learnku.com/docs/elasticsearch73/7.3) 推荐
- [ES-CN 官网/Elasticsearch 集群协调迎来新时代](https://www.elastic.co/cn/blog/a-new-era-for-cluster-coordination-in-elasticsearch) 对于 ES7 的集群发现机制介绍较为详细，推荐
- [程序羊-CentOS7上ElasticSearch安装填坑记](https://www.jianshu.com/p/04f4d7b4a1d3) FAQ 有帮助
- [搭建ELFK日志采集系统](https://jeremy-xu.oschina.io/2018/10/搭建elfk日志采集系统/)
- [静觅—Ubuntu 搭建 Elasticsearch 6 集群流程](https://cuiqingcai.com/6255.html)
- [ELK 架构之 Elasticsearch 和 Kibana 安装配置](https://www.cnblogs.com/xishuai/p/elk-elasticsearch-kibana.html)
- [使用 ELK(Elasticsearch + Logstash + Kibana) 搭建日志集中分析平台实践](https://wsgzao.github.io/post/elk/)
- [手把手教你，在CentOS上安装ELK，进行服务器日志收集](http://www.justdojava.com/2019/08/11/elk-install/)

- 本文作者： Michael翔
- 本文链接： https://michael728.github.io/2020/04/12/elk-es-install/



