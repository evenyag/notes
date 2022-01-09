# 论文阅读：《Monarch: Google’s Planet-Scale In-Memory Time Series Database》

Monarch 是 Google 用来替代 Borgmon 的下一代监控系统

## Borgmon
2004 ~ 2014 年间，Google 的主要监控系统是 Borgmon 。随着部署规模的增长， Borgmon 暴露了不少问题
- Borgmon 的架构鼓励去中心化部署，每个团队需要自己运维一套 Borgmon 实例。这给各个团队使用 Borgmon 带来不少额外的心智负担，同时也难以进行跨应用或者基础设施的监控数据分析
- Borgmon 是的维度和指标值的缺少 schema 约束，查询时容易碰到 schema 的歧义，也限制了查询语言的表达能力
- Borgmon 缺少对分位数的支持(如计算 P99)
- Borgmon 需要用户手动对大指标分片和进行组织

而 Monarch 就是为了解决这些问题而开发的，以提供一个多租户的统一监控服务

## 架构
### 特点
由于定位监控系统， Monarch 为了保证可用性和网络分区容忍能力（AP），牺牲了一定的一致性
- 数据没有直接存储到 Spanner 等强一致的系统
- 保证时效性，忽略延迟的写入
- 允许查询只返回部分数据
- 网络分区后，仍能提供监控和告警服务

在关键监控告警链路上，尽量减少外部依赖
- 数据存储在内存
- 不使用 Google 本身的各种分布式存储基础服务，避免循环依赖

Monarch 是多 Zone 部署的，而每个 Zone 可以看做是一个独立的集群。这种部署模式可以更好地应对网络分区的情况，同时也提高了整体的可用性
- Monarch 本身也有一套全局组件，来提供全局视角

### 组件
组件可以分为三类
- 状态存储
    - Leaves 负责在内存里存储监控数据
    - Recovery logs 负责在磁盘上存储 leaves 中的数据，并进一步写到长期的时序数据存储中
    - 全局的配置服务 configuration server 和每个 zone 中的副本 zonal mirrors ，这些数据时存储在 Spanner 中的
- 数据摄入
    - Ingestion routers 负责根据时序 key 中包含的路由信息路由数据到相应 zone 的 leaf routers
    - Leaf routers 接收该 zone 的数据并路由到负责存储的 leaves
    - Range assigners 管理 zone 内数据在 leaves 中的分布和负载均衡
- 查询
    - Mixers 将查询分解成多个子查询并路由到相应的 leaves 中执行，并对结果进行合并
        - 查询可以由 root mixer 或者 zone mixer 发起
        - root mixer 发起的查询需要经过 root mixer 和 zone mixer 处理
    - Index servers 负责 zone 和 leaf 数据的索引并指导分布式查询的执行
    - Evaluators 负责定期向 mixer 发起查询并将结果写回 leaves

## 数据模型
Monarch 的时序模型类型带 schema 的表模型，每个表包含若干 key 列(key columns)组成的时序 key(time series key) 和一个值列(value column)来存储时序点

### Target
而 key 列，又称为 fields ，由两部分组成， targets 和 metrics
- targets 表示要监控的实体，例如某个继承或者容器，遵循定义好的 target schema
- target schema 中有一个列会负责标识数据应该路由到哪个 zone （就近原则，路由到就近的 zone），例如图中的 cluster 列
- 相同 target 的数据会放到一个 leave 里
- 一个 target 的 schema 名和 key 值组成 target string ，如 `ComputeTask::sql-dba::db.server::aa::0876`
- Monarch 用 leaves 存储数据时，通过 target string 对数据进行 range 分区

### Metrics
Metric 同样有 metric schema ，包含若干 key 列（又称为 metric fields）和一个时序值类型(value type)。 Metric 命名类似文件名，如 `/rpc/server/latency`。

