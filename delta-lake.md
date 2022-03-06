# Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores

基于对象存储实现 ACID 事务和高性能是比较困难的
- 元数据操作，如 list 对象开销较大
- 一致性保证比较有限

Delta Lake 就是为了基于对象存储实现 ACID 的表格存储层而开发的。通过使用和 parquet 格式紧密结合的 transaction log 机制实现了
- ACID 特性
- 时间回溯（time travel）
- 大量表格数据下更快的元数据操作（亿级别表分区下的快速检索能力）

基于这些能力， Delta Lake 还提供了部分高级特性
- 自动数据布局优化（automatic data layout optimization）
- 更新或插入（upsert）
- 缓存
- 审计日志（audit log）

Delta Lake 也可以很方便地作为 Spark/Hive/Presto/Redshift 这些系统的数据源

# INTRODUCTION
尽管很多大数据系统都支持读写对象存储了，但是要在上面实现高性能和可修改的表格系统还是很有挑战性的，因为大部分对象存储都没提供不同 key 之间的一致性保证
- 同时更新多个对象不是原子的
    - 可能看到部分更新的数据
    - 回滚写入是困难的，如果更新操作中途失败了，那么表的实际上就处于损坏（corrupted）的状态了
- 对象太多的时候，元数据操作代价就会很高
    - 在 HDFS 里读一个 parquet 文件的 footer 可能只需要几毫秒，而在对象存储里耗时会很高，有时读 min/max 这些元数据做过滤的耗时比实际读取数据的耗时还大

而 Delta Lake 就是为了解决这些问题而设计的，核心就是通过一个 write-ahead log (wal) 以 ACID 的方式维护每个 delta table 的对象信息，而这个 wal 自身也是存在对象存储上的。这个日志还包括每个文件的 min/max 统计之类的元数据信息，可以更快地进行元数据检索

最关键的就是，所有元数据都是存储在对象存储上，事务是通过基于对象存储的乐观并发控制协议实现的（具体协议因云厂商不同而有差异），因此不需要服务器维护 delta table 的状态，做到了存储计算分离，用户可以按需启动服务来执行查询

除此之外，还实现了以下传统数据湖没有的功能
- Time travel ，可查询任意时间点的快照和回滚有问题的更新
- UPSERT/DELETE/MERGE operations
- Efficient streaming I/O ，允许流式作业低延时地往 delta table 写入小对象，后续在将其合并成大对象提高查询性能，还支持读最新的数据，完全可以将 delta table 当做消息总线来使用
- Caching ，因为对象不可变，因此很方便做缓存（Databricks 做了一层高速的 SSD 缓存）
- Data layout optimization 可以自动优化对象的大小以及数据的聚集方式(clustering)，而不影响实时查询
- Schema evolution 即使 schema 变了，也无需重建 parquet 文件，允许继续读老的 parquet 文件
- Audit logging 基于 trasaction log 实现

# MOTIVATION: CHARACTERISTICS AND CHALLENGES OF OBJECT STORES
## Object Store APIs
对象存储的 API 大同小异
- 允许用户创建 bucket 来存储多个对象，每个对象就是一个二进制的 blob ，大小不超过数 TB
- 每个对象通过 key 定位
- 用 key 来模拟目录
- rename 操作代价很高
- 提供 list 操作
    - S3 的 list 一次最多返回 1000 个 keys，每次都需要化 10 到几百毫秒
- 支持只读部分内容
- 更新需要整个对象重新覆盖，这个更新操作是原子的（有些对象存储允许 append）
- 有些云厂商基于对象存储包装了分布式文件服务
    - 不过还是有不支持跨目录原子更新等能力

## Consistency Properties
大部分对象存储对单个 key 提供最终一致性，而多个 key 间不提供一致性保证
- 一个客户端如果上传了一个对象，其他客户端不保证能马上通过 list 操作查到或者直接访问到
- 不保证其他客户端能马上看到被更新的对象
- 不同厂商的一致性模型有差异

以 S3 为例
- 对写入新对象的客户端提供 read-after-write 保证
- 当客户端用 PUT 写入一个新对象，那么该客户端可以保证 GET 能读到该对象，除了下面一种情况
    - 如果客户端在 PUT 之前先查了一次那个不存在的 key
    - 则不保证能马上读到后续写入该 key 的对象
    - 这个应该是做了类似不存在的缓存

