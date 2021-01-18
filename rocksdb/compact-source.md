# Level Compaction 分析

主要关心的问题
* 触发条件
* 调用入口
* 执行流程

## DBImpl::MaybeScheduleFlushOrCompaction()
RocksDB compaction 的调用情况分为手动和自动两种
* 手动的情况是用户调用 `DB::CompactRange()` 和 `DB::CompactFiles()` 等会触发 compaction 的接口
* 自动的情况是调用 `DBImpl::MaybeScheduleFlushOrCompaction()` 这个函数

`DBImpl::MaybeScheduleFlushOrCompaction()` 这个函数在分析 RocksDB 的 flush 操作时我们已经了解到，触发 flush 时都会调用这个函数。这个函数除了可能会触发 flush 外，也会检查是否需要进行 compaction ，如果需要，就触发 compaction 。相关的实现如下

```c++
void DBImpl::MaybeScheduleFlushOrCompaction() {
  // ...
  if (HasExclusiveManualCompaction()) {
    // only manual compactions are allowed to run. don't schedule automatic
    // compactions
    TEST_SYNC_POINT("DBImpl::MaybeScheduleFlushOrCompaction:Conflict");
    // 如果 DBImpl::HasExclusiveManualCompaction() 返回 true
    // 返回
    return;
  }

  while (bg_compaction_scheduled_ < bg_job_limits.max_compactions &&
         unscheduled_compactions_ > 0) {
    // 如果 DBImpl::bg_compaction_scheduled_ < BGJobLimits::max_compactions
    // 且 DBImpl::unscheduled_compactions_ > 0
    // 创建 CompactionArg
    CompactionArg* ca = new CompactionArg;
    ca->db = this;
    ca->prepicked_compaction = nullptr;
    // 自增 DBImpl::bg_compaction_scheduled_
    bg_compaction_scheduled_++;
    // 自减 DBImpl::unscheduled_compactions_
    unscheduled_compactions_--;
    // 调用 Env::Schedule() 调度 low-pri 线程池执行 DBImpl::BGWorkCompaction()
    env_->Schedule(&DBImpl::BGWorkCompaction, ca, Env::Priority::LOW, this,
                   &DBImpl::UnscheduleCallback);
  }
}
```

可以看到
* 如果有手动的 compaction 需要执行时，就不会触发自动的 compaction
* 变量 `unscheduled_compactions_` 需要大于 0 才可能触发 compaction
* 和 flush 一样， compaction 也会调度到后台线程池中执行，并且后台 compaction 数不会超过 `BGJobLimits::max_compactions` 的限制，这个变量是由 `max_background_compactions` 选项控制的

`unscheduled_compactions_` 表示需要被 compact 的 column family （简称 cf）队列长度。和 flush 类似， `DBImpl` 内部也有一条队列 `DBImpl::compaction_queue_` 用于存放待 compact 的 cf
```c++
// Column families are in this queue when they need to be flushed or
// compacted. Consumers of these queues are flush and compaction threads. When
// column family is put on this queue, we increase unscheduled_flushes_ and
// unscheduled_compactions_. When these variables are bigger than zero, that
// means we need to schedule background threads for compaction and thread.
// Once the background threads are scheduled, we decrease unscheduled_flushes_
// and unscheduled_compactions_. That way we keep track of number of
// compaction and flush threads we need to schedule. This scheduling is done
// in MaybeScheduleFlushOrCompaction()
// invariant(column family present in flush_queue_ <==>
// ColumnFamilyData::pending_flush_ == true)
std::deque<FlushRequest> flush_queue_;
// invariant(column family present in compaction_queue_ <==>
// ColumnFamilyData::pending_compaction_ == true)
std::deque<ColumnFamilyData*> compaction_queue_;
```

## DBImpl::SchedulePendingCompaction()
队列 `compaction_queue_` 的更新和 `unscheduled_compactions_` 的更新是同步的，而更新的地方主要在调用 `DBImpl::SchedulePendingCompaction()` 时
```c++
void DBImpl::SchedulePendingCompaction(ColumnFamilyData* cfd) {
  if (!cfd->queued_for_compaction() && cfd->NeedsCompaction()) {
    // 如果 ColumnFamilyData::queued_for_compaction() 返回 false
    // 且 ColumnFamilyData::NeedsCompaction() 返回 true
    // 调用 DBImpl::AddToCompactionQueue() 将 cfd 加入 DBImpl::compaction_queue_
    AddToCompactionQueue(cfd);
    // 自增 DBImpl::unscheduled_compactions_
    ++unscheduled_compactions_;
  }
}
```

`DBImpl::SchedulePendingCompaction()` 只有在 cf 不在队列中且需要进行 compaction 时才会将 cf 加入队列，并增加 `unscheduled_compactions_`

函数 `DBImpl::SchedulePendingCompaction()` 主要是在 `DBImpl::InstallSuperVersionAndScheduleWork()` 中被调用的。函数 `DBImpl::InstallSuperVersionAndScheduleWork()` 在很多地方都被调用，这里不做分析，因为最终是否需要进行 compaction 是由 `DBImpl::SchedulePendingCompaction()` 决定的

## ColumnFamilyData::NeedsCompaction()
判断是否需要 compaction 的核心在函数 `ColumnFamilyData::NeedsCompaction()` 中
```c++
bool ColumnFamilyData::NeedsCompaction() const {
  return compaction_picker_->NeedsCompaction(current_->storage_info());
}
```

`CompactionPicker::NeedsCompaction()` 实际上是一个接口，而对于 level compaction 来说，其实现在 `LevelCompactionPicker::NeedsCompaction()` 中
```c++
bool LevelCompactionPicker::NeedsCompaction(
    const VersionStorageInfo* vstorage) const {
  if (!vstorage->ExpiredTtlFiles().empty()) {
    return true;
  }
  if (!vstorage->BottommostFilesMarkedForCompaction().empty()) {
    return true;
  }
  if (!vstorage->FilesMarkedForCompaction().empty()) {
    return true;
  }
  for (int i = 0; i <= vstorage->MaxInputLevel(); i++) {
    if (vstorage->CompactionScore(i) >= 1) {
      return true;
    }
  }
  return false;
}
```

当满足以下条件之一时，就可以更新 compaction 队列
* 有超时的 sst
* `VersionStorageInfo::bottommost_files_marked_for_compaction_` 队列不为空
* `VersionStorageInfo::files_marked_for_compaction_` 队列不为空
* 遍历所有 level 的 sst ，如果有 level 的 compaction score 大于等于 1

## Compaction Score
这里会涉及到两个变量 `VersionStorageInfo::compaction_score_` 和 `VersionStorageInfo::compaction_level_` ，这两个变量分别保存了待 compact 的 level 和 level 对应的 score 。其中 score 越高表示 compact 优先级越高，小于 1 表示不需要 compact
```c++
// Level that should be compacted next and its compaction score.
// Score < 1 means compaction is not strictly needed.  These fields
// are initialized by Finalize().
// The most critical level to be compacted is listed first
// These are used to pick the best compaction level
std::vector<double> compaction_score_;
std::vector<int> compaction_level_;
```

## VersionStorageInfo::ComputeCompactionScore()
Compaction score 是在 `VersionStorageInfo::ComputeCompactionScore()` 中计算并更新的，其中 level 0 的 compaction score 计算方法和其他 level 是不同的

Level 0 的计算过程如下
* 计算 level 0 下所有文件的总大小 `total_size` 和 文件数 `num_sorted_runs`
* score 设置为 `num_sorted_runs` 除以配置选项 `level0_file_num_compaction_trigger`
* 如果 level 不止一层，将 score 设置为 `max(score, total_size/max_bytes_for_level_base)` ，其中 `max_bytes_for_level_base` 为配置选项