Metric 的时序数据类型支持 boolean, int64, double, string, distribution 或者前面几种类型的组成的 tuple
- distribution 类似 prometheus 的 histogram 类型，包含若干 bucket ，用于计算 p99 等百分位(percentiles)数据
- distribution 的每个 bucket 可以包含一个 exemplar ，用于存储其他信息，例如上图中 RPC trace 以及原始数据的 target/metric fields ，这样在分析一些慢请求的时候就十分有用
- Metrics 类型分为 gauge 和 cumulative 两种，前者不可累计，类似 prometheus 的 gauge ，而后者可以累计，就例如 distribution
    - cumulative 更可靠，因为 target 很可能会出现重启等行为，导致一部分数据缺失

## 可伸缩的数据采集能力
### 架构
采集分为 2 层
- ingestion routers 通过 location field 将数据划分到 zone
- leaf routers 通过 range assigner 将数据按 range 划分到 leaves

往 Monarch 写入数据的流程为
- 上报数据的客户端发送数据到临近的 ingestion routers
- ingestion routers 根据 location field 路由到 zone 的 leaf router 。这个映射关系是可动态配置
- leaf router 中会维护一个 range map ，为每个 target range 记录 3 个 leaf 副本的位置。需要注意，这个映射不是通过 range assigner 获取的，而是通过 leaves 获取的。这样的设计也是为了尽可能提高可用性，可以容忍 range assigner 挂掉
- leaf 负责将数据写到内存和 recovery logs 里。内存存储做了以下优化
    - 相同 target 的 timestamp 会做合并并编码，平均一个 target 有 10 根线可以共享时间戳
    - 值会做 delta 和 run-length 等编码
    - 支持快速的读写和快照
    - 查询和 range 漂移时可以做到不中断服务
    - 减少内存碎片，平衡 cpu 和内存使用

对于持久化层
- Recovery logs 是异步写入的
- Leaves 将 log 写到分布式文件系统 Colossus 里
- 系统能容忍 Colossus 挂掉
- Recovery logs 是对写友好的格式，但会通过压缩，并转为读友好地格式存储到一个长期时序数据存储中
- Recovery logs 会在三个集群间进行异步复制

Leaves 的数据采集也会触发 zone 和 root index server 的更新

### Zone 内负载均衡
由于提了 target schema 出来，相同 target 的数据可以一次性上报，并最多只需要写到 3 个 leaves 中。而根据 target range 进行 range 分区也保证了水平扩展的能力，也提高了数据的局部性。

不同 range 实际上可以选择不同的副本数(1 到 3)副本。

Range assigner 可以移动，分裂，合并 range 来实现负载均衡。以移动一个 range R 的流程为例
- range assigner 分配 R 给目标 leaf ，目标 leaf 会通知 range router 他负责收集该 range 的数据，并写 recovery log
- 等 1 秒之后（大部分情况源 leaf 已经将数据刷到 recovery log 了），目标 leaf 开始从新数据往老数据回放 recovery log
- 待目标 leaf 回放完成，会通知 range assigner 将 R 从源 leaf 移除。随后源 leaf 停止收集 R 的数据并清理内存中对应的数据

可见整个过程也是可用性优先， range 移动的过程中，源和目标 leaf 都仍然提供服务

### 采集过程中的聚合
海量的时序数据如果直接存储原始数据，不论存储成本还是读写成本都相当高，而用户往往关心的是一定程度聚合后的数据。 Monarch 在采集过程中就会对数据进行一定程度的聚合(Collection aggregation)
- 首先数据上报是以 delta 的形式上报的，即每个时间窗口内相邻点的 delta ，这样就算发生了重启之类的操作导致起始值变化，也不会影响 monarch 侧的数值。在上报过程中，客户端， leaf routers 可以做一定的预聚合，并由 leaf 做最终的聚合。
- 采集聚合时，会根据 delta 的 end time 映射到对应的 bucket 里。这里的 bucket 边界做了一定的偏移，应该是为了避免所有线的边界值都相同，导致时间上的热点
- 数据由准入窗口(Admission window)，即太老的数据会被丢弃掉

## 可伸缩的查询
### 查询语言
特有的语法，除了常规的 filter ， group by 外， 也能支持 join ，另外提供了一些针对时序的查询功能如对齐， offset 等