其他厂商有的会提供更强的保证，但基本没有多 key 操作的保证

## Performance Characteristics
性能方面，需要做好顺序 IO 和并行的平衡
- 读操作
    - 一般需要有 5 到 10 毫秒的基础延迟
    - 随后读取数据的时候，速度可以到 50 ~ 100 MB/s
    - 这意味着要达到较高的吞吐，一次读取的数据量不能太小
    - 另外一般需要并发发起若干个读请求以把带宽打满
- List 操作
    - 也需要大量的并行
    - 可能需要上百的并行度
- 写入通常需要整个覆盖对象
    - 对于少量更新（主要是 point updates）的情况，对象最好比较小，但不利于读

这意味着表格存储需要
- 保证经常访问的数据分布比较近，所以需要用列存
- 保证对象够大，但不能太大
- 避免 list 操作

## Existing Approaches for Table Storage
现在基于对象存储实现表格存储有以下几种方式

### Directories of Files
最直观也是最常见的方式，就是直接将表作为一系列对象，用列存的格式如 parquet 存起来。另外，会根据部分字段做下分区，放到不同的目录下，减少 list 的开销和查询的分区数
- 代表的系统是 Hive

遇到的挑战
- 没有多对象的原子性
    - 部分更新
    - 数据损坏
- 最终一致性
    - 客户端可能只能看到部分更新
- 性能较差
    - 即使做了分区， list 操作效率还是比较差
    - 读对象里的统计信息也比较慢
- 缺少管理功能
    - 没有类似表的版本管理和审计日志等功能

### Custom Storage Engines
Snowflake 的做法，将元数据的管理作为单独的服务，并提供强一致保证
- 绕过对象存储的一致性问题
- 需要一个高可用的服务来管理元数据，可能开销会比较大
- 外界计算引擎进行查询时会引入额外的开销，也会造成平台绑定

遇到的挑战
- 所有 IO 操作都需要和元数据服务交互
    - 增加资源开销
    - 降低性能
- 不利于和其他计算系统对接
    - 需要额外开发，而不像用 parquet 之类的有现成的方案（Spark/TensorFlow/PyTorch）
- 把用户绑定在元数据服务所在的提供商

### Metadata in Object Stores
Delta lake 的方案
- 直接将事务日志和元数据存储到对象存储里
- 通过一系列基于对象存储操作的协议实现串行性
- 数据以 Parquet 格式存放
- 有其他系统如 Hudi 和 Iceberg 有类似的方案，不过 Delta Lake 还提供了类似 Z-order clustering/缓存/后台优化之类其他系统没有的特性

# DELTA LAKE STORAGE FORMAT AND ACCESS PROTOCOLS

delta lake 表对应对象存储或者文件系统里的一个目录，里面存放表数据的数据对象（data objects）和事务操作（以及相关的检查点， checkpoints）的日志

## Storage Format
### Data Objects
表的数据文件
- Parquet 格式
- 可以 Hive 的分区命名风格分多目录存放
- 每个数据对象都有一个唯一的名字，由写入方生成，一般就是个 GUID
- 对象属于表的哪个版本则由事务日志决定

### Log
事务日志
- 存放在 _delta_log 目录
- 一系列 json 对象
    - 文件名是一个数字 id ，开头填充 0 ，这样可以保证字典序
- 定期做检查点，即 checkpoint
    - parquet 格式

每个日志对象都是一个数组，记录了要应用到上个版本的操作
- Change Metadata
    - 记录元数据的修改
    - 第一个版本必须包含 metaData 操作
    - 主要记录 schema ，分区的列名，存储格式，配置（append only）等
- Add or Remove Files
    - 数据对象的增删
    - 增加一个对象时，还会记录对象的统计信息
        - 总记录数
        - min/max 索引
        - null 的数量
    - 删除对象时，还会记录删除的时间
        - 对象真正的物理删除，会等到用户指定的一个过期时间之后才会删
            - 保证正在读该对象的查询不会出错
            - 可以用于实现 time travel
        - 在没有物理删除前，这个 remove 操作还需要留在日志和 checkpoint 里，表示该对象已经标记为删除了
    - 还有一个 dataChange 标记表示数据是不是被修改了
        - 像 compaction ，统计更新之类的不修改数据本身的操作就可以被标记
        - 流计算引擎在读取最新的日志时，可以跳过这种操作，因为不影响计算结果
