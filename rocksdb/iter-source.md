# NewIterator 分析

RocksDB 迭代时，需要从 memtable 、 immutable memtables 、以及 sst 中获取数据并汇总成最终的全局视图，过程相比 get 要更为麻烦

## DBImpl::NewIterator()
创建迭代器的入口在 `DBImpl::NewIterator()` ，默认情况下会调用 `DBImpl::NewIteratorImpl()` 来创建迭代器
```c++
Iterator* DBImpl::NewIterator(const ReadOptions& read_options,
                              ColumnFamilyHandle* column_family) {
  // ...
  auto cfh = reinterpret_cast<ColumnFamilyHandleImpl*>(column_family);
  auto cfd = cfh->cfd();
  ReadCallback* read_callback = nullptr;  // No read callback provided.
if (read_options.tailing) {
    // ...
  } else {
    // Note: no need to consider the special case of
    // last_seq_same_as_publish_seq_==false since NewIterator is overridden in
    // WritePreparedTxnDB
    auto snapshot = read_options.snapshot != nullptr
                        ? read_options.snapshot->GetSequenceNumber()
                        : versions_->LastSequence();
    // 调用 DBImpl::NewIteratorImpl() 创建迭代器
    result = NewIteratorImpl(read_options, cfd, snapshot, read_callback);
  }
  // 返回迭代器
  return result;
}
```

## DBImpl::NewIteratorImpl()
RocksDB 的迭代器是个包含了若干子迭代器的树结构，为了提高这个结构的 cache 友好性， RocksDB 在创建迭代器树时会传入一个段连续内存 `Arena` ，整个迭代器树都从这个 `Arena` 中申请内存，这样迭代器树就相当于分配在一个连续的内存上
```c++
ArenaWrappedDBIter* DBImpl::NewIteratorImpl(const ReadOptions& read_options,
                                            ColumnFamilyData* cfd,
                                            SequenceNumber snapshot,
                                            ReadCallback* read_callback,
                                            bool allow_blob,
                                            bool allow_refresh) {
  SuperVersion* sv = cfd->GetReferencedSuperVersion(&mutex_);

  ArenaWrappedDBIter* db_iter = NewArenaWrappedDbIterator(
      env_, read_options, *cfd->ioptions(), sv->mutable_cf_options, snapshot,
      sv->mutable_cf_options.max_sequential_skip_in_iterations,
      sv->version_number, read_callback, this, cfd, allow_blob,
      ((read_options.snapshot != nullptr) ? false : allow_refresh));

  InternalIterator* internal_iter =
      NewInternalIterator(read_options, cfd, sv, db_iter->GetArena(),
                          db_iter->GetRangeDelAggregator());
  db_iter->SetIterUnderDBIter(internal_iter);

  return db_iter;
}
```

可以看到，首先会调用 `NewArenaWrappedDbIterator()` 创建一个 `ArenaWrappedDBIter` ，然后会调用 `DBImpl::NewInternalIterator()` 创建一个 `InternalIterator` ，最后将 `InternalIterator` 设置到 `ArenaWrappedDBIter` 中

`ArenaWrappedDBIter` 实际上是对 `DBIter` 的一个封装，已提供连续内存 `Arena` 并从 `Arena` 中分配 `DBIter` 的内存分配

`DBIter` 则是对 `InternalIterator` 的封装，因此创建好 `InternalIterator` 还会将其设置到 `DBIter` 中

在 RocksDB 中， memtable 、 sst 等内部用的迭代器的 key 都是包含了 sequence number 的，因为这些数据都是带版本的，并且删除也是通过插入一条新的数据来实现的。但是用户并不关心这些，用户看到的只是 user key ，因此在返回给用户之前，还需要封装上一层 `DBIter` ，由 `DBIter` 来处理数据的版本、删除等问题，最终将 user key 相同、版本不同的数据汇总为用户需要看到的最新版本的数据，使得用户看到 user key 对应的只有一个值

