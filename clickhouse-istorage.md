# ClickHouse IStorage 笔记

## 多引擎支持
ClickHouse 本身就支持多引擎特性

### MergeTree 
ClickHouse 的核心卖点就是其高效的 MergeTree 引擎，对应 Replicated* 的版本则支持复制。 MergeTree 引擎也是 ClickHouse 中最为复杂的引擎，很多存储的接口也是基于 MergeTree 做了些定制的
- MergeTree
- ReplacingMergeTree
- SummingMergeTree
- AggregatingMergeTree
- CollapsingMergeTree
- VersionedCollapsingMergeTree
- GraphiteMergeTree

### 日志 
具有最小功能的轻量级引擎
- TinyLog
- StripeLog
- Log

这部分引擎的实现比较简单，适合用来了解 ClickHouse 存储接口的一些最基本的函数

### 集成引擎 
与其他的数据存储与处理系统集成的引擎
- Kafka
- MySQL
- ODBC
- JDBC
- HDFS

对于集成外部系统来说，可以参考这部分的一些实现

### 特定功能
用于其他特定功能的引擎 
- Distributed
- MaterializedView
- Dictionary
- Merge
- File
- Null
- Set
- Join
- URL
- View
- Memory
- Buffer

## IStorage 接口
IStorage 是 ClickHouse 单表存储引擎的接口，具体的声明位于 `dbms/src/Storages/IStorage.h`

```c++
/** Storage. Describes the table. Responsible for
  * - storage of the table data;
  * - the definition in which files (or not in files) the data is stored;
  * - data lookups and appends;
  * - data storage structure (compression, etc.)
  * - concurrent access to data (locks, etc.)
  */
class IStorage : public std::enable_shared_from_this<IStorage>
{
public:
    IStorage() = default;
    explicit IStorage(ColumnsDescription columns_);
    IStorage(ColumnsDescription columns_, ColumnsDescription virtuals_);

    virtual ~IStorage() = default;
    IStorage(const IStorage &) = delete;
    IStorage & operator=(const IStorage &) = delete;

    /// The main name of the table type (for example, StorageMergeTree).
    virtual std::string getName() const = 0;

    /// The name of the table.
    virtual std::string getTableName() const = 0;
    virtual std::string getDatabaseName() const = 0;

    // ...snipped...

    /** Read a set of columns from the table.
      * Accepts a list of columns to read, as well as a description of the query,
      *  from which information can be extracted about how to retrieve data
      *  (indexes, locks, etc.)
      * Returns a stream with which you can read data sequentially
      *  or multiple streams for parallel data reading.
      * The `processed_stage` must be the result of getQueryProcessingStage() function.
      *
      * context contains settings for one query.
      * Usually Storage does not care about these settings, since they are used in the interpreter.
      * But, for example, for distributed query processing, the settings are passed to the remote server.
      *
      * num_streams - a recommendation, how many streams to return,
      *  if the storage can return a different number of streams.
      *
      * It is guaranteed that the structure of the table will not change over the lifetime of the returned streams (that is, there will not be ALTER, RENAME and DROP).
      */
    virtual BlockInputStreams read(
        const Names & /*column_names*/,
        const SelectQueryInfo & /*query_info*/,
        const Context & /*context*/,
        QueryProcessingStage::Enum /*processed_stage*/,
        size_t /*max_block_size*/,
        unsigned /*num_streams*/)
    {
        throw Exception("Method read is not supported by storage " + getName(), ErrorCodes::NOT_IMPLEMENTED);
    }

    /** Writes the data to a table.
      * Receives a description of the query, which can contain information about the data write method.
      * Returns an object by which you can write data sequentially.
      *
      * It is guaranteed that the table structure will not change over the lifetime of the returned streams (that is, there will not be ALTER, RENAME and DROP).
      */
    virtual BlockOutputStreamPtr write(
        const ASTPtr & /*query*/,
        const Context & /*context*/)
    {
        throw Exception("Method write is not supported by storage " + getName(), ErrorCodes::NOT_IMPLEMENTED);
    }

    // ...snipped...
};
```

这个接口声明了 ClickHouse 引擎需要实现的所有函数和功能
- 引擎支持的特性
    - 是否从 remote 获取数据
    - 是否支持 sampling
    - 是否支持去重
    - ...
- 元信息
    - 列
    - 索引
    - partition key 表达式
    - sort key 表达式
    - ...
- 参数检查
- 读取 Block
- 写入 Block
- 常用的修改操作
    - drop
    - truncate
    - alter
    - mutate
    - ...
- 启动 (startup)
- 停止 (shutdown)
- ...


其中最为核心的就是
- 读取 Block 的函数 read()
- 写入 Block 的函数 write()

## Block
ClickHouse 中数据处理的基本单位，可以看作是多行数据的集合，不过数据组织方式是列存的方式
- ClickHouse 的查询和写入都是构建在 BlockStream 之上的
- Block 包含了多列的数据以及列的元信息 (ColumnWithTypeAndName)
- Block 本身也提供了一些增/删的接口
- Block 的 DataType 也记录了如何序列化和反序列化数据