- Protocol Evolution
    - 协议兼容用
- Add Provenance Information
    - 有 commitInfo 的操作，可以记录哪个用户做了这个操作
- Update Application Transaction IDs
    - 可以让应用将自己的一些数据放到日志记录中
    - 流计算可以写入一个 txn 操作，包含 appId 和 version 字段来记录当前状态，例如消费位点
        - 可以用这个来避免重复消费和写入，提供 exactly-once 语义
        - 用到该机制： Delta Lake connector for Spark Structured Streaming

### Log Checkpoints
检查点，记录了某个 log record ID 以及之前的所有日志最终的状态信息
- 0000003.parquet 就包含了 0000003.json 和之前的所有 json 对象的信息
- 如果一个对象被 add 然后被 remove 了，那么 add 的操作无需再记录， delete 还需要记录，直到对象物理删除了
- 同一个对象更新多次的话，只需要保留最后的 add 记录
- 相同 appId 的 txn 操作只需要保留最后一个
- changeMetaData/protocol 操作都可以被合并
- 有了检查点， list 的时候就只需要 list 检查点之后的对象，减少 list 的开销
    - 比用 list 找到 parquet 文件并读取 footer 快
- 默认每 10 个事务做一次 checkpoint
- 最新的检查点 ID 记录在 _delta_log/_last_checkpoint 文件

## Access Protocols
### Reading from Tables
只读事务可以安全地读取到表的某个版本的内容
- 从 _last_checkpoint 读取最近的 checkpoint id
- 用刚刚的 id （没有 checkpoint 则用 0）去 list 日志路径下所有更新的 json/parquet 文件
    - 可以用拿到的最新的文件 id 作为版本
    - 由于对象存储最终一致性的问题，所以 list 的对象中间可能有空洞
    - 对于空洞，客户端可以等该对象可读后再读取该对象
- 用 checkpoint 和后续的对象确定表的最终状态
    - 这一步可以并行化
        - 我认为主要是文件的读取可以并行化，数据文件的统计信息等处理可能也可以并行化
- 通过统计信息找到查询相关的对象
- 并行地查询所有对象

这里检查点的读取可以容忍最终一致性问题，因为读到老的 _last_checkpoint 也不影响后续的查询，只不过要多 list 些对象

### Write Transactions
主要分 5 步
- 通过只读事务的 1, 2 步确定最新的日志 id ，假设是 r ，那么表的下个版本就是 r + 1
- 如果有需要，用读事务中同样的步骤读取版本 r 的数据
    - 更新之类的情况，或者读写事务
- 将事务涉及的对象写到对象存储
- 尝试写 r + 1 版本对象的日志文件，并且要保证没有其他人写过 r + 1 版本的对象
    - 需要是原子操作
    - 如果有其他人已经写过了，说明事务冲突了，就需要重试事务
- 有需要的话，创建一个检查点，将其指向版本 r + 1

这里最复杂的是如何原子地保证 r + 1 版本的日志对象只写了一次，不同云平台不一样
- Google Cloud Storage 和 Azure Blob Store 有原子的 put-if-absent 操作
- HDFS/Azure Data Lake Storage 通过原子的 rename 操作实现
- S3 没有这种操作，因此是通过额外引入一个轻量级的协调服务来保证只有一个客户端写入日志
    - 只有写入依赖这个服务，因此开销不大
    - 开源版本里没提供这种服务

## Available Isolation Levels
简单来说
- 写事务提供 serializable 级别
- 读事务提供 snapshot isolation 或者 serializable 级别
    - serializable 是通过一个 dummy 的 read-write 事务实现的（等于走一次写的流程）
- 只支持单表事务

## Transaction Rates
写事务的速度取决于底层 put-if-absent 操作的速度
    - 对象存储写入延迟在几十到几百毫秒
    - 因此事务速度可能也就每秒若干个
    - 对 delta lake 用户来说基本都够用
    - 如果后面需要更高的事务速度，可以考虑引入定制的 LogStore 服务