## DBImpl::NewInternalIterator()
`InternalIterator` 是 RocksDB 内部迭代器的接口。 `DBImpl::NewInternalIterator()` 会创建一个汇总了 memtable 、 immutable memtables 以及 sst 文件的数据的迭代器，该迭代器会实现 `InternalIterator` 接口，并且迭代器的内存是从传入的 `Arena` 中分配的
```c++
InternalIterator* DBImpl::NewInternalIterator(
    const ReadOptions& read_options, ColumnFamilyData* cfd,
    SuperVersion* super_version, Arena* arena,
    RangeDelAggregator* range_del_agg) {
  InternalIterator* internal_iter;
  assert(arena != nullptr);
  assert(range_del_agg != nullptr);
  // Need to create internal iterator from the arena.
  // 创建 MergeIteratorBuilder 用于合并迭代器结果
  MergeIteratorBuilder merge_iter_builder(
      &cfd->internal_comparator(), arena,
      !read_options.total_order_seek &&
          super_version->mutable_cf_options.prefix_extractor != nullptr);
  // Collect iterator for mutable mem
  // 通过 SuperVersion::mem 的 MemTable::NewIterator() 创建 mutable memtable 的迭代器
  merge_iter_builder.AddIterator(
      super_version->mem->NewIterator(read_options, arena));
  std::unique_ptr<InternalIterator> range_del_iter;
  Status s;
  // ...
  // Collect all needed child iterators for immutable memtables
  if (s.ok()) {
    // 调用 SuperVersion::imm MemTableListVersion::AddIterators() 创建 immutable memtables
    // 的迭代器
    super_version->imm->AddIterators(read_options, &merge_iter_builder);
    // ...
  }
  TEST_SYNC_POINT_CALLBACK("DBImpl::NewInternalIterator:StatusCallback", &s);
  if (s.ok()) {
    // Collect iterators for files in L0 - Ln
    if (read_options.read_tier != kMemtableTier) {
      // 调用 SuperVersion::current Version::AddIterators() 获取 sst 文件迭代器
      super_version->current->AddIterators(read_options, env_options_,
                                           &merge_iter_builder, range_del_agg);
    }
    // 调用 MergeIteratorBuilder::Finish() 创建 InternalIterator
    internal_iter = merge_iter_builder.Finish();
    IterState* cleanup =
        new IterState(this, &mutex_, super_version,
                      read_options.background_purge_on_iterator_cleanup);
    internal_iter->RegisterCleanup(CleanupIteratorState, cleanup, nullptr);

    // 返回迭代器
    return internal_iter;
  } else {
    CleanupSuperVersion(super_version);
  }
  return NewErrorInternalIterator<Slice>(s, arena);
}
```

`DBImpl::NewInternalIterator()` 的主要流程为
* 创建 `MergeIteratorBuilder` 用于合并迭代器
* 调用 `MemTable::NewIterator()` 创建 memtable 的迭代器并加入到 `MergeIteratorBuilder` 中
* 调用 `MemTableListVersion::AddIterators()` 创建 immutable memtables 的迭代器并加入到 `MergeIteratorBuilder` 中
* 调用 `Version::AddIterators()` 创建 sst 的迭代器并加入到 `MergeIteratorBuilder`
* 从 `MergeIteratorBuilder` 中获取合并后的迭代器 `MergingIterator` ，这个迭代器会汇总前面添加的所有迭代器的数据

## MemTable::NewIterator()
该函数用于创建 memtable 的迭代器
```c++
InternalIterator* MemTable::NewIterator(const ReadOptions& read_options,
                                        Arena* arena) {
  assert(arena != nullptr);
  auto mem = arena->AllocateAligned(sizeof(MemTableIterator));
  return new (mem) MemTableIterator(*this, read_options, arena);
}
```
实际上就是在内存上创建了一个 `MemTableIterator` 对象

在 `MemTableIterator` 的构造函数中，会调用 `MemTableRep::GetIterator()` 来创建具体 memtable 实现的迭代器
```c++
class MemTableIterator : public InternalIterator {
 public:
  MemTableIterator(const MemTable& mem, const ReadOptions& read_options,
                   Arena* arena, bool use_range_del_table = false)
      : bloom_(nullptr),
        prefix_extractor_(mem.prefix_extractor_),
        comparator_(mem.comparator_),
        valid_(false),
        arena_mode_(arena != nullptr),
        value_pinned_(
            !mem.GetImmutableMemTableOptions()->inplace_update_support) {
    if (use_range_del_table) {
      iter_ = mem.range_del_table_->GetIterator(arena);
    } else if (prefix_extractor_ != nullptr && !read_options.total_order_seek) {
      bloom_ = mem.prefix_bloom_.get();
      iter_ = mem.table_->GetDynamicPrefixIterator(arena);
    } else {
      iter_ = mem.table_->GetIterator(arena);
    }
  }
  // ...
};
```

## MemTableListVersion::AddIterators()
这个函数的实现很简单，就是遍历所有 immutable memtables 、创建迭代器并添加到 `MergeIteratorBuilder` 中
```c++
void MemTableListVersion::AddIterators(
    const ReadOptions& options, MergeIteratorBuilder* merge_iter_builder) {
  for (auto& m : memlist_) {
    merge_iter_builder->AddIterator(
        m->NewIterator(options, merge_iter_builder->GetArena()));
  }
}
```

## Version::AddIterators()
该函数用于从创建 sst 的迭代器，具体的实现就是遍历所有非空 level 并调用 `Version::AddIteratorsForLevel()`
```c++
void Version::AddIterators(const ReadOptions& read_options,
                           const EnvOptions& soptions,
                           MergeIteratorBuilder* merge_iter_builder,
                           RangeDelAggregator* range_del_agg) {
  assert(storage_info_.finalized_);

  for (int level = 0; level < storage_info_.num_non_empty_levels(); level++) {
    AddIteratorsForLevel(read_options, soptions, merge_iter_builder, level,
                         range_del_agg);
  }
}
```

