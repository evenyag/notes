# Get 分析

在 RocksDB 中，删除或更新数据是通过插入一条新的数据来实现的，而插入的数据都会带有一个递增的 sequence number 和一个类型字段来标识数据的版本以及操作的类型（ put/delete/merge 等）

写入时，数据会先写入到 memtable 。随后 memtable 变为 immutable memtable ，最后 flush 成为 sst 文件

因此，当需要查询一条数据时，需要依次在 memtable 、 immutable memtable 、 和 sst 文件中查询，直到找到目标数据

## 一些概念
### ValueType
RocksDB 中，数据的操作类型使用 `ValueType` 来表示，其中
* `kTypeDeletion` 和 `kTypeColumnFamilyDeletion` 分别标识删除 `default` cf （column family）和其他 cf 的数据
* `kTypeValue` 和 `kTypeColumnFamilyValue` 分别表示插入 `default` 和其他 cf 的数据
* `kTypeMerge` 和 `kTypeColumnFamilyMerge` 分别表示 merge `default` 和其他 cf 的数据

```c++
enum ValueType : unsigned char {
  kTypeDeletion = 0x0,
  kTypeValue = 0x1,
  kTypeMerge = 0x2,
  // ...
  kTypeColumnFamilyDeletion = 0x4,  // WAL only.
  kTypeColumnFamilyValue = 0x5,     // WAL only.
  kTypeColumnFamilyMerge = 0x6,     // WAL only.
  // ...
  kTypeBlobIndex = 0x11,                  // Blob DB only
  // ...
  kMaxValue = 0x7F                // Not used for storing records.
};
```

### SequenceNumber
RocksDB 中的版本 sequence number 通过 `SequenceNumber` 类型来表示，其中只有 56 bits 是用来存储 sequence number 的，另外 8 bits 可以存储 `ValueType` ，这样有需要时就可以将两个信息打包存储到一个 64 bits 整型中
```c++
typedef uint64_t SequenceNumber;

// We leave eight bits empty at the bottom so a type and sequence#
// can be packed together into 64-bits.
static const SequenceNumber kMaxSequenceNumber =
    ((0x1ull << 56) - 1);

uint64_t PackSequenceAndType(uint64_t seq, ValueType t) {
  assert(seq <= kMaxSequenceNumber);
  assert(IsExtendedValueType(t));
  return (seq << 8) | t;
}
```

### Keys
用户传入进行查找的 key ，称为 user key ，不带版本等信息

RocksDB 内部进行操作时，使用的 key 是会带上 `SequenceNumber` 和 `ValueType` 这些信息的。这些信息可以直接包装为 `ParsedInternalKey` 对象或 `InternalKey` 对象
```c++
struct ParsedInternalKey {
  Slice user_key;
  SequenceNumber sequence;
  ValueType type;

  ParsedInternalKey()
      : sequence(kMaxSequenceNumber)  // Make code analyzer happy
        {} // Intentionally left uninitialized (for speed)
  ParsedInternalKey(const Slice& u, const SequenceNumber& seq, ValueType t)
      : user_key(u), sequence(seq), type(t) { }
  // ...
};
```

其中 `InternalKey` 可以看作是 `ParsedInternalKey` 序列化后得到的， `InternalKey` 可以看作是数据的真实的 key
```c++
class InternalKey {
 private:
  std::string rep_;
 public:
  InternalKey() { }   // Leave rep_ as empty to indicate it is invalid
  InternalKey(const Slice& _user_key, SequenceNumber s, ValueType t) {
    AppendInternalKey(&rep_, ParsedInternalKey(_user_key, s, t));
  }
  // ...
  Slice Encode() const {
    assert(!rep_.empty());
    return rep_;
  }
  // ...
};

void AppendInternalKey(std::string* result, const ParsedInternalKey& key) {
  result->append(key.user_key.data(), key.user_key.size());
  PutFixed64(result, PackSequenceAndType(key.sequence, key.type));
}
```

RocksDB 内部通过 `InternalKey` 对数据进行排序，原则如下
* `InternalKey` 按 user key 递增
* `InternalKey` 按 sequence number 递减
* `InternalKey` 按 `ValueType` 递减（但实际上通过 sequence number 已经可以保证唯一性了）

因此，通过越新的数据， sequence number 越大，就能排到越前面

`InternalKey` 之间并不能直接进行比较，而是需要通过 `InternalKeyComparator` 来进行比较，实现如下
```c++
int InternalKeyComparator::Compare(const ParsedInternalKey& a,
                                   const ParsedInternalKey& b) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(a.user_key, b.user_key);
  PERF_COUNTER_ADD(user_key_comparison_count, 1);
  if (r == 0) {
    if (a.sequence > b.sequence) {
      r = -1;
    } else if (a.sequence < b.sequence) {
      r = +1;
    } else if (a.type > b.type) {
      r = -1;
    } else if (a.type < b.type) {
      r = +1;
    }
  }
  return r;
}

inline int InternalKeyComparator::Compare(
    const InternalKey& a, const InternalKey& b) const {
  return Compare(a.Encode(), b.Encode());
}

int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  PERF_COUNTER_ADD(user_key_comparison_count, 1);
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

用户在查找时，传入的 user key 实际上会被 RocksDB 加上 sequence 来构造出 `LookupKey` 用于查找
```c++
// A helper class useful for DBImpl::Get()
class LookupKey {
 public:
  // Initialize *this for looking up user_key at a snapshot with
  // the specified sequence number.
  LookupKey(const Slice& _user_key, SequenceNumber sequence);

  ~LookupKey();

  // Return a key suitable for lookup in a MemTable.
  Slice memtable_key() const {
    return Slice(start_, static_cast<size_t>(end_ - start_));
  }

  // Return an internal key (suitable for passing to an internal iterator)
  Slice internal_key() const {
    return Slice(kstart_, static_cast<size_t>(end_ - kstart_));
  }

  // Return the user key
  Slice user_key() const {
    return Slice(kstart_, static_cast<size_t>(end_ - kstart_ - 8));
  }

 private:
  // We construct a char array of the form:
  //    klength  varint32               <-- start_
  //    userkey  char[klength]          <-- kstart_
  //    tag      uint64
  //                                    <-- end_
  // The array is a suitable MemTable key.
  // The suffix starting with "userkey" can be used as an InternalKey.
  const char* start_;
  const char* kstart_;
  const char* end_;
  char space_[200];      // Avoid allocation for short keys
  // ...
};

