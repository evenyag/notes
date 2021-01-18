# Flush 流程
主要关心的问题
* Flush memtable 的触发条件
* 调用入口
* 执行 flush 的线程
* 相关参数？

数据写入 RocksDB 时，会先写到 WAL 和 memtable 中。当 memtable 写满后，会转变为 immutable memtable ，里面的数据将不再变化，最终 RocksDB 通过 flush 操作将 immutable memtable 持久化为 level 0 的 sst 文件

## DBImpl::SwitchMemtable()
前面提到 memtable 写满后会进行 memtable 的切换，该 memtable 会切换为 immutable memtable 。这个过程调用 `DBImpl::SwitchMemtable()` 来实现的

调用这个函数时需要持有 db 的大锁 `DBImpl::mutex_` 并且位于 Writer 队列的头部，也就是写入队列的 leader ，以保证操作过程的线程安全性

主要的流程包括
* 检查当前的 WAL 文件是否为空，如果不为空就需要创建新的 WAL 文件（如果 `recycle_log_file_num` 选项开启了就会复用 WAL 文件）
* 获取当前最新的 sequence number 并构建新的 memtable ，然后创建新的 super version
* 当前的 WAL 文件刷盘，保证 WAL 的数据都落盘了
* 将新的 WAL 文件加入到 WAL 文件队列 `DBImpl::logs_` 中
* 将之前的 memtable 变为 immutable memtable ，然后切换到新创建的 memtable
* 调用 `DBImpl::InstallSuperVersionAndScheduleWork()` 函数启用新的 super version 并且有可能会触发 flush 或 compaction 操作

