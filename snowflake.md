# The Snowflake Elastic Data Warehouse

Snowflake 是一个基于云（aws）构建的一个弹性数仓，这篇论文主要是介绍 snowflake 整个数仓系统的架构和其中的亮点的

# 核心特点
- 纯粹的 Saas (Software-as-a-Service) 用户体验，用户无需关心使用数据库时的底层细节
- 关系型数据库，提供 ANSI SQL 和 ACID 事务
- 半结构化，内置函数和 SQL 扩展用于方便处理半结构化/schema-less 数据，支持 JSON 和 Avro 等格式
- 弹性，存储计算分离，可以分开扩容并且不影响数据可靠性和查询性能
- 高可用
- 持久化，还支持克隆， undrop ，跨区域备份等能力
- 低成本，数据会做压缩
- 安全，数据存储和传输时都做了加密

# STORAGE VERSUS COMPUTE
这里提到了传统数仓 shared-nothing 架构在以下场景里遇到的缺陷
- 异构负载：节点都是同构的，但是负载往往不是同构的。重 IO 和重计算所需要的硬件和系统配置往往不同，最终只能通过降低机器的平均水位来达到一个折中的状态
- 成员变更：节点变更往往导致数据需要重新做均衡（进行 shuffle），同时也对查询性能带来较为明显的影响
- 在线升级：在线升级需要影响所有节点，要做到上层无感知比较困难

而在云上环境，厂商根据配置特点的不同提供了多种不同类型的节点（例如存储型，计算型），最好的做法就是将数据放到合适的节点上。此外，云上节点的故障也会更频繁，而且即使是相同类型的实例，他们之间也可能较大的性能差距。因此，节点上下线可能会是个常态。还有一点比较重要的，如果能很方便地进行在线升级和扩缩容，那么不仅可以缩短迭代周期，而且也能帮助用户快速扩容来应对短暂的业务流量峰值

因此 snowflake 做了存储计算的分离，将数据放到了对象存储上，而本地的磁盘只用来存储临时数据和缓存，不需要存储全量的数据

# ARCHITECTURE
分为三层，每层之间通过 RESTful 接口交互
- Data Storage 负责使用 S3 存储表的数据和查询结果
- Virtual Warehouses ，虚拟数仓，由 vm (virtual machine) 组成的弹性集群，负责执行查询
- Cloud Services ，一系列负责管理所有虚拟数仓，查询，事务，元数据（schema， 权限，秘钥，统计信息等）的服务

## Data Storage
存储层基于 AWS S3
- 对象存储 (PUT/GET/DELETE)
- 性能不是非常稳定（延迟要比本地盘大）
- 重在高可靠，高可用

重点投入在使用本地盘做缓存，优化访问 S3 的延迟

存储格式采用 PAX 形式的列存（类似 parquet），每个文件都有 header 记录每个列数据的偏移量。 S3 支持只读取一个对象的一部分，因此可以只下载 header 和所需列的数据

元数据，例如 catalog 对象，表包含的 S3 文件，统计信息，锁，事务日志等都存在一个分布式事务 KV 数据库，属于 Cloud Service 层

## Virtual Warehouses
虚拟数仓，简称 VW ，由一系列 worker node 构成，一个 worker node 就是一个 aws EC2 实例。每个 VW 用类似衣服的码数来定义规格，从 X-Small 一直到 XX-Large ，用户只需要关心这个规格就行了。这种规格定义也方便定价，同时屏蔽掉底层基础设施的差异

### Elasticity and Isolation
一个查询只会跑在一个 VW 上， VW 之间不共享 worker node ，即 VW 之间是资源隔离的。查询时 worker node 会 spawn 一个新的进程来处理该查询

一个用户可以运行多个 VW ，而 VW 之间可以共享一套表，这样用户可以解锁更多用法，例如可以直接根据需要起一个 VW 来导入数据

### Local Caching and File Stealing
节点可以缓存文件的 header 和每个列的数据，这个缓存在 worker 的所有查询进程间共享

查询优化器会负责按照表明将文件一致性哈希到 VW 的节点上，每个节点负责缓存和查询这部分文件

查询时，还支持 file stealing ，即如果一个节点处理完任务了，会从旁边节点那里看看能否分担一部分文件查询任务

### Execution Engine
SQL 执行层也是自研的
- 列存，对 CPU 缓存， SIMD 更友好
- 向量化， pipeline 形式
- Push-based ，基于推模型，比基于 pull 的火山模型对 cache 更友好

此外，主要的算子如 join/group by/sort 都支持在内存不够时将数据放到磁盘上，而不仅仅只是做一个纯内存引擎，这样应用范围能更广些

## Cloud Services
### Query Management and Optimization
优化器采用 cascades-style 实现，支持自顶向下的基于代价优化

查询执行时 Cloud Services 会持续跟踪查询的状态，统计信息，节点故障等情况。所有的查询和统计信息都会记录下来用于后续的审计和性能分析，用户也能够监控和分析过往和当前的查询

### Concurrency Control
通过 Snapshot Isolation 实现了 ACID 事务