### Version::AddIteratorsForLevel()
由于 level 0 的 sst 比较特殊，可能存在 key 范围重叠的情况，因此创建迭代器的时候，需要区别对待 level 0 的文件和其他 level 的文件
* 对于 level 0 的 sst ，需要遍历所有文件，调用 `TableCache::NewIterator()` 创建每个文件的迭代器并加入到 `MergeIteratorBuilder` 中
* 对于其他 level 的 sst ，一个 level 只需要创建一个 `LevelIterator` 并加入到 `MergeIteratorBuilder` 中即可
```c++
void Version::AddIteratorsForLevel(const ReadOptions& read_options,
                                   const EnvOptions& soptions,
                                   MergeIteratorBuilder* merge_iter_builder,
                                   int level,
                                   RangeDelAggregator* range_del_agg) {
  // ...
  auto* arena = merge_iter_builder->GetArena();
  if (level == 0) {
    // Merge all level zero files together since they may overlap
    // 如果 level 为 0 ， merge 所有 level 0 文件
    for (size_t i = 0; i < storage_info_.LevelFilesBrief(0).num_files; i++) {
      // 遍历 level 0 的所有文件
      const auto& file = storage_info_.LevelFilesBrief(0).files[i];
      merge_iter_builder->AddIterator(cfd_->table_cache()->NewIterator(
          read_options, soptions, cfd_->internal_comparator(), *file.file_metadata,
          range_del_agg, mutable_cf_options_.prefix_extractor.get(), nullptr,
          cfd_->internal_stats()->GetFileReadHist(0), false, arena,
          false /* skip_filters */, 0 /* level */));
    }
    // ...
  } else if (storage_info_.LevelFilesBrief(level).num_files > 0) {
    // For levels > 0, we can use a concatenating iterator that sequentially
    // walks through the non-overlapping files in the level, opening them
    // lazily.
    auto* mem = arena->AllocateAligned(sizeof(LevelIterator));
    merge_iter_builder->AddIterator(new (mem) LevelIterator(
        cfd_->table_cache(), read_options, soptions,
        cfd_->internal_comparator(), &storage_info_.LevelFilesBrief(level),
        mutable_cf_options_.prefix_extractor.get(), should_sample_file_read(),
        cfd_->internal_stats()->GetFileReadHist(level),
        false /* for_compaction */, IsFilterSkipped(level), level,
        range_del_agg));
  }
}
```

从这里我们也可以看到，应该避免 level 0 有太多的文件，否则迭代的时候就需要创建和合并很多的迭代器

### LevelIterator
对于非 level 0 的 sst ，它们的范围是不重叠的，因此不需要像 level 0 的 sst 那样一个文件一个迭代器，而是创建一个用于整个 level 的迭代器 `LevelIterator` 即可

`LevelIterator` 在进行 `Seek()` 等操作时，会先根据 key 定位到具体的 sst ，然后调用 `LevelIterator::InitFileIterator()` 创建该 sst 的迭代器
```c++
void LevelIterator::Seek(const Slice& target) {
  size_t new_file_index = FindFile(icomparator_, *flevel_, target);

  InitFileIterator(new_file_index);
  if (file_iter_.iter() != nullptr) {
    file_iter_.Seek(target);
  }
  SkipEmptyFileForward();
}
```

`LevelIterator::InitFileIterator()` 会调用 `LevelIterator::NewFileIterator()` 创建迭代器
```c++
void LevelIterator::InitFileIterator(size_t new_file_index) {
  if (new_file_index >= flevel_->num_files) {
    file_index_ = new_file_index;
    SetFileIterator(nullptr);
    return;
  } else {
    // If the file iterator shows incomplete, we try it again if users seek
    // to the same file, as this time we may go to a different data block
    // which is cached in block cache.
    //
    if (file_iter_.iter() != nullptr && !file_iter_.status().IsIncomplete() &&
        new_file_index == file_index_) {
      // file_iter_ is already constructed with this iterator, so
      // no need to change anything
    } else {
      file_index_ = new_file_index;
      InternalIterator* iter = NewFileIterator();
      SetFileIterator(iter);
    }
  }
}
```

`LevelIterator::NewFileIterator()` 也是通过调用 `TableCache::NewIterator()` 来创建 sst 的迭代器的
```c++
InternalIterator* NewFileIterator() {
  assert(file_index_ < flevel_->num_files);
  auto file_meta = flevel_->files[file_index_];
  if (should_sample_) {
    sample_file_read_inc(file_meta.file_metadata);
  }

  return table_cache_->NewIterator(
      read_options_, env_options_, icomparator_, *file_meta.file_metadata,
      range_del_agg_, prefix_extractor_,
      nullptr /* don't need reference to table */,
      file_read_hist_, for_compaction_, nullptr /* arena */, skip_filters_,
      level_);
}
```

可以看到，不管是哪个 level ，创建迭代器的入口都是 `TableCache::NewIterator()`

## TableCache::NewIterator()
`TableCache` 在 get 的时候我们就了解过，它主要用于缓存 `TableReader`

