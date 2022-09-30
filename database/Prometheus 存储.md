### Prometheus 存储

原文： https://mp.weixin.qq.com/s/cbbDOAOROh1SacfImLRxSg

Prometheus 是当下最流行的监控平台之一，它的主要职责是从各个目标节点中采集监控数据，后持久化到本地的时序数据库中，并向外部提供便捷的查询接口。

#### TSDB：Vertical writes, horizontal(-ish) reads

Time series Database 是数据库大家庭中的一员，专门存储随时间变化的数据，如股票价格、传感器数据、机器状态监控等等。时序（Time Series）指的是某个变量随时间变化的所有历史，而样本（Sample）指的是历史中该变量的瞬时值。

每个样本由时序标识、时间戳和数值 3 部分构成，其所属的时序就由一系列样本构成。由于时间是连续的，我们不可能、也没有必要记录时序在每个时刻的数值，因此采样间隔（Interval）也是时序的重要组成部分。采样间隔越小、样本总量越大、捕获细节越多；采样间隔越大、样本总量越小、遗漏细节越多。以服务器机器监控为例，通常采样间隔为 15 秒。

数据的高效查询离不开索引，对于时序数据而言，唯一的、天然的索引就是时间（戳）。因此通常时序数据库的存储层相比于关系型数据库要简单得多。仔细思考，你可能会发现时序数据在某种程度上就是键值数据的一个子集，因此键值数据库天然地可以作为时序数据的载体。通常一个时序数据库能容纳百万量级以上的时序数据，要从其中搜索到其中少量的几个时序也非易事，因此对时序本身建立高效的索引也很重要。



研究过存储引擎结构和性能优化的工程师都会知道，许多数据库的奇技淫巧都是在解决内存与磁盘的读写模式、性能的不匹配问题。

时序数据库也是数据库的一种，只要它想持久化，自然不能例外。但与键值数据库相比，时序数据库存储的数据有更特殊的读写特征，Björn Rabenstein 将称其为：Vertical writes, horizontal(-ish) reads（垂直写，水平读）。

图中每条横线就是一个时序，每个时序由按照（准）固定间隔采集的样本数据构成，通常在时序数据库中会有很多活跃时序，因此数据写入可以用一个垂直的窄方框表示，即每个时序都要写入新的样本数据；用户在查询时，通常会观察某个、某几个时序在某个时间段内的变化趋势，或对其进行聚合计算，因此数据读取可以用一个水平的方框表示。是谓“垂直写、水平读”。

##### Data Model

时间序列由 metric 名和一组 key-value 标签组成。

