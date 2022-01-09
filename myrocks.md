# 论文阅读：《MyRocks: LSM-Tree Database Storage Engine Serving Facebook's Social Graph》

主要讲了 Facebook 是如何用 MyRocks 替换 InnoDB 引擎的，以及这个过程中的实践经验和心得

## INTRODUCTION
> When we started the project, our goal was to reduce the number of UDB servers by 50%. That required the MyRocks space usage to be no more than 50% of the compressed InnoDB format, while maintaining comparable CPU and I/O utilization

替换的出发点是希望通过使用基于 LSM-Tree 的引擎来减少基于 B+Tree 的引擎 InnoDB 带来的空间放大和写放大问题，换句话说就是降成本。项目启动时，定下了降低 50% 成本的目标

> MyRocks also reduced the instance size by 62.3% for UDB data sets and performed fewer I/O operations than InnoDB. Finally, MyRocks consumed less CPU time for serving the same production traffic workload.

从结果来看最后确实达到了目标

而这篇文章提到了使用 MyRocks 去替换 InnoDB 的过程中，面临的诸多挑战
- 成本减半意味着单个 db 实例需要承担 2 倍的读写压力
- LSM-tree 的逆序 scan 性能比顺序 scan 差不少
- LSM-tree 需要更频繁的 key 比较
- MyRocks 范围查询性能要比 InnoDB 要差
- LSM-tree 的查询性能提升很依赖各种 cache ，这也会带来相应的内存占用和内存压力
- 频繁删除/更新对性能的影响
- 大量写入对合并造成压力，可能会导致 write stall

而 MyRocks/RocksDB 也针对这些情况做了很多针对性的优化或者改造

## BACKGROUND AND MOTIVATION
介绍了下 UDB (User Database) 的一些背景以及为何要改用 LSM-tree 。UDB 基本上就是个 cache + db 的结构， db 也是做了主从复制的。一开始 db 也是纯 HDD 的，后来逐渐用 SSD 作为 cache ，最后发展成纯 SSD 。到了纯 SSD 时代后， I/O 吞吐不再是瓶颈，但是每 GB 的综合成本也随之增加。要降低成本就需要降低空间占用，而尽管 InnoDB 支持压缩，并且压缩开启后能降低近一半的存储空间，但还是不够。主要原因在于 B+Tree 存储存在索引碎片化的问题，每个块都会存在 25% 到 30% 的空间浪费。

InnoDB 本身的压缩也存在局限，块大小是固定的，例如配置 8K 一个块，即使数据被压缩成了 5K ，但是还是会占用 8K 的空间。除了空间放大，写放大的问题也存在。在 UDB 场景下，由于有了前端的缓存， MySQL 中的 working set 大多数不在缓存里，即使是一行的更新，最后也需要把整个脏页回刷磁盘，再加上 InnoDB 的 double-write 特性，就使得写放大非常严重。因为这些原因， Facebook 开始尝试使用 LSM-tree 的引擎代替 InnoDB

而 RocksDB 在空间放大和写放大上有一定优势。首先写放大方面
- 由于 LSM 无需原地更新 page
- 写入和更新现在内存里积攒， Flush 的数据只包含新写入/更新的数据

而空间放大方面
- LSM-tree 在存储上更紧凑，空间的浪费主要在于被删除的数据占用的空间
- 通过对合并做调优，能做到被删除的数据占比低于 10% ，这样浪费掉的空间就少了很多
- LSM-tree 更容易做合并
- InnoDB 每行额外需要 13 bytes (6-byte transaction id and 7-byte roll pointer) 来实现事务，而 RocksDB 则占用 7-byte sequence number for each row
    - 同时 RocksDB 在合并后，对于没有事务引用的 row ，就可以把 sequence number 转为 0 ，节省存储空间，而 Lmax 层的数据基本上 sequence number 都是 0

## MYROCKS/ROCKSDB DEVELOPMENT
这里提到了对 MyRocks/RocksDB 的很多优化，这些优化也都是从实际应用场景出发的

### Mem-comparable Keys
LSM-tree 查找时需要进行更多的 key 比较
- B-tree 只需要做一次二分查找
- LSM-tree 需要在每层上做查找，并通过堆合并每层的查找结果
- 另外 B-tree 前进不需要比较，而 LSM-tree 需要至少一次比较来调整堆（从堆里拿到下一个 top key），还往往需要检查 key 的版本

为此， MyRocks 都是使用 mem-comparable 的格式来保证比较尽量高效

### Reverse Key Comparator
RocksDB 对逆序 scan 不友好
- key 是 delta 压缩的
- 记录是按照版本逆序排序的，逆序也需要先跳过这些数据
- memtable 的 skiplist 实现对逆序不友好，每次逆序迭代实际上都需要一次二分查找（重新找到比当前 key 小的 key），复杂度 O(logN)

为此 MyRocks 增对需要逆序扫描的场景，用逆序的 key comparator ，反过来存储 key (当然，这时顺序扫就会变慢)

### Faster Approximate Size to Scan Calculation
主要针对优化器代价估计，高效计算扫描一个 key range 的开销，这些计算本身也会带来开销

而优化有两块
- 一块是上层业务在 sql 里直接加 hint 指定走索引，跳过这个代价计算，很暴力，但有效
- 直接估算 key range 范围内的 sst 文件总大小

### Prefix Bloom Filter
在 B-tree 中，小范围的 Range 查询只需要读取一两个叶子节点就完成了，而 LSM-tree 则需要通过 seek + next 实现，特别是 seek 的开销是比较大的，需要在每层上查找，即使该层上可能根本没这个 key 。为了优化这个问题， RocksDB 引入了 prefix bloom filter ，用来加速判断 key 前缀在不在 sst 中。因为 range scan 实际上就是根据前缀做 scan

