# 写入流程
主要关心的问题
* 写 WAL
* 写 Memtable
* Flush 和 Compaction 可以后续再讨论

## DBImpl::WriteImpl()
`DB::Put()` 和 `DB::Write()` 等写入接口最终都会调用 `DBImpl::WriteImpl()` 函数，实现在 `db/db_impl_write.cc` 中。写入的主要逻辑都实现在这里
```c++
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, WriteCallback* callback,
                         uint64_t* log_used, uint64_t log_ref,
                         bool disable_memtable, uint64_t* seq_used,
                         size_t batch_cnt,
                         PreReleaseCallback* pre_release_callback) {
    // ...
}
```

在写入前，首先会进行参数的合法性检验，随后会根据 `write_options` 和其他配置参数决定调用哪种写入实现，后面只考虑默认情况下的写入实现

为了提高写入性能， `DBImpl::WriteImpl()` 在写入时采用了 group commit 机制，代码也因此复杂了不少。在写入前，首先会创建一个 `WriteThread::Writer`，随后调用 `WriteThread::JoinBatchGroup()` 将其加入到 `Writer` 队列当中。`Writer` 队列被划分为一个一个 group ，每个 group 有一个 leader ，其他为 follower ， leader 负责批量写 WAL 。
```c++
WriteThread::Writer w(write_options, my_batch, callback, log_ref,
                      disable_memtable, batch_cnt, pre_release_callback);
// ...
// 调用 WriteThread::JoinBatchGroup() 将 w 注册到 DBImpl::write_thread_
write_thread_.JoinBatchGroup(&w);
```

`WriteThread::JoinBatchGroup()` 将 `Writer` 加入到队列后，会根据 `Writer` 在当前 group 的身份更新 `Writer` 的状态

一个 `WriteThread::Writer` 会处于以下状态之一
```c++
enum State : uint8_t {
  // The initial state of a writer.  This is a Writer that is
  // waiting in JoinBatchGroup.  This state can be left when another
  // thread informs the waiter that it has become a group leader
  // (-> STATE_GROUP_LEADER), when a leader that has chosen to be
  // non-parallel informs a follower that its writes have been committed
  // (-> STATE_COMPLETED), or when a leader that has chosen to perform
  // updates in parallel and needs this Writer to apply its batch (->
  // STATE_PARALLEL_FOLLOWER).
  STATE_INIT = 1,

  // The state used to inform a waiting Writer that it has become the
  // leader, and it should now build a write batch group.  Tricky:
  // this state is not used if newest_writer_ is empty when a writer
  // enqueues itself, because there is no need to wait (or even to
  // create the mutex and condvar used to wait) in that case.  This is
  // a terminal state unless the leader chooses to make this a parallel
  // batch, in which case the last parallel worker to finish will move
  // the leader to STATE_COMPLETED.
  STATE_GROUP_LEADER = 2,

  // The state used to inform a waiting writer that it has become the
  // leader of memtable writer group. The leader will either write
  // memtable for the whole group, or launch a parallel group write
  // to memtable by calling LaunchParallelMemTableWrite.
  STATE_MEMTABLE_WRITER_LEADER = 4,

  // The state used to inform a waiting writer that it has become a
  // parallel memtable writer. It can be the group leader who launch the
  // parallel writer group, or one of the followers. The writer should then
  // apply its batch to the memtable concurrently and call
  // CompleteParallelMemTableWriter.
  STATE_PARALLEL_MEMTABLE_WRITER = 8,

  // A follower whose writes have been applied, or a parallel leader
  // whose followers have all finished their work.  This is a terminal
  // state.
  STATE_COMPLETED = 16,

  // A state indicating that the thread may be waiting using StateMutex()
  // and StateCondVar()
  STATE_LOCKED_WAITING = 32,
};
```

`WriteThread::JoinBatchGroup()` 返回后，需要检查 `Writer::state` ，根据 `Writer` 的状态做相应的处理
```c++
// 检查 state 的状态，如果状态为 STATE_PARALLEL_MEMTABLE_WRITER ，说明 w
// 属于一个 parallel group 里的非 leader
if (w.state == WriteThread::STATE_PARALLEL_MEMTABLE_WRITER) {
  // we are a non-leader in a parallel group

  // 如果 Writer::ShouldWriteToMemtable() 返回 true
  if (w.ShouldWriteToMemtable()) {
    PERF_TIMER_STOP(write_pre_and_post_process_time);
    PERF_TIMER_GUARD(write_memtable_time);

    ColumnFamilyMemTablesImpl column_family_memtables(
        versions_->GetColumnFamilySet());
    // 调用 WriteBatchInternal::InsertInto() 插入 memtable
    w.status = WriteBatchInternal::InsertInto(
        &w, w.sequence, &column_family_memtables, &flush_scheduler_,
        write_options.ignore_missing_column_families, 0 /*log_number*/, this,
        true /*concurrent_memtable_writes*/, seq_per_batch_, w.batch_cnt);

    PERF_TIMER_START(write_pre_and_post_process_time);
  }

  // 调用 WriteThread::CompleteParallelMemTableWriter() ，会等待非最后一个退出的
  // writer 状态变为 STATE_COMPLETED ，如果返回 true
  // 则说明是最后一个退出的，状态仍然保持 STATE_PARALLEL_MEMTABLE_WRITER
  if (write_thread_.CompleteParallelMemTableWriter(&w)) {
    // we're responsible for exit batch group
    // 说明当前线程是最后一个退出的，需要负责整个 batch group 的退出工作
    // 遍历 write group 中的每一个 writer
    for (auto* writer : *(w.write_group)) {
      if (!writer->CallbackFailed() && writer->pre_release_callback) {
        assert(writer->sequence != kMaxSequenceNumber);
        // 调用 Writer::pre_release_callback
        Status ws = writer->pre_release_callback->Callback(writer->sequence,
                                                           disable_memtable);
        if (!ws.ok()) {
          // 如果返回错误，跳出循环
          status = ws;
          break;
        }
      }
    }
    // TODO(myabandeh): propagate status to write_group
    // 从 write_group 中获取 last_sequence
    auto last_sequence = w.write_group->last_sequence;
    // 调用 DBImpl::versions_ 的 VersionSet::SetLastSequence() 更新 sequence
    versions_->SetLastSequence(last_sequence);
    MemTableInsertStatusCheck(w.status);
    // 调用 WriteThread::ExitAsBatchGroupFollower()
    write_thread_.ExitAsBatchGroupFollower(&w);
  }
  assert(w.state == WriteThread::STATE_COMPLETED);
  // STATE_COMPLETED conditional below handles exit

  // 调用 Writer::FinalStatus() 获取当前 status
  status = w.FinalStatus();
}
```