首先，会先获取 `TableReader` ，某些情况下，会直接创建新的 `TableReader`
* 如果是用于 compaction 的迭代器且 `new_table_reader_for_compaction_inputs` 选项为 true ，则创建新的 `TableReader` ，该选项默认是关闭的
* 如果 `ReadOptions::readahead_size` 选项设置了一个大于 0 的值，也会创建新的 `TableReader` ，这个选项主要是用于提升机械硬盘上的迭代性能，默认是关闭的
* 其他情况则会调用 `TableCache::FindTable()` 创建 `TableReader`
```c++
InternalIterator* TableCache::NewIterator(
    const ReadOptions& options, const EnvOptions& env_options,
    const InternalKeyComparator& icomparator, const FileMetaData& file_meta,
    RangeDelAggregator* range_del_agg, const SliceTransform* prefix_extractor,
    TableReader** table_reader_ptr, HistogramImpl* file_read_hist,
    bool for_compaction, Arena* arena, bool skip_filters, int level) {
  PERF_TIMER_GUARD(new_table_iterator_nanos);

  Status s;
  bool create_new_table_reader = false;
  TableReader* table_reader = nullptr;
  Cache::Handle* handle = nullptr;
  if (table_reader_ptr != nullptr) {
    *table_reader_ptr = nullptr;
  }
  size_t readahead = 0;
  if (for_compaction) {
    // ...
    if (ioptions_.new_table_reader_for_compaction_inputs) {
      // get compaction_readahead_size from env_options allows us to set the
      // value dynamically
      readahead = env_options.compaction_readahead_size;
      create_new_table_reader = true;
    }
  } else {
    readahead = options.readahead_size;
    create_new_table_reader = readahead > 0;
  }

  auto& fd = file_meta.fd;
  if (create_new_table_reader) {
    unique_ptr<TableReader> table_reader_unique_ptr;
    // 调用 TableCache::GetTableReader() 创建 TableReader
    s = GetTableReader(
        env_options, icomparator, fd, true /* sequential_mode */, readahead,
        !for_compaction /* record stats */, nullptr, &table_reader_unique_ptr,
        prefix_extractor, false /* skip_filters */, level,
        true /* prefetch_index_and_filter_in_cache */, for_compaction);
    if (s.ok()) {
      table_reader = table_reader_unique_ptr.release();
    }
  } else {
    table_reader = fd.table_reader;
    if (table_reader == nullptr) {
      // 调用 TableCache::FindTable() 获取 TableReader
      s = FindTable(env_options, icomparator, fd, &handle, prefix_extractor,
                    options.read_tier == kBlockCacheTier /* no_io */,
                    !for_compaction /* record read_stats */, file_read_hist,
                    skip_filters, level);
      if (s.ok()) {
        table_reader = GetTableReaderFromHandle(handle);
      }
    }
  }
  // ...
}
```

随后，会调用 `TableReader::NewIterator()` 来创建迭代器。 `TableReader` 是一个接口，而默认的 sst 实现是 `BlockBasedTable` ，因此实际上调用的是 `BlockBasedTable::NewIterator()`
```c++
InternalIterator* TableCache::NewIterator(
    const ReadOptions& options, const EnvOptions& env_options,
    const InternalKeyComparator& icomparator, const FileMetaData& file_meta,
    RangeDelAggregator* range_del_agg, const SliceTransform* prefix_extractor,
    TableReader** table_reader_ptr, HistogramImpl* file_read_hist,
    bool for_compaction, Arena* arena, bool skip_filters, int level) {
  // ...
  InternalIterator* result = nullptr;
  if (s.ok()) {
    if (options.table_filter &&
        !options.table_filter(*table_reader->GetTableProperties())) {
      // sst 被过滤掉了
      result = NewEmptyInternalIterator<Slice>(arena);
    } else {
      // 默认为 BlockBasedTable::NewIterator()
      result = table_reader->NewIterator(options, prefix_extractor, arena,
                                         skip_filters, for_compaction);
    }
    if (create_new_table_reader) {
      assert(handle == nullptr);
      result->RegisterCleanup(&DeleteTableReader, table_reader, nullptr);
    } else if (handle != nullptr) {
      result->RegisterCleanup(&UnrefEntry, cache_, handle);
      handle = nullptr;  // prevent from releasing below
    }

    if (for_compaction) {
      table_reader->SetupForCompaction();
    }
    if (table_reader_ptr != nullptr) {
      *table_reader_ptr = table_reader;
    }
  }
  // ...
}
```