```
{ fetch ComputeTask::/rpc/server/latency
    | filter user=="monarch"
    | align delta (1h)
; fetch ComputeTask::/build/label
    | filter user=="monarch" && job=~"mixer.*"
} | join
  | group_by [ label ], aggregate ( latency )
```

### 查询执行
查询分两种
- 在线查询(ad hoc queries)
    - 用户发起的查询
- 常驻查询(standing queries)
    - 周期地执行的物化视图查询
    - 提高查询速度
    - 用于生成告警
    - 常驻查询会通过 hash 分配给不同的 mixer 处理，保证查询的扩展能力

全局查询会通过 root mixers 处理， zone 级别的常驻查询则直接通过 zone mixer 处理。在查询时， mixers 会查询 index servers 来拿到哪些 zone 和 leaves 是和查询相关的。

前面提到副本的数量是可配置的，而且副本也有迁移等过程，因此 zone 查询时会有一个决定副本(replica resolution)的过程，找到副本质量最高的 leaf 。对于一个 range ，只会选择一个数据质量最高的副本。质量主要根据数据时间范围，密度，完整度等因素综合判断。可以看到，这里没有追求严格的数据一致性，而是选择用数据最齐全的副本来提供查询，也不需要考虑副本间的数据合并。决定副本的过程是通过 leaf 自身来实现的，无需经过 range assigner ，保证可用性

在用户隔离方面，用户查询时本地使用的和在各个节点用的内存会被跟踪起来。使用过多内存的查询会被取消掉。查询线程本身会被放到用户级别的 cgroup (per-user cgroups)，保证 cpu 时间的公平使用

### 查询下推
Monarch 支持将查询下推到尽可能接近数据的节点

#### 下推到 zone
如果查询只涉及一个 zone 的数据，则可以下推到 zone 级别，例如，以下的查询也可以下推到 zone
- 如果 group by 的列里有包含 location field
- join 的输入里包含了相同的 location field

因此，常驻查询里，只有以下的查询需要通过 root evaluators 发起
- 需要查询的输入数据不包含 location 信息，即不分布在确定的 zone 里
- 需要跨 zone 进行数据聚合

#### 下推到 leaf
由于数据按 target range 进行分区，因此一个 target 的数据只会全部分布在一个 leaf 里。因此 target 相关的查询操作会下推到 leaf
- group by target
- 或者 join 条件包含 target

#### 固定的 fields
一部分 field 的值可以认为是固定的，可以用来帮助判断查询是否可以下推。例如查询条件中如果有 cluster 字段，就可以判断查询只会落在固定的 zone ，就可以下推到对应的 zone

### Field 索引（Field Hints Index）
Monarch 的 index servers 负责存储 field hints index (FHI), 用来筛掉查询树上不符合条件的子节点。 FHI 和布隆过滤器类似，存在 False positives ，即也可能返回不符合条件的子节点

FHI 是 field 值的摘要（excerpt），在 Monarch 中就是一系列三元组(trigram)。这个三元组就是从头到尾遍历 field 值构造出来的，例如下面就列出了 `monarch` 的所有 trigrams 。其中 `^` 和 `$` 表示文本的开始和结束
```
^^m, ^mo, mon, ona, nar, arc, rch, ch$, h$$
```

一个 field hint index 就是一个 multimap ，其中 key 是通过 (schema name, field name, excerpt 也就是 trigram) 计算的到的一个 int64 fingerprint ，而 value 是所有包含该 fingerprint 的节点

查询时会将查询条件转化为对应的 trigrams ，例如 `mixer.*` 转为 `^^m, ^mi, mix, ixe, xer` ，然后所有符合这些 trigrams 的节点才会被查询（取交集）

Monarch 的这些索引是全部放在内存里的。需要注意的是，不同 field 的过滤结果是取并集的，也就是说 leaf 的两个 targets `user:'monarch',job:'leaf'` 和 `user:'foo',job:'mixer.root'` 都会被认为是符合条件 `user=='monarch'&&job=~'mixer.*'` 的。除了节省空间外， FHI 的过滤效果也很好，目前 zone 上的 FHI 可以过滤掉 99.5% 的子节点，在 root 上的 FHI 能过滤掉 80% 的子节点