如果状态为 `STATE_PARALLEL_MEMTABLE_WRITER`，则说明当前 `Writer` 为 follower ，不需要写 WAL ，但是也是需要调用 `WriteBatchInternal::InsertInto()` 写 memtable

写完 memtable 后，会调用 `WriteThread::CompleteParallelMemTableWriter()` 退出当前 group
* 如果 `Writer` 不是最后一个退出的，则等待状态变为 `STATE_COMPLETED`
* 如果 `Writer` 是最后一个退出的，则还需要负责执行 group 结束时的收尾工作，这时 `WriteThread::CompleteParallelMemTableWriter()` 会返回 true ，同时 `Writer` 的状态仍然保持 `STATE_PARALLEL_MEMTABLE_WRITER`

在收尾工作完成后，调用 `WriteThread::ExitAsBatchGroupFollower()` 最终退出 group ，同时其状态也变为 `STATE_COMPLETED`

如果 `Writer` 的状态为 `STATE_COMPLETED` ，则说明整个 group 的写入工作均已完成，可以返回
```c++
// 如果 state 为 STATE_COMPLETED
if (w.state == WriteThread::STATE_COMPLETED) {
  if (log_used != nullptr) {
    *log_used = w.log_used;
  }
  if (seq_used != nullptr) {
    *seq_used = w.sequence;
  }
  // write is complete and leader has updated sequence
  // 说明写入已经结束， leader 已经更新了 sequence ，直接返回 Writer::FinalStatus()
  return w.FinalStatus();
}
```

否则，则说明当前 `Writer` 已经成为 group 的 leader ，需要负责写 WAL ，首先会进行一些预处理工作，其中最主要的工作在 `DBImpl::PreprocessWrite()` 中完成
```c++
WriteContext write_context;
WriteThread::WriteGroup write_group;
bool in_parallel_group = false;
uint64_t last_sequence = kMaxSequenceNumber;
if (!two_write_queues_) {
  // 如果 DBImpl::two_write_queues_ 为 false ，设置 last_sequence 为 DBImpl::versions_
  // 的 VersionSet::LastSequence()
  last_sequence = versions_->LastSequence();
}

// 加锁 DBImpl::mutex_
mutex_.Lock();

// 如果用户设置了 WriteOptions::sync ，设置 need_log_sync 为 true
bool need_log_sync = write_options.sync;
bool need_log_dir_sync = need_log_sync && !log_dir_synced_;
if (!two_write_queues_ || !disable_memtable) {
  // With concurrent writes we do preprocess only in the write thread that
  // also does write to memtable to avoid sync issue on shared data structure
  // with the other thread
  // 如果 DBImpl::two_write_queues_ 为 false 且 disable_memtable 为 false

  // PreprocessWrite does its own perf timing.
  PERF_TIMER_STOP(write_pre_and_post_process_time);

  // 调用 DBImpl::PreprocessWrite() 做些写入前的处理
  // 涉及 WAL 的切换 和 sync 等
  status = PreprocessWrite(write_options, &need_log_sync, &write_context);

  PERF_TIMER_START(write_pre_and_post_process_time);
}
// 从 DBImpl::logs_ 中获取队尾的 log 的 writer
log::Writer* log_writer = logs_.back().writer;

// 解锁 DBImpl::mutex_
mutex_.Unlock();

// 调用 WriteThread::EnterAsBatchGroupLeader() 设置 write_group ，返回
// write_group 的大小并设置到 DBImpl::last_batch_group_size_ 
last_batch_group_size_ =
    write_thread_.EnterAsBatchGroupLeader(&w, &write_group);
```
TODO 简要介绍 `DBImpl::PreprocessWrite()` 的作用

完成预处理之后，会调用 `WriteThread::EnterAsBatchGroupLeader()` 初始化 leader 和 group ，然后就可以开始 leader 的写入工作了