LookupKey::LookupKey(const Slice& _user_key, SequenceNumber s) {
  size_t usize = _user_key.size();
  size_t needed = usize + 13;  // A conservative estimate
  char* dst;
  if (needed <= sizeof(space_)) {
    dst = space_;
  } else {
    dst = new char[needed];
  }
  start_ = dst;
  // NOTE: We don't support users keys of more than 2GB :)
  dst = EncodeVarint32(dst, static_cast<uint32_t>(usize + 8));
  kstart_ = dst;
  memcpy(dst, _user_key.data(), usize);
  dst += usize;
  EncodeFixed64(dst, PackSequenceAndType(s, kValueTypeForSeek));
  dst += 8;
  end_ = dst;
}
```

可以看到 `LookupKey` 的结构如下

| start | kstart | end |
| --- | --- | --- |
| user key len (varint32) | user key (user key len) | `SequenceNumber + ValueType` (uint64) |

获取 memtable key 时返回 [start, end) ，获取 internal key 时返回 [kstart, end)

```c++
// kValueTypeForSeek defines the ValueType that should be passed when
// constructing a ParsedInternalKey object for seeking to a particular
// sequence number (since we sort sequence numbers in decreasing order
// and the value type is embedded as the low 8 bits in the sequence
// number in internal keys, we need to use the highest-numbered
// ValueType, not the lowest).
const ValueType kValueTypeForSeek = kTypeBlobIndex;
const ValueType kValueTypeForSeekForPrev = kTypeDeletion;
```
另外，可以看到查找时会将 `ValueType` 设置为 `kValueTypeForSeek` ，即一个值尽可能大的 `ValueType` ，实际上就是 `kTypeBlobIndex` 的值

之所以选用尽可能大的值，是因为在用 key1 进行 seek 时，是要找到大于等于 key1 的数据。而 `ValueType` 是降序，因此需要保证 key1 的 `ValueType` 是可能出现的 `ValueType` 中最大的， seek 时才能返回所有 `ValueType` 小于等于 key1 的 `ValueType` 的数据

## DBImpl::GetImpl()
获取数据的实现在 `DBImpl::GetImpl()` 中
```c++
Status DBImpl::GetImpl(const ReadOptions& read_options,
                       ColumnFamilyHandle* column_family, const Slice& key,
                       PinnableSlice* pinnable_val, bool* value_found,
                       ReadCallback* callback, bool* is_blob_index) {
  // ...
  auto cfh = reinterpret_cast<ColumnFamilyHandleImpl*>(column_family);
  // 调用 ColumnFamilyHandleImpl::cfd() 获取 cf handle 的 ColumnFamilyData()
  auto cfd = cfh->cfd();
  // ...

  // Acquire SuperVersion
  // 调用 DBImpl::GetAndRefSuperVersion() 获取 SuperVersion
  SuperVersion* sv = GetAndRefSuperVersion(cfd);
  // ...
  SequenceNumber snapshot;
  if (read_options.snapshot != nullptr) {
    // ...
    snapshot =
        reinterpret_cast<const SnapshotImpl*>(read_options.snapshot)->number_;
    if (callback) {
      snapshot = std::max(snapshot, callback->MaxUnpreparedSequenceNumber());
    }
  } else {
    // Since we get and reference the super version before getting
    // the snapshot number, without a mutex protection, it is possible
    // that a memtable switch happened in the middle and not all the
    // data for this snapshot is available. But it will contain all
    // the data available in the super version we have, which is also
    // a valid snapshot to read from.
    // We shouldn't get snapshot before finding and referencing the super
    // version because a flush happening in between may compact away data for
    // the snapshot, but the snapshot is earlier than the data overwriting it,
    // so users may see wrong results.
    snapshot = last_seq_same_as_publish_seq_
                   ? versions_->LastSequence()
                   : versions_->LastPublishedSequence();
  }
  // ...

  // First look in the memtable, then in the immutable memtable (if any).
  // s is both in/out. When in, s could either be OK or MergeInProgress.
  // merge_operands will contain the sequence of merges in the latter case.
  // 用 key 和 snapshot 生成 LookupKey
  LookupKey lkey(key, snapshot);
  PERF_TIMER_STOP(get_snapshot_time);

  // 判断是否需要跳过 memtable 查找
  bool skip_memtable = (read_options.read_tier == kPersistedTier &&
                        has_unpersisted_data_.load(std::memory_order_relaxed));
  bool done = false;
  if (!skip_memtable) {
    // 如果不跳过 memtable 查找
    // 调用 SuperVersion::mem 的 MemTable::Get() 从 memtable 查找
    if (sv->mem->Get(lkey, pinnable_val->GetSelf(), &s, &merge_context,
                     &range_del_agg, read_options, callback, is_blob_index)) {
      // 如果找到了，调用 PinnableSlice::PinSelf()
      done = true;
      pinnable_val->PinSelf();
      RecordTick(stats_, MEMTABLE_HIT);
    } else if ((s.ok() || s.IsMergeInProgress()) &&
               sv->imm->Get(lkey, pinnable_val->GetSelf(), &s, &merge_context,
                            &range_del_agg, read_options, callback,
                            is_blob_index)) {
      // 否则如果从 SuperVersion::imm 调用 MemTableListVersion::Get() 找到了
      done = true;
      // 调用 PinnableSlice::PinSelf()
      pinnable_val->PinSelf();
      RecordTick(stats_, MEMTABLE_HIT);
    }
    if (!done && !s.ok() && !s.IsMergeInProgress()) {
      // 如果找不到且出错了，则调用 DBImpl::ReturnAndCleanupSuperVersion() 并返回
      ReturnAndCleanupSuperVersion(cfd, sv);
      return s;
    }
  }
  if (!done) {
    PERF_TIMER_GUARD(get_from_output_files_time);
    // 如果从 memtable 中找不到或者数据不全
    // 调用 SuperVersion::current 的 Version::Get() 从 sst 中查找
    sv->current->Get(read_options, lkey, pinnable_val, &s, &merge_context,
                     &range_del_agg, value_found, nullptr, nullptr, callback,
                     is_blob_index);
    RecordTick(stats_, MEMTABLE_MISS);
  }

  // ...
  return s;
}
```

基本流程如下
* 获取 `SuperVersion`
* 如果指定了 snapshot 则用 snapshot 的版本，否则使用 `VersionSet::LastSequence()` 的版本构造 `LookupKey` （可以看作是当前 version 最后一次写成功的 sequence number）
* 调用 `MemTable::Get()` 从 memtable 中查找
* 调用 `MemTableListVersion::Get()` 从 immutable memtable 中查找
* 调用 `Version::Get()` 从 sst 中查找

这里先获取 `SuperVersion` 再获取 sequence number 的原因在注释中做了解释，应该是为了保证 `SuperVersion` 的版本不会比 sequence number 新，避免用户看到错误的结果
* 先获取 `SuperVersion` 时，如果在获取 sequence number 前发生了 memtable 切换，产生了新的 `SuperVersion` ，那么尽管新版本下写入的数据在旧的 `SuperVersion` 下没有，但是旧的 `SuperVersion` 仍然是一个合法可用的版本
* 先获取 sequence number 的话，如果发生 flush 产生了新的 `SuperVersion` ，期间一些数据可能会被 compact 掉，这时，用旧的 sequence number 去读可能会得到有问题的结果

RocksDB 中 cf 的 immutable memtable 可以是多个，而 `MemTableListVersion` 实际上就是 immutable memtable 列表。 `MemTableListVersion::Get()` 实际上就是遍历列表并调用 `MemTable::Get()` 从列表的 immutable memtable 中查找数据

因此， get 操作的入口主要是 `MemTable::Get()` 以及 `Version::Get()`

## MemTable::Get()
从 memtable 中获取数据的实现如下
```c++
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s,
                   MergeContext* merge_context,
                   RangeDelAggregator* range_del_agg, SequenceNumber* seq,
                   const ReadOptions& read_opts, ReadCallback* callback,
                   bool* is_blob_index) {
  // The sequence number is updated synchronously in version_set.h
  if (IsEmpty()) {
    // Avoiding recording stats for speed.
    // 如果 memtable 为空，直接返回 false
    return false;
  }
  // ...

  // 调用 LookupKey::user_key() 获取 user_key
  Slice user_key = key.user_key();
  // 初始化 found_final_value 为 false
  bool found_final_value = false;
  // 初始化 merge_in_progress 为 Status::IsMergeInProgress()
  bool merge_in_progress = s->IsMergeInProgress();
  // 先通过 bloom filter 查找
  // 如果 MemTable::prefix_bloom_ 为空，设置 may_contain 为 false
  // 否则设置为 MemTable::prefix_bloom_ DynamicBloom::MayContain() 的值
  bool const may_contain =
      nullptr == prefix_bloom_
          ? false
          : prefix_bloom_->MayContain(prefix_extractor_->Transform(user_key));
  if (prefix_bloom_ && !may_contain) {
    // iter is null if prefix bloom says the key does not exist
    PERF_COUNTER_ADD(bloom_memtable_miss_count, 1);
    // 如果 bloom filter 发现这个 key 不存在
    // 设置 *seq 为 kMaxSequenceNumber
    *seq = kMaxSequenceNumber;
  } else {
    // 否则
    if (prefix_bloom_) {
      PERF_COUNTER_ADD(bloom_memtable_hit_count, 1);
    }
    // 创建 Saver
    Saver saver;
    saver.status = s;
    // 设置 Saver::found_final_value 指向 found_final_value
    saver.found_final_value = &found_final_value;
    // 设置 Saver::merge_in_progress 指向 merge_in_progress
    saver.merge_in_progress = &merge_in_progress;
    saver.key = &key;
    saver.value = value;
    saver.seq = kMaxSequenceNumber;
    saver.mem = this;
    saver.merge_context = merge_context;
    saver.range_del_agg = range_del_agg;
    saver.merge_operator = moptions_.merge_operator;
    saver.logger = moptions_.info_log;
    saver.inplace_update_support = moptions_.inplace_update_support;
    saver.statistics = moptions_.statistics;
    saver.env_ = env_;
    saver.callback_ = callback;
    saver.is_blob_index = is_blob_index;
    // 调用 MemTable::table_ MemTableRep::Get()
    // 参数为 Saver ， callback 为 SaveValue()
    table_->Get(key, &saver, SaveValue);

    // 设置 *seq 为 Saver::seq
    *seq = saver.seq;
  }

  // No change to value, since we have not yet found a Put/Delete
  if (!found_final_value && merge_in_progress) {
    *s = Status::MergeInProgress();
  }
  PERF_COUNTER_ADD(get_from_memtable_count, 1);
  return found_final_value;
}
```

函数主要是为了构造查找时的上下文 `Saver` ，然后调用 `MemTableRep::Get()` 函数。其中第三个参数是会调用函数

## MemTableRep::Get()
`MemTableRep` 实际上是个虚类， MemTable 底层可以采用不同的实现。默认的实现是基于 skiplist 的 `SkipListRep`
```c++
class SkipListRep : public MemTableRep {
  InlineSkipList<const MemTableRep::KeyComparator&> skip_list_;
  // ...
public:
  // ...
  virtual void Get(const LookupKey& k, void* callback_args,
                   bool (*callback_func)(void* arg,
                                         const char* entry)) override {
    // 创建 SkipListRep::Iterator
    SkipListRep::Iterator iter(&skip_list_);
    Slice dummy_slice;
    // 调用 Iterator::Seek() 根据 key 进行 seek
    // 如果 Iterator::Valid() 且传入的 callback_func() 返回 true
    // 则循环调用 Iterator::Next()
    for (iter.Seek(dummy_slice, k.memtable_key().data());
         iter.Valid() && callback_func(callback_args, iter.key());
         iter.Next()) {
    }
  }
  // ...
  // Iteration over the contents of a skip list
  class Iterator : public MemTableRep::Iterator {
    InlineSkipList<const MemTableRep::KeyComparator&>::Iterator iter_;