## BlockBasedTable::NewIterator()
该函数主要是调用 `BlockBasedTable::NewIndexIterator()` 创建一个 `InternalIteratorBase<BlockHandle>` ，然后构造一个 `BlockBasedTableIterator<DataBlockIter>` 并返回
```c++
InternalIterator* BlockBasedTable::NewIterator(
    const ReadOptions& read_options, const SliceTransform* prefix_extractor,
    Arena* arena, bool skip_filters, bool for_compaction) {
  bool need_upper_bound_check =
      PrefixExtractorChanged(rep_->table_properties.get(), prefix_extractor);
  const bool kIsNotIndex = false;
  if (arena == nullptr) {
    return new BlockBasedTableIterator<DataBlockIter>(
        this, read_options, rep_->internal_comparator,
        NewIndexIterator(
            read_options,
            need_upper_bound_check &&
                rep_->index_type == BlockBasedTableOptions::kHashSearch),
        !skip_filters && !read_options.total_order_seek &&
            prefix_extractor != nullptr,
        need_upper_bound_check, prefix_extractor, kIsNotIndex,
        true /*key_includes_seq*/, for_compaction);
  } else {
    auto* mem =
        arena->AllocateAligned(sizeof(BlockBasedTableIterator<DataBlockIter>));
    return new (mem) BlockBasedTableIterator<DataBlockIter>(
        this, read_options, rep_->internal_comparator,
        NewIndexIterator(read_options, need_upper_bound_check),
        !skip_filters && !read_options.total_order_seek &&
            prefix_extractor != nullptr,
        need_upper_bound_check, prefix_extractor, kIsNotIndex,
        true /*key_includes_seq*/, for_compaction);
  }
}
```

`BlockBasedTableIterator<DataBlockIter>` 内部有两个迭代器，一个是用于读取 index block 的 `InternalIteratorBase<BlockHandle>` ，由 `BlockBasedTable::NewIndexIterator()` 创建，另一个是用于读取 data block 的 `DataBlockIter` ，在通过 index block 定位到具体的 data block 后通过调用 `BlockBasedTable::NewDataBlockIterator()` 创建

### BlockBasedTable::NewIndexIterator()
该函数负责调用 `IndexReader::NewIterator()` 来创建 index block 的迭代器

首先会检查 `BlockBasedTable::rep_` 中是否有已经创建好的迭代器，有的话就可以使用已有的迭代器
```c++
InternalIteratorBase<BlockHandle>* BlockBasedTable::NewIndexIterator(
    const ReadOptions& read_options, bool disable_prefix_seek,
    IndexBlockIter* input_iter, CachableEntry<IndexReader>* index_entry,
    GetContext* get_context) {
  // index reader has already been pre-populated.
  if (rep_->index_reader) {
    return rep_->index_reader->NewIterator(
        input_iter, read_options.total_order_seek || disable_prefix_seek,
        read_options.fill_cache);
  }
  // we have a pinned index block
  if (rep_->index_entry.IsSet()) {
    // 返回 BlockBasedTable::rep_ Rep::index_entry TableReader::NewIterator()
    return rep_->index_entry.value->NewIterator(
        input_iter, read_options.total_order_seek || disable_prefix_seek,
        read_options.fill_cache);
  }
  // ...
}
```

否则的话，就会尝试从 block cache 中查找 `IndexReader`
* 首先构造 cache key
* 从 `BlockBasedTableOptions::block_cache` 中根据 cache key 查找 `IndexReader`
* 如果没找到，会调用 `BlockBasedTable::CreateIndexReader()` 创建 `IndexReader` ，然后将它插入 block cache
* 最后同样会调用 `IndexReader::NewIterator()` 创建迭代器

这部分的代码如下
```c++
InternalIteratorBase<BlockHandle>* BlockBasedTable::NewIndexIterator(
    const ReadOptions& read_options, bool disable_prefix_seek,
    IndexBlockIter* input_iter, CachableEntry<IndexReader>* index_entry,
    GetContext* get_context) {
  // ...
  const bool no_io = read_options.read_tier == kBlockCacheTier;
  Cache* block_cache = rep_->table_options.block_cache.get();
  char cache_key[kMaxCacheKeyPrefixSize + kMaxVarint64Length];
  auto key =
      GetCacheKeyFromOffset(rep_->cache_key_prefix, rep_->cache_key_prefix_size,
                            rep_->dummy_index_reader_offset, cache_key);
  Statistics* statistics = rep_->ioptions.statistics;
  auto cache_handle = GetEntryFromCache(
      block_cache, key, BLOCK_CACHE_INDEX_MISS, BLOCK_CACHE_INDEX_HIT,
      get_context ? &get_context->get_context_stats_.num_cache_index_miss
                  : nullptr,
      get_context ? &get_context->get_context_stats_.num_cache_index_hit
                  : nullptr,
      statistics, get_context);

  // ...
  IndexReader* index_reader = nullptr;
  if (cache_handle != nullptr) {
    // 从 block_cache 中根据 cache_handle 获取 index_reader
    index_reader =
        reinterpret_cast<IndexReader*>(block_cache->Value(cache_handle));
  } else {
    // Create index reader and put it in the cache.
    // 创建 index reader 并插入 cache
    Status s;
    // ...
    s = CreateIndexReader(nullptr /* prefetch_buffer */, &index_reader);
    // ...
    size_t charge = 0;
    if (s.ok()) {
      assert(index_reader != nullptr);
      charge = index_reader->ApproximateMemoryUsage();
      s = block_cache->Insert(
          key, index_reader, charge, &DeleteCachedIndexEntry, &cache_handle,
          rep_->table_options.cache_index_and_filter_blocks_with_high_priority
              ? Cache::Priority::HIGH
              : Cache::Priority::LOW);
    }
    // ...
  }

  assert(cache_handle);
  auto* iter = index_reader->NewIterator(
      input_iter, read_options.total_order_seek || disable_prefix_seek);

  // ...
  return iter;
}
```