首先会判断能否并发写 memtable ，可以看到，当有 merge 操作时，是不能并发写 memtable 的
```c++
// Rules for when we can update the memtable concurrently
// 1. supported by memtable
// 2. Puts are not okay if inplace_update_support
// 3. Merges are not okay
//
// Rules 1..2 are enforced by checking the options
// during startup (CheckConcurrentWritesSupported), so if
// options.allow_concurrent_memtable_write is true then they can be
// assumed to be true.  Rule 3 is checked for each batch.  We could
// relax rules 2 if we could prevent write batches from referring
// more than once to a particular key.
// 判断能否并发写 memtable, 如果可以，设置 parallel 为 true
bool parallel = immutable_db_options_.allow_concurrent_memtable_write &&
                write_group.size > 1;
size_t total_count = 0;
size_t valid_batches = 0;
size_t total_byte_size = 0;
// 遍历 write_group 中的每一个 writer
for (auto* writer : write_group) {
  if (writer->CheckCallback(this)) {
    // 如果 Writer::CheckCallback() 返回 true
    // 增加 Writer::batch_cnt 到 valid_batches
    valid_batches += writer->batch_cnt;
    if (writer->ShouldWriteToMemtable()) {
      // 如果 Writer::ShouldWriteToMemtable() 返回 true
      // 增加待写入项总数到 total_count
      total_count += WriteBatchInternal::Count(writer->batch);
      // 如果 batch 的 WriteBatch::HasMerge() 为 true ，则不能并发写
      // 这时 parallel 改为 false
      parallel = parallel && !writer->batch->HasMerge();
    }

    total_byte_size = WriteBatchInternal::AppendedByteSize(
        total_byte_size, WriteBatchInternal::ByteSize(writer->batch));
  }
}
```

随后会写 WAL ，这里只关注常规情况
```c++
if (!two_write_queues_) {
  // 如果 DBImpl::two_write_queues_ 为 false
  if (status.ok() && !write_options.disableWAL) {
    // 如果 WriteOptions::disableWAL 为 false
    PERF_TIMER_GUARD(write_wal_time);
    // 调用 DBImpl::WriteToWAL()
    status = WriteToWAL(write_group, log_writer, log_used, need_log_sync,
                        need_log_dir_sync, last_sequence + 1);
  }
}
```

写入 WAL 完成后就会开始写 memtable ，写入前会根据 `parallel` 判断是否能并发写 memtable
```c++
if (!parallel) {
  // 如果 parallel 为 false ，不能并行写
  // w.sequence will be set inside InsertInto
  // 调用 WriteBatchInternal::InsertInto()
  w.status = WriteBatchInternal::InsertInto(
      write_group, current_sequence, column_family_memtables_.get(),
      &flush_scheduler_, write_options.ignore_missing_column_families,
      0 /*recovery_log_number*/, this, parallel, seq_per_batch_,
      batch_per_txn_);
} else {
  // 设置 next_sequence 为 current_sequence
  SequenceNumber next_sequence = current_sequence;
  // Note: the logic for advancing seq here must be consistent with the
  // logic in WriteBatchInternal::InsertInto(write_group...) as well as
  // with WriteBatchInternal::InsertInto(write_batch...) that is called on
  // the merged batch during recovery from the WAL.
  // 遍历 write_group 中的所有 writer
  for (auto* writer : write_group) {
    if (writer->CallbackFailed()) {
      // 如果 Writer::CallbackFailed() 返回 true ，直接处理下一个 writer
      continue;
    }
    // Writer::sequence 设置为 next_sequence
    writer->sequence = next_sequence;
    // 根据 DBImpl::seq_per_batch_ 增加 next_sequence
    if (seq_per_batch_) {
      assert(writer->batch_cnt);
      // 如果 DBImpl::seq_per_batch_ 为 true 则 next_sequence 增加 batch 数
      next_sequence += writer->batch_cnt;
    } else if (writer->ShouldWriteToMemtable()) {
      // 否则， next_sequence 增加 batch 中插入的条数
      next_sequence += WriteBatchInternal::Count(writer->batch);
    }
  }
  // 设置 WriteGroup::last_sequence 为 last_sequence
  write_group.last_sequence = last_sequence;
  // 设置 WriteGroup::running 为 WriteGroup::size
  write_group.running.store(static_cast<uint32_t>(write_group.size),
                            std::memory_order_relaxed);
  // 调用 WriteThread::LaunchParallelMemTableWriters() ，将 write_group 中
  // 的每个 writer 的 state 设为 STATE_PARALLEL_MEMTABLE_WRITER
  write_thread_.LaunchParallelMemTableWriters(&write_group);
  // 设置 in_parallel_group 为 true ，说明处于并行写 memtable 的 group 中
  in_parallel_group = true;

  // Each parallel follower is doing each own writes. The leader should
  // also do its own.
  // 这时每个并行的 follower 都会各自写 memtable , leader 也同样写 memtable
  if (w.ShouldWriteToMemtable()) {
    ColumnFamilyMemTablesImpl column_family_memtables(
        versions_->GetColumnFamilySet());
    assert(w.sequence == current_sequence);
    // 调用 WriteBatchInternal::InsertInto() 写 memtable
    w.status = WriteBatchInternal::InsertInto(
        &w, w.sequence, &column_family_memtables, &flush_scheduler_,
        write_options.ignore_missing_column_families, 0 /*log_number*/,
        this, true /*concurrent_memtable_writes*/, seq_per_batch_,
        w.batch_cnt, batch_per_txn_);
  }
}
```
如果不可以并发写，则 leader 会负责将整个 group 中的数据都写入 memtable

如果可以并发写，则会调用 `WriteThread::LaunchParallelMemTableWriters()` 将 group 中的所有 `Writer` 的状态更新为 `STATE_PARALLEL_MEMTABLE_WRITER` 并将所有 follower 都唤醒