- metric 名：语义的名字，一般用来表示 metric 功能，比如： http_requests_total， http 总请求数。metric 名由 ASCII 字符，数字，下划线，冒号组成，必须满足 `[a-zA-Z_:][a-zA-Z0-9_:]*`[1](https://einverne.github.io/post/2020/04/prometheus-monitoring-system-and-tsdb.html#fn:name)
- 标签：一个标签就是一个维度，`http_requests_total{method="Get"}` 表示所有 http 请求中 Get 请求，method 就是一个标签，标签需要满足 `[a-zA-Z_:][a-zA-Z0-9_:]*`[1](https://einverne.github.io/post/2020/04/prometheus-monitoring-system-and-tsdb.html#fn:name)
- 样本：实际时间序列，每个序列包括 float64 值和一个毫秒级时间戳





尽管数据模型是存储层之上的抽象，理论上它不应该影响存储层的设计。但理解数据模型能够帮助我们更快地理解存储层。

> 带时间戳的键值对》？？？

在 Prometheus 中，每个时序实际上由多个标签（labels）标识，如：

```
api_http_requests_total{path="/users",status=200,method="GET",instance="10.111.201.26"} 
```

该时序的名字为 api_http_requests_total，标签为 path、status、method 和 instance，**只有时序名字和标签键值完全相同的时序才是同一个时序**。事实上，时序名字就是一个隐藏标签：

```
{__name__="api_http_requests_total",path="/users",status=200,method="GET",
instance="10.111.201.26"} 
```

对于用户来说，标签之间不存在先后顺序，用户可能关注：

- 所有 API 调用的 status
- 某个 path 调用的成功率、QPS
- 某个实例、某个 path 调用的成功率
- ……

##### metric 类型

Client 提供如下 metric 类型：[2](https://einverne.github.io/post/2020/04/prometheus-monitoring-system-and-tsdb.html#fn:metrictype)

- Counter: 累加 metric，只增不减的计数器，默认值为 0，典型应用场景：请求个数，错误次数，执行任务次数
- Gauge: 计量器，与时间无关的瞬时值，值可增可减，比如：当前温度，CPU 负载，运行的 goroutines 个数，数值可以任意加减，node_memory_MemFree 主机当前空闲大小，node_memory_MemAvailable 可用内存
- Histogram：柱状图，直方图，表示一段时间内的资料信息，用于请求时间，响应大小，可以对结果进行采样，分组和统计
- Summary: 类似 Histogram，但提供了 quantiles 功能，昆虫安装百分比划分结果，比如 quantile 为 0.99，表示取采样数据中的 95 数据。

##### LevelDB

https://chienlungcheung.github.io/2020/09/11/leveldb-annotations-0-usage-and-examples/

##### Gorilla

Gorilla 是 Facebook 于 2015 年开放的一个快速, 可扩展的, 内存式时序数据库. 它的一些设计理念影响了后来的 Prometheus. 本文就其设计和实现进行深入分析希望能为各位后续在系统研发中提供灵感.

https://chienlungcheung.github.io/2020/12/05/Gorilla-%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F-%E5%8F%AF%E6%89%A9%E5%B1%95%E7%9A%84-%E5%86%85%E5%AD%98%E5%BC%8F%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/



#### **Compression**

##### **Why Compression**

假设监控系统的需求如下：

- 500 万活跃时序
- 30 秒采样间隔
- 1 个月数据留存

那么经过计算可以得到具体的存储要求：

- 平均每秒采集 166000 个样本
- 存储样本总量为 4320 亿个样本

假设没有任何压缩，不算时序标识，每个样本需要 16 个字节存储空间 (时间戳 8 个字节、数值 8 个字节)，整个系统的存储总量为 7 TB，假设数据需要留存 6 个月，则总量为 42 TB，那么如果能找到一种有效的方式压缩数据，就能在单机的内存和磁盘中存放更多、更长的时序数据。

##### **Timestamp Compression：Double Delta**

时序数据库中常见的一种时间戳或者数值压缩方法：delta-of-delta 算法，可以极大地降低数据存储的成本和提高数据写入、查询的性能。

有两种常用的实现思路：

* 存储相邻两个时间戳差值 Delta(n) = T(n) - T(n-1)
* 存储与起始时间戳的差值 Delta(n) = T(n) - T(0)

由于通常数据采样间隔是固定值，因此前后时间戳的差值几乎固定，如 15 s，30 s。但如果我们更近一步，只存储差值的差值，那么几乎不用再为新的时间戳花费额外的空间，这便是所谓的“Double Delta”。本质上，如果未来所有的采集时间戳都可以精准预测，那么每个新时间戳的信息熵为 0 bit。但现实并不完美，网络可能延迟、中断，实例可能遇到 GC、重启，采样间隔随时有可能波动：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKSKicGr7drdKalkypRFJqkyTy6Ln89aEkuhgVSiaKP6Biax9vuaCECoQSg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



但这种波动的幅度有限，Prometheus 采用了和 FB 的内存时序数据库 Gorilla 类似的方式编码时间戳，详情可以参考 Gorilla) 以及 Björn Rabenstein 在 PromCon 2016 的演讲 PPT，细节比较琐碎，这里不赘述。

##### **Value Compression**

Prometheus 和 Gorilla 中的每个样本值都是 float64 类型。Gorilla 利用 float64 的二进制表示（IEEE754）将前后两个样本值 XOR 来寻找压缩的空间，能获得 1.37 bytes/sample 的压缩能力。Prometheus V2 采用的方式比较简单：

- 如果可能的话，使用整型（8/16/32 位）存储，否则用 float32，最后实在不行就直接存储 float64
- 如果数值增长得很规律，则不使用额外的空间存储

以上做法给 Prometheus V1 带来了 3.3 bytes/sample 的压缩能力。相比于为完全存储于内存中的 Gorilla 相比，这样的压缩能力对于 Prometheus 已经够用，但在 V2 中，Prometheus 也融合了 Gorilla 采用的压缩技术。



#### Storage Layer of Prometheus

Prometheus 是为云原生环境中的数据监控而生，在其设计过程中至少需要考虑以下两个方面：

1、在云原生环境中，实例可能随时出现、消失，因此时序也可能随时出现或消失，即系统中存在大量时序，其中部分处于活跃状态，这会在多方面带来挑战：

- 如何存储大量时序避免资源浪费
- 如何定位被查询的少数几个时序

2、监控系统本身应该尽量少地依赖外部服务，否则外部服务失效将引发监控系统失效。

对于第 2 点，Prometheus 团队选择放弃集群，使用单机架构，并且在单机系统中使用本地 TSDB 做数据持久化，完全不依赖外部服务；第 1 点是需要存储、索引、查询引擎层合作解决的问题，在下文中我们将进一步分析存储层在其中的作用。Prometheus 存储层的演进可以分成 3 个阶段：

- 1st Generation：Prototype
- 2nd Generation：Prometheus V1
- 3rd Generation：Prometheus V2



注意：本节只关注 Prometheus 时序数据的存储，不涉及索引、WAL 等其它数据的存储。





##### 1st Generation：Prototype

在 Prototype 阶段，Prometheus 直接利用开源的键值数据库（LevelDB）作为本地持久化存储，并采用与 BigTable 推荐的时序数据方案类似的 schema 设计：

> LevelDB: https://github.com/google/leveldb

将时序名称、标签（固定顺序）、时间戳拼接成每个样本的键，于是同一个时序的数据就能够连续存储在键值数据库中，提高范围查询的效率。但从图中可以看出，这种方式存储的键很长，尽管键值数据库内部会对数据进行压缩，但是在内存中这样存储数据很浪费空间，这无法满足项目的设计要求。Prometheus 希望在内存中压缩数据，使得内存中可以容纳更多活跃的时序数据，同时在磁盘中也能按类似的方式压缩编码，提高效率。时序数据比通用键值数据有更显著的特征。即使键值数据库能够压缩数据，但针对时序数据的特征，使用特殊的压缩算法能够取得更好的压缩率。因此在 Prototype 阶段，使用三方键值数据库的方案最终流产。

##### 2nd Generation：Prometheus V1



###### **Chunked Storage Abstraction**

上文提到 TSDB 的根本问题是“垂直写，水平读”，每次采样都会需要为每个活跃时序写入一条样本数据，但如果每次为每个时序写入 16 个字节到 HDD/SSD 中，显然这对块存储设备十分不友好，效率低下。**因此 Prometheus V2 将数据按固定长度切割相同大小的分段（Chunks），方便压缩、批量读写。**

访问时序数据时，Prometheus 使用 3 层抽象，如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKundwsIDY3C7kicKkvBEXbGeHQ21Ce6a2e6KwB65gAod78qDIicEPqGVA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



应用层使用 Series Iterator 顺序访问时序中的样本，而 Series Iterator 底下由一个个 Chunk Iterator 拼接而成，每个 Chunk Iterator 负责将压缩编码的时序数据解码返回。这样做的好处是，每个 Chunk 甚至可以使用完全不同的方式编码，方便开发团队尝试不同的编码方案。





**Chunk Encoding**

Prometheus V1 将每个时序分割成大小为 1KB 的 chunks，如下图所示：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKvOjSQjtmlvJtk0jjicpUiaEh8Rwnaict7OG7n1HpTO2FMLMicSFCSSKZrA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在内存中保留着最近写入的 chunk，其中 head chunk 正在接收新的样本。每当一个 head chunk 写满 1KB 时，会立即被冻结，我们称之为完整的 chunk，从此刻开始该 chunk 中的数据就是不可变的（immutable），同时生成一个新的 head chunk 负责消化新的请求。每个完整的 chunk 会被尽快地持久化到磁盘中。内存中保存着每个时序最近被写入或被访问的 chunks，当 chunks 数量过多时，存储引擎会将超过的 chunks 通过 LRU 策略清出。



在 Prometheus V1 中，每个时序都会被存储到在一个独占的文件中，这也意味着大量的时序将产生大量的文件。存储引擎会定期地去检查磁盘中的时序文件，是否已经有 chunk 数据超过保留时间，如果有则将其删除（复制后删除）。



Prometheus 的查询引擎的查询过程必须完全在内存中进行。因此在执行之前，存储引擎需要将不在内存中的 chunks 预加载到内存中：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKQXRruQfpjyaEUbOV9IfroVDLkRcWicvsMe7QlaLaCDibUI8btuH9oFgw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如果在内存中的 chunks 持久化之前系统发生崩溃，则会产生数据丢失。为了减少数据丢失，Prometheus V1 还使用了额外的 checkpoint 文件，用于存储各个时序中尚未写入磁盘的 chunks：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKLV6R0hQoLfv1BT93Kw3cdRCtlCz5DhMVYBt8rbmAwA7dWS5QOm5bgw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**Prometheus V1 vs. Gorilla**

正因为 Prometheus V1 与 Gorilla 的设计理念、需求有所不同，我们可以通过对比二者来理解其设计过程中使用不同决策的原因。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKoBcX89AFiaVu31JZlQa1jrOsg84HJ4aGaoANpZ9K4icf6tM8ld7FhD1w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



##### 3rd Generation：Prometheus V2

**The Main Problem With 2nd Generation**

Prometheus V1 中，每个时序数据对应一个磁盘文件的方式给系统带来了比较大的麻烦：

- 由于在云原生环境下，会不断产生新的时序、废弃旧的时序（Series Churn），因此实际上存储层需要的文件数量远远高于活跃的时序数量。任其发展迟早会将文件系统的 inodes 消耗殆尽。而且一旦发生，恢复系统将异常麻烦。不仅如此，在新旧时序大量更迭时，由于旧时序数据尚未从内存中清出，系统的内存消耗量也会飙升，造成 OOM。
- 即便使用 chunks 来批量读写数据，从整体上看，系统每秒钟仍要向磁盘写入数千个 chunks，造成 I/O 压力；如果通过增大每批写入的量来减少 I/O 次数，又将造成内存的压力。
- 同时将所有时序文件保持打开状态很不合理，需要消耗大量的资源。如果在查询前后打开、关闭文件，又会增加查询的时延。
- 当数据超过留存时间时需要删除相关的 chunks，这意味着每隔一段时间就要对数百万的文件执行一次删除数据操作，这个过程可能需要持续数小时。
- 通过周期性地将未持久化的 chunks 写入 checkpoint 文件理论上确实可以减少数据丢失，但是如果执行数据恢复需要很长时间，那么实际上又错过了新的数据，还不如不恢复。

因此 Prometheus 的第三代存储引擎，主要改变就是放弃“一个时序对应一个文件”的设计理念。

###### **Macro Design**

第三代存储引擎在磁盘中的文件结构如下图所示：

补一个: `tree ./data`

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKt1z3gcrF4gt5QiaMK9lDYgX3ynwfiaNz1ricdGGe312icP58ncXZ1jLYicw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



根目录下，顺序排列着编了号的 blocks，每个 block 中包含 index 和 chunk 文件夹，后者里面包含编了号的 chunks，每个 chunk 包含许多不同时序的样本数据。其中 index 文件中的信息可以帮我我们快速锁定时序的标签及其可能的取值，进而找到相关的时序和持有该时序样本数据的 chunks。值得注意的是，最新的 block 文件夹中还包含一个 wal 文件夹，后者将承担故障恢复的职责。

**Many Little Databases**

第三代存储引擎将所有时序数据按时间分片，即在时间维度上将数据划分成互不重叠的 blocks，如下图所示：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKwFGQZeVZD7xy6E1KypyPyUgicOtiaOUQWDkeJdZqGRbqHhEomW0piaxicg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



每个 block 实际上就是一个小型数据库，内部存储着该时间窗口内的所有时序数据，因此它需要拥有自己的 index 和 chunks。除了最新的、正在接收新鲜数据的 block 之外，其它 blocks 都是不可变的。由于新数据的写入都在内存中，数据的写效率较高：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKpjfTeMXDNicDkgK9ubUmQZJw4T8xygA2ibNwFiaXlnicMibgNazsOqQAgaQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



为了防止数据丢失，所有新采集的数据都会被写入到 WAL 日志中，在系统恢复时能快速地将其中的数据恢复到内存中。在查询时，我们需要将查询发送到不同的 block 中，再将结果聚合。

按时间将数据分片赋予了存储引擎新的能力：

- 当查询某个时间范围内的数据，我们可以直接忽略在时间范围外的 blocks
- 写完一个 block 后，我们可以将轻易地其持久化到磁盘中，因为只涉及到少量几个文件的写入
- 新的数据，也是最常被查询的数据会处在内存中，提高查询效率（第二代同样支持）
- 每个 chunk 不再是固定的 1KB 大小，我们可以选择任意合适的大小，选择合适的压缩方式
- 删除超过留存时间的数据变得异常简单，直接删除整个文件夹即可



**mmap**

第三代引擎将数百万的小文件合并成少量大文件，也让 mmap 成为可能。利用 mmap 将文件 I/O 、缓存管理交给操作系统，降低 OOM 发生的频率。



**Compaction**

在 Macro Design 中，我们将所有时序数据按时间切割成许多 blocks，当新写满的 block 持久化到磁盘后，相应的 WAL 文件也会被清除。写入数据时，我们希望每个 block 不要太大，比如 2 小时左右，来避免在内存中积累过多的数据。读取数据时，若查询涉及到多个时间段，就需要对许多个 block 分别执行查询，然后再合并结果。假如需要查询一周的数据，那么这个查询将涉及到 80 多个 blocks，降低数据读取的效率。

为了既能写得快，又能读得快，我们就得引入 compaction，后者将一个或多个 blocks 中的数据合并成一个更大的 block，在合并的过程中会自动丢弃被删除的数据、合并多个版本的数据、重新结构化 chunks 来优化查询效率，如下图所示：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKJhHsaWUMibZfMA3fPNX47BOvWicFQtVAw78n30MyHjHoHIqhaDeyVdew/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**Retention**

当数据超过留存时间时，删除旧数据非常容易：



![Image](https://mmbiz.qpic.cn/mmbiz_jpg/A1HKVXsfHNlPPAPA5kViasqGz2BSXFOIKXmvEBYPibW4lcoeleQnJVZLiaialHKavO4p98CJXQDNfF07Bh2Vkjkr9g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



直接删除在边界之外的 block 文件夹即可。如果边界在某个 block 之内，则暂时将它留存，知道边界超出为止。当然，在 Compaction 中，我们会将旧的 blocks 合并成更大的 block；在 Retention 时，我们又希望能够粒度更小。所以 Compaction 与 Retention 的策略之间存在着一定的互斥关系。Prometheus 的系统参数可以对单个 block 的大小作出限制，来寻找二者之间的平衡。



看到这里，相信你已经发现了，这不就是 LSM Tree 吗？每个 block 就是按时间排序的 SSTable，内存中的 block 就是 MemTable。

**Compression**

第三代存储引擎融合了 Gorilla 的 XOR float encoding 方案，将压缩能力提升到 1-2 bytes/sample。具体方案可以概括为按顺序采用以下第一条适用的策略：

1. Zero encoding：如果完全可预测，则无需额外空间
2. Integer double-delta encoding：如果是整型，可以利用 double-delta 原理，将不等的前后间隔分成 6/13/20/33 bits 几种，来优化空间使用
3. XOR float encoding：参考 Gorilla
4. Direct encoding：直接存 float64

平均下来能取得 1.28 bytes/sample 的压缩能力。

##### OSS

使用Thons可以将prometheus数据持久化到oss

##### WAL

WAL： write-ahead log 

https://zhuanlan.zhihu.com/p/261118971

https://zhuanlan.zhihu.com/p/261267784

Head块是数据库的内存部分，灰色块是磁盘上不可更改的持久块。我们有一个预写日志（WAL）用于持久写入。传入的sample（粉红色的方框）首先进入Head块，并在内存中停留一段时间，然后刷新到磁盘并映射到内存（蓝色的方框）。当这些内存映射的块或内存中的块变旧到一定程度时，它们会作为持久性块被刷新到磁盘。随着它们变旧，将合并更多的块，并在超过保留期限后将其最终删除。

WAL是数据库中发生的事件的顺序日志。在写入/修改/删除数据库中的数据之前，首先将事件记录（附加）到WAL中，然后在数据库中执行必要的操作。

不管出于何种原因，如果机器或程序崩溃，都会在此WAL中记录事件，您可以按照相同的顺序重播这些事件以恢复数据。这对于内存数据库尤其有用，在内存数据库中，如果数据库崩溃，则如果不是WAL，则内存中的所有数据都会丢失。

这在关系数据库中被广泛使用，以为数据库提供持久性（[ACID](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/ACID)中的D）。同样，普罗米修斯（Prometheus）通过WAL，为其Head block提供可用性。 Prometheus还使用WAL进行正常重启以恢复内存中状态。

> 原子，一致，隔离，持久

在Prometheus中，WAL仅用于记录事件并在启动时恢复内存中状态。它不以任何其他方式涉及读取或写入操作。

TSDB中的写请求由 [series](https://link.zhihu.com/?target=https%3A//prometheus.io/docs/concepts/data_model/) / metric name的标签值及其对应的sample组成。这给了我们两种记录类型，[series](https://link.zhihu.com/?target=https%3A//prometheus.io/docs/concepts/data_model/) 和sample。

> Samples
>
> Samples form the actual time series data. Each sample consists of:
>
> - a float64 value
> - a millisecond-precision timestamp
>
> Series
>
> Every time series is uniquely identified by its *metric name* and optional key-value pairs called *labels*..

默认情况下，WAL存储为带有128MiB的编号文件序列。这里的WAL文件称为“segment”。

```text
data
└── wal
   ├── 000000
   ├── 000001
   └── 000002
```

文件的大小必然会使旧文件的垃圾回收更加简单。可以猜到，序列号总是增加的。

##### Head Chunk

https://zhuanlan.zhihu.com/p/261321039

一旦chunk填满（默认120个sample，或是到 了chunk/block 范围，默认为`chunkRange`），默认情况下为2h，则将剪切一个新chunk，并称旧块为“full”。

当一个chunk已满时，我们剪切了一个新chunk，而较旧的chunk变得不可变，只能从中读取

这些chunks保留在其自己的目录中，称为`chunks_head`，其文件序列类似于WAL（除了以1开头）。

例如：

```text
data
├── chunks_head
|   ├── 000001
|   └── 000002
└── wal
    ├── checkpoint.000003
    |   ├── 000000
    |   └── 000001
    ├── 000004
    └── 000005
```

文件的最大大小保持为128MiB。



##### index & 重启

Index 在哪里？

它在内存中并存储为倒排索引。有关该索引的总体概念的更多信息，请参见Fabian的博客文章。当Head块的压缩发生时，创建了一个持久性块，Head块被截断以删除旧的块，并且对该索引进行了垃圾回收，以删除Head中不再存在的任何系列条目。

**处理重启**

需要定期删除旧的WAL段，否则，磁盘最终将填满，TSDB的启动将花费大量时间，因为它必须重播此WAL中的所有事件（其中大部分会被丢弃，因为旧）。

如果TSDB必须重新启动（优雅地或突然地），它将使用磁盘上的内存映射的chunk和WAL恢复数据和事件，并重新构造内存中的索引和chunk。

WAL重播是启动最慢的部分。主要是，（1）从磁盘解码WAL记录和（2）从各个sample重建压缩chunk是重放的缓慢部分。内存映射chunks的迭代相对较快

#### LSM-Tree

十多年前，谷歌发布了大名鼎鼎的"三驾马车"的论文，分别是GFS(2003年)，MapReduce（2004年），BigTable（2006年），为开源界在大数据领域带来了无数的灵感，其中在 “BigTable” 的论文中很多很酷的方面之一就是它所使用的文件组织方式，这个方法更一般的名字叫 Log Structured-Merge Tree。在面对亿级别之上的海量数据的存储和检索的场景下，我们选择的数据库通常都是各种强力的NoSQL，比如Hbase，Cassandra，Leveldb，RocksDB等等，这其中前两者是Apache下面的顶级开源项目数据库，后两者分别是Google和Facebook开源的数据库存储引擎。而这些强大的NoSQL数据库都有一个共性，就是其底层使用的数据结构，都是仿照“BigTable”中的文件组织方式来实现的，也就是LSM-Tree。

https://cloud.tencent.com/developer/article/1441835

LSM-Tree全称是Log Structured Merge Tree，是一种分层，有序，面向磁盘的数据结构，其核心思想是充分了利用了，磁盘批量的顺序写要远比随机写性能高出很多。

LSM tree （**log-structured merge-tree** ）通过一种叫做 **SSTable (Sorted Strings Table)** 的格式，持久化到硬盘上。正如其名，SSTable 是一种用来存储有序的键值对的格式，其中键的组织是有序存储的。一个SSTable 会包括多个有序的子文件，被称为 *segment* 。 这些 *segments* 一旦被写入硬盘，就不可以再修改了。

##### 写入数据

由于 LSM tree 只会进行顺序写入，所以自然而然地就会引出这样一个问题，写入的数据可能是任意顺序的，我们又如何保证数据能够保持 SSTable 要求的有序组织呢？
这就需要引入新的常驻内存 *(in-memory)* 数据结构: *memtable_了, _memtable* 的底层数据结构则有点像[红黑树](https://en.wikipedia.org/wiki/Red–black_tree),当由新的写入操作则将数据插入到红黑树中。

写入操作会先把数据存储到红黑树中，直至红黑树的大小达到了预先定义的大小。一旦红黑树的大小达到阈值，就会把数据整个刷到磁盘中，这个过程就可以把数据保证有序写入了。经过一层数据结构的承接，就可以保证单向顺序写入的同时，也能保证数据的有序了。

##### 读取数据  

如何从SSTable中查找数据的呢？一种naive的方法就是遍历所有的 segments，寻找我们需要的key。从最新的 segment 到最老的 segment 一一遍历，知道找到目标key为止。显然，这种方式在寻找刚刚写入的数据是比较快的，但是文件一多就不太行了。因此也有针对这个问题的优化，[稀疏索引](https://yetanotherdevblog.com/dense-vs-sparse-index/) 就是一种在内存中对数据检索进行优化的技术。



LSM tree 引擎是如何工作的：

1. 写入操作是先写入内存的（被成为 memtable）。所有的用于加速查询的数据结构（布隆过滤器和稀疏索引）都会被同时更新；
2. 当内存中的 memtable 太大了，将会被刷到磁盘中，注意是有序的；
3. 当查询时我们先回查询布隆过滤器，如果布隆过滤器返回说键不存在，则实际不存在，如果布隆过滤器说存在，进一步遍历 segment 文件；
4. 对于遍历 segment 文件的过程，我们将会先通过稀疏索引找到最小的文件范围，并开始由新到老开始遍历，找到一个key则直接返回。

#### 红黑树

红黑树的查询时间复杂度是O(log2 N)，2为底N的对数，10亿数据最坏情况需要30次。

流弊-