   public:
    // ...
    // Returns true iff the iterator is positioned at a valid node.
    virtual bool Valid() const override {
      return iter_.Valid();
    }

    // Returns the key at the current position.
    // REQUIRES: Valid()
    virtual const char* key() const override {
      return iter_.key();
    }

    // Advances to the next position.
    // REQUIRES: Valid()
    virtual void Next() override {
      iter_.Next();
    }

    // Advances to the previous position.
    // REQUIRES: Valid()
    virtual void Prev() override {
      iter_.Prev();
    }

    // Advance to the first entry with a key >= target
    virtual void Seek(const Slice& user_key, const char* memtable_key)
        override {
      if (memtable_key != nullptr) {
        iter_.Seek(memtable_key);
      } else {
        iter_.Seek(EncodeKey(&tmp_, user_key));
      }
    }
    // ...
   protected:
    std::string tmp_;       // For passing to EncodeKey
  };
  // ...
};
```

查找的过程实际上就是通过 `MemTableRep` 的迭代器来实现的，在找到数据之后，会调用之前传入的回调函数，也就是 `SaveValue()`

## SaveValue()
`SaveValue()` 的第一个参数是之前保存的 `Saver` 对象，第二个参数是在 skiplist 中定位到的记录

`SaveValue()` 首先会判断获取到的 key 是否和通过 `Saver` 传入的 user key 相同，如果不同，说明没找到，直接返回 false ，否则，会根据记录的 `ValueType` 做相应的处理，这里暂时不考虑 merge 的情况
* 如果是 `kTypeValue` 则会设置到 `Saver::value` 中以返回给用户
* 如果是 `kTypeDeletion` 则说明没找到

```c++
// 返回 true 表示要继续处理，返回 false 则不用继续处理
static bool SaveValue(void* arg, const char* entry) {
  Saver* s = reinterpret_cast<Saver*>(arg);
  // ...

  // entry format is:
  //    klength  varint32
  //    userkey  char[klength-8]
  //    tag      uint64
  //    vlength  varint32
  //    value    char[vlength]
  // Check that it belongs to same user key.  We do not check the
  // sequence number since the Seek() call above should have skipped
  // all entries with overly large sequence numbers.
  uint32_t key_length;
  // 调用 GetVarint32Ptr() 获取 key_ptr 以及 key 的长度 key_length
  const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);
  if (s->mem->GetInternalKeyComparator().user_comparator()->Equal(
          Slice(key_ptr, key_length - 8), s->key->user_key())) {
    // Correct user key
    // 如果 key 和 user key 相同
    const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
    // 调用 UnPackSequenceAndType() 取出 ValueType 和 SequenceNumber
    ValueType type;
    SequenceNumber seq;
    UnPackSequenceAndType(tag, &seq, &type);
    // If the value is not in the snapshot, skip it
    // 调用 Saver::CheckCallback() 检查 sequence number ，如果返回 false
    if (!s->CheckCallback(seq)) {
      // 返回 true
      return true;  // to continue to the next seq
    }

    // 否则，将 Saver::seq 设置为 seq
    s->seq = seq;

    // ...
    // 根据 type 的类型进行对应的处理
    switch (type) {
      // ...
      case kTypeValue: {
        // 如果是 kTypeValue (或者前面没有返回)
        if (s->inplace_update_support) {
          s->mem->GetLock(s->key->user_key())->ReadLock();
        }
        Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
        *(s->status) = Status::OK();
        if (*(s->merge_in_progress)) {
          if (s->value != nullptr) {
            *(s->status) = MergeHelper::TimedFullMerge(
                merge_operator, s->key->user_key(), &v,
                merge_context->GetOperands(), s->value, s->logger,
                s->statistics, s->env_, nullptr /* result_operand */, true);
          }
        } else if (s->value != nullptr) {
          s->value->assign(v.data(), v.size());
        }
        if (s->inplace_update_support) {
          s->mem->GetLock(s->key->user_key())->ReadUnlock();
        }
        *(s->found_final_value) = true;
        if (s->is_blob_index != nullptr) {
          *(s->is_blob_index) = (type == kTypeBlobIndex);
        }
        return false;
      }
      case kTypeDeletion:
      case kTypeSingleDeletion:
      case kTypeRangeDeletion: {
        // 如果为 kTypeDeletion 或 kTypeSingleDeletion 或 kTypeRangeDeletion
        if (*(s->merge_in_progress)) {
          if (s->value != nullptr) {
            *(s->status) = MergeHelper::TimedFullMerge(
                merge_operator, s->key->user_key(), nullptr,
                merge_context->GetOperands(), s->value, s->logger,
                s->statistics, s->env_, nullptr /* result_operand */, true);
          }
        } else {
          *(s->status) = Status::NotFound();
        }
        *(s->found_final_value) = true;
        return false;
      }
      case kTypeMerge: {
        // 如果为 kTypeMerge
        if (!merge_operator) {
          *(s->status) = Status::InvalidArgument(
              "merge_operator is not properly initialized.");
          // Normally we continue the loop (return true) when we see a merge
          // operand.  But in case of an error, we should stop the loop
          // immediately and pretend we have found the value to stop further
          // seek.  Otherwise, the later call will override this error status.
          *(s->found_final_value) = true;
          return false;
        }
        Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
        *(s->merge_in_progress) = true;
        merge_context->PushOperand(
            v, s->inplace_update_support == false /* operand_pinned */);
        if (merge_operator->ShouldMerge(merge_context->GetOperandsDirectionBackward())) {
          *(s->status) = MergeHelper::TimedFullMerge(
              merge_operator, s->key->user_key(), nullptr,
              merge_context->GetOperands(), s->value, s->logger, s->statistics,
              s->env_, nullptr /* result_operand */, true);
          *(s->found_final_value) = true;
          return false;
        }
        return true;
      }
      default:
        assert(false);
        return true;
    }
  }

  // s->state could be Corrupt, merge or notfound
  // 否则，返回 false
  return false;
}
```

## Version::Get()
`Version::Get()` 用于从 sst 中获取数据

主要流程包括
* 通过 `FilePicker` 找到范围覆盖了 key 的 sst 文件
* 循环调用 `FilePicker::GetNextFile()` 获取找到的 sst 文件
* 调用 `TableCache::Get()` 从 `FilePicker::GetNextFile()` 返回的文件中查找 key 对应的数据
* 根据返回的结果做相应的处理，如果没有找到会继续调用 `FilePicker::GetNextFile()` 查找下一个文件

```c++
void Version::Get(const ReadOptions& read_options, const LookupKey& k,
                  PinnableSlice* value, Status* status,
                  MergeContext* merge_context,
                  RangeDelAggregator* range_del_agg, bool* value_found,
                  bool* key_exists, SequenceNumber* seq, ReadCallback* callback,
                  bool* is_blob) {
  // ...
  PinnedIteratorsManager pinned_iters_mgr;
  GetContext get_context(
      user_comparator(), merge_operator_, info_log_, db_statistics_,
      status->ok() ? GetContext::kNotFound : GetContext::kMerge, user_key,
      value, value_found, merge_context, range_del_agg, this->env_, seq,
      merge_operator_ ? &pinned_iters_mgr : nullptr, callback, is_blob);

  // Pin blocks that we read to hold merge operands
  if (merge_operator_) {
    pinned_iters_mgr.StartPinning();
  }

  FilePicker fp(
      storage_info_.files_, user_key, ikey, &storage_info_.level_files_brief_,
      storage_info_.num_non_empty_levels_, &storage_info_.file_indexer_,
      user_comparator(), internal_comparator());
  FdWithKeyRange* f = fp.GetNextFile();

  while (f != nullptr) {
    // ...

    *status = table_cache_->Get(
        read_options, *internal_comparator(), *f->file_metadata, ikey,
        &get_context, mutable_cf_options_.prefix_extractor.get(),
        cfd_->internal_stats()->GetFileReadHist(fp.GetHitFileLevel()),
        IsFilterSkipped(static_cast<int>(fp.GetHitFileLevel()),
                        fp.IsHitFileLastInLevel()),
        fp.GetCurrentLevel());
    // TODO: examine the behavior for corrupted key
    if (!status->ok()) {
      return;
    }
    // ...

    // 根据 GetContext::State() 的结果进行对应的处理
    switch (get_context.State()) {
      case GetContext::kNotFound:
        // Keep searching in other files
        break;
      case GetContext::kMerge:
        break;
      case GetContext::kFound:
        // ...
        return;
      case GetContext::kDeleted:
        // Use empty error message for speed
        *status = Status::NotFound();
        return;
      case GetContext::kCorrupt:
        *status = Status::Corruption("corrupted key for ", user_key);
        return;
      case GetContext::kBlobIndex:
        ROCKS_LOG_ERROR(info_log_, "Encounter unexpected blob index.");
        *status = Status::NotSupported(
            "Encounter unexpected blob index. Please open DB with "
            "rocksdb::blob_db::BlobDB instead.");
        return;
    }
    f = fp.GetNextFile();
  }
  // ...
}
```

## FilePicker
`FilePicker` 用于根据 key 查找范围覆盖了 key 的 sst 文件

`FilePicker` 在构造时会调用 `FilePicker::PrepareNextLevel()` 来准备 level 0 的查找
```c++
FilePicker(std::vector<FileMetaData*>* files, const Slice& user_key,
           const Slice& ikey, autovector<LevelFilesBrief>* file_levels,
           unsigned int num_levels, FileIndexer* file_indexer,
           const Comparator* user_comparator,
           const InternalKeyComparator* internal_comparator)
    : num_levels_(num_levels),
      curr_level_(static_cast<unsigned int>(-1)),
      returned_file_level_(static_cast<unsigned int>(-1)),
      hit_file_level_(static_cast<unsigned int>(-1)),
      search_left_bound_(0),
      search_right_bound_(FileIndexer::kLevelMaxIndex),
      // ...
      level_files_brief_(file_levels),
      is_hit_file_last_in_level_(false),
      curr_file_level_(nullptr),
      user_key_(user_key),
      ikey_(ikey),
      file_indexer_(file_indexer),
      user_comparator_(user_comparator),
      internal_comparator_(internal_comparator) {
  // ...
  // Setup member variables to search first level.
  search_ended_ = !PrepareNextLevel();
  if (!search_ended_) {
    // Prefetch Level 0 table data to avoid cache miss if possible.
    // 如果 FilePicker::search_ended_ 为 false ，则预取 level 0 的数据
    for (unsigned int i = 0; i < (*level_files_brief_)[0].num_files; ++i) {
      // 遍历 level 0 的所有文件
      auto* r = (*level_files_brief_)[0].files[i].fd.table_reader;
      if (r) {
        // 调用 TableReader::Prepare()
        r->Prepare(ikey);
      }
    }
  }
}
```

### FilePicker::GetNextFile()
`FilePicker::GetNextFile()` 会返回下一个符合条件的 sst ，主要流程为
* 遍历所有 level
    * 如果是 level 0 ，由于 level 0 的文件是无序且范围存在重叠的，遍历 level 0 的所有文件查找满足条件的文件
    * 如果非 level 0 ，则通过二分查找找到对应的文件，如果找不到，就到下一 level 中进行查找
* 开始下一 level 的查找时，会调用 `FilePicker::PrepareNextLevel()` 定位到下一 level 查找开始的地方，二分查找实际上是在这里面进行的

```c++
// 逐层按顺序查找文件，如果找到一个符合条件的文件，则返回
// 找不到文件，返回 nullptr
FdWithKeyRange* GetNextFile() {
  while (!search_ended_) {  // Loops over different levels.
    // 遍历不同的 level
    while (curr_index_in_curr_level_ < curr_file_level_->num_files) {
      // Loops over all files in current level.
      // 遍历当前 level 的所有文件 FdWithKeyRange f
      FdWithKeyRange* f = &curr_file_level_->files[curr_index_in_curr_level_];
      hit_file_level_ = curr_level_;
      is_hit_file_last_in_level_ =
          curr_index_in_curr_level_ == curr_file_level_->num_files - 1;
      int cmp_largest = -1;

      // Do key range filtering of files or/and fractional cascading if:
      // (1) not all the files are in level 0, or
      // (2) there are more than 3 current level files
      // If there are only 3 or less current level files in the system, we skip
      // the key range filtering. In this case, more likely, the system is
      // highly tuned to minimize number of tables queried by each query,
      // so it is unlikely that key range filtering is more efficient than
      // querying the files.
      if (num_levels_ > 1 || curr_file_level_->num_files > 3) {
        // Check if key is within a file's range. If search left bound and
        // right bound point to the same find, we are sure key falls in
        // range.
        assert(
            curr_level_ == 0 ||
            curr_index_in_curr_level_ == start_index_in_curr_level_ ||
            user_comparator_->Compare(user_key_,
              ExtractUserKey(f->smallest_key)) <= 0);

        int cmp_smallest = user_comparator_->Compare(user_key_,
            ExtractUserKey(f->smallest_key));
        if (cmp_smallest >= 0) {
          // 如果 cmp_smallest ，即查找的 key >= smallest key
          // 设置 cmp_largest 为 FilePicker::user_key_ 和 FdWithKeyRange::largest_key
          // 的比较结果
          cmp_largest = user_comparator_->Compare(user_key_,
              ExtractUserKey(f->largest_key));
        }

        // Setup file search bound for the next level based on the
        // comparison results
        if (curr_level_ > 0) {
          // 根据比较结果准备下一层 level 的 search bound
          file_indexer_->GetNextLevelIndex(curr_level_,
                                          curr_index_in_curr_level_,
                                          cmp_smallest, cmp_largest,
                                          &search_left_bound_,
                                          &search_right_bound_);
        }
        // Key falls out of current file's range
        // 如果 cmp_smallest < 0 或 cmp_largest > 0 ，说明 key 在当前文件外
        if (cmp_smallest < 0 || cmp_largest > 0) {
          if (curr_level_ == 0) {
            // 如果当前 level 为 level 0 ，自增 FilePicker::curr_index_in_curr_level_
            // 到下一个文件查找
            ++curr_index_in_curr_level_;
            // 继续处理下一个文件
            continue;
          } else {
            // Search next level.
            // 否则，跳出循环，到下一层查找
            break;
          }
        }
      }
      // ...

      // 说明找到了一个符合条件的文件，需要返回
      returned_file_level_ = curr_level_;
      if (curr_level_ > 0 && cmp_largest < 0) {
        // No more files to search in this level.
        // 如果当前 level 大于 0 且 cmp_largest < 0 ，说明命中了文件，当前 level
        // 没有其他文件需要查找了
        // 调用 FilePicker::PrepareNextLevel() 准备下一层，返回值取反设置到
        // FilePicker::search_ended_
        search_ended_ = !PrepareNextLevel();
      } else {
        // 否则， level 0 的文件由于范围可能重叠
        // 自增 FilePicker::curr_index_in_curr_level_ ，查找下一个文件
        ++curr_index_in_curr_level_;
      }
      // 返回找到的文件 f
      return f;
    }
    // Start searching next level.
    // 当前 level 查找完了，开始查找下一个 level
    search_ended_ = !PrepareNextLevel();
  }
  // Search ended.
  // 查找结束，没找到文件，返回 nullptr
  return nullptr;
}
```

### FileIndexer::GetNextLevelIndex()
RocksDB 通过 `FileIndexer` 来加快二分查找的速度。每次应用（ apply ）一个新的 `Version` 前，都会调用 `FileIndexer::UpdateIndex()` 来生成 `FileIndexer` 索引

`FileIndexer` 主要保存每一个 level n 和 level n + 1 的 key 范围的关联信息。当在 level n 查找时，如果没有找到匹配的文件，就可以通过这个索引快速得到 level n + 1 需要查找的文件范围

假设 key 在 level n 中没找到符合的文件，当前所在 level n 文件 fileN 的范围为 [smallest, largest] ，则 key 的比较结果有以下三种
* key 小于 fileN.smallest ，则 level n + 1 中 smallest 大于 fileN.smallest 的文件可以作为右边界被跳过
* key 位于 fileN.smallest 和 fileN.largest 之间
    * level n + 1 中 largest 小于 fileN.smallest 的文件可以作为左边界被跳过
    * level n + 1 中 smallest 大于 fileN.largest 的文件可以作为右边界被跳过
* key 大于 fileN.largest ，则 level n + 1 中 largest 小于 fileN.largest 的文件可以作为左边界被跳过

`FileIndexer` 中会记录下面三个值
```c++
// Example:
//    level 1:                     [50 - 60]
//    level 2:        [1 - 40], [45 - 55], [58 - 80]
// A key 35, compared to be less than 50, 3rd file on level 2 can be
// skipped according to rule (1). LB = 0, RB = 1.
// A key 53, sits in the middle 50 and 60. 1st file on level 2 can be
// skipped according to rule (2)-a, but the 3rd file cannot be skipped
// because 60 is greater than 58. LB = 1, RB = 2.
// A key 70, compared to be larger than 60. 1st and 2nd file can be skipped
// according to rule (3). LB = 2, RB = 2.
//
// Point to a left most file in a lower level that may contain a key,
// which compares greater than smallest of a FileMetaData (upper level)
// level n + 1 中位于最左边的且可能包含比 fileN.smallest 大的 key 的文件
int32_t smallest_lb;
// Point to a left most file in a lower level that may contain a key,
// which compares greater than largest of a FileMetaData (upper level)
// level n + 1 中位于最左边的且可能包含比 fileN.largest 大的 key 的文件
int32_t largest_lb;
// Point to a right most file in a lower level that may contain a key,
// which compares smaller than smallest of a FileMetaData (upper level)
// level n + 1 中位于最右边的且可能包含比 fileN.smallest 小的 key 的文件
int32_t smallest_rb;
// Point to a right most file in a lower level that may contain a key,
// which compares smaller than largest of a FileMetaData (upper level)
// level n + 1 中位于最右边的且可能包含比 fileN.largest 小的 key 的文件
int32_t largest_rb;
```

在 level n 中查找时，会先通过二分查找找到第一个 largest 大于等于 user key 的文件 f （如果 user key 小于 level n 全局最小的 key 则 f 为 level n 第一个文件，如果 user key 大于 level n 全局最大的 key ，则 f 为 level n 最后一个文件），则当 f 不命中时，就会执行 `FileIndexer::GetNextLevelIndex()` 来确定 level n + 1 中查找的范围 left_bound 和 right_bound

和 f 比较时，可以有以下几种情况
* user key 小于 smallest_key
    * 如果 f 是 level n 最左边的文件，则 left_bound = 0
    * 否则， left_bound = f 前一个文件对应的 `largest_lb`
    * right_bound = f 的 `smallest_rb`
* user key == smallest_key
    * left_bound = f 的 `smallest_lb`
    * right_bound = f 的 `smallest_rb`
* user key 大于 smallest_key 且小于 largest_key
    * left_bound = f 的 `smallest_lb`
    * right_bound = f 的 `largest_rb`
* user key 等于 largest_key
    * left_bound = f 的 `largest_lb`
    * right_bound = f 的 `largest_rb`
* user key 大于 largest_key
    * left_bound = f 的 `largest_lb`
    * right_bound = level n + 1 中最右的 sst （注意这里 f 肯定是 level n 最后一个文件了）

可以看到, `smallest_lb` 和 `largest_lb` 用于 left_bound ， `smallest_rb` 和 `largest_rb` 用于 right_bound

实现如下
```c++
void FileIndexer::GetNextLevelIndex(const size_t level, const size_t file_index,
                                    const int cmp_smallest,
                                    const int cmp_largest, int32_t* left_bound,
                                    int32_t* right_bound) const {
  assert(level > 0);

  // Last level, no hint
  if (level == num_levels_ - 1) {
    *left_bound = 0;
    *right_bound = -1;
    return;
  }

  assert(level < num_levels_ - 1);
  assert(static_cast<int32_t>(file_index) <= level_rb_[level]);

  const IndexUnit* index_units = next_level_index_[level].index_units;
  const auto& index = index_units[file_index];

  if (cmp_smallest < 0) {
    *left_bound = (level > 0 && file_index > 0)
                      ? index_units[file_index - 1].largest_lb
                      : 0;
    *right_bound = index.smallest_rb;
  } else if (cmp_smallest == 0) {
    *left_bound = index.smallest_lb;
    *right_bound = index.smallest_rb;
  } else if (cmp_smallest > 0 && cmp_largest < 0) {
    *left_bound = index.smallest_lb;
    *right_bound = index.largest_rb;
  } else if (cmp_largest == 0) {
    *left_bound = index.largest_lb;
    *right_bound = index.largest_rb;
  } else if (cmp_largest > 0) {
    *left_bound = index.largest_lb;
    *right_bound = level_rb_[level + 1];
  } else {
    assert(false);
  }

  assert(*left_bound >= 0);
  assert(*left_bound <= *right_bound + 1);
  assert(*right_bound <= level_rb_[level + 1]);
}
```

## TableCache::Get()
`TableCache::Get()` 用于从 sst 中查找数据，同时会缓存相应的文件信息、在 `row_cache` 选项打开后还会对当前 key 找到的 value 进行缓存。 `row_cache` 选项默认是关闭的

在 `row_cache` 打开的情况下， RocksDB 首先会计算 row cache 的 key ，然后尝试从 row cache 中查找。可以看到 row cache 的 key 就是 `row_cache_id + fd_number + seq_no + user_key`
```c++
Status TableCache::Get(const ReadOptions& options,
                       const InternalKeyComparator& internal_comparator,
                       const FileMetaData& file_meta, const Slice& k,
                       GetContext* get_context,
                       const SliceTransform* prefix_extractor,
                       HistogramImpl* file_read_hist, bool skip_filters,
                       int level) {
  auto& fd = file_meta.fd;
  std::string* row_cache_entry = nullptr;
  bool done = false;
  // ...
  IterKey row_cache_key;
  std::string row_cache_entry_buffer;

  // Check row cache if enabled. Since row cache does not currently store
  // sequence numbers, we cannot use it if we need to fetch the sequence.
  if (ioptions_.row_cache && !get_context->NeedToReadSequence()) {
    uint64_t fd_number = fd.GetNumber();
    auto user_key = ExtractUserKey(k);
    // We use the user key as cache key instead of the internal key,
    // otherwise the whole cache would be invalidated every time the
    // sequence key increases. However, to support caching snapshot
    // reads, we append the sequence number (incremented by 1 to
    // distinguish from 0) only in this case.
    uint64_t seq_no =
        options.snapshot == nullptr ? 0 : 1 + GetInternalKeySeqno(k);

    // Compute row cache key.
    // 生成 row cache key
    // TableCache::row_cache_id_ + fd_number + seq_no + user_key
    row_cache_key.TrimAppend(row_cache_key.Size(), row_cache_id_.data(),
                             row_cache_id_.size());
    AppendVarint64(&row_cache_key, fd_number);
    AppendVarint64(&row_cache_key, seq_no);
    row_cache_key.TrimAppend(row_cache_key.Size(), user_key.data(),
                             user_key.size());

    // 先从 row cache 中查找
    if (auto row_handle =
            ioptions_.row_cache->Lookup(row_cache_key.GetUserKey())) {
      // 如果 Cache::Lookup() 返回值不为 nullptr
      // Cleanable routine to release the cache entry
      Cleanable value_pinner;
      auto release_cache_entry_func = [](void* cache_to_clean,
                                         void* cache_handle) {
        ((Cache*)cache_to_clean)->Release((Cache::Handle*)cache_handle);
      };
      // 调用 Cache::Value() 获取缓存的具体内容
      auto found_row_cache_entry = static_cast<const std::string*>(
          ioptions_.row_cache->Value(row_handle));
      // If it comes here value is located on the cache.
      // found_row_cache_entry points to the value on cache,
      // and value_pinner has cleanup procedure for the cached entry.
      // After replayGetContextLog() returns, get_context.pinnable_slice_
      // will point to cache entry buffer (or a copy based on that) and
      // cleanup routine under value_pinner will be delegated to
      // get_context.pinnable_slice_. Cache entry is released when
      // get_context.pinnable_slice_ is reset.
      value_pinner.RegisterCleanup(release_cache_entry_func,
                                   ioptions_.row_cache.get(), row_handle);
      replayGetContextLog(*found_row_cache_entry, user_key, get_context,
                          &value_pinner);
      RecordTick(ioptions_.statistics, ROW_CACHE_HIT);
      done = true;
    } else {
      // Not found, setting up the replay log.
      // 从 Cache 中没找到
      RecordTick(ioptions_.statistics, ROW_CACHE_MISS);
      // row_cache_entry 指向 row_cache_entry_buffer
      row_cache_entry = &row_cache_entry_buffer;
    }
  }
  // ...
}
```

如果在 row cache 中没找到，就需要在 sst 文件中进行查找。可以看到每个 `fd` 都包含了一个 `TableReader` 对象用于读取 sst 的内容。 `TableCache` 主要就是用于缓存 `TableReader` 的，如果 `fd` 的 `TableReader` 指针为 nullptr ，就会调用 `TableCache::FindTable()` 来获取 `TableReader`

获取到 `TableReader` 后，就可以调用 `TableReader::Get()` 从 sst 中查找数据了。 `TableReader` 同样也是一个接口，可以对接不同的 sst 实现，默认的实现是 `BlockBasedTable`
```c++
Status TableCache::Get(const ReadOptions& options,
                       const InternalKeyComparator& internal_comparator,
                       const FileMetaData& file_meta, const Slice& k,
                       GetContext* get_context,
                       const SliceTransform* prefix_extractor,
                       HistogramImpl* file_read_hist, bool skip_filters,
                       int level) {
  // ...
  Status s;
  TableReader* t = fd.table_reader;
  Cache::Handle* handle = nullptr;
  if (!done && s.ok()) {
    if (t == nullptr) {
      // 调用 TableCache::FindTable() 查找并获取 TableReader
      s = FindTable(
          env_options_, internal_comparator, fd, &handle, prefix_extractor,
          options.read_tier == kBlockCacheTier /* no_io */,
          true /* record_read_stats */, file_read_hist, skip_filters, level);
      if (s.ok()) {
        t = GetTableReaderFromHandle(handle);
      }
    }
    // ...
    if (s.ok()) {
      // 如果状态为 ok ，从 table 中找到了
      // 传入 row_cache_entry 调用 GetContext::SetReplayLog() 设置到 GetContext
      // 中用于记录 replay log ，随后可以将 replay log 插入 row cache
      get_context->SetReplayLog(row_cache_entry);  // nullptr if no cache.
      // 调用 TableReader::Get()
      s = t->Get(options, k, get_context, prefix_extractor, skip_filters);
      // 传入 nullptr 调用 GetContext::SetReplayLog()
      get_context->SetReplayLog(nullptr);
    } else if (options.read_tier == kBlockCacheTier && s.IsIncomplete()) {
      // Couldn't find Table in cache but treat as kFound if no_io set
      get_context->MarkKeyMayExist();
      s = Status::OK();
      done = true;
    }
  }
  // ...
}
```

如果找到了 key ，还会将对应的 value 缓存到 row_cache 中
```c++
Status TableCache::Get(const ReadOptions& options,
                       const InternalKeyComparator& internal_comparator,
                       const FileMetaData& file_meta, const Slice& k,
                       GetContext* get_context,
                       const SliceTransform* prefix_extractor,
                       HistogramImpl* file_read_hist, bool skip_filters,
                       int level) {
  // ...
  // Put the replay log in row cache only if something was found.
  // 将 table 中找到的 replay log 加入 row cache 中
  if (!done && s.ok() && row_cache_entry && !row_cache_entry->empty()) {
    size_t charge =
        row_cache_key.Size() + row_cache_entry->size() + sizeof(std::string);
    void* row_ptr = new std::string(std::move(*row_cache_entry));
    // 调用 Cache::Insert() 将 replay log 插入 row cache
    ioptions_.row_cache->Insert(row_cache_key.GetUserKey(), row_ptr, charge,
                                &DeleteEntry<std::string>);
  }
  // ...
}
```

### TableCache::FindTable()
`TableCache::FindTable()` 用于实现对应 `TableReader` 的读取以及缓存
```c++
Status TableCache::FindTable(const EnvOptions& env_options,
                             const InternalKeyComparator& internal_comparator,
                             const FileDescriptor& fd, Cache::Handle** handle,
                             const SliceTransform* prefix_extractor,
                             const bool no_io, bool record_read_stats,
                             HistogramImpl* file_read_hist, bool skip_filters,
                             int level,
                             bool prefetch_index_and_filter_in_cache) {
  PERF_TIMER_GUARD(find_table_nanos);
  Status s;
  uint64_t number = fd.GetNumber();
  // 调用 GetSliceForFileNumber() 将 number 转为 key
  Slice key = GetSliceForFileNumber(&number);
  *handle = cache_->Lookup(key);
  TEST_SYNC_POINT_CALLBACK("TableCache::FindTable:0",
                           const_cast<bool*>(&no_io));

  if (*handle == nullptr) {
    // 如果 *handle 为 nullptr ，则没在 cache 中找到
    if (no_io) {  // Don't do IO and return a not-found status
      return Status::Incomplete("Table not found in table_cache, no_io is set");
    }
    unique_ptr<TableReader> table_reader;
    // 调用 TableCache::GetTableReader() 创建 table reader
    s = GetTableReader(env_options, internal_comparator, fd,
                       false /* sequential mode */, 0 /* readahead */,
                       record_read_stats, file_read_hist, &table_reader,
                       prefix_extractor, skip_filters, level,
                       prefetch_index_and_filter_in_cache);
    if (!s.ok()) {
      assert(table_reader == nullptr);
      RecordTick(ioptions_.statistics, NO_FILE_ERRORS);
      // We do not cache error results so that if the error is transient,
      // or somebody repairs the file, we recover automatically.
    } else {
      // 调用 TableCache::cache_ Cache::Insert() 将 TableReader 加入 cache
      s = cache_->Insert(key, table_reader.get(), 1, &DeleteEntry<TableReader>,
                         handle);
      if (s.ok()) {
        // Release ownership of table reader.
        table_reader.release();
      }
    }
  }
  return s;
}
```

可以看到，读取的工作在 `TableCache::GetTableReader()` 中实现，其中核心是调用配置选项中 table factory 的 `TableFactory::NewTableReader()` 来创建 `TableReader` ，默认创建的是 `BlockBasedTable`
```c++
Status TableCache::GetTableReader(
    const EnvOptions& env_options,
    const InternalKeyComparator& internal_comparator, const FileDescriptor& fd,
    bool sequential_mode, size_t readahead, bool record_read_stats,
    HistogramImpl* file_read_hist, unique_ptr<TableReader>* table_reader,
    const SliceTransform* prefix_extractor, bool skip_filters, int level,
    bool prefetch_index_and_filter_in_cache, bool for_compaction) {
  // 调用 TableFileName() 获取对应的 sst 文件名
  std::string fname =
      TableFileName(ioptions_.cf_paths, fd.GetNumber(), fd.GetPathId());
  unique_ptr<RandomAccessFile> file;
  // 调用 Env::NewRandomAccessFile() 打开文件
  Status s = ioptions_.env->NewRandomAccessFile(fname, &file, env_options);

  RecordTick(ioptions_.statistics, NO_FILE_OPENS);
  if (s.ok()) {
    // ...
    std::unique_ptr<RandomAccessFileReader> file_reader(
        new RandomAccessFileReader(
            std::move(file), fname, ioptions_.env,
            record_read_stats ? ioptions_.statistics : nullptr, SST_READ_MICROS,
            file_read_hist, ioptions_.rate_limiter, for_compaction));
    // 调用 TableFactory::NewTableReader() 创建新的 TableReader
    s = ioptions_.table_factory->NewTableReader(
        TableReaderOptions(ioptions_, prefix_extractor, env_options,
                           internal_comparator, skip_filters, immortal_tables_,
                           level, fd.largest_seqno),
        std::move(file_reader), fd.GetFileSize(), table_reader,
        prefetch_index_and_filter_in_cache);
    TEST_SYNC_POINT("TableCache::GetTableReader:0");
  }
  return s;
}
```

## 参考
* [MySQL · RocksDB · 数据的读取(一)](http://mysql.taobao.org/monthly/2018/11/05/)
* [MySQL · RocksDB · 数据的读取(二)](http://mysql.taobao.org/monthly/2018/12/08/)
* leveldb 实现解析 by 那岩
* [【Rocksdb实现分析及优化】FileIndexer](http://kernelmaker.github.io/Rocksdb_file_indexers)