唤醒 follower 后，所有 follower 都会各自并行地写 memtable ，同时 leader 也会写 memtable 。写完 memtable 后，各个 follower 就可以开始退出 group ， 而 leader 还不能直接退出

如果 `need_log_sync` 为 true ，则 leader 还需要将 WAL 刷盘
```c++
if (need_log_sync) {
  // 如果 need_log_sync 为 true
  // 加锁 DBImpl::mutex_
  mutex_.Lock();
  // 调用 DBImpl::MarkLogsSynced() 通知 log 已经 sync 了并且将已经 sync
  // 的旧 log 移动到 DBImpl::logs_to_free_
  MarkLogsSynced(logfile_number_, need_log_dir_sync, status);
  // 解锁 DBImpl::mutex_
  mutex_.Unlock();
  // Requesting sync with two_write_queues_ is expected to be very rare. We
  // hence provide a simple implementation that is not necessarily efficient.
  if (two_write_queues_) {
    // 如果 DBImpl::two_write_queues_ 为 true
    if (manual_wal_flush_) {
      // 如果 DBImpl::manual_wal_flush_ 为 true, 调用 DBImpl::FlushWAL()
      status = FlushWAL(true);
    } else {
      // 否则调用 DBImpl::SyncWAL()
      status = SyncWAL();
    }
  }
}
```

完成 WAL 刷盘后， leader 也可退出 group 了
```c+++
bool should_exit_batch_group = true;
if (in_parallel_group) {
  // CompleteParallelWorker returns true if this thread should
  // handle exit, false means somebody else did
  // 如果在 parallel group 里，调用 WriteThread::CompleteParallelMemTableWriter()
  // 并将返回值设置到 should_exit_batch_group ，如果返回 true 则说明 leader 是最后
  // 一个退出的，此时 should_exit_batch_group 也为 true
  should_exit_batch_group = write_thread_.CompleteParallelMemTableWriter(&w);
}
if (should_exit_batch_group) {
  // 如果 should_exit_batch_group 为 true 则说明 leader 负责最后的清理
  if (status.ok()) {
    // 遍历 write_group 中的每个 writer
    for (auto* writer : write_group) {
      if (!writer->CallbackFailed() && writer->pre_release_callback) {
        assert(writer->sequence != kMaxSequenceNumber);
        // 如有必要，调用 Writer::pre_release_callback
        Status ws = writer->pre_release_callback->Callback(writer->sequence,
                                                           disable_memtable);
        if (!ws.ok()) {
          status = ws;
          break;
        }
      }
    }
    // 调用 VersionSet::SetLastSequence() 更新 sequence number
    versions_->SetLastSequence(last_sequence);
  }
  MemTableInsertStatusCheck(w.status);
  // 调用 WriteThread::ExitAsBatchGroupLeader()
  write_thread_.ExitAsBatchGroupLeader(write_group, status);
}
```
如果 `in_parallel_group` 为 true ，则说明进行了并发写 memtable ，则同样需要调用 `WriteThread::CompleteParallelMemTableWriter()` 进行退出

如果 `WriteThread::CompleteParallelMemTableWriter()` 返回 true 则说明 leader 是最后一个退出当前 group 的，需要进行一些收尾工作。这些收尾工作和前面 follower 做的收尾工作雷同

在完成收尾工作之后，会调用 `WriteThread::ExitAsBatchGroupLeader()` 最终退出 group

需要注意的是，在写入时，每一条数据都有对应的 `SequenceNumber`
* 写入 batch 前会统计 batch 中有几条数据并提前算好 `SequenceNumber` 应该增加的量
* 写 WAL 时，会将整个 batch 起始的 `SequenceNumber` 设置到 batch 中，然后写入 WAL
* 写 memtable 时，每插入一条数据都会增加 `SequenceNumber` 并将 `SequenceNumber` 拼装记录到内部的 key 中

## WriteThread::JoinBatchGroup()
实现在 `db/write_thread.cc` 中
```c++
// 将 w 注册到 batch group 中，并一直等待直到调用方需要做进一步操作，会更新 w 中
// 的状态值 state
void WriteThread::JoinBatchGroup(Writer* w) {
  TEST_SYNC_POINT_CALLBACK("WriteThread::JoinBatchGroup:Start", w);
  assert(w->batch != nullptr);

  // 调用 WriteThread::LinkOne() ，如果 w 被放到 leader 的位置，返回 true
  bool linked_as_leader = LinkOne(w, &newest_writer_);

  if (linked_as_leader) {
    // 如果被放到 leader 的位置，即称为 leader ，则调用 WriteThread::SetState() 将
    // state 更新为 STATE_GROUP_LEADER
    SetState(w, STATE_GROUP_LEADER);
  }

  TEST_SYNC_POINT_CALLBACK("WriteThread::JoinBatchGroup:Wait", w);

  if (!linked_as_leader) {
    /**
     * Wait util:
     * 1) An existing leader pick us as the new leader when it finishes
     * 2) An existing leader pick us as its follewer and
     * 2.1) finishes the memtable writes on our behalf
     * 2.2) Or tell us to finish the memtable writes in pralallel
     * 3) (pipelined write) An existing leader pick us as its follower and
     *    finish book-keeping and WAL write for us, enqueue us as pending
     *    memtable writer, and
     * 3.1) we become memtable writer group leader, or
     * 3.2) an existing memtable writer group leader tell us to finish memtable
     *      writes in parallel.
     */
    TEST_SYNC_POINT_CALLBACK("WriteThread::JoinBatchGroup:BeganWaiting", w);
    // 否则，说明不是 leader ，则等待直到 w 的 state 发生变化
    AwaitState(w, STATE_GROUP_LEADER | STATE_MEMTABLE_WRITER_LEADER |
                      STATE_PARALLEL_MEMTABLE_WRITER | STATE_COMPLETED,
               &jbg_ctx);
    TEST_SYNC_POINT_CALLBACK("WriteThread::JoinBatchGroup:DoneWaiting", w);
  }
}
```
`Writer` 有两个指针，其中 `link_older` 指向前一个节点， `link_newer` 指向下一个节点，以此构成一个双向链表