### BlockBasedTable::CreateIndexReader()
`IndexReader` 也是一个接口，具体可以有很多种实现。 `BlockBasedTable::CreateIndexReader()` 创建 `IndexReader` 的时候会根据 `BlockBasedTable::UpdateIndexType()` 返回的类型来创建具体的 `IndexReader`
```c++
Status BlockBasedTable::CreateIndexReader(
    FilePrefetchBuffer* prefetch_buffer, IndexReader** index_reader,
    InternalIterator* preloaded_meta_index_iter, int level) {
  auto index_type_on_file = UpdateIndexType();

  auto file = rep_->file.get();
  const InternalKeyComparator* icomparator = &rep_->internal_comparator;
  const Footer& footer = rep_->footer;

  // kHashSearch requires non-empty prefix_extractor but bypass checking
  // prefix_extractor here since we have no access to MutableCFOptions.
  // Add need_upper_bound_check flag in  BlockBasedTable::NewIndexIterator.
  // If prefix_extractor does not match prefix_extractor_name from table
  // properties, turn off Hash Index by setting total_order_seek to true

  switch (index_type_on_file) {
    case BlockBasedTableOptions::kTwoLevelIndexSearch: {
      return PartitionIndexReader::Create(
          this, file, prefetch_buffer, footer, footer.index_handle(),
          rep_->ioptions, icomparator, index_reader,
          rep_->persistent_cache_options, level,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_key_is_user_key == 0,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_value_is_delta_encoded == 0);
    }
    case BlockBasedTableOptions::kBinarySearch: {
      return BinarySearchIndexReader::Create(
          file, prefetch_buffer, footer, footer.index_handle(), rep_->ioptions,
          icomparator, index_reader, rep_->persistent_cache_options,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_key_is_user_key == 0,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_value_is_delta_encoded == 0);
    }
    case BlockBasedTableOptions::kHashSearch: {
      std::unique_ptr<Block> meta_guard;
      std::unique_ptr<InternalIterator> meta_iter_guard;
      auto meta_index_iter = preloaded_meta_index_iter;
      if (meta_index_iter == nullptr) {
        auto s =
            ReadMetaBlock(rep_, prefetch_buffer, &meta_guard, &meta_iter_guard);
        if (!s.ok()) {
          // we simply fall back to binary search in case there is any
          // problem with prefix hash index loading.
          ROCKS_LOG_WARN(rep_->ioptions.info_log,
                         "Unable to read the metaindex block."
                         " Fall back to binary search index.");
          return BinarySearchIndexReader::Create(
              file, prefetch_buffer, footer, footer.index_handle(),
              rep_->ioptions, icomparator, index_reader,
              rep_->persistent_cache_options,
              rep_->table_properties == nullptr ||
                  rep_->table_properties->index_key_is_user_key == 0,
              rep_->table_properties == nullptr ||
                  rep_->table_properties->index_value_is_delta_encoded == 0);
        }
        meta_index_iter = meta_iter_guard.get();
      }

      return HashIndexReader::Create(
          rep_->internal_prefix_transform.get(), footer, file, prefetch_buffer,
          rep_->ioptions, icomparator, footer.index_handle(), meta_index_iter,
          index_reader, rep_->hash_index_allow_collision,
          rep_->persistent_cache_options,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_key_is_user_key == 0,
          rep_->table_properties == nullptr ||
              rep_->table_properties->index_value_is_delta_encoded == 0);
    }
    default: {
      std::string error_message =
          "Unrecognized index type: " + ToString(index_type_on_file);
      return Status::InvalidArgument(error_message.c_str());
    }
  }
}
```

默认情况下， `BlockBasedTable::UpdateIndexType()` 返回的是 `BlockBasedTableOptions::kBinarySearch` ，因此创建的是 `BinarySearchIndexReader`

具体的 `IndexReader` 区别在 `BlockBasedTableOptions` 中也有说明
```c++
struct BlockBasedTableOptions {
  // ...
  // The index type that will be used for this table.
  enum IndexType : char {
    // A space efficient index block that is optimized for
    // binary-search-based index.
    kBinarySearch,

    // The hash index, if enabled, will do the hash lookup when
    // `Options.prefix_extractor` is provided.
    kHashSearch,

    // A two-level index implementation. Both levels are binary search indexes.
    kTwoLevelIndexSearch,
  };

  IndexType index_type = kBinarySearch;
  // ...
};
```

## BinarySearchIndexReader
`BinarySearchIndexReader::NewIterator()` 会调用 `Block::NewIterator<IndexBlockIter>()` 来创建 index block 的迭代器，最后返回的类型是 `IndexBlockIter`
```c++
// Index that allows binary search lookup for the first key of each block.
// This class can be viewed as a thin wrapper for `Block` class which already
// supports binary search.
class BinarySearchIndexReader : public IndexReader {
 public:
  // ...
  virtual InternalIteratorBase<BlockHandle>* NewIterator(
      IndexBlockIter* iter = nullptr, bool /*dont_care*/ = true,
      bool /*dont_care*/ = true) override {
    Statistics* kNullStats = nullptr;
    return index_block_->NewIterator<IndexBlockIter>(
        icomparator_, icomparator_->user_comparator(), iter, kNullStats, true,
        index_key_includes_seq_, index_value_is_full_);
  }
 private:
  // ...
  std::unique_ptr<Block> index_block_;
  const bool index_key_includes_seq_;
  const bool index_value_is_full_;
};
```