可以看到，会同时考虑 level 0 的文件数和文件大小，对应的代码如下
```c++
void VersionStorageInfo::ComputeCompactionScore(
    const ImmutableCFOptions& immutable_cf_options,
    const MutableCFOptions& mutable_cf_options) {
  // ...
    double score;
    if (level == 0) {
      // We treat level-0 specially by bounding the number of files
      // instead of number of bytes for two reasons:
      //
      // (1) With larger write-buffer sizes, it is nice not to do too
      // many level-0 compactions.
      //
      // (2) The files in level-0 are merged on every read and
      // therefore we wish to avoid too many files when the individual
      // file size is small (perhaps because of a small write-buffer
      // setting, or very high compression ratios, or lots of
      // overwrites/deletions).
      // 如果 level 为 0
      int num_sorted_runs = 0;
      uint64_t total_size = 0;
      for (auto* f : files_[level]) {
        if (!f->being_compacted) {
          total_size += f->compensated_file_size;
          num_sorted_runs++;
        }
      }
      // ...
        score = static_cast<double>(num_sorted_runs) /
                mutable_cf_options.level0_file_num_compaction_trigger;
        if (compaction_style_ == kCompactionStyleLevel && num_levels() > 1) {
          // Level-based involves L0->L0 compactions that can lead to oversized
          // L0 files. Take into account size as well to avoid later giant
          // compactions to the base level.
          score = std::max(
              score, static_cast<double>(total_size) /
                     mutable_cf_options.max_bytes_for_level_base);
        }
      // ...
    }
    // ...
    compaction_level_[level] = level;
    compaction_score_[level] = score;
  // ...
}
```

非 level 0 的 score 计算过程如下
```c++
void VersionStorageInfo::ComputeCompactionScore(
    const ImmutableCFOptions& immutable_cf_options,
    const MutableCFOptions& mutable_cf_options) {
    double score;
  // ...
      // Compute the ratio of current size to size limit.
      uint64_t level_bytes_no_compacting = 0;
      for (auto f : files_[level]) {
        if (!f->being_compacted) {
          level_bytes_no_compacting += f->compensated_file_size;
        }
      }
      score = static_cast<double>(level_bytes_no_compacting) /
              MaxBytesForLevel(level);
    // ...
    compaction_level_[level] = level;
    compaction_score_[level] = score;
  // ...
}
```
可以看到会先计算该 level 的文件总大小，然后再除以 `VersionStorageInfo::MaxBytesForLevel()` 得到 score

`VersionStorageInfo::MaxBytesForLevel()` 的实现如下，就是直接返回 `VersionStorageInfo::level_max_bytes_` 里对应的值
```c++
uint64_t VersionStorageInfo::MaxBytesForLevel(int level) const {
  // Note: the result for level zero is not really used since we set
  // the level-0 compaction threshold based on number of files.
  assert(level >= 0);
  assert(level < static_cast<int>(level_max_bytes_.size()));
  return level_max_bytes_[level];
}
```

`level_max_bytes_` 可以看作是非 0 level 中文件总大小的上限，超过这个上限就需要进行 compact 了，具体的值是在函数 `VersionStorageInfo::CalculateBaseBytes()` 中设置的

在计算出 `compaction_level_` 和 `compaction_score_` 后，将里面的值按照 score 的大小排序，让 score 高的排在前面。另外，可以看到 `bottommost_files_marked_for_compaction_` 、 `files_marked_for_compaction_` 以及超时的 sst 也都是在这里计算的
```c++
void VersionStorageInfo::ComputeCompactionScore(
    const ImmutableCFOptions& immutable_cf_options,
    const MutableCFOptions& mutable_cf_options) {
  // ...
    compaction_level_[level] = level;
    // 设置 VersionStorageInfo::compaction_score_[level] 为 score
    compaction_score_[level] = score;
  // ...
  // sort all the levels based on their score. Higher scores get listed
  // first. Use bubble sort because the number of entries are small.
  // 冒泡排序， 将 level 按照 score 从高到低排序
  for (int i = 0; i < num_levels() - 2; i++) {
    for (int j = i + 1; j < num_levels() - 1; j++) {
      if (compaction_score_[i] < compaction_score_[j]) {
        double score = compaction_score_[i];
        int level = compaction_level_[i];
        compaction_score_[i] = compaction_score_[j];
        compaction_level_[i] = compaction_level_[j];
        compaction_score_[j] = score;
        compaction_level_[j] = level;
      }
    }
  }
  // 调用 VersionStorageInfo::ComputeFilesMarkedForCompaction()
  ComputeFilesMarkedForCompaction();
  // 调用 VersionStorageInfo::ComputeBottommostFilesMarkedForCompaction()
  ComputeBottommostFilesMarkedForCompaction();
  if (mutable_cf_options.ttl > 0) {
    // 如果 ttl > 0
    // 调用 VersionStorageInfo::ComputeExpiredTtlFiles()
    ComputeExpiredTtlFiles(immutable_cf_options, mutable_cf_options.ttl);
  }
  // ...
}
```

TODO 后续有时间再分析 `ComputeFilesMarkedForCompaction()` 、 `ComputeBottommostFilesMarkedForCompaction()` 、 `ComputeExpiredTtlFiles()`

## VersionStorageInfo::CalculateBaseBytes()
这个函数在每次创建新的 `Version` 时调用

RocksDB 有一个选项叫 `level_compaction_dynamic_level_bytes` 会影响这个函数的执行逻辑，默认为 false

当这个选项为 false 时，计算过程如下
* 如果是 level 1，则 `level_max_bytes_[1]` 设置为选项 `max_bytes_for_level_base` 的值
* 如果 level 大于 1，则对于 level i 有
`level_max_bytes_[i] = level_max_bytes_[i - 1] * max_bytes_for_level_multiplier * max_bytes_for_level_multiplier_additional[i - 1]`

代码如下
```c++
void VersionStorageInfo::CalculateBaseBytes(const ImmutableCFOptions& ioptions,
                                            const MutableCFOptions& options) {
  // ...
  level_max_bytes_.resize(ioptions.num_levels);
  if (!ioptions.level_compaction_dynamic_level_bytes) {
    // 如果 level_compaction_dynamic_level_bytes 为 false ，常见情况
    base_level_ = (ioptions.compaction_style == kCompactionStyleLevel) ? 1 : -1;

    // Calculate for static bytes base case
    for (int i = 0; i < ioptions.num_levels; ++i) {
      if (i == 0 && ioptions.compaction_style == kCompactionStyleUniversal) {
        level_max_bytes_[i] = options.max_bytes_for_level_base;
      } else if (i > 1) {
        level_max_bytes_[i] = MultiplyCheckOverflow(
            MultiplyCheckOverflow(level_max_bytes_[i - 1],
                                  options.max_bytes_for_level_multiplier),
            options.MaxBytesMultiplerAdditional(i - 1));
      } else {
        level_max_bytes_[i] = options.max_bytes_for_level_base;
      }
    }
  }
  // ...
}
```