`WriteThread::LinkOne()` 会尝试将 `Writer` 插入 `Writer` 链表尾部，其中 `newest_writer_` 指向链表尾部，如果 `Writer` 插入后成为新的链表头，则返回 true ，说明成为了新的 leader

如果当前 `Writer` 成为了 leader ，则会更新状态为 STATE_GROUP_LEADER ，否则，就会调用 `WriteThread::AwaitState()` 等待状态发生变化

`WriteThread::LinkOne()` 的实现如下
```c++
// 将 writer w 加入到链表尾部，如果 w 直接设置到了 leader 的位置，即队头
// 则返回 true
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  // 注意这里只是检查 newest_writer 不能是空指针，而不是里面的值不能是空指针
  assert(newest_writer != nullptr);
  assert(w->state == STATE_INIT);
  // 获取旧的队尾
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    // If write stall in effect, and w->no_slowdown is not true,
    // block here until stall is cleared. If its true, then return
    // immediately
    if (writers == &write_stall_dummy_) {
      if (w->no_slowdown) {
        w->status = Status::Incomplete("Write stall");
        SetState(w, STATE_COMPLETED);
        return false;
      }
      // Since no_slowdown is false, wait here to be notified of the write
      // stall clearing
      {
        MutexLock lock(&stall_mu_);
        writers = newest_writer->load(std::memory_order_relaxed);
        if (writers == &write_stall_dummy_) {
          stall_cv_.Wait();
          // Load newest_writers_ again since it may have changed
          writers = newest_writer->load(std::memory_order_relaxed);
          continue;
        }
      }
    }
    // w 的 link_older 指向旧尾节点
    w->link_older = writers;
    // 将尾节点改为 w
    if (newest_writer->compare_exchange_weak(writers, w)) {
      // 如果最新的队尾和 writers 一样，就尝试将 newest_writer 指向 w ，
      // 即将 w 作为新的队尾，如果不一样，则 writers 会更新为最新的队尾
      // 如果旧尾节点的值为 nullptr 则说明队列之前是空的了， w 被放到
      // 了 leader 的位置，返回 true
      return (writers == nullptr);
    }
  }
}
```
注意插入时只需要设置 `link_older` 指针，而 `link_newer` 指针可以在后面反向遍历链表的时候再设置好

`WriteThread::AwaitState()` 的实现如下
```c++
uint8_t WriteThread::AwaitState(Writer* w, uint8_t goal_mask,
                                AdaptationContext* ctx) {
  uint8_t state;

  // 1. Busy loop using "pause" for 1 micro sec
  // 2. Else SOMETIMES busy loop using "yield" for 100 micro sec (default)
  // 3. Else blocking wait

  // On a modern Xeon each loop takes about 7 nanoseconds (most of which
  // is the effect of the pause instruction), so 200 iterations is a bit
  // more than a microsecond.  This is long enough that waits longer than
  // this can amortize the cost of accessing the clock and yielding.
  for (uint32_t tries = 0; tries < 200; ++tries) {
    state = w->state.load(std::memory_order_acquire);
    if ((state & goal_mask) != 0) {
      return state;
    }
    port::AsmVolatilePause();
  }

  // This is below the fast path, so that the stat is zero when all writes are
  // from the same thread.
  PERF_TIMER_GUARD(write_thread_wait_nanos);

  const size_t kMaxSlowYieldsWhileSpinning = 3;

  // Whether the yield approach has any credit in this context. The credit is
  // added by yield being succesfull before timing out, and decreased otherwise.
  auto& yield_credit = ctx->value;
  // Update the yield_credit based on sample runs or right after a hard failure
  bool update_ctx = false;
  // Should we reinforce the yield credit
  bool would_spin_again = false;
  // The samling base for updating the yeild credit. The sampling rate would be
  // 1/sampling_base.
  const int sampling_base = 256;

  if (max_yield_usec_ > 0) {
    update_ctx = Random::GetTLSInstance()->OneIn(sampling_base);

    if (update_ctx || yield_credit.load(std::memory_order_relaxed) >= 0) {
      // we're updating the adaptation statistics, or spinning has >
      // 50% chance of being shorter than max_yield_usec_ and causing no
      // involuntary context switches
      auto spin_begin = std::chrono::steady_clock::now();

      // this variable doesn't include the final yield (if any) that
      // causes the goal to be met
      size_t slow_yield_count = 0;

      auto iter_begin = spin_begin;
      while ((iter_begin - spin_begin) <=
             std::chrono::microseconds(max_yield_usec_)) {
        std::this_thread::yield();

        state = w->state.load(std::memory_order_acquire);
        if ((state & goal_mask) != 0) {
          // success
          would_spin_again = true;
          break;
        }

        auto now = std::chrono::steady_clock::now();
        if (now == iter_begin ||
            now - iter_begin >= std::chrono::microseconds(slow_yield_usec_)) {
          // conservatively count it as a slow yield if our clock isn't
          // accurate enough to measure the yield duration
          ++slow_yield_count;
          if (slow_yield_count >= kMaxSlowYieldsWhileSpinning) {
            // Not just one ivcsw, but several.  Immediately update yield_credit
            // and fall back to blocking
            update_ctx = true;
            break;
          }
        }
        iter_begin = now;
      }
    }
  }

  if ((state & goal_mask) == 0) {
    state = BlockingAwaitState(w, goal_mask);
  }

  if (update_ctx) {
    // Since our update is sample based, it is ok if a thread overwrites the
    // updates by other threads. Thus the update does not have to be atomic.
    auto v = yield_credit.load(std::memory_order_relaxed);
    // fixed point exponential decay with decay constant 1/1024, with +1
    // and -1 scaled to avoid overflow for int32_t
    //
    // On each update the positive credit is decayed by a facor of 1/1024 (i.e.,
    // 0.1%). If the sampled yield was successful, the credit is also increased
    // by X. Setting X=2^17 ensures that the credit never exceeds
    // 2^17*2^10=2^27, which is lower than 2^31 the upperbound of int32_t. Same
    // logic applies to negative credits.
    v = v - (v / 1024) + (would_spin_again ? 1 : -1) * 131072;
    yield_credit.store(v, std::memory_order_relaxed);
  }

  assert((state & goal_mask) != 0);
  return state;
}
```
这里将等待分成了3步来做，以减少条件锁的使用以及尽可能减少 Context Switches
* Busy loop
* Short-Wait: Loop + `std::this_thread::yield()`
* Long-Wait: `std::condition_variable::wait()`