## BlockBasedTableIterator<DataBlockIter>
前面提到， `BlockBasedTable::NewIterator()` 最后是构造并返回一个 `BlockBasedTableIterator<DataBlockIter>` ，并且这时已经创建好了 index block 的迭代器 `IndexBlockIter` ，而 data block 的迭代器则需要通过 index block 定位到 data block 之后才会创建

从 `BlockBasedTableIterator::Seek()` 为例进行分析
```c++
template <class TBlockIter, typename TValue>
void BlockBasedTableIterator<TBlockIter, TValue>::Seek(const Slice& target) {
  is_out_of_bound_ = false;
  if (!CheckPrefixMayMatch(target)) {
    ResetDataIter();
    return;
  }

  SavePrevIndexValue();

  index_iter_->Seek(target);

  if (!index_iter_->Valid()) {
    ResetDataIter();
    return;
  }

  InitDataBlock();

  block_iter_.Seek(target);

  FindKeyForward();
  assert(
      !block_iter_.Valid() ||
      (key_includes_seq_ && icomp_.Compare(target, block_iter_.key()) <= 0) ||
      (!key_includes_seq_ &&
       icomp_.user_comparator()->Compare(ExtractUserKey(target),
                                         block_iter_.key()) <= 0));
}
```

可以看到，通过 `IndexBlockIter::Seek()` 后会调用 `BlockBasedTableIterator::InitDataBlock()` 。该函数会通过负责调用 `BlockBasedTable::NewDataBlockIterator<DataBlockIter>()`
```c++
template <class TBlockIter, typename TValue>
void BlockBasedTableIterator<TBlockIter, TValue>::InitDataBlock() {
  BlockHandle data_block_handle = index_iter_->value();
  if (!block_iter_points_to_real_block_ ||
      data_block_handle.offset() != prev_index_value_.offset() ||
      // if previous attempt of reading the block missed cache, try again
      block_iter_.status().IsIncomplete()) {
    if (block_iter_points_to_real_block_) {
      ResetDataIter();
    }
    auto* rep = table_->get_rep();

    // ...
    Status s;
    // 调用 BlockBasedTable::NewDataBlockIterator()
    BlockBasedTable::NewDataBlockIterator<TBlockIter>(
        rep, read_options_, data_block_handle, &block_iter_, is_index_,
        key_includes_seq_, index_key_is_full_,
        /* get_context */ nullptr, s, prefetch_buffer_.get());
    // 设置 BlockBasedTableIterator::block_iter_points_to_real_block_ 为 true
    block_iter_points_to_real_block_ = true;
  }
}
```

### BlockBasedTable::NewDataBlockIterator<DataBlockIter>()
该函数的主要流程如下
* 调用 `BlockBasedTable::MaybeLoadDataBlockToCache()` ，该函数在开启了 block cache 的情况下会尝试从 block cache 中读取 data block ，如果不在 cache ，则会从文件中读取并放入 cache
* 如果前面没找到，则调用 `ReadBlockFromFile()` 从文件中读取 data block
* 调用 `Block::NewIterator<DataBlockIter>()` 来创建 data block 的迭代器 `DataBlockIter` ，最后返回的类型是 `DataBlockIter`