当选项 `level_compaction_dynamic_level_bytes` 为 true 时， RocksDB 刚开始的时候会先将 level 0 的文件合并到最后的 level ，然后用最后 level 的文件大小逐层除以选项 `max_bytes_for_level_multiplier` 反推出前面每一个 level 的大小。详见 RocksDB 对该参数的 [解释](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction#level_compaction_dynamic_level_bytes-is-true)

代码如下
```c++
void VersionStorageInfo::CalculateBaseBytes(const ImmutableCFOptions& ioptions,
                                            const MutableCFOptions& options) {
  // ...
    uint64_t max_level_size = 0;

    int first_non_empty_level = -1;
    // Find size of non-L0 level of most data.
    // Cannot use the size of the last level because it can be empty or less
    // than previous levels after compaction.
    // 计算每个非 level 0 的文件大小，找到最大的 level 大小和第一个非空 level
    for (int i = 1; i < num_levels_; i++) {
      uint64_t total_size = 0;
      for (const auto& f : files_[i]) {
        total_size += f->fd.GetFileSize();
      }
      if (total_size > 0 && first_non_empty_level == -1) {
        first_non_empty_level = i;
      }
      if (total_size > max_level_size) {
        max_level_size = total_size;
      }
    }

    // Prefill every level's max bytes to disallow compaction from there.
    for (int i = 0; i < num_levels_; i++) {
      level_max_bytes_[i] = std::numeric_limits<uint64_t>::max();
    }

    if (max_level_size == 0) {
      // No data for L1 and up. L0 compacts to last level directly.
      // No compaction from L1+ needs to be scheduled.
      // level 1 及以上 level 都没有数据，直接将 level 0 compact 到最后一层 level
      // 即最后一层作为 base_level_
      base_level_ = num_levels_ - 1;
    } else {
      uint64_t base_bytes_max = options.max_bytes_for_level_base;
      uint64_t base_bytes_min = static_cast<uint64_t>(
          base_bytes_max / options.max_bytes_for_level_multiplier);

      // Try whether we can make last level's target size to be max_level_size
      // 用 max_level_size 反向计算最小的非空 level 的大小
      uint64_t cur_level_size = max_level_size;
      for (int i = num_levels_ - 2; i >= first_non_empty_level; i--) {
        // Round up after dividing
        cur_level_size = static_cast<uint64_t>(
            cur_level_size / options.max_bytes_for_level_multiplier);
      }

      // Calculate base level and its size.
      uint64_t base_level_size;
      if (cur_level_size <= base_bytes_min) {
        // Case 1. If we make target size of last level to be max_level_size,
        // target size of the first non-empty level would be smaller than
        // base_bytes_min. We set it be base_bytes_min.
        // 如果 cur_level_size 小于 base_bytes_min ，则用 base_bytes_min 作为
        // 该层的大小并将最小的非空 level 作为 base_level_
        base_level_size = base_bytes_min + 1U;
        base_level_ = first_non_empty_level;
        ROCKS_LOG_WARN(ioptions.info_log,
                       "More existing levels in DB than needed. "
                       "max_bytes_for_level_multiplier may not be guaranteed.");
      } else {
        // Find base level (where L0 data is compacted to).
        // 否则，从第一个非空 level 开始向前找大小小于 base_bytes_max 的 level
        // 作为 base_level_
        base_level_ = first_non_empty_level;
        while (base_level_ > 1 && cur_level_size > base_bytes_max) {
          --base_level_;
          cur_level_size = static_cast<uint64_t>(
              cur_level_size / options.max_bytes_for_level_multiplier);
        }
        if (cur_level_size > base_bytes_max) {
          // Even L1 will be too large
          assert(base_level_ == 1);
          base_level_size = base_bytes_max;
        } else {
          base_level_size = cur_level_size;
        }
      }

      // 从 base_level_ 往后算出每一 level 的 level_max_bytes_
      uint64_t level_size = base_level_size;
      for (int i = base_level_; i < num_levels_; i++) {
        if (i > base_level_) {
          level_size = MultiplyCheckOverflow(
              level_size, options.max_bytes_for_level_multiplier);
        }
        // Don't set any level below base_bytes_max. Otherwise, the LSM can
        // assume an hourglass shape where L1+ sizes are smaller than L0. This
        // causes compaction scoring, which depends on level sizes, to favor L1+
        // at the expense of L0, which may fill up and stall.
        level_max_bytes_[i] = std::max(level_size, base_bytes_max);
      }
    }
  // ...
}
```

具体的流程为
* 遍历每一个非 0 level ，找到第一个非空的 level `first_non_empty_level` 并统计每一层的大小，找出其中的最大值 `max_level_size`
* 如果 `max_level_size` 为 0 ，说明 level 1 及后面还没有数据， `base_level_` 设置为最后一层，即 level 0 会合并到最后一层
* 否则，假设从倒数第二层开始，往前到 level 0 前的每一层的大小都是后一层的 `1/max_bytes_for_level_multiplier`
* 从倒数第二层往前到 `first_non_empty_level` ，假设共 n 层，计算 `first_non_empty_level` 的大小上限 `cur_level_size = max_level_size / (max_bytes_for_level_multiplier^n)`
* 如果 `cur_level_size` 大于选项 `max_bytes_for_level_base` ，则说明还需要更小的 level ，所以继续逐层向前遍历，直到找到大小上限小于等于 `max_bytes_for_level_base` 的 level ，将 `base_level_` 设置为该 level ，每向前一层 `cur_level_size` 都会除一次 `max_bytes_for_level_multiplier`
* 从 `base_level_` 开始往后，用该层的 `cur_level_size` 乘以 `max_bytes_for_level_multiplier` 计算后面每一层的大小上限

至此就基本了解了什么条件下会触发 compaction 了

## DBImpl::BackgroundCompaction()
接下来关注 compact 的具体过程，和 flush 类似， compact 的操作是在函数 `DBImpl::BackgroundCompaction()` 中进行的

首先会从 compaction 队列 `DBImpl::compaction_queue_` 中取出第一个需要 compact 的 cf ，然后调用 `ColumnFamilyData::PickCompaction()` 获取需要 compact 的内容 `Compaction`
```c++
Status DBImpl::BackgroundCompaction(bool* made_progress,
                                    JobContext* job_context,
                                    LogBuffer* log_buffer,
                                    PrepickedCompaction* prepicked_compaction) {
  // ...
  unique_ptr<Compaction> c;
  // ...
    // 调用 DBImpl::PopFirstFromCompactionQueue() 获取 cfd
    auto cfd = PopFirstFromCompactionQueue();
    // We unreference here because the following code will take a Ref() on
    // this cfd if it is going to use it (Compaction class holds a
    // reference).
    // This will all happen under a mutex so we don't have to be afraid of
    // somebody else deleting it.
    if (cfd->Unref()) {
      // 如果 ColumnFamilyData::Unref() 返回 true
      // 删除 cfd
      delete cfd;
      // This was the last reference of the column family, so no need to
      // compact.
      // 直接返回 ok
      return Status::OK();
    }
    // ...
      c.reset(cfd->PickCompaction(*mutable_cf_options, log_buffer));
  // ...
}
```

cf 选取 `Compaction` 是通过调用 `CompactionPicker::PickCompaction()` 来实现的

根据 `Compaction` 对象中的信息，在 compact 时可以采取不同的操作

一些 compaction 直接通过删除 sst 文件就可以完成
```c++
Status DBImpl::BackgroundCompaction(bool* made_progress,
                                    JobContext* job_context,
                                    LogBuffer* log_buffer,
                                    PrepickedCompaction* prepicked_compaction) {
  // ...
  if (!c) {
    // Nothing to do
    // 如果 c 为 nullptr
    // 什么也不用做
    ROCKS_LOG_BUFFER(log_buffer, "Compaction nothing to do");
  } else if (c->deletion_compaction()) {
    // 否则，如果 Compaction::deletion_compaction() 为 true ，说明 compaction
    // 只需要删除文件即可完成
    // TODO(icanadi) Do we want to honor snapshots here? i.e. not delete old
    // file if there is alive snapshot pointing to it

    // ...
    // 遍历 Compaction::inputs(0) 的文件
    for (const auto& f : *c->inputs(0)) {
      // 调用 Compaction::edit() VersionEdit::DeleteFile() 记录文件的删除
      c->edit()->DeleteFile(c->level(), f->fd.GetNumber());
    }
    // 调用 VersionSet::LogAndApply()
    status = versions_->LogAndApply(c->column_family_data(),
                                    *c->mutable_cf_options(), c->edit(),
                                    &mutex_, directories_.GetDbDir());
    // 调用 DBImpl::InstallSuperVersionAndScheduleWork()
    InstallSuperVersionAndScheduleWork(
        c->column_family_data(), &job_context->superversion_contexts[0],
        *c->mutable_cf_options(), FlushReason::kAutoCompaction);
    ROCKS_LOG_BUFFER(log_buffer, "[%s] Deleted %d files\n",
                     c->column_family_data()->GetName().c_str(),
                     c->num_input_files(0));
    // 设置 *made_progress 为 true
    *made_progress = true;
  }
  // ...
}
```

一些 compaction 直接通过移动 sst 文件就可以完成
```c++
Status DBImpl::BackgroundCompaction(bool* made_progress,
                                    JobContext* job_context,
                                    LogBuffer* log_buffer,
                                    PrepickedCompaction* prepicked_compaction) {
  // ...
  } else if (!trivial_move_disallowed && c->IsTrivialMove()) {
    // 如果 trivial_move_disallowed 为 false 且 Compaction::IsTrivialMove()
    // 返回 true ，说明 compaction 只需要将文件移动到下一 level 即可完成

    // ...
    // Move files to next level
    // 移动文件到下一个 level
    int32_t moved_files = 0;
    int64_t moved_bytes = 0;
    // 从 level 0 开始遍历每个 level
    for (unsigned int l = 0; l < c->num_input_levels(); l++) {
      if (c->level(l) == c->output_level()) {
        // 如果当前 level == Compaction::output_level()
        // 继续处理下一个 level
        continue;
      }
      for (size_t i = 0; i < c->num_input_files(l); i++) {
        // 遍历当前 level 的所有文件
        FileMetaData* f = c->input(l, i);
        // 调用 Compaction::edit() VersionEdit::DeleteFile() 删除文件
        c->edit()->DeleteFile(c->level(l), f->fd.GetNumber());
        // 调用 Compaction::edit() VersionEdit::AddFile() 将对应文件
        // 添加到 Compaction::output_level() 层
        c->edit()->AddFile(c->output_level(), f->fd.GetNumber(),
                           f->fd.GetPathId(), f->fd.GetFileSize(), f->smallest,
                           f->largest, f->fd.smallest_seqno,
                           f->fd.largest_seqno, f->marked_for_compaction);

        ROCKS_LOG_BUFFER(
            log_buffer,
            "[%s] Moving #%" PRIu64 " to level-%d %" PRIu64 " bytes\n",
            c->column_family_data()->GetName().c_str(), f->fd.GetNumber(),
            c->output_level(), f->fd.GetFileSize());
        // 自增 moved_files
        ++moved_files;
        // 增加 moved_bytes
        moved_bytes += f->fd.GetFileSize();
      }
    }

    // 调用 DBImpl::versions_ VersionSet::LogAndApply()
    status = versions_->LogAndApply(c->column_family_data(),
                                    *c->mutable_cf_options(), c->edit(),
                                    &mutex_, directories_.GetDbDir());
    // Use latest MutableCFOptions
    // 调用 DBImpl::InstallSuperVersionAndScheduleWork()
    InstallSuperVersionAndScheduleWork(
        c->column_family_data(), &job_context->superversion_contexts[0],
        *c->mutable_cf_options(), FlushReason::kAutoCompaction);
    // ...
    // 设置 *made_progress 为 true
    *made_progress = true;
    // ...
  }
  // ...
}
```

最后是普通的 compact 情况
```c++
Status DBImpl::BackgroundCompaction(bool* made_progress,
                                    JobContext* job_context,
                                    LogBuffer* log_buffer,
                                    PrepickedCompaction* prepicked_compaction) {
  // ...
  } else {
    // ...
    // 创建 CompactionJob
    CompactionJob compaction_job(
        job_context->job_id, c.get(), immutable_db_options_,
        env_options_for_compaction_, versions_.get(), &shutting_down_,
        preserve_deletes_seqnum_.load(), log_buffer, directories_.GetDbDir(),
        GetDataDir(c->column_family_data(), c->output_path_id()), stats_,
        &mutex_, &error_handler_, snapshot_seqs, earliest_write_conflict_snapshot,
        snapshot_checker, table_cache_, &event_logger_,
        c->mutable_cf_options()->paranoid_file_checks,
        c->mutable_cf_options()->report_bg_io_stats, dbname_,
        &compaction_job_stats);
    // 调用 CompactionJob::Prepare()
    compaction_job.Prepare();

    // 解锁 DBImpl::mutex_
    mutex_.Unlock();
    // 调用 CompactionJob::Run()
    compaction_job.Run();
    TEST_SYNC_POINT("DBImpl::BackgroundCompaction:NonTrivial:AfterRun");
    // 加锁 DBImpl::mutex_
    mutex_.Lock();

    // 调用 CompactionJob::Install()
    status = compaction_job.Install(*c->mutable_cf_options());
    if (status.ok()) {
      // 如果 ok
      // 调用 DBImpl::InstallSuperVersionAndScheduleWork()
      InstallSuperVersionAndScheduleWork(
          c->column_family_data(), &job_context->superversion_contexts[0],
          *c->mutable_cf_options(), FlushReason::kAutoCompaction);
    }
    // 设置 *made_progress 为 true
    *made_progress = true;
  }
  // ...
}
```
可以看到，每次 compact 时都会
* 创建一个 `CompactionJob`
* 调用 `CompactionJob::Prepare()`
* 调用 `CompactionJob::Run()` ，具体的 compact 逻辑在 `CompactionJob::Run()` 中
* 调用 `CompactionJob::Install()` 应用 compaction 的结果，即记录文件的增删情况
* 调用 `DBImpl::InstallSuperVersionAndScheduleWork()` 产生并应用新的 super version

## CompactionPicker::PickCompaction()
对于 level compaction 来说，`CompactionPicker::PickCompaction()` 实际上会调用 `LevelCompactionPicker::PickCompaction()` 并最终调用 `LevelCompactionBuilder::PickCompaction()`
```c++
Compaction* LevelCompactionBuilder::PickCompaction() {
  // Pick up the first file to start compaction. It may have been extended
  // to a clean cut.
  // 调用 LevelCompactionBuilder::SetupInitialFiles() ，会
  // 设置 LevelCompactionBuilder::start_level_inputs_
  SetupInitialFiles();
  // 如果 LevelCompactionBuilder::start_level_inputs_ 为空
  if (start_level_inputs_.empty()) {
    // 返回 nullptr
    return nullptr;
  }
  assert(start_level_ >= 0 && output_level_ >= 0);

  // If it is a L0 -> base level compaction, we need to set up other L0
  // files if needed.
  // 如果是从 L0->base level 的 compaction ，如果有需要，还会将其他一些 L0
  // 文件加入，因为 level 0 的文件是有可能重叠的
  if (!SetupOtherL0FilesIfNeeded()) {
    // 如果 LevelCompactionBuilder::SetupOtherL0FilesIfNeeded() 返回 false
    // 返回 nullptr
    return nullptr;
  }

  // Pick files in the output level and expand more files in the start level
  // if needed.
  // 已经找到了 start level 的文件，再从 output level 中找到参与 compact 的文件
  // 如有需要，扩展 start level 的文件
  if (!SetupOtherInputsIfNeeded()) {
    // 如果 LevelCompactionBuilder::SetupOtherInputsIfNeeded() 返回 false
    // 返回 nullptr
    return nullptr;
  }

  // Form a compaction object containing the files we picked.
  // 调用 LevelCompactionBuilder::GetCompaction() 创建 Compaction
  Compaction* c = GetCompaction();

  TEST_SYNC_POINT_CALLBACK("LevelCompactionPicker::PickCompaction:Return", c);

  // 返回 Compaction
  return c;
}
```

### LevelCompactionBuilder::SetupInitialFiles()
这个函数用来初始化需要 compact 的文件，即 compaction 的输入文件，主要的流程为
* 根据 compaction score 的大小从大到小找到一个可以进行 compact 的 level ，从该 level 中挑选文件，通常在这里就已经找到需要 compact 的文件了
* 如果在上一步中没找到文件，则会依次尝试下面几种方法直到找到需要 compact 的文件
    * 调用 `CompactionPicker::PickFilesMarkedForCompaction()` 尝试从 `files_marked_for_compaction_` 中找文件
    * 尝试从 `bottommost_files_marked_for_compaction_` 中找文件
    * 调用 `LevelCompactionBuilder::PickExpiredTtlFiles()` 尝试从超时的 sst 中找文件
    * 调用 `LevelCompactionBuilder::GetCompaction()` 创建 `Compaction`

TODO 后续考虑分析其他几步

这里主要关注第一步，这是最常见的情况
```c++
void LevelCompactionBuilder::SetupInitialFiles() {
  // Find the compactions by size on all levels.
  // 初始化 skipped_l0_to_base 为 false
  bool skipped_l0_to_base = false;
  for (int i = 0; i < compaction_picker_->NumberLevels() - 1; i++) {
    // i 从 0 到 num_levels - 2
    // 调用 LevelCompactionBuilder::vstorage_ VersionStorageInfo::CompactionScore(i)
    // 并设置到 LevelCompactionBuilder::start_level_score_
    start_level_score_ = vstorage_->CompactionScore(i);
    // 调用 LevelCompactionBuilder::vstorage_ VersionStorageInfo::CompactionScoreLevel(i)
    // 并设置到 LevelCompactionBuilder::start_level_
    start_level_ = vstorage_->CompactionScoreLevel(i);
    assert(i == 0 || start_level_score_ <= vstorage_->CompactionScore(i - 1));
    if (start_level_score_ >= 1) {
      // 如果 LevelCompactionBuilder::start_level_score_ >= 1
      if (skipped_l0_to_base && start_level_ == vstorage_->base_level()) {
        // If L0->base_level compaction is pending, don't schedule further
        // compaction from base level. Otherwise L0->base_level compaction
        // may starve.
        // 如果 skipped_l0_to_base 为 true 且 LevelCompactionBuilder::start_level_ ==
        // VersionStorage::base_level() ，则进行下一次循环，避免调度更多的从 base level
        // 开始的 compaction ，以防 L0->base_level 的 compaction 被饿死
        continue;
      }
      // 如果 LevelCompactionBuilder::start_level_ == 0
      // 则设置 LevelCompactionBuilder::output_level_ 为 VersionStorageInfo::base_level()
      // 否则设置为 LevelCompactionBuilder::start_level_ + 1
      output_level_ =
          (start_level_ == 0) ? vstorage_->base_level() : start_level_ + 1;
      if (PickFileToCompact()) {
        // found the compaction!
        // 如果 LevelCompactionBuilder::PickFileToCompact() 返回 true ，则找到了 compaction
        if (start_level_ == 0) {
          // L0 score = `num L0 files` / `level0_file_num_compaction_trigger`
          // 如果 LevelCompactionBuilder::start_level_ 为 0
          // 设置 LevelCompactionBuilder::compaction_reason_
          // 为 CompactionReason::kLevelL0FilesNum
          compaction_reason_ = CompactionReason::kLevelL0FilesNum;
        } else {
          // L1+ score = `Level files size` / `MaxBytesForLevel`
          // 否则
          // 设置 LevelCompactionBuilder::compaction_reason_
          // 为 CompactionReason::kLevelMaxLevelSize
          compaction_reason_ = CompactionReason::kLevelMaxLevelSize;
        }
        // 跳出循环
        break;
      } else {
        // didn't find the compaction, clear the inputs
        // 否则，没有找到 compaction ，清理输入
        // 清空 LevelCompactionBuilder::start_level_inputs_
        start_level_inputs_.clear();
        if (start_level_ == 0) {
          // 如果 LevelCompactionBuilder::start_level_ == 0
          // 设置 skipped_l0_to_base 为 true
          skipped_l0_to_base = true;
          // L0->base_level may be blocked due to ongoing L0->base_level
          // compactions. It may also be blocked by an ongoing compaction from
          // base_level downwards.
          //
          // In these cases, to reduce L0 file count and thus reduce likelihood
          // of write stalls, we can attempt compacting a span of files within
          // L0.
          // L0->base_level 的 compact 暂时无法进行，为了减少 level 0 的文件数
          // 尝试将 level 0 相邻的文件做合并
          if (PickIntraL0Compaction()) {
            // 如果 LevelCompactionBuilder::PickIntraL0Compaction() 返回 true
            // 设置 LevelCompactionBuilder::output_level_ 为 0
            output_level_ = 0;
            // 设置 LevelCompactionBuilder::compaction_reason_
            // 为 CompactionReason::kLevelL0FilesNum
            compaction_reason_ = CompactionReason::kLevelL0FilesNum;
            // 跳出循环
            break;
          }
        }
      }
    }
  }
}
```
只有当 score 大于等于 1 才会进行 compact ，如果是对 level 0 进行 compact ，则 `output_level_` 会设为 `VersionStorageInfo::base_level()` ，否则就是下一 level 。这里的 `base_level_` 就是前面计算得到的

主要关注 `LevelCompactionBuilder::PickFileToCompact()` 函数

### LevelCompactionBuilder::PickFileToCompact()
实现如下
```c++
// 查找文件用于 compaction ，找到了会返回 true
bool LevelCompactionBuilder::PickFileToCompact() {
  // ...
  // Pick the largest file in this level that is not already
  // being compacted
  // LevelCompactionBuilder::start_level_ 即当前需要 compaction 的 level
  // 调用 VersionStorageInfo::FilesByCompactionPri() 获取 file_size
  // 调用 VersionStorageInfo::LevelFiles() 获取 level_files
  const std::vector<int>& file_size =
      vstorage_->FilesByCompactionPri(start_level_);
  const std::vector<FileMetaData*>& level_files =
      vstorage_->LevelFiles(start_level_);

  unsigned int cmp_idx;
  // 遍历所有待 compact 的文件
  for (cmp_idx = vstorage_->NextCompactionIndex(start_level_);
       cmp_idx < file_size.size(); cmp_idx++) {
    // 遍历 cmp_idx [VersionStorageInfo::NextCompactionIndex(), file_size.size())
    // 设置 index 为 file_size[cmp_idx] ，即通过 file_size 获取下标
    int index = file_size[cmp_idx];
    // 设置 f 为 level_files[index]
    auto* f = level_files[index];

    // do not pick a file to compact if it is being compacted
    // from n-1 level.
    if (f->being_compacted) {
      // 如果 FileMetaData::being_compacted 为 true ， continue
      continue;
    }

    // 将 f 插入 LevelCompactionBuilder::start_level_inputs_
    // 的 CompactionInputFiles::files
    start_level_inputs_.files.push_back(f);
    // 设置 LevelCompactionBuilder::start_level_inputs_
    // 的 CompactionInputFiles::level 为 LevelCompactionBuilder::start_level_
    start_level_inputs_.level = start_level_;
    if (!compaction_picker_->ExpandInputsToCleanCut(cf_name_, vstorage_,
                                                    &start_level_inputs_) ||
        compaction_picker_->FilesRangeOverlapWithCompaction(
            {start_level_inputs_}, output_level_)) {
      // A locked (pending compaction) input-level file was pulled in due to
      // user-key overlap.
      // 如果 CompactionPicker::ExpandInputsToCleanCut() 返回 false
      // 或 CompactionPicker::FilesRangeOverlapWithCompaction() 返回 true
      // 则存在 key range 重叠的情况
      // 清空 LevelCompactionBuilder::start_level_inputs_ ， continue
      start_level_inputs_.clear();
      continue;
    }

    // Now that input level is fully expanded, we check whether any output files
    // are locked due to pending compaction.
    //
    // Note we rely on ExpandInputsToCleanCut() to tell us whether any output-
    // level files are locked, not just the extra ones pulled in for user-key
    // overlap.
    // 检查 output level 会不会出现覆盖的情况
    InternalKey smallest, largest;
    // 调用 CompactionPicker::GetRange() 获取 start level 的 smallest 和 largest
    compaction_picker_->GetRange(start_level_inputs_, &smallest, &largest);
    // 创建 CompactionInputFiles output_level_inputs
    CompactionInputFiles output_level_inputs;
    // 设置 CompactionInputFiles::level 为 LevelCompactionBuilder::output_level_
    output_level_inputs.level = output_level_;
    // 调用 VersionStorageInfo::GetOverlappingInputs() 获取 output level
    // 和 smallest 和 largest 返回有重叠的文件
    vstorage_->GetOverlappingInputs(output_level_, &smallest, &largest,
                                    &output_level_inputs.files);
    // 如果 CompactionInputFiles::empty() 返回 false
    // 且 CompactionPicker::ExpandInputsToCleanCut() 返回 false
    if (!output_level_inputs.empty() &&
        !compaction_picker_->ExpandInputsToCleanCut(cf_name_, vstorage_,
                                                    &output_level_inputs)) {
      // 清空 LevelCompactionBuilder::start_level_inputs_ 然后 continue
      start_level_inputs_.clear();
      continue;
    }
    // 设置 LevelCompactionBuilder::base_index_ 为 index
    base_index_ = index;
    // 跳出
    break;
  }

  // store where to start the iteration in the next call to PickCompaction
  // 调用 VersionStorageInfo::SetNextCompactionIndex() 并设置为当前 cmp_index
  // 则下次调用 LevelCompactionBuilder::PickCompaction() 时就会从这个地方
  // 开始继续遍历了
  vstorage_->SetNextCompactionIndex(start_level_, cmp_idx);

  // 如果 LevelCompactionBuilder::start_level_inputs_ 大小 > 0 ，返回 true
  // 否则 false
  return start_level_inputs_.size() > 0;
}
```
主要流程为
* 遍历待 compact 的文件，选择一个文件 f
    * 扩展文件的 key 范围到 `clean cut` 的范围，将这个范围内的文件都加入到 `LevelCompactionBuilder::start_level_inputs_` ，作为输入文件
    * 检查 `output_level_` 中目前正在参与其他 compaction 的文件中是否有和这次 compaction 输入文件 key 范围重叠的，如果有的话，则只能跳过文件 f。这是因为 compaction 是并行进行的，不能和其他 compaction 冲突
    * 从 `output_level_` 中找到范围和输入文件重叠且 key 范围扩展到 `clean cut` 后的文件，检查这些文件是否被其他尚未执行的 compaction 占用，如果是的话也只能跳过文件 f
    * 如果通过了上面的检查，则找到了需要的输入文件，可以结束查找

### LevelCompactionBuilder::SetupOtherL0FilesIfNeeded()
接着分析 `LevelCompactionBuilder::PickCompaction()` 所调用的 `LevelCompactionBuilder::SetupOtherL0FilesIfNeeded()`
```c++
bool LevelCompactionBuilder::SetupOtherL0FilesIfNeeded() {
  if (start_level_ == 0 && output_level_ != 0) {
    // 如果 LevelCompactionBuilder::start_level_ == 0
    // 且 LevelCompactionBuilder::output_level_ 不为 0
    // 返回 CompactionPicker::GetOverlappingL0Files()
    return compaction_picker_->GetOverlappingL0Files(
        vstorage_, &start_level_inputs_, output_level_, &parent_index_);
  }
  return true;
}
```

在 RocksDB 中，只有 level 0 的文件是无序的，因为它们是由 memtable flush 产生的，它们的 key 范围也有可能重叠。前面找输入文件的过程中，并没有针对 level 0 的情况做特殊处理，因此需要通过 `SetupOtherL0FilesIfNeeded()` 找到所有在给定范围内的 level 0 文件。该函数检查如果是从 level 0 到其他 level 的 compact ，就会调用 `CompactionPicker::GetOverlappingL0Files()` 进行处理
```c++
bool CompactionPicker::GetOverlappingL0Files(
    VersionStorageInfo* vstorage, CompactionInputFiles* start_level_inputs,
    int output_level, int* parent_index) {
  // Two level 0 compaction won't run at the same time, so don't need to worry
  // about files on level 0 being compacted.
  assert(level0_compactions_in_progress()->empty());
  InternalKey smallest, largest;
  // 调用 CompactionPicker::GetRange() 获取 smallest 和 largest
  GetRange(*start_level_inputs, &smallest, &largest);
  // Note that the next call will discard the file we placed in
  // c->inputs_[0] earlier and replace it with an overlapping set
  // which will include the picked file.
  // 清空 start_level_inputs
  start_level_inputs->files.clear();
  // 调用 VersionStorageInfo::GetOverlappingInputs() 获取 level 0
  // smallest 和 largest 范围内的文件到 start_level_inputs
  vstorage->GetOverlappingInputs(0, &smallest, &largest,
                                 &(start_level_inputs->files));

  // If we include more L0 files in the same compaction run it can
  // cause the 'smallest' and 'largest' key to get extended to a
  // larger range. So, re-invoke GetRange to get the new key range
  // 重新调用 CompactionPicker::GetRange() 获取 smallest 和 largest
  GetRange(*start_level_inputs, &smallest, &largest);
  // 检查输出文件是否有正在进行 compaction 的
  if (IsRangeInCompaction(vstorage, &smallest, &largest, output_level,
                          parent_index)) {
    // 如果 CompactionPicker::IsRangeInCompaction() 返回 true
    // 返回 false
    return false;
  }
  assert(!start_level_inputs->files.empty());

  // 返回 true
  return true;
}
```
主要流程为
* 找到 level 0 中所有和给定 key 范围重叠的文件到 `start_level_inputs`
* 由于可能有新的 level 0 文件加入， key 的范围也会变大，因此重新计算 `start_level_inputs` 的 key 范围
* 检查输出 level 中与范围重叠的文件是否正在 compaction

### LevelCompactionBuilder::SetupOtherInputsIfNeeded()
接着分析 `LevelCompactionBuilder::PickCompaction()` 所调用的 `LevelCompactionBuilder::SetupOtherInputsIfNeeded()`

```c++
// 会设置 LevelCompactionBuilder::compaction_inputs_
bool LevelCompactionBuilder::SetupOtherInputsIfNeeded() {
  // Setup input files from output level. For output to L0, we only compact
  // spans of files that do not interact with any pending compactions, so don't
  // need to consider other levels.
  if (output_level_ != 0) {
    output_level_inputs_.level = output_level_;
    if (!compaction_picker_->SetupOtherInputs(
            cf_name_, mutable_cf_options_, vstorage_, &start_level_inputs_,
            &output_level_inputs_, &parent_index_, base_index_)) {
      return false;
    }

    // 将 LevelCompactionBuilder::start_level_inputs_
    // 插入 LevelCompactionBuilder::compaction_inputs_
    compaction_inputs_.push_back(start_level_inputs_);
    if (!output_level_inputs_.empty()) {
      // 如果 LevelCompactionBuilder::output_level_inputs_ 不为空
      // 将 LevelCompactionBuilder::output_level_inputs_
      // 插入 LevelCompactionBuilder::compaction_inputs_
      compaction_inputs_.push_back(output_level_inputs_);
    }

    // In some edge cases we could pick a compaction that will be compacting
    // a key range that overlap with another running compaction, and both
    // of them have the same output level. This could happen if
    // (1) we are running a non-exclusive manual compaction
    // (2) AddFile ingest a new file into the LSM tree
    // We need to disallow this from happening.
    if (compaction_picker_->FilesRangeOverlapWithCompaction(compaction_inputs_,
                                                            output_level_)) {
      // This compaction output could potentially conflict with the output
      // of a currently running compaction, we cannot run it.
      // 如果 CompactionPicker::FilesRangeOverlapWithCompaction() 返回 true
      // 说明这个 compaction 的输出可能会和其他同时运行的 compaction 的输出
      // 重叠
      // 直接返回 false
      return false;
    }
    // 调用 CompactionPicker::GetGrandparents()
    compaction_picker_->GetGrandparents(vstorage_, start_level_inputs_,
                                        output_level_inputs_, &grandparents_);
  } else {
    // 否则，将 LevelCompactionBuilder::start_level_inputs_
    // 插入 LevelCompactionBuilder::compaction_inputs_
    compaction_inputs_.push_back(start_level_inputs_);
  }
  // 返回 true
  return true;
}
```
主要流程包括
* 由于输入文件被扩展过了，因此会调用 `CompactionPicker::SetupOtherInputs()` 扩展输出 level 的文件到 `LevelCompactionBuilder::output_level_inputs_` , `SetupOtherInputs()` 在扩展输出 level 的文件后，还会在保证不影响输出文件的 key 范围的情况下尝试能否进一步扩展输入文件
* 将 `start_level_inputs_` 和 `output_level_inputs_` 插入 `LevelCompactionBuilder::compaction_inputs_`
* 检查 `LevelCompactionBuilder::compaction_inputs_` 的范围不与其他 compaction 冲突
* 获取输出 level 后一层中和 compact 范围重叠的文件到 `LevelCompactionBuilder::grandparents_` ，这个范围包括了输入和输出文件的 key 范围

### LevelCompactionBuilder::GetCompaction()
该函数会创建一个 `Compaction` 对象并返回，而 compaction 的输入文件就是之前设置的 `compaction_inputs_`
```c++
Compaction* LevelCompactionBuilder::GetCompaction() {
  // 创建新的 Compaction
  auto c = new Compaction(
      vstorage_, ioptions_, mutable_cf_options_, std::move(compaction_inputs_),
      output_level_,
      MaxFileSizeForLevel(mutable_cf_options_, output_level_,
                          ioptions_.compaction_style, vstorage_->base_level(),
                          ioptions_.level_compaction_dynamic_level_bytes),
      mutable_cf_options_.max_compaction_bytes,
      GetPathId(ioptions_, mutable_cf_options_, output_level_),
      GetCompressionType(ioptions_, vstorage_, mutable_cf_options_,
                         output_level_, vstorage_->base_level()),
      GetCompressionOptions(ioptions_, vstorage_, output_level_),
      /* max_subcompactions */ 0, std::move(grandparents_), is_manual_,
      start_level_score_, false /* deletion_compaction */, compaction_reason_);

  // If it's level 0 compaction, make sure we don't execute any other level 0
  // compactions in parallel
  // 调用 CompactionPicker::RegisterCompaction()
  compaction_picker_->RegisterCompaction(c);

  // Creating a compaction influences the compaction score because the score
  // takes running compactions into account (by skipping files that are already
  // being compacted). Since we just changed compaction score, we recalculate it
  // here
  // 调用 LevelCompactionBuilder::vstorage_ VersionStorageInfo::ComputeCompactionScore()
  // 重新计算 compaction score
  vstorage_->ComputeCompactionScore(ioptions_, mutable_cf_options_);
  // 返回 Compaction
  return c;
}
```

另外，在 `Compaction` 的构造函数中，会调用 `Compaction::MarkFilesBeingCompacted()` 将这次 compaction 的文件的 `FileMetaData::being_compacted` 设置为 true ，已标记文件正在被 compact ，这样下次计算 compact 范围时就会跳过这些文件

可以看到，进行 compact 之后会重新调用 `VersionStorageInfo::ComputeCompactionScore()` 计算 compaction score ，因为计算 score 的时候我们会跳过正在 compact 的文件

## CompactionJob::Prepare()
该函数主要执行一些 compact 前的准备工作，主要是判断能否进行 sub compaction ，如果可以，调用 `CompactionJob::GenSubcompactionBoundaries()` 划分 sub compaction 并将结果记录到 `CompactionState::sub_compact_states`
```c++
void CompactionJob::Prepare() {
  // ...
  // Generate file_levels_ for compaction berfore making Iterator
  auto* c = compact_->compaction;
  // ...
  // Is this compaction producing files at the bottommost level?
  bottommost_level_ = c->bottommost_level();

  if (c->ShouldFormSubcompactions()) {
    const uint64_t start_micros = env_->NowMicros();
    GenSubcompactionBoundaries();
    MeasureTime(stats_, SUBCOMPACTION_SETUP_TIME,
                env_->NowMicros() - start_micros);

    assert(sizes_.size() == boundaries_.size() + 1);

    for (size_t i = 0; i <= boundaries_.size(); i++) {
      Slice* start = i == 0 ? nullptr : &boundaries_[i - 1];
      Slice* end = i == boundaries_.size() ? nullptr : &boundaries_[i];
      compact_->sub_compact_states.emplace_back(c, start, end, sizes_[i]);
    }
    MeasureTime(stats_, NUM_SUBCOMPACTIONS_SCHEDULED,
                compact_->sub_compact_states.size());
  } else {
    compact_->sub_compact_states.emplace_back(c, nullptr, nullptr);
  }
}
```
Sub compaction 主要用于将 level 0 到 level 1 的 compaction 并行化

### CompactionJob::GenSubcompactionBoundaries()
该函数用于计算每个 sub compaction 的边界，主要流程为
* 遍历所有参与 compaction 的 level ，获取每个 level 的边界到 `bounds`
* 对 `bounds` 进行排序和去重
* 边界两个两个一组作为 smallest key 和 largest key ， [start_lvl, out_lvl] 之间被这个范围覆盖到的文件大小和作为 size 打包成 RangeWithSize 对象并记录到 `ranges` 中
* 计算 sub compaction 数
* 根据 sub compaction 数从 `ranges` 中划分出各个 sub compaction 的范围，分别将范围和这个范围的文件大小记录到 `CompactionJob::boundaries_` 和 `CompactionJob::sizes_` 中

实现如下
```c++
// Generates a histogram representing potential divisions of key ranges from
// the input. It adds the starting and/or ending keys of certain input files
// to the working set and then finds the approximate size of data in between
// each consecutive pair of slices. Then it divides these ranges into
// consecutive groups such that each group has a similar size.
void CompactionJob::GenSubcompactionBoundaries() {
  auto* c = compact_->compaction;
  auto* cfd = c->column_family_data();
  const Comparator* cfd_comparator = cfd->user_comparator();
  std::vector<Slice> bounds;
  int start_lvl = c->start_level();
  int out_lvl = c->output_level();

  // Add the starting and/or ending key of certain input files as a potential
  // boundary
  for (size_t lvl_idx = 0; lvl_idx < c->num_input_levels(); lvl_idx++) {
    int lvl = c->level(lvl_idx);
    if (lvl >= start_lvl && lvl <= out_lvl) {
      // 遍历所有参与 compaction 的 level
      const LevelFilesBrief* flevel = c->input_levels(lvl_idx);
      size_t num_files = flevel->num_files;

      if (num_files == 0) {
        continue;
      }

      // 获取所有 level 的边界
      if (lvl == 0) {
        // For level 0 add the starting and ending key of each file since the
        // files may have greatly differing key ranges (not range-partitioned)
        for (size_t i = 0; i < num_files; i++) {
          bounds.emplace_back(flevel->files[i].smallest_key);
          bounds.emplace_back(flevel->files[i].largest_key);
        }
      } else {
        // For all other levels add the smallest/largest key in the level to
        // encompass the range covered by that level
        bounds.emplace_back(flevel->files[0].smallest_key);
        bounds.emplace_back(flevel->files[num_files - 1].largest_key);
        if (lvl == out_lvl) {
          // For the last level include the starting keys of all files since
          // the last level is the largest and probably has the widest key
          // range. Since it's range partitioned, the ending key of one file
          // and the starting key of the next are very close (or identical).
          for (size_t i = 1; i < num_files; i++) {
            bounds.emplace_back(flevel->files[i].smallest_key);
          }
        }
      }
    }
  }

  // 排序并去重
  std::sort(bounds.begin(), bounds.end(),
            [cfd_comparator](const Slice& a, const Slice& b) -> bool {
              return cfd_comparator->Compare(ExtractUserKey(a),
                                             ExtractUserKey(b)) < 0;
            });
  // Remove duplicated entries from bounds
  bounds.erase(
      std::unique(bounds.begin(), bounds.end(),
                  [cfd_comparator](const Slice& a, const Slice& b) -> bool {
                    return cfd_comparator->Compare(ExtractUserKey(a),
                                                   ExtractUserKey(b)) == 0;
                  }),
      bounds.end());

  // Combine consecutive pairs of boundaries into ranges with an approximate
  // size of data covered by keys in that range
  // 边界两个两个一组作为 smallest key 和 largest key
  // [start_lvl, out_lvl] 之间被这个范围覆盖到的文件大小和作为 size
  // 打包成 RangeWithSize 对象并记录到 ranges 中
  uint64_t sum = 0;
  std::vector<RangeWithSize> ranges;
  auto* v = cfd->current();
  for (auto it = bounds.begin();;) {
    const Slice a = *it;
    it++;

    if (it == bounds.end()) {
      break;
    }

    const Slice b = *it;
    uint64_t size = versions_->ApproximateSize(v, a, b, start_lvl, out_lvl + 1);
    ranges.emplace_back(a, b, size);
    sum += size;
  }

  // Group the ranges into subcompactions
  // 计算 sub compaction 数 subcompactions
  const double min_file_fill_percent = 4.0 / 5;
  int base_level = v->storage_info()->base_level();
  // 设置 max_output_files 为 sum / min_file_fill_percent / max_file_size[out_lvl]
  // MaxFileSizeForLevel() 返回对应 level 中单个文件的最大大小
  uint64_t max_output_files = static_cast<uint64_t>(std::ceil(
      sum / min_file_fill_percent /
      MaxFileSizeForLevel(*(c->mutable_cf_options()), out_lvl,
          c->immutable_cf_options()->compaction_style, base_level,
          c->immutable_cf_options()->level_compaction_dynamic_level_bytes)));
  uint64_t subcompactions =
      std::min({static_cast<uint64_t>(ranges.size()),
                static_cast<uint64_t>(c->max_subcompactions()),
                max_output_files});

  // 根据 sub compaction 数划分 range
  if (subcompactions > 1) {
    double mean = sum * 1.0 / subcompactions;
    // Greedily add ranges to the subcompaction until the sum of the ranges'
    // sizes becomes >= the expected mean size of a subcompaction
    sum = 0;
    for (size_t i = 0; i < ranges.size() - 1; i++) {
      sum += ranges[i].size;
      if (subcompactions == 1) {
        // If there's only one left to schedule then it goes to the end so no
        // need to put an end boundary
        continue;
      }
      if (sum >= mean) {
        boundaries_.emplace_back(ExtractUserKey(ranges[i].range.limit));
        sizes_.emplace_back(sum);
        subcompactions--;
        sum = 0;
      }
    }
    sizes_.emplace_back(sum + ranges.back().size);
  } else {
    // Only one range so its size is the total sum of sizes computed above
    sizes_.emplace_back(sum);
  }
}
```
在计算 sub compaction 个数时，会选取下面几项中的最小值
* 用户配置的选项 `max_subcompaction` 参数
* `ranges` 的元素个数
* `max_output_files`

## CompactionJob::Run()
`CompactionJob::Run()` 的主要流程为
* 为每个 sub compaction 创建一个线程
* 每个线程分别调用 `CompactionJob::ProcessKeyValueCompaction()` 执行相应的 sub compaction
* 检查 compaction 结果并校验生成的 sst ，其中校验过程也会为每个 sub compaction 创建一个线程

这里节选部分实现
```c++
Status CompactionJob::Run() {
  // ...
  // 获取 CompactionJob::compact_ CompactionState::sub_compact_states
  // 的大小到 num_threads
  const size_t num_threads = compact_->sub_compact_states.size();
  assert(num_threads > 0);
  const uint64_t start_micros = env_->NowMicros();

  // Launch a thread for each of subcompactions 1...num_threads-1
  // 创建线程池用来执行 1 到 num_threads - 1 的 subcompaction
  std::vector<port::Thread> thread_pool;
  thread_pool.reserve(num_threads - 1);
  for (size_t i = 1; i < compact_->sub_compact_states.size(); i++) {
    // 创建线程执行 CompactionJob::ProcessKeyValueCompaction()
    thread_pool.emplace_back(&CompactionJob::ProcessKeyValueCompaction, this,
                             &compact_->sub_compact_states[i]);
  }

  // Always schedule the first subcompaction (whether or not there are also
  // others) in the current thread to be efficient with resources
  // 在当前线程执行第 0 个 subcompaction
  // 调用 ProcessKeyValueCompaction()
  ProcessKeyValueCompaction(&compact_->sub_compact_states[0]);

  // Wait for all other threads (if there are any) to finish execution
  for (auto& thread : thread_pool) {
    // join 线程池的所有线程
    thread.join();
  }
  // ...
  if (status.ok() && output_directory_) {
    status = output_directory_->Fsync();
  }
  // ...
}
```

`CompactionJob::ProcessKeyValueCompaction()` 会读取旧 sst 的数据并生成新的 sst ，具体的过程暂时不分析了

TODO 后续考虑分析 `CompactionJob::ProcessKeyValueCompaction()`

## 相关选项
以下是涉及到的一些配置选项
* `max_background_compactions`
* `level0_file_num_compaction_trigger`
* `max_bytes_for_level_base`
* `level_compaction_dynamic_level_bytes`
* `max_bytes_for_level_base`
* `max_bytes_for_level_multiplier`
* `max_bytes_for_level_multiplier_additional`
* `max_subcompaction`

## 参考
* [Compaction](https://github.com/facebook/rocksdb/wiki/Compaction)
* [MySQL · RocksDB · Level Compact 分析](http://mysql.taobao.org/monthly/2018/10/08/)
* [Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)
* [【Rocksdb实现分析及优化】level_compaction_dynamic_level_bytes](https://kernelmaker.github.io/Rocksdb_dynamic)
* [【Rocksdb实现分析及优化】 subcompaction](http://kernelmaker.github.io/Rocksdb_Study_5)
* [Sub Compaction](https://github.com/facebook/rocksdb/wiki/Sub-Compaction)