其中在 `WriteThread::BlockingAwaitState()` 进行 Long-Wait ，并且会将 `Writer` 的状态设置为 `STATE_LOCKED_WAITING`
```c++
uint8_t WriteThread::BlockingAwaitState(Writer* w, uint8_t goal_mask) {
  // We're going to block.  Lazily create the mutex.  We guarantee
  // propagation of this construction to the waker via the
  // STATE_LOCKED_WAITING state.  The waker won't try to touch the mutex
  // or the condvar unless they CAS away the STATE_LOCKED_WAITING that
  // we install below.
  w->CreateMutex();

  auto state = w->state.load(std::memory_order_acquire);
  assert(state != STATE_LOCKED_WAITING);
  if ((state & goal_mask) == 0 &&
      w->state.compare_exchange_strong(state, STATE_LOCKED_WAITING)) {
    // we have permission (and an obligation) to use StateMutex
    std::unique_lock<std::mutex> guard(w->StateMutex());
    w->StateCV().wait(guard, [w] {
      return w->state.load(std::memory_order_relaxed) != STATE_LOCKED_WAITING;
    });
    state = w->state.load(std::memory_order_relaxed);
  }
  // else tricky.  Goal is met or CAS failed.  In the latter case the waker
  // must have changed the state, and compare_exchange_strong has updated
  // our local variable with the new one.  At the moment WriteThread never
  // waits for a transition across intermediate states, so we know that
  // since a state change has occurred the goal must have been met.
  assert((state & goal_mask) != 0);
  return state;
}
```

更新状态的实现如下
```c++
void WriteThread::SetState(Writer* w, uint8_t new_state) {
  auto state = w->state.load(std::memory_order_acquire);
  if (state == STATE_LOCKED_WAITING ||
      !w->state.compare_exchange_strong(state, new_state)) {
    assert(state == STATE_LOCKED_WAITING);

    std::lock_guard<std::mutex> guard(w->StateMutex());
    assert(w->state.load(std::memory_order_relaxed) != new_state);
    w->state.store(new_state, std::memory_order_relaxed);
    w->StateCV().notify_one();
  }
}
```

## WriteThread::CompleteParallelMemTableWriter()
实现如下
```c++
// This method is called by both the leader and parallel followers
// leader 和 follower 写完 memtable 后调用该函数
bool WriteThread::CompleteParallelMemTableWriter(Writer* w) {

  auto* write_group = w->write_group;
  if (!w->status.ok()) {
    std::lock_guard<std::mutex> guard(write_group->leader->StateMutex());
    // 如果 w 的 status 不成功，则将 write_group 的 status 设置为 w 的 status
    write_group->status = w->status;
  }

  // 减少 WriteGroup::running (该变量是原子变量)
  if (write_group->running-- > 1) {
    // we're not the last one
    // 如果不是最后一个 writer ，则等待 state 变为 STATE_COMPLETED
    AwaitState(w, STATE_COMPLETED, &cpmtw_ctx);
    // 返回 false
    return false;
  }
  // else we're the last parallel worker and should perform exit duties.
  // 否则说明这是最后一个 parallel worker ，将 w 的 status 设置为 write_group 的 status
  w->status = write_group->status;
  // 返回 true
  return true;
}
```
在这里，如果 `Writer` 不是最后一个退出的，就会一直等待状态变为 `STATE_COMPLETED` 并返回 false

否则，说明是最后一个退出的，返回 true