```c++
// Convert an index iterator value (i.e., an encoded BlockHandle)
// into an iterator over the contents of the corresponding block.
// If input_iter is null, new a iterator
// If input_iter is not null, update this iter and return it
template <typename TBlockIter>
TBlockIter* BlockBasedTable::NewDataBlockIterator(
    Rep* rep, const ReadOptions& ro, const BlockHandle& handle,
    TBlockIter* input_iter, bool is_index, bool key_includes_seq,
    bool index_key_is_full, GetContext* get_context, Status s,
    FilePrefetchBuffer* prefetch_buffer) {
  PERF_TIMER_GUARD(new_table_block_iter_nanos);

  const bool no_io = (ro.read_tier == kBlockCacheTier);
  Cache* block_cache = rep->table_options.block_cache.get();
  CachableEntry<Block> block;
  Slice compression_dict;
  if (s.ok()) {
    if (rep->compression_dict_block) {
      compression_dict = rep->compression_dict_block->data;
    }
    s = MaybeLoadDataBlockToCache(prefetch_buffer, rep, ro, handle,
                                  compression_dict, &block, is_index,
                                  get_context);
  }

  TBlockIter* iter;
  if (input_iter != nullptr) {
    iter = input_iter;
  } else {
    iter = new TBlockIter;
  }
  // Didn't get any data from block caches.
  if (s.ok() && block.value == nullptr) {
    if (no_io) {
      // Could not read from block_cache and can't do IO
      iter->Invalidate(Status::Incomplete("no blocking io"));
      return iter;
    }
    std::unique_ptr<Block> block_value;
    {
      StopWatch sw(rep->ioptions.env, rep->ioptions.statistics,
                   READ_BLOCK_GET_MICROS);
      s = ReadBlockFromFile(
          rep->file.get(), prefetch_buffer, rep->footer, ro, handle,
          &block_value, rep->ioptions, rep->blocks_maybe_compressed,
          compression_dict, rep->persistent_cache_options,
          is_index ? kDisableGlobalSequenceNumber : rep->global_seqno,
          rep->table_options.read_amp_bytes_per_bit, rep->immortal_table);
    }
    if (s.ok()) {
      block.value = block_value.release();
    }
  }

  if (s.ok()) {
    assert(block.value != nullptr);
    const bool kTotalOrderSeek = true;
    iter = block.value->NewIterator<TBlockIter>(
        &rep->internal_comparator, rep->internal_comparator.user_comparator(),
        iter, rep->ioptions.statistics, kTotalOrderSeek, key_includes_seq,
        index_key_is_full);
    if (block.cache_handle != nullptr) {
      iter->RegisterCleanup(&ReleaseCachedEntry, block_cache,
                            block.cache_handle);
    } else {
      // ...
    }
  } else {
    assert(block.value == nullptr);
    iter->Invalidate(s);
  }
  return iter;
}
```

## MergingIterator
遍历过程中创建的 memtable 、 imuutable memtables 、 sst 的迭代器会通过 `MergingIterator` 来汇总

`MergingIterator` 内部会通过堆来维护所有的子迭代器，堆顶迭代器 `MergingIterator::current_` 的值就是当前遍历到的值，调用 `Next()` 时使用的是 min heap ，调用 `Prev()` 时使用的是 max heap ，这样迭代器读取下一个值的复杂度就可以降到 O(logN) 
```c++
virtual void Next() override {
  assert(Valid());

  // Ensure that all children are positioned after key().
  // If we are moving in the forward direction, it is already
  // true for all of the non-current children since current_ is
  // the smallest child and key() == current_->key().
  if (direction_ != kForward) {
    SwitchToForward();
    // The loop advanced all non-current children to be > key() so current_
    // should still be strictly the smallest key.
    assert(current_ == CurrentForward());
  }

  // For the heap modifications below to be correct, current_ must be the
  // current top of the heap.
  assert(current_ == CurrentForward());

  // as the current points to the current record. move the iterator forward.
  current_->Next();
  if (current_->Valid()) {
    // current is still valid after the Next() call above.  Call
    // replace_top() to restore the heap property.  When the same child
    // iterator yields a sequence of keys, this is cheap.
    assert(current_->status().ok());
    minHeap_.replace_top(current_);
  } else {
    // current stopped being valid, remove it from the heap.
    considerStatus(current_->status());
    minHeap_.pop();
  }
  current_ = CurrentForward();
}

IteratorWrapper* CurrentForward() const {
  assert(direction_ == kForward);
  // 如果 MergingIterator::minHeap_ 非空，返回 MergingIterator::minHeap_
  // 顶部元素，否则返回 nullptr
  return !minHeap_.empty() ? minHeap_.top() : nullptr;
}
```

另外， min heap 是在构造 `MergingIterator` 时就会构造好，而 max heap 则是在需要的时候构造，这是因为使用 `Next()` 的场景要比 `Prev()` 多
```c++
MergingIterator(const InternalKeyComparator* comparator,
                InternalIterator** children, int n, bool is_arena_mode,
                bool prefix_seek_mode)
    : is_arena_mode_(is_arena_mode),
      comparator_(comparator),
      current_(nullptr),
      direction_(kForward),
      minHeap_(comparator_),
      prefix_seek_mode_(prefix_seek_mode),
      pinned_iters_mgr_(nullptr) {
  children_.resize(n);
  for (int i = 0; i < n; i++) {
    children_[i].Set(children[i]);
  }
  for (auto& child : children_) {
    if (child.Valid()) {
      assert(child.status().ok());
      // 将 child 插入 MergingIterator::minHeap_
      minHeap_.push(&child);
    } else {
      considerStatus(child.status());
    }
  }
  current_ = CurrentForward();
}

virtual void AddIterator(InternalIterator* iter) {
  assert(direction_ == kForward);
  children_.emplace_back(iter);
  if (pinned_iters_mgr_) {
    iter->SetPinnedItersMgr(pinned_iters_mgr_);
  }
  auto new_wrapper = children_.back();
  if (new_wrapper.Valid()) {
    assert(new_wrapper.status().ok());
    // 插入 MergingIterator::minHeap_
    minHeap_.push(&new_wrapper);
    current_ = CurrentForward();
  } else {
    considerStatus(new_wrapper.status());
  }
}
```

## 参考
* [【Rocksdb实现分析及优化】Iterator](https://kernelmaker.github.io/Rocksdb_Iterator)