```c++
/** Container for set of columns for bunch of rows in memory.
  * This is unit of data processing.
  * Also contains metadata - data types of columns and their names
  *  (either original names from a table, or generated names during temporary calculations).
  * Allows to insert, remove columns in arbitrary position, to change order of columns.
  */
class Block
{
private:
    using Container = ColumnsWithTypeAndName;
    using IndexByName = std::map<String, size_t>;

    Container data;
    IndexByName index_by_name;

public:
    BlockInfo info;
    // ...snipped...

    /// insert the column at the specified position
    void insert(size_t position, const ColumnWithTypeAndName & elem);
    void insert(size_t position, ColumnWithTypeAndName && elem);
    /// insert the column to the end
    void insert(const ColumnWithTypeAndName & elem);
    void insert(ColumnWithTypeAndName && elem);
    // ...snipped...
    /// remove the column at the specified position
    void erase(size_t position);
    /// remove the columns at the specified positions
    void erase(const std::set<size_t> & positions);
    /// remove the column with the specified name
    void erase(const String & name);

    // ...snipped...

    const ColumnsWithTypeAndName & getColumnsWithTypeAndName() const;
    NamesAndTypesList getNamesAndTypesList() const;
    Names getNames() const;
    DataTypes getDataTypes() const;

    /// Returns number of rows from first column in block, not equal to nullptr. If no columns, returns 0.
    size_t rows() const;

    size_t columns() const { return data.size(); }

    /// Checks that every column in block is not nullptr and has same number of elements.
    void checkNumberOfRows() const;

    /// Approximate number of bytes in memory - for profiling and limits.
    size_t bytes() const;

    /// Approximate number of allocated bytes in memory - for profiling and limits.
    size_t allocatedBytes() const;
    // ...snipped...
};

using ColumnPtr = COW<IColumn>::Ptr;
using DataTypePtr = std::shared_ptr<const IDataType>;
using ColumnsWithTypeAndName = std::vector<ColumnWithTypeAndName>;

/** Column data along with its data type and name.
  * Column data could be nullptr - to represent just 'header' of column.
  * Name could be either name from a table or some temporary generated name during expression evaluation.
  */
struct ColumnWithTypeAndName
{
    ColumnPtr column;
    DataTypePtr type;
    String name;
};
```

## 自定义引擎
如果要实现自定义的引擎，并非所有引擎的接口都需要实现，可以参考一些比较简单的引擎
- StorageLog
- StorageMySQL

```c++
/** Implements simple table engine without support of indices.
  * The data is stored in a compressed form.
  */
class StorageLog : public ext::shared_ptr_helper<StorageLog>, public IStorage
{
friend class LogBlockInputStream;
friend class LogBlockOutputStream;

public:
    std::string getName() const override { return "Log"; }
    std::string getTableName() const override { return table_name; }
    std::string getDatabaseName() const override { return database_name; }

    BlockInputStreams read(
        const Names & column_names,
        const SelectQueryInfo & query_info,
        const Context & context,
        QueryProcessingStage::Enum processed_stage,
        size_t max_block_size,
        unsigned num_streams) override;

    BlockOutputStreamPtr write(const ASTPtr & query, const Context & context) override;

    void rename(const String & new_path_to_db, const String & new_database_name, const String & new_table_name) override;

    CheckResults checkData(const ASTPtr & /* query */, const Context & /* context */) override;

    void truncate(const ASTPtr &, const Context &) override;

    std::string full_path() const { return path + escapeForFileName(table_name) + '/';}

    String getDataPath() const override { return full_path(); }

    // ...snipped...
};
```

不过基本上都需要实现 BlockInputStream 和 BlockOutputStream ，对接外部数据源时，还需要看如何将数据格式转换为 Block 的格式

这里可以参考 ParquetBlockInputStream ，实现了 arrow 格式转为 Block 格式

实现好引擎之后，需要将引擎注册到 ClickHouse 中， ClickHouse 提供了类似的插件机制
```c++
void registerStorages()
{
    auto & factory = StorageFactory::instance();

    registerStorageLog(factory);
    registerStorageTinyLog(factory);
    registerStorageStripeLog(factory);
    registerStorageMergeTree(factory);
    registerStorageNull(factory);
    registerStorageMerge(factory);
    registerStorageBuffer(factory);
    registerStorageDistributed(factory);
    registerStorageMemory(factory);
    registerStorageFile(factory);
    registerStorageURL(factory);
    registerStorageDictionary(factory);
    registerStorageSet(factory);
    registerStorageJoin(factory);
    registerStorageView(factory);
    registerStorageMaterializedView(factory);

    // ...snipped...
}
```

每个引擎都会往 StorageFactory 中注册一个回调，用于在建表时创建引擎的实例，同时还支持用户传递额外的参数给引擎
```c++
void registerStorageLog(StorageFactory & factory)
{
    factory.registerStorage("Log", [](const StorageFactory::Arguments & args)
    {
        if (!args.engine_args.empty())
            throw Exception(
                "Engine " + args.engine_name + " doesn't support any arguments (" + toString(args.engine_args.size()) + " given)",
                ErrorCodes::NUMBER_OF_ARGUMENTS_DOESNT_MATCH);

        return StorageLog::create(
            args.data_path, args.database_name, args.table_name, args.columns,
            args.context.getSettings().max_compress_block_size);
    });
}
```

## 参考
- [表引擎](https://clickhouse.tech/docs/zh/engines/table-engines/)