## WriteThread::EnterAsBatchGroupLeader()
实现如下，主要工作包括
* 将 group 的 leader 设置为当前 `Writer`
* 调用 `WriteThread::CreateMissingNewerLinks()` 设置好 `link_newer` 指针
* 从 leader 开始向后遍历链表，将连续的符合条件的 `Writer` 都加入到当前的 group
* 最终 `last_writer` 会指向 group 中最后一个 `Writer`
```c++
// 设置 leader 和 write group
size_t WriteThread::EnterAsBatchGroupLeader(Writer* leader,
                                            WriteGroup* write_group) {
  assert(leader->link_older == nullptr);
  assert(leader->batch != nullptr);
  assert(write_group != nullptr);

  size_t size = WriteBatchInternal::ByteSize(leader->batch);

  // Allow the group to grow up to a maximum size, but if the
  // original write is small, limit the growth so we do not slow
  // down the small write too much.
  // 计算 write group 的最大字节数 max_size
  size_t max_size = 1 << 20;
  if (size <= (128 << 10)) {
    max_size = size + (128 << 10);
  }

  // 设置 leader->write group
  leader->write_group = write_group;
  // 设置 write_group->leader
  write_group->leader = leader;
  // 将 writer_group->last_writer 初始化为 leader
  write_group->last_writer = leader;
  // write_group->size 初始化为 1
  write_group->size = 1;
  // 获取队尾到 newest_writer
  Writer* newest_writer = newest_writer_.load(std::memory_order_acquire);

  // This is safe regardless of any db mutex status of the caller. Previous
  // calls to ExitAsGroupLeader either didn't call CreateMissingNewerLinks
  // (they emptied the list and then we added ourself as leader) or had to
  // explicitly wake us up (the list was non-empty when we added ourself,
  // so we have already received our MarkJoined).
  // 设置 newest_writer 到队头的 link_newer 指针
  CreateMissingNewerLinks(newest_writer);

  // Tricky. Iteration start (leader) is exclusive and finish
  // (newest_writer) is inclusive. Iteration goes from old to new.
  // 将队列中的直到 newest_writer (包括 newest_writer) 的 writer 加入
  // write group 当中，当一定条件满足时，会提前退出循环
  // 从 leader 开始
  Writer* w = leader;
  while (w != newest_writer) {
    // 获取 link_newer 作为将要加入 write group 的 writer
    w = w->link_newer;

    // 进行一系列检查确定 write group 是否仍然能加入 writer
    if (w->sync && !leader->sync) {
      // Do not include a sync write into a batch handled by a non-sync write.
      break;
    }

    if (w->no_slowdown != leader->no_slowdown) {
      // Do not mix writes that are ok with delays with the ones that
      // request fail on delays.
      break;
    }

    if (!w->disable_wal && leader->disable_wal) {
      // Do not include a write that needs WAL into a batch that has
      // WAL disabled.
      break;
    }

    if (w->batch == nullptr) {
      // Do not include those writes with nullptr batch. Those are not writes,
      // those are something else. They want to be alone
      break;
    }

    if (w->callback != nullptr && !w->callback->AllowWriteBatching()) {
      // dont batch writes that don't want to be batched
      break;
    }

    // write group 的字节数不能超过 max_size
    auto batch_size = WriteBatchInternal::ByteSize(w->batch);
    if (size + batch_size > max_size) {
      // Do not make batch too big
      break;
    }

    // 设置 w 的 write_group
    w->write_group = write_group;
    size += batch_size;
    // write_group->last_writer 更新为 w
    write_group->last_writer = w;
    // 增加 write_group->size
    write_group->size++;
  }
  TEST_SYNC_POINT_CALLBACK("WriteThread::EnterAsBatchGroupLeader:End", w);
  // 返回 write_group 的字节数
  return size;
}
```

## WriteThread::LaunchParallelMemTableWriters()
该函数会将 group 中所有的 `Writer` 状态都更新为 `STATE_PARALLEL_MEMTABLE_WRITER` ，包括 leader 自身，随后 group 中所有处于等待的 `Writer` 都被唤醒
```c++
void WriteThread::LaunchParallelMemTableWriters(WriteGroup* write_group) {
  assert(write_group != nullptr);
  write_group->running.store(write_group->size);
  for (auto w : *write_group) {
    SetState(w, STATE_PARALLEL_MEMTABLE_WRITER);
  }
}
```