### Reducing Tombstone on Deletes and Updates
这个也是针对实际场景的一个比较定制的优化。对于索引列的更新， MyRocks 是通过 delete + put 来实现的，而 delete 在 LSM-tree 中是通过写入一个 delete tombstone 来实现的。一个 delete tombstone 需要到达 Lmax 层后才能删除，因为中间层可能也存在这个 key ，如果提前把 tombstone 删除了，就会出现被删掉的 key 又出现的情况。对于频繁更新的索引， delete 越来越多， tombstone 越来越多，就会严重影响读性能。

为此 RocksDB 提供了一个 single delete 的 api ，在删除匹配的 put 后就会直接删除这个类型的 tombstone 。因为 MyRocks 本身保证了二级索引不会有重复的 put 出现，所以能够用到这个优化

### Triggering Compaction based on Tombstones
Deletion Triggered Compaction (DTC) ，当大量密集的 tombstone 出现时，会触发合并，目的也是减少 tombstone 的数量，提高 scan 性能

### DRAM Usage Regression
由于 bloom filter 本身也会占用空间，因此 MyRocks 做了取舍， Lmax 不产生 bloom filter

### SSD Slowness Because of Compaction
对合并的 io 请求增加了流控，减少对用户查询的 io 请求影响

### Physically Removing Stale Data
周期性地合并老 sst 直到合并到 Lmax ，保证对于老数据 (Lmax) 的 delete/put 操作最终能到达 Lmax ，减少过期数据占用的空间。没有这一机制，很多 delete/put 操作可能到达不了 Lmax ，造成空间浪费

### Bulk Loading
直接生成 sst 并导入，提供数据导入性能，这个需要数据和已有的数据的 key range 不重叠。在 MyRocks 场景下，主要是导入全新的表，所以十分适合使用这个特性

## Extra Benefits of Using MyRocks
除此之外使用 MyRocks 也带来了预料之外的好处，包括
- 数据备份恢复更方便
    - RocksDB 有 checkpoint 机制
- 对有大量二级索引场景更友好
    - 对于有二级索引的表写入吞吐更好
    - 不需要 Get/GetForUpdate （我理解写入/更新时，为了能用 single delete ，应该还需需要 get 的）
- 替换/插入时可以不用把数据读出来，不过对于插入 MyRocks 没有默认支持跳过数据 unique key constraints 检查，因为容易在使用触发器/复制功能时碰到数据一致性问题
- 更多的压缩机会，不同层可以用不同的压缩算法
    - 非 Lmax 数据用 Lz4 压缩，
    - 90% 数据在 Lmax ，用 Zstd 压缩，压缩率更高

## PRODUCTION MIGRATION
讲述了从 InnoDB 迁移到 MyRocks 的过程。整个迁移过程持续了较长时间，期间也非常慎重。包括
- 专门搞了个 MyShadow 来 mirror 查询，做查询测试
- 专门的工具持续地检查数据正确性
- 修复不一致的查询行为，前面的实际生产流量，能验证出很多单元测试或者 sysbench 和 LinkBench 这些工具测不出来的查询兼容性问题
- 实际的迁移类似逐步灰度替换的过程
    - 先将 MyRocks 作为副本提供服务，而 InnoDB 仍然作为主副本
    - 逐步验证并将 MyRocks 提升作为主副本
    - 保留 InnoDB 用于回滚
    - 逐步去掉 InnoDB

## RESULTS
从实际数据来看， MyRocks 实例占用的磁盘空间是开启压缩后的 InnoDB 实例的 37.7% ，可见空间占用上确实有不少优势。另外，对于没有读流量的场景下， MyRocks 比 InnoDB 节省 40% 的 CPU 资源。当然在读写混合的场景下，两者就差不多了。不过由于 InnoDB 的实例空间占用更小，单机可以承载更多的 mysql 实例，提高单机利用率

除此之外， Fackbook 还尝试了将 Facebook Messenger 的 backend 从 HBase 切换到 MySQL + MyRocks ，因为 MyRocks 对 SSD 更友好

## LESSONS LEARNED
这里提到了一些经验心得，列举一些我的理解
- 理解 workload 很重要
- 理解底层如何工作的很重要
- 要关注 outlier ，如果只关注 p90 或者 p99 往往发现不了一些问题
- SQL 很好地解决了协议兼容的问题， SQL 大法好
- RocksDB 可调参数太多了，要调好参不容易，这也是个优化方向
- 解决 Memory Stalls ，提高内存利用率
    - RocksDB 写数据时需要为每个 block 申请内存，大量依赖内存分配，同时早期 RocksDB 也是直接使用 buffered I/O ，给 page cache 分配带来不少压力，于是将内核从 4.0 升级到 4.6
    - 内核对于管理大文件的内存开销比较大，每 1TB 的 RocksDB SST 文件就需要申请 2~3GB 的 slab memory
        - Fackbook 内核组用 radix tree 管理大文件，减少内存开销
        - 考虑到开源用户， RocksDB 也支持了 direct I/O ，使得平均 slab 大小减少了 80% ，不过这需要保证同一文件不能混用 buffered I/O 和 direct I/O

## 小结
这篇文章包含了相当多开发，使用和优化 RocksDB/MyRocks 过程中的经验，介绍了非常多存储系统在生产下才会碰到的问题，很好地阐述了什么叫理论结合实践