# HIGHER-LEVEL FEATURES IN DELTA
基于事务能力，解锁了很多新功能

## Time Travel and Rollbacks
对象的不可变性让 time travel 更容易实现，用户可以指定数据文件的版本保留多久，需要时通过 SQL AS OF timestamp 以及 VERSION AS OF commit_id 等语法访问过去的版本

客户端也有 api 获取最近读写的 commit id

另外，也支持 SQL MERGE 语句来用老数据修正新数据
```sql
MERGE INTO mytable target
USING mytable TIMESTAMP AS OF <old_date> source
ON source.userId = target.userId
WHEN MATCHED THEN UPDATE SET *
```

## Efficient UPSERT, DELETE and MERGE
SQL 的 UPSERT/DELETE/MERGE 等操作都支持

## Streaming Ingest and Consumption
一定程度上， Delta Lake 可以当做消息队列来使用，而不需要通过额外的消息队列将数据导入数据湖，或者做实时消费
- Write Compaction
    - 用户导入小文件，降低延迟
    - 通过后台进程将数据文件做 compaction
- Exactly-Once Streaming Writes
    - 前面提到的，可以通过 txn 记录实现
- Efficient Log Tailing
    - 可以读取最新的日志文件来做消费

## Data Layout Optimization
可以执行后台任务，优化数据的分布
- OPTIMIZE 命令
    - 用户手动执行
    - 合并小对象成 1G 左右的大对象，更新统计
- Z-Ordering by Multiple Attributes
    - 如果分区字段有多个，很可能产生大量小分区，降低性能
    - 支持按照 Z-Order 重新组织数据，让数据分布更加有局部性
    - 用户可以设置 Z-Order 的字段然后通过 OPTIMIZE 命令重新组织数据，并且后续也可以修改字段的组合
- AUTO-OPTIMIZE
    - Databricks 云服务的用户可以设置自动优化，自动合并小对象

## Caching
不可变的对象方便缓存， Databricks 服务也提供了一层缓存层

## Audit Logging
commitInfo 记录可以用于实现审计日志，用户可以通过 DESCRIBE HISTORY 命令查看表的历史

## Schema Evolution and Enforcement
由于有 schema 的变更记录，因此处理不同 schema 的对象更方便
- 删列的时候，可以通过事务将对象都更新掉
- 加列的时候，由于有记录，无需将所有对象都更新掉
- 写入数据会校验 schema ，帮助提前发现问题

## Connectors to Query and ETL Engines
支持在数据目录生成 Hive 的 symlink manifest files (_symink_format_manifest) ，可以很方便地和 Hive 等第三方查询引擎集成

# DELTA LAKE USE CASES
主要是一些应用场景，和前面的内容其实比较类似，不详细记录了

# PERFORMANCE EXPERIMENTS
## Impact of Many Objects or Partitions
主要测试随着分区数增多，查询的性能表现
- 相比 Hive / Presto 和直接读取 parquet ，在多分区下性能表现更优异
- 能支持更多（百万量级）分区，而 Hive/Presto 在这个场景下查询耗时要非常久（小时以上）

## Impact of Z-Ordering
Z-Ordering 对于多维查询效果比数据按单一顺序排序更好，可以筛选掉更多的对象
- 数据有 4 个维度 (sourceIP, sourcePort, destIP, destPort)
- 全局顺序排序，即按照 (sourceIP, sourcePort, destIP, destPort) 排序，只有按照 sourceIP 做筛选才能发挥过滤效果
- Z-Order 排序，则分别用 4 个维度去筛选，最低也能跳过 43% 的对象

## TPC-DS Performance
相比 Spark + Parquet/Presto 都有提升

## Write Performance
Spark 任务写入 delta lake 和直接写 parquet 文件差不多，可见写入时维护数据对象统计信息对写入的影响不大

# DISCUSSION AND LIMITATIONS
Delta Lake 的最大特点是基于对象存储实现了 ACID 事务，而且不依赖其他第三方系统来做协调

不过当前的设计和实现也还有局限性和值得优化的地方
- 只支持单表事务
- 对于流式场景，受限于对象存储的延迟
- 不支持二级索引，不过当前在规划基于 bloom filter 的索引了