## WriteThread::ExitAsBatchGroupFollower()
在 follower 是最后一个退出 group 的 `Writer` 时调用，实现如下
```c++
void WriteThread::ExitAsBatchGroupFollower(Writer* w) {
  // 获取 w 的 write_group
  auto* write_group = w->write_group;

  assert(w->state == STATE_PARALLEL_MEMTABLE_WRITER);
  assert(write_group->status.ok());
  // 调用 WriteThread::ExitAsBatchGroupLeader(write_group) 执行清理
  // 会将非 leader 的 writer 的状态都设置为 STATE_COMPLETED
  ExitAsBatchGroupLeader(*write_group, write_group->status);
  assert(w->status.ok());
  assert(w->state == STATE_COMPLETED);
  // 将 write_group 的 leader 状态设置为 STATE_COMPLETED
  SetState(write_group->leader, STATE_COMPLETED);
}
实际上又会调用 `WriteThread::ExitAsBatchGroupLeader()`

## WriteThread::ExitAsBatchGroupLeader()
实现如下，主要关心非 pipelined write 的情况。该函数的主要工作包括
* 将当前 group 的 `Writer` 从 `Writer` 链表中移除
* 如果当前 group 后面有新的 `Writer` 在链表中，则将头部的 `Writer` 设置为 leader ，因为该 `Writer` 已经处于等待状态，是不会将自己设置为 leader 的
* 将除 leader 外的 `Writer` 状态都设置为 `STATE_COMPLETED`
```c++
// 执行退出清理，由最后一个退出的线程负责调用
void WriteThread::ExitAsBatchGroupLeader(WriteGroup& write_group,
                                         Status status) {
  // 获取 write group 的 leader
  Writer* leader = write_group.leader;
  // 获取 write group 的 last_writer ，就是 write group 的队尾
  Writer* last_writer = write_group.last_writer;
  assert(leader->link_older == nullptr);

  // Propagate memtable write error to the whole group.
  if (status.ok() && !write_group.status.ok()) {
    // 如果 write_group 有错误则将错误码设置到 status
    status = write_group.status;
  }

  if (enable_pipelined_write_) {
    // 如果 WriteThread::enable_pipelined_write_ 为 true ，暂不分析
    // ...
  } else {
    // 如果不是 pipeline write
    // 从 WriteThread::newest_writer_ 中读取出 head ，实际上就是
    // 最新加入链表的 writer
    Writer* head = newest_writer_.load(std::memory_order_acquire);
    // 如果 head 不是 last_writer ，说明当前 group 的最后一个 writer 不在
    // 队尾，后面有新的 writer 加入了链表，这时需要将 last_writer 后的第一个
    // writer 设置为下一个 group 的 leader
    // 如果 head 是 last_writer ，则通过 cas 将 WriteThread::newest_writer_
    // 设置为 nullptr ，相当于清空链表
    if (head != last_writer ||
        !newest_writer_.compare_exchange_strong(head, nullptr)) {
      // Either w wasn't the head during the load(), or it was the head
      // during the load() but somebody else pushed onto the list before
      // we did the compare_exchange_strong (causing it to fail).  In the
      // latter case compare_exchange_strong has the effect of re-reading
      // its first param (head).  No need to retry a failing CAS, because
      // only a departing leader (which we are at the moment) can remove
      // nodes from the list.
      assert(head != last_writer);

      // After walking link_older starting from head (if not already done)
      // we will be able to traverse w->link_newer below. This function
      // can only be called from an active leader, only a leader can
      // clear newest_writer_, we didn't, and only a clear newest_writer_
      // could cause the next leader to start their work without a call
      // to MarkJoined, so we can definitely conclude that no other leader
      // work is going on here (with or without db mutex).
      // 调用 WriteThread::CreateMissingNewerLinks() 生成完整的 link_newer
      // 链表，使得可以正向遍历整个 writer 链表
      CreateMissingNewerLinks(head);
      assert(last_writer->link_newer->link_older == last_writer);
      // last_writer->link_newer 这时指向了下一个 group 的 leader ，将该 leader
      // 的 link_older 设置为 nullptr ，断开和当前 group 的连接
      last_writer->link_newer->link_older = nullptr;

      // Next leader didn't self-identify, because newest_writer_ wasn't
      // nullptr when they enqueued (we were definitely enqueued before them
      // and are still in the list).  That means leader handoff occurs when
      // we call MarkJoined
      // 将 last_writer->link_newer 即下一个 group 的 leader 的状态
      // 设置为 STATE_GROUP_LEADER ，唤醒下一个 write group 的 leader
      SetState(last_writer->link_newer, STATE_GROUP_LEADER);
    }
    // else nobody else was waiting, although there might already be a new
    // leader now

    while (last_writer != leader) {
      // 从 laster_writer 开始反向遍历链表直到 leader ，更新其状态
      // 设置 last_writer 的状态为 status
      last_writer->status = status;
      // we need to read link_older before calling SetState, because as soon
      // as it is marked committed the other thread's Await may return and
      // deallocate the Writer.
      // 从 last_writer->link_older 读取下一个 writer
      auto next = last_writer->link_older;
      // 将 last_writer 的状态设置为 STATE_COMPLETED
      SetState(last_writer, STATE_COMPLETED);

      // last_writer 设置为 next
      last_writer = next;
    }
  }
}
```


## 参考
* [【Rocksdb实现及优化分析】 JoinBatchGroup](http://kernelmaker.github.io/Rocksdb_Study_1)
* [RocksDB Writer](https://zhuanlan.zhihu.com/p/45952903)
* [MySQL · myrocks · myrocks写入分析](http://mysql.taobao.org/monthly/2017/07/05/)
* [MySQL · RocksDB · 写入逻辑的实现](http://mysql.taobao.org/monthly/2018/07/04/)
* [MySQL · RocksDB · Memtable flush分析](http://mysql.taobao.org/monthly/2018/09/04/)
* [MySQL · RocksDB · Level Compact 分析](http://mysql.taobao.org/monthly/2018/10/08/)
* [MySQL · myrocks · MyRocks之memtable切换与刷盘](http://mysql.taobao.org/monthly/2017/06/08/)
* [MySQL · RocksDB · MANIFEST文件介绍](http://mysql.taobao.org/monthly/2018/05/08/)
* [MySQL · RocksDB · WAL(WriteAheadLog)介绍](http://mysql.taobao.org/monthly/2018/04/09/)
* [MySQL · RocksDB · 数据的读取(一)](http://mysql.taobao.org/monthly/2018/11/05/)
* [MySQL · RocksDB · 数据的读取(二)](http://mysql.taobao.org/monthly/2018/12/08/)