这里提到的 super version 指的是某一时刻 RocksDB 中所有的 sst 文件和 memtable 的集合，可以理解为 RocksDB 的一个版本，具体的解释详见 RocksDB 的官方 [wiki](https://github.com/facebook/rocksdb/wiki/Terminology)

在 RocksDB 中，以下几个地方都会调用 `SwitchMemtable()`
* `DBImpl::SwitchWAL()`
* `DBImpl::HandleWriteBufferFull()`
* `DBImpl::FlushMemTable()`
* `DBImpl::FlushAllCFs()`
* `DBImpl::ScheduleFlushes()`

## DBImpl::SwitchWAL()
这个函数的调用条件是判断 WAL 的大小是否已经超过 `max_total_wal_size`
```c++
Status DBImpl::PreprocessWrite(const WriteOptions& write_options,
                               bool* need_log_sync,
                               WriteContext* write_context) {
  // ...
  if (UNLIKELY(status.ok() && !single_column_family_mode_ &&
               total_log_size_ > GetMaxTotalWalSize())) {
    status = SwitchWAL(write_context);
  }
  // ...
}
```

`DBImpl::SwitchWAL()` 会找到要清理的 WAL 对应的 memtable 并调用 `SwitchMemtable()` 刷盘
```c++
Status DBImpl::SwitchWAL(WriteContext* write_context) {
  // ...
  // 创建 FlushRequest
  FlushRequest flush_req;
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    // 遍历所有 cf
    if (cfd->IsDropped()) {
      // 如果 cf 已被 drop ，遍历下一个
      continue;
    }
    if (cfd->OldestLogToKeep() <= oldest_alive_log) {
      // 如果 ColumnFamilyData::OldestLogToKeep() <= oldest_alive_log ，则
      // 需要 flush memtable
      // 调用 DBImpl::SwitchMemtable() 切换对应 cfd 的 memtable
      status = SwitchMemtable(cfd, write_context);
      if (!status.ok()) {
        // 如果出错，跳出循环
        break;
      }
      // 向 flush_req 尾部插入 (cfd, MemTableList::GetLatestMemTableID())
      flush_req.emplace_back(cfd, cfd->imm()->GetLatestMemTableID());
      // 调用 MemTableList::FlushRequested() 标记所有 memtable 需要 flush
      cfd->imm()->FlushRequested();
    }
  }
  if (status.ok()) {
    // 如果没有出错
    // 调用 DBImpl::SchedulePendingFlush() 
    // flush_reason 为 FlushReason::kWriteBufferManager 将 FlushRequest 入队列
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferManager);
    // 调用 DBImpl::MaybeScheduleFlushOrCompaction() 调度执行 flush 和 compaction
    MaybeScheduleFlushOrCompaction();
  }
  return status;
}
```

## DBImpl::HandleWriteBufferFull()
这个函数在 `WriteBufferManager::ShouldFlush()` 返回 true 时调用，主要处理所有 cf 的 memtable 内存用量超过限制的情况
```c++
Status DBImpl::PreprocessWrite(const WriteOptions& write_options,
                               bool* need_log_sync,
                               WriteContext* write_context) {
  // ...
  if (UNLIKELY(status.ok() && write_buffer_manager_->ShouldFlush())) {
    // Before a new memtable is added in SwitchMemtable(),
    // write_buffer_manager_->ShouldFlush() will keep returning true. If another
    // thread is writing to another DB with the same write buffer, they may also
    // be flushed. We may end up with flushing much more DBs than needed. It's
    // suboptimal but still correct.
    // 如果 WriteBufferManager::ShouldFlush() 返回 true ，调用
    // DBImpl::HandleWriteBufferFull()
    status = HandleWriteBufferFull(write_context);
  }
  // ...
}
```

`WriteBufferManager::ShouldFlush()` 会检查 write buffer ，也就是 memtable 使用的内存是否超过限制，如果超过了，就说明需要进行 flush 操作
```c++
// Should only be called from write thread
bool ShouldFlush() const {
  if (enabled()) {
    if (mutable_memtable_memory_usage() > mutable_limit_) {
      return true;
    }
    if (memory_usage() >= buffer_size_ &&
        mutable_memtable_memory_usage() >= buffer_size_ / 2) {
      // If the memory exceeds the buffer size, we trigger more aggressive
      // flush. But if already more than half memory is being flushed,
      // triggering more flush may not help. We will hold it instead.
      return true;
    }
  }
  return false;
}
```
而 `WriteBufferManager::buffer_size_` 就是 `db_write_buffer_size` 选项

`DBImpl::HandleWriteBufferFull()` 的实现如下，可以看到，实际上只会选出一个 cf 进行 flush
```c++
Status DBImpl::HandleWriteBufferFull(WriteContext* write_context) {
  // ...
  // cfd_picked 初始化为 nullptr
  ColumnFamilyData* cfd_picked = nullptr;
  // seq_num_for_cf_picked 初始化为 kMaxSequenceNumber
  SequenceNumber seq_num_for_cf_picked = kMaxSequenceNumber;

  for (auto cfd : *versions_->GetColumnFamilySet()) {
    // 遍历所有 cf
    if (cfd->IsDropped()) {
      // 如果 cf 被 drop 则处理下一个 cf
      continue;
    }
    if (!cfd->mem()->IsEmpty()) {
      // We only consider active mem table, hoping immutable memtable is
      // already in the process of flushing.
      // 如果 mutable memtable 的 MemTable::IsEmpty() 返回 false
      // 调用 MemTable::GetCreationSeq() 获取创建 memtable 时的 seq
      uint64_t seq = cfd->mem()->GetCreationSeq();
      if (cfd_picked == nullptr || seq < seq_num_for_cf_picked) {
        // 如果 cfd_picked 为 nullptr 或 seq 小于 seq_num_for_cf_picked
        // 设置 cfd_picked 为当前 cfd
        cfd_picked = cfd;
        // 设置 seq_num_for_cf_picked 为 seq
        seq_num_for_cf_picked = seq;
      }
    }
  }

  autovector<ColumnFamilyData*> cfds;
  if (cfd_picked != nullptr) {
    // 如果 cfd_picked 不为 nullptr
    // 将 cfd_picked 插入 cfds 队列
    cfds.push_back(cfd_picked);
  }
  FlushRequest flush_req;
  for (const auto cfd : cfds) {
    // 遍历 cfds ，实际上这里 cfds 应该只有一个 cfd
    // 调用 ColumnFamilyData::Ref()
    cfd->Ref();
    // 调用 DBImpl::SwitchMemtable() 切换 memtable
    status = SwitchMemtable(cfd, write_context);
    // 调用 ColumnFamilyData::Unref()
    cfd->Unref();
    if (!status.ok()) {
      break;
    }
    // 调用 immutable memtable 的 MemTableList::GetLatestMemTableID()
    // 获取 flush_memtable_id ，实际上就是刚刚被 switch 的 memtable
    uint64_t flush_memtable_id = cfd->imm()->GetLatestMemTableID();
    // 调用 MemTableList::FlushRequested()
    cfd->imm()->FlushRequested();
    // 将 request 插入 FlushRequest
    flush_req.emplace_back(cfd, flush_memtable_id);
  }
  if (status.ok()) {
    // 调用 DBImpl::SchedulePendingFlush() ， flush_reason 为 FlushReason::kWriteBufferFull
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferFull);
    // 调用 DBImpl::MaybeScheduleFlushOrCompaction()
    MaybeScheduleFlushOrCompaction();
  }
  return status;
}
```

## DBImpl::FlushMemTable()
该函数用于强制 flush memtable 到磁盘，例如用户直接调用 `DB::Flush()` 接口时会触发该函数
```c++
Status DBImpl::FlushMemTable(ColumnFamilyData* cfd,
                             const FlushOptions& flush_options,
                             FlushReason flush_reason, bool writes_stopped) {
  FlushRequest flush_req;
  {
    WriteContext context;
    InstrumentedMutexLock guard_lock(&mutex_);
    // ...
    if (cfd->imm()->NumNotFlushed() != 0 || !cfd->mem()->IsEmpty() ||
        !cached_recoverable_state_empty_.load()) {
      s = SwitchMemtable(cfd, &context);
      flush_memtable_id = cfd->imm()->GetLatestMemTableID();
      flush_req.emplace_back(cfd, flush_memtable_id);
    }

    if (s.ok() && !flush_req.empty()) {
      for (auto& elem : flush_req) {
        ColumnFamilyData* loop_cfd = elem.first;
        loop_cfd->imm()->FlushRequested();
      }
      SchedulePendingFlush(flush_req, flush_reason);
      MaybeScheduleFlushOrCompaction();
    }
    // ...
  }
  // ...
  return s;
}
```

## DBImpl::FlushAllCFs()
该函数是用户调用 `DB::Resume()` 接口时触发，会强制所有 cf 的 memtable 刷盘
```c++
Status DBImpl::FlushAllCFs(FlushReason flush_reason) {
  // ...
  FlushRequest flush_req;
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    if (cfd->imm()->NumNotFlushed() == 0 && cfd->mem()->IsEmpty() &&
        cached_recoverable_state_empty_.load()) {
      // Nothing to flush
      continue;
    }

    // SwitchMemtable() will release and reacquire mutex during execution
    s = SwitchMemtable(cfd, &context);
    if (!s.ok()) {
      break;
    }

    cfd->imm()->FlushRequested();

    flush_req.emplace_back(cfd, cfd->imm()->GetLatestMemTableID());
  }

  // schedule flush
  if (s.ok() && !flush_req.empty()) {
    SchedulePendingFlush(flush_req, flush_reason);
    MaybeScheduleFlushOrCompaction();
  }
  // ...

  flush_req.clear();
  return s;
}

## DBImpl::ScheduleFlushes()
当 `DBImpl::PreprocessWrite()` 判断 `FlushScheduler` 不为空时会调用该函数
```c++
Status DBImpl::PreprocessWrite(const WriteOptions& write_options,
                               bool* need_log_sync,
                               WriteContext* write_context) {
  // ...
  if (UNLIKELY(status.ok() && !flush_scheduler_.Empty())) {
    // 如果 FlushScheduler::Empty() 返回 false ,调用
    // DBImpl::ScheduleFlushes()
    status = ScheduleFlushes(write_context);
  }
  // ...
}
```

`FlushScheduler` 通过一个链表来管理需要 flush 的 cf
```c++
class FlushScheduler {
 public:
  FlushScheduler() : head_(nullptr) {}

  // May be called from multiple threads at once, but not concurrent with
  // any other method calls on this instance
  void ScheduleFlush(ColumnFamilyData* cfd);

  // Removes and returns Ref()-ed column family. Client needs to Unref().
  // Filters column families that have been dropped.
  ColumnFamilyData* TakeNextColumnFamily();

  bool Empty();

  void Clear();

 private:
  struct Node {
    ColumnFamilyData* column_family;
    Node* next;
  };

  std::atomic<Node*> head_;
  // ...
};
```
而 `FlushScheduler::Empty()` 实际上就是判断该链表是否为空
```c++
bool FlushScheduler::Empty() {
  // ...
  auto rv = head_.load(std::memory_order_relaxed) == nullptr;
  // ...
  return rv;
}
```

当 cf 需要 flush 时，会将该 cf 通过调用 `FlushScheduler::ScheduleFlush()` 加入到链表中
```c++
void FlushScheduler::ScheduleFlush(ColumnFamilyData* cfd) {
  // ...
  cfd->Ref();
// Suppress false positive clang analyzer warnings.
#ifndef __clang_analyzer__
  Node* node = new Node{cfd, head_.load(std::memory_order_relaxed)};
  while (!head_.compare_exchange_strong(
      node->next, node, std::memory_order_relaxed, std::memory_order_relaxed)) {
    // failing CAS updates the first param, so we are already set for
    // retry.  TakeNextColumnFamily won't happen until after another
    // inter-thread synchronization, so we don't even need release
    // semantics for this CAS
  }
#endif  // __clang_analyzer__
}
```

那么 `FlushScheduler::ScheduleFlush()` 什么时候会调用呢？

首先，每个 memtable 都有一个 `MemTable::flush_state_` 状态
```c++
enum FlushStateEnum { FLUSH_NOT_REQUESTED, FLUSH_REQUESTED, FLUSH_SCHEDULED };
```

在调用 `MemTable::UpdateFlushState()` 时，如果该 memtable 之前不需要 flush 且 `MemTable::ShouldFlushNow()` 返回 true ，就会将状态会更新为 `FLUSH_REQUESTED`
```c++
void MemTable::UpdateFlushState() {
  auto state = flush_state_.load(std::memory_order_relaxed);
  if (state == FLUSH_NOT_REQUESTED && ShouldFlushNow()) {
    // ignore CAS failure, because that means somebody else requested
    // a flush
    flush_state_.compare_exchange_strong(state, FLUSH_REQUESTED,
                                         std::memory_order_relaxed,
                                         std::memory_order_relaxed);
  }
}
```

而 `MemTable::ShouldFlushNow()` 主要是判断 memtable 使用的内存是否超过了 `Write_buffer_size` 的限制，如果超过了，就返回 true
```c++
bool MemTable::ShouldFlushNow() const {
  size_t write_buffer_size = write_buffer_size_.load(std::memory_order_relaxed);
  // In a lot of times, we cannot allocate arena blocks that exactly matches the
  // buffer size. Thus we have to decide if we should over-allocate or
  // under-allocate.
  // This constant variable can be interpreted as: if we still have more than
  // "kAllowOverAllocationRatio * kArenaBlockSize" space left, we'd try to over
  // allocate one more block.
  const double kAllowOverAllocationRatio = 0.6;

  // If arena still have room for new block allocation, we can safely say it
  // shouldn't flush.
  auto allocated_memory = table_->ApproximateMemoryUsage() +
                          range_del_table_->ApproximateMemoryUsage() +
                          arena_.MemoryAllocatedBytes();

  // if we can still allocate one more block without exceeding the
  // over-allocation ratio, then we should not flush.
  if (allocated_memory + kArenaBlockSize <
      write_buffer_size + kArenaBlockSize * kAllowOverAllocationRatio) {
    return false;
  }

  // if user keeps adding entries that exceeds write_buffer_size, we need to
  // flush earlier even though we still have much available memory left.
  if (allocated_memory >
      write_buffer_size + kArenaBlockSize * kAllowOverAllocationRatio) {
    return true;
  }

  return arena_.AllocatedAndUnused() < kArenaBlockSize / 4;
}
```

`MemTable::UpdateFlushState()` 会在每次写 memtable 时，即执行 `MemTable::Add` 或 `MemTable::UpdateCallback()` 操作时会被调用，这样就能够判断是否需要 flush memtable

在 `flush_state_` 更新为 `FLUSH_REQUESTED` 后， `MemTable::ShouldScheduleFlush()` 就会返回 true ，表示 memtable 需要 flush
```c++
// This method heuristically determines if the memtable should continue to
// host more data.
bool ShouldScheduleFlush() const {
  return flush_state_.load(std::memory_order_relaxed) == FLUSH_REQUESTED;
}
```

`MemTable::ShouldScheduleFlush()` 会在 `MemTableInserter::CheckMemtableFull()` 中被调用
```c++
void CheckMemtableFull() {
  if (flush_scheduler_ != nullptr) {
    // 如果设置了 MemTableInserter::flush_scheduler_
    // 调用 ColumnFamilyMemTables::current() 获取当前 ColumnFamilyData 到 cfd
    auto* cfd = cf_mems_->current();
    assert(cfd != nullptr);
    // 如果 cfd 的 MemTable::ShouldScheduleFlush() 和 MemTable::MarkFlushScheduled()
    // 都返回 true
    if (cfd->mem()->ShouldScheduleFlush() &&
        cfd->mem()->MarkFlushScheduled()) {
      // MarkFlushScheduled only returns true if we are the one that
      // should take action, so no need to dedup further
      // 调用 FlushScheduler::ScheduleFlush()
      flush_scheduler_->ScheduleFlush(cfd);
    }
  }
}
```
`CheckMemtableFull()` 会检查 memtable 是否需要 flush ，如果需要 flush ，就会调用 `MemTable::MarkFlushScheduled()` 将 memtable 的状态从 `FLUSH_REQUESTED` 改为 `FLUSH_SCHEDULED` ，表示 flush 已经被调用，然后调用 `FlushScheduler::ScheduleFlush()` 将 cf 加入到链表中

`MarkFlushScheduled()` 的实现如下，就是用来改变状态的
```c++
// Returns true if a flush should be scheduled and the caller should
// be the one to schedule it
bool MarkFlushScheduled() {
  auto before = FLUSH_REQUESTED;
  return flush_state_.compare_exchange_strong(before, FLUSH_SCHEDULED,
                                              std::memory_order_relaxed,
                                              std::memory_order_relaxed);
}
```

`MemTableInserter::CheckMemtableFull()` 就是在数据从 WriteBatch 写入 memtable 的过程中调用的，包括以下几种情况
* Put 操作
* Delete 操作
* Merge 操作

可见，在这些操作过程中，如果检测到 memtable 需要 flush 了，并不会马上 flush memtable ，因为这种情况的紧急程度不是很高，而是将需要 flush 的 cf 加入到 `FlushScheduler` 的链表中，待下次写入时再执行 flush

`DBImpl::ScheduleFlushes()` 的实现如下，主要是循环从 `FlushScheduler` 中取出需要 flush 的 cf 进行 flush 操作
```c++
Status DBImpl::ScheduleFlushes(WriteContext* context) {
  ColumnFamilyData* cfd;
  FlushRequest flush_req;
  Status status;
  // 循环调用 FlushScheduler::TakeNextColumnFamily() 取出 cfd
  while ((cfd = flush_scheduler_.TakeNextColumnFamily()) != nullptr) {
    // 如果 cfd 不为 nullptr
    // 调用 DBImpl::SwitchMemtable()
    status = SwitchMemtable(cfd, context);
    // 初始化 should_schedule 为 true
    bool should_schedule = true;
    if (cfd->Unref()) {
      // 如果 ColumnFamilyData::Unref() 返回 true
      // 删除 cfd
      delete cfd;
      // 设置 should_schedule 为 false
      should_schedule = false;
    }
    if (!status.ok()) {
      // 如果遇到错误
      // 跳出循环
      break;
    }
    if (should_schedule) {
      // 如果 should_schedule 为 true
      // 调用 MemTableList::GetLatestMemTableID() 获取 flush_memtable_id
      uint64_t flush_memtable_id = cfd->imm()->GetLatestMemTableID();
      // 插入 FlushRequest
      flush_req.emplace_back(cfd, flush_memtable_id);
    }
  }
  if (status.ok()) {
    // 调用 DBImpl::SchedulePendingFlush() ， flush_reason 为 FlushReason::kWriteBufferFull
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferFull);
    // 调用 DBImpl::MaybeScheduleFlushOrCompaction()
    MaybeScheduleFlushOrCompaction();
  }
  return status;
}
```

## DBImpl::SchedulePendingFlush()
具体的 flush 操作是通过调用 `DBImpl::SchedulePendingFlush()` 来实现的，可以看到前面完成 memtable 的切换后，都会调用该函数来触发 flush 操作

该函数实际上就是将 flush 的请求 FlushRequest 插入到 flush 请求队列 `DBImpl::flush_queue_` 中
```c++
void DBImpl::SchedulePendingFlush(const FlushRequest& flush_req,
                                  FlushReason flush_reason) {
  if (flush_req.empty()) {
    // 如果 FlushRequest 为空，直接返回
    return;
  }
  for (auto& iter : flush_req) {
    // 遍历 FlushRequest
    ColumnFamilyData* cfd = iter.first;
    // 调用 ColumnFamilyData::Ref()
    cfd->Ref();
    // 调用 ColumnFamilyData::SetFlushReason() 设置 flush reason
    cfd->SetFlushReason(flush_reason);
  }
  // 增加 DBImpl::unscheduled_flushes_ 为 FlushRequest 的数量
  unscheduled_flushes_ += static_cast<int>(flush_req.size());
  // 将 FlushRequest 插入 DBImpl::flush_queue_
  flush_queue_.push_back(flush_req);
}
```

## DBImpl::MaybeScheduleFlushOrCompaction()
`SchedulePendingFlush()` 仅仅是将请求入队列了，并不会真正地执行 flush 请求，因此可以看到调用 `SchedulePendingFlush()` 后，会再调用 `DBImpl::MaybeScheduleFlushOrCompaction()` 函数

`DBImpl::MaybeScheduleFlushOrCompaction()` 函数，该函数会检查当前是否能够执行 flush 操作，如果可以，就会在后台线程中执行 `DBImpl::BGWorkFlush()` 函数
```c++
void DBImpl::MaybeScheduleFlushOrCompaction() {
  // ...
  // 调用 DBImpl::GetBGJobLimits() 获取 bg_job_limits
  auto bg_job_limits = GetBGJobLimits();
  // 获取 high-pri 线程池大小
  // 如果 Env::GetBackgroundThreads() 返回 0 ，设置 is_flush_pool_empty 为 true
  // 否则，设置为 false
  bool is_flush_pool_empty =
      env_->GetBackgroundThreads(Env::Priority::HIGH) == 0;
  while (!is_flush_pool_empty && unscheduled_flushes_ > 0 &&
         bg_flush_scheduled_ < bg_job_limits.max_flushes) {
    // 循环
    // 如果 is_flush_pool_empty 为 false 且 DBImpl::unscheduled_flushes_ > 0
    // 且 DBImpl::bg_flush_scheduled_ < BGJobLimits::max_flushes
    // 自增 DBImpl::bg_flush_scheduled_
    bg_flush_scheduled_++;
    // 调用 Env::Schedule() 调度 high-pri 线程池执行 DBImpl::BGWorkFlush()
    env_->Schedule(&DBImpl::BGWorkFlush, this, Env::Priority::HIGH, this);
  }
  // ...
}
```

## DBImpl::BackgroundFlush()
`DBImpl::BGWorkFlush()` 函数最终会调用 `DBImpl::BackgroundFlush()` 函数
```c++
Status DBImpl::BackgroundFlush(bool* made_progress, JobContext* job_context,
                               LogBuffer* log_buffer, FlushReason* reason) {
  // ...
  while (!flush_queue_.empty()) {
    // 循环，如果 DBImpl::flush_queue_ 不为空
    // This cfd is already referenced
    // 调用 DBImpl::PopFirstFromFlushQueue() 从 DBImpl::flush_queue_
    // 头部取出 FlushRequest
    const FlushRequest& flush_req = PopFirstFromFlushQueue();
    superversion_contexts.clear();
    superversion_contexts.reserve(flush_req.size());

    // 遍历 FlushRequest
    for (const auto& iter : flush_req) {
      // 获取 ColumnFamilyData
      ColumnFamilyData* cfd = iter.first;
      if (cfd->IsDropped() || !cfd->imm()->IsFlushPending()) {
        // can't flush this CF, try next one
        // 如果 cf 已经 drop 或者 MemTableList::IsFlushPending() 返回 false
        if (cfd->Unref()) {
          // unref cfd ，如果返回 true ，删除 cfd
          delete cfd;
        }
        // 这个 cf 无需 flush ，继续处理下一个
        continue;
      }
      // 构造 SuperVersionContext 并插入 superversion_contexts
      superversion_contexts.emplace_back(SuperVersionContext(true));
      // bg_flush_args 插入 BGFlushArg
      bg_flush_args.emplace_back(cfd, iter.second,
                                 &(superversion_contexts.back()));
    }
    if (!bg_flush_args.empty()) {
      // 如果 bg_flush_args 不为空，因此一次只会成功处理一个 FlushRequest
      // 跳出循环
      break;
    }
  }

  if (!bg_flush_args.empty()) {
    // 如果 bg_flush_args 不为空
    // 调用 DBImpl::GetBGJobLimits() 获取 bg_job_limits
    auto bg_job_limits = GetBGJobLimits();
    for (const auto& arg : bg_flush_args) {
      // 遍历 bg_flush_args
      // 获取 ColumnFamilyData
      ColumnFamilyData* cfd = arg.cfd_;
      // ...
    }
    // 调用 DBImpl::FlushMemTablesToOutputFiles()
    status = FlushMemTablesToOutputFiles(bg_flush_args, made_progress,
                                         job_context, log_buffer);
    // ...
  }
  return status;
}
```
可以看到，该函数会从 flush 请求队列中找到一个 cf 的 flush 请求，然后调用 `DBImpl::FlushMemTablesToOutputFiles()` 将 cf 的 memtable 刷新到磁盘

`FlushMemTablesToOutputFiles()` 最终会调用 `DBImpl::FlushMemTableToOutputFile()`

另外，注意只有 `MemTableList::IsFlushPending()` 返回 true 的 cf 才会进行 flush ，其实现如下
```c++
// Returns true if there is at least one memtable on which flush has
// not yet started.
bool MemTableList::IsFlushPending() const {
  if ((flush_requested_ && num_flush_not_started_ >= 1) ||
      (num_flush_not_started_ >= min_write_buffer_number_to_merge_)) {
    assert(imm_flush_needed.load(std::memory_order_relaxed));
    return true;
  }
  return false;
}
```
`num_flush_not_started_` 是尚未 flush 的 memtable 的数量

`flush_requested_` 在调用 `MemTableList::FlushRequested()` 时会设置为 true ，所以可以看到下面几个触发 flush 的地方都会调用该函数
* `DBImpl::SwitchWAL()`
* `DBImpl::HandleWriteBufferFull()`
* `DBImpl::FlushMemTable()`
* `DBImpl::FlushAllCFs()`

而 `DBImpl::ScheduleFlushes()` 则没有设置该值，这应该是因为这种情况下 flush 的需求没有另外几种那么紧急。但是这种情况并不是就不会触发 flush 了，当 `num_flush_not_started_` 大于配置选项 `min_write_buffer_number_to_merge_` 设置的值时，同样也会触发 flush

`min_write_buffer_number_to_merge` 这个参数也会受 `max_write_buffer_number` 参数的影响，在检查 cf 的选项时， RocksDB 会保证 `min_write_buffer_number_to_merge` 不会超过 `max_write_buffer_number` - 1
```c++
ColumnFamilyOptions SanitizeOptions(const ImmutableDBOptions& db_options,
                                    const ColumnFamilyOptions& src) {
  ColumnFamilyOptions result = src;
  // ...
  result.min_write_buffer_number_to_merge =
      std::min(result.min_write_buffer_number_to_merge,
               result.max_write_buffer_number - 1);
  if (result.min_write_buffer_number_to_merge < 1) {
    result.min_write_buffer_number_to_merge = 1;
  }
  // ...
}
```

## DBImpl::FlushMemTableToOutputFile()
可以看到，最终的 flush 工作是通过 `FlushJob` 来完成的
```c++
Status DBImpl::FlushMemTableToOutputFile(
    ColumnFamilyData* cfd, const MutableCFOptions& mutable_cf_options,
    bool* made_progress, JobContext* job_context,
    SuperVersionContext* superversion_context, LogBuffer* log_buffer) {
  // ...
  // 创建 FlushJob
  FlushJob flush_job(
      dbname_, cfd, immutable_db_options_, mutable_cf_options,
      env_options_for_compaction_, versions_.get(), &mutex_, &shutting_down_,
      snapshot_seqs, earliest_write_conflict_snapshot, snapshot_checker,
      job_context, log_buffer, directories_.GetDbDir(), GetDataDir(cfd, 0U),
      GetCompressionFlush(*cfd->ioptions(), mutable_cf_options), stats_,
      &event_logger_, mutable_cf_options.report_bg_io_stats);

  FileMetaData file_meta;

  // 调用 FlushJob::PickMemTable()
  flush_job.PickMemTable();

  // ...

  // Within flush_job.Run, rocksdb may call event listener to notify
  // file creation and deletion.
  //
  // Note that flush_job.Run will unlock and lock the db_mutex,
  // and EventListener callback will be called when the db_mutex
  // is unlocked by the current thread.
  if (s.ok()) {
    // 如果没有错误
    // 调用 FlushJob::Run()
    s = flush_job.Run(&logs_with_prep_tracker_, &file_meta);
  } else {
    // 否则，调用 FlushJob::Cancel()
    flush_job.Cancel();
  }

  if (s.ok()) {
    // 如果成功了
    // 调用 DBImpl::InstallSuperVersionAndScheduleWork()
    InstallSuperVersionAndScheduleWork(cfd, superversion_context,
                                       mutable_cf_options);
    // ...
  }
  // ...
  return s;
}
```

`FlushJob` 通过 `FlushJob::PickMemTable()` 从 immutable memtable 列表中找到所有没有在 flush 的 memtable ，然后通过 `FlushJob::Run()` 将这些 memtable 刷到磁盘

## FlushJob::Run()
`FlushJob::Run()` 会调用 `FlushJob::WriteLevel0Table()` 最终将 memtable 持久化为 level 0 的 sst 文件
```c++
Status FlushJob::Run(LogsWithPrepTracker* prep_tracker,
                     FileMetaData* file_meta) {
  // ...
  // This will release and re-acquire the mutex.
  // 调用 FlushJob::WriteLevel0Table()
  Status s = WriteLevel0Table();

  if (s.ok() &&
      (shutting_down_->load(std::memory_order_acquire) || cfd_->IsDropped())) {
    // 如果状态为 ok 且 FlushJob::shshutting_down_ 为 true 或 cf 被删除
    // 设置状态为 Status::ShutdownInProgress()
    s = Status::ShutdownInProgress(
        "Database shutdown or Column family drop during flush");
  }

  if (!s.ok()) {
    // 如果状态有问题
    // 调用 MemTableList::RollbackMemtableFlush()
    cfd_->imm()->RollbackMemtableFlush(mems_, meta_.fd.GetNumber());
  } else {
    // 否则
    TEST_SYNC_POINT("FlushJob::InstallResults");
    // Replace immutable memtable with the generated Table
    // 调用 MemTableList::InstallMemtableFlushResults()
    s = cfd_->imm()->InstallMemtableFlushResults(
        cfd_, mutable_cf_options_, mems_, prep_tracker, versions_, db_mutex_,
        meta_.fd.GetNumber(), &job_context_->memtables_to_free, db_directory_,
        log_buffer_);
  }
  // ...
  return s;
}
```

## 触发条件
简单总结一下 flush 的触发条件
* 总的 memtable 大小超过 `db_write_buffer_size`
* 某个 memtable 大小超过 `write_buffer_size` ，另外还需要满足 cf 中待 flush 的 memtable 数量大于等于 `min_write_buffer_number_to_merge`
* WAL 文件大小超过 `max_total_wal_size` ，这时需要清理 WAL ，所以需要将 WAL 数据对应的 memtable 都刷盘
* 主动触发 flush 操作，例如调用 `DB::Flush()`/`DB::Resume()` 等接口

因此，也可以通过调整上面的参数来调节 RocksDB flush 的行为

## 参考
* [rocksdb SwitchMemtable 源码解析](https://bravoboy.github.io/2018/11/30/SwitchMemtable/)
* [MySQL · myrocks · MyRocks之memtable切换与刷盘](http://mysql.taobao.org/monthly/2017/06/08/)
* [MySQL · RocksDB · Memtable flush分析](http://mysql.taobao.org/monthly/2018/09/04/)
* [Write Buffer Manager](https://github.com/facebook/rocksdb/wiki/Write-Buffer-Manager)
* [Terminology](https://github.com/facebook/rocksdb/wiki/Terminology)
* [Background Error Handling](https://github.com/facebook/rocksdb/wiki/Background-Error-Handling)