除此之外， FHI 还有一些额外的特性
- 可以支持正则， RE2 库可以将正则转成一系列 trigrams 的集合关系表达式（交并集）。处理正则时， Monarch 会查找 FHI 里的 trigrams 并应用这些表达式
- 针对某些 field 的 也可以用 `fourgrams` 四元组和完整的字符串做 FHI ，在准确性和空间占用上有调整空间，十分灵活
- 对于 join 的场景，可以进一步下推过滤条件做优化
- 针对 metric 名字也做了索引， metric 名字索引时是作为一个叫 `:metric` 的特殊列

FHI 并不会持久化，每次重启会重新根据各个 leaf 的数据构建

### 查询可靠性
#### Zone 裁剪(Zone pruning)
文章提到，根据统计， 99.998% 成功的查询都有个特点，就是在大约超时时间的一半时就能开始收到各个 zone 的结果响应了。因此他们引入了一个 zone 级别的软查询超时(soft query deadline) ，如果一个 zone 在超过 soft query deadline 的时间还没有响应时，就会把这个 zone 从查询里裁剪掉，即不需要他的结果。而被裁剪掉的 zone 的信息也会通过查询返回给用户

这可以很好地保障查询能及时返回，而 Monarch 也能容忍只返回部分结果

#### Hedged reads
在一个 zone 内，查询仍然可能涉及超过 10000 个 leaves 。为了保证查询不受慢节点影响， Monarch 会从响应快的副本查数据。除此之外， zone mixer 在发现某个 leaf 特别慢时，也会发起等价的查询到候选 leaves ，并从中取出更快返回的结果，进行去重等处理后返回

## 配置管理
这部分相对比较简单，就是一套分布式的配置管理系统

### 配置分发
Configuration server 将配置存储在一个全局的 spanner 数据库里，并负责处理所有配置的修改，并将这些配置转换为底层对分布式更友好，更易于其他组件缓存的形式。每个 zone 都有 configuration mirrors 将配置复制到各个 zone ，并分发到 zone 内的各个组件里。每个组件都会在启动时将配置缓存起来，并定期更新。这样的架构保证出现网络分区或者 configuration mirror 不可用时的可用性

### 配置的种类
Monarch 有一套默认的采集，查询，告警等配置来做到“开箱即用”，除此之外，也能让用户灵活地添加后面提到的一些自定义配置

#### Schemas
用户可以自定义 target schema ，而 Monarch 也提供了一个可编程的客户端来方便用户定义自己的带 schema 的 metric ，用户可以很方便地为 metric 加列，设置 metric 命名空间的访问控制，保证别人不会修改他们的 schema

#### 采集，聚合和保存时长
每个指标采集的 target 和保存的时长都是可配置的，以及采集周期，存储介质，副本数，如何降采样等

#### 常驻查询
用户可以配置周期性发起的常驻查询，并将结果写回 Monarch ，以及配置告警，一种结果是 boolean 的查询，以及告警如何通知

## 测试
测试部分不做详述，有兴趣可以去看论文

## 经验教训
Monarch 本身也演进了快十年，其中也收获了不少经验教训
- 时序 key 的字典分区可以提高写入和查询的可伸缩性
    - 相同 target 的所有指标数据可以在一个请求里上报上来
    - 对一个 target 的查询时可以在一个 leaf 里完成
    - 涉及相邻 target 的查询也能减少节点间的数据传输
- 基于推模式的数据采集可以提高系统可靠性，同时简化系统架构
- 强 schema 的模型可以提高系统可靠性和性能
    - 有利于查询前的校验和优化
- 提高系统的可伸缩性是一个长期的过程
    - Index servers ，采集层聚合，常驻查询的分区都是为了提高可伸缩性而在后期添加的功能，这些都不在一开始的架构设计里
- 运行 Monarch 这样一个多租户的服务方便了用户，同时也给开发者提出了很大的挑战
    - 用户的使用模式多种多样，也为保证系统的可靠性带来很大挑战
    - 用量统计，数据检验，用户隔离，限流功能都是为了保证 Monarch 的 SLOs (service-level objectives) 所不可缺的功能
    - 向前兼容很重要