基于不可变性来实现的，即对文件的修改都是需要将老文件替换为包含这些修改的新文件来实现，这样可以很方便地实现版本管理和快照

### Pruning
基于 min-max 做分区裁剪，这个是比较常见的做法。但值得注意的是， snowflake 支持对半结构化的数据里的字段做裁剪

# FEATURE HIGHLIGNTS
## Pure Software-as-a-Service Experience
这个没啥好多的，产品化做的好

## Continuous Availability
### Fault Resilience
S3 提供 99.99% 的数据可用性和 99.999999999% 的持久性

Snowflake 自身的元数据存储也是跨 AZ 做复制的

不过目前 VW 是单 AZ 内的，整个 AZ 挂了之后（基本不可能）用户需要手动另外起起来

### Online Upgrade
简单来说就是滚动升级，升级过程不停机

## Semi-Structured and Schema-Less Data
这部分是很大的亮点，提供了半结构化类型 VARIANT/ARRAY/OBJECT
- ARRAY 类似数组
- OBJECT 类似 json 对象
- 底层都是 VARINT ，一种自描述的二进制序列化格式，支持 kv 形式的查找

### Post-relational Operations
支持通过 functional SQL notation 或者类似 JavaScript 的 path 语法访问非结构化里的字段

同时也支持把嵌套的字段 flatten 多行便于 SQL 处理，以及 ARRAY_AGG/OBJECT_AGG 等函数来聚合这些类型

### Columnar Storage and Processing
这里 snowflake 能够分析半结构化数据并为每个字段推断类型，并自动转成列来处理，完全不需要用户显示指定类型，十分强大

对于每个字段的 path ，会统一生成一个布隆过滤器，这样查询时可以跳过不含有该字段的文件

### Optimistic Conversion
有时转换可能导致信息丢失的情况，例如字符串可能并不是一个整型，而是一个数字编码，开头的 0 不能被忽略掉，或者一个日期字符串实际上可能是个 message

针对这种情况， snowflake 在类型不能对等地进行互相转换时，就把原来的字符串也记录到单独的一个列里

### 性能
针对弱 schema 的这一能力， snowflake 进行了查询测试，最终发现和显式指定类型的情况下性能相当，除去个别因为 bug 导致的性能差距过大外，整体增加的性能开销只有 10% 左右

## Time Travel and Cloning
多版本，文件删除后最多可以保留 90 天。同时扩展了 SQL 语法访问老版本的数据
```sql
SELECT * FROM my_table AT(TIMESTAMP =>
'Mon, 01 May 2015 16:20:00 -0700'::timestamp);
SELECT * FROM my_table AT(OFFSET => -60*5); -- 5 min ago
SELECT * FROM my_table BEFORE(STATEMENT =>
'8e5d0ca9-005e-44e6-b858-a8f5b37c5726');

SELECT new.key, new.value, old.value FROM my_table new
JOIN my_table AT(OFFSET => -86400) old -- 1 day ago
ON new.key = old.key WHERE new.value <> old.value;
```

回滚 drop
```sql
DROP DATABASE important_db; -- whoops!
UNDROP DATABASE important_db;
```

克隆
```sql
CREATE DATABASE recovered_db CLONE important_db BEFORE(
STATEMENT => ’8e5d0ca9-005e-44e6-b858-a8f5b37c5726’);
```

## Security
### Key Hierarchy
基于 AWS CloudHSM 的多级秘钥机制
- root keys
- account keys
- table keys
- file keys

每一层的 key 加密了下一层的 key

### Key Life Cycle
分为 key rotation 和 rekeying 两种机制
- key rotation ，定期产生新的 key ，此后数据用新 key 加密
- rekeying ， key 定期过期，在 key 过期前需要对该 key 加密的数据用新 key 重新加密
- 两种机制各自独立运行

其中 file key 比较特别，是直接根据表名和唯一的文件名加密得到的，这样省去记录并维护大量 file key 的开销

### End-to-End Security
简要介绍了下 AWS CloudHSM ，一种硬件加密模块(hardware security modules, HSMs)

此外，介绍了其他环节也做了加密，以及 role-based access control 等安全机制

# LESSONS LEARNED AND OUTLOOK
这里提到的一些经验教训，其中印象较深的包括
- 用户还是需要关系型数据库， Hadoop 之类的并没有替换掉 RDBMS ，反而是作为他的一种补充
- 半结构化扩展非常有用，因为用户不需要再自己对数据进行清洗处理后再导入 RDBMS 之类的系统
- 数据库开发经验丰富，避免了不少坑，包括
    - 一些关系算子实现过于简单
    - 没有早期纳入所有的数据类型到引擎中
    - 没有在早期关注资源管理
    - 没在早期就处理好复杂的日期和时间处理功能
- 用户其实没有那么关注性能，只要你伸缩性够强，能快速利用弹性资源解决偶然的需求
- 构建一个支持百万用户并发的元数据层是个有挑战性，复杂的任务

# 参考
- [https://zhuanlan.zhihu.com/p/390025973](https://zhuanlan.zhihu.com/p/390025973)
