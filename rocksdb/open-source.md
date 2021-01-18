# 打开流程
主要关心的问题
* 目录的创建
* 相关文件的创建
* 数据的恢复
* 相关对象的创建
* 不存在的 cf 的创建

## 目录结构
```
ls /tmp/rocksdb_column_families_example/
000009.log  CURRENT  IDENTITY  LOCK  LOG  LOG.old.1554188694245688  MANIFEST-000008  OPTIONS-000011  OPTIONS-000013

ls /tmp/rocksdb_open_example/
000003.log  CURRENT  IDENTITY  LOCK  LOG  MANIFEST-000001  OPTIONS-000005
```

## DB::Open()
打开 DB 的入口在 `DB::Open()` 函数，实现在 `db/db_impl_open.cc` 中。跟打开 DB 相关的逻辑大部分都在该文件中
```c++
Status DB::Open(const DBOptions& db_options, const std::string& dbname,
                const std::vector<ColumnFamilyDescriptor>& column_families,
                std::vector<ColumnFamilyHandle*>* handles, DB** dbptr) {
  const bool kSeqPerBatch = true;
  const bool kBatchPerTxn = true;
  return DBImpl::Open(db_options, dbname, column_families, handles, dbptr,
                      !kSeqPerBatch, kBatchPerTxn);
}
```

`DB::Open()` 的主体逻辑都在 `DBImpl::Open()` 中

## DBImpl::Open()
`DBImpl::Open()` 首先会对参数进行一些校验和初始化
```c++
// 调用各个 cf 的 table factory 对参数做校验
Status s = SanitizeOptionsByTable(db_options, column_families);
if (!s.ok()) {
  return s;
}

// 参数合法化校验
s = ValidateOptions(db_options, column_families);
if (!s.ok()) {
  return s;
}

// *dbptr 设置为空
*dbptr = nullptr;
// 清空 handles
handles->clear();
```

随后，创建 `DBImpl` 对象。在这时，会根据 `dbname` 创建 DB 目录
```c++
DBImpl* impl = new DBImpl(db_options, dbname, seq_per_batch, batch_per_txn);
```

创建 wal 和 sst 文件的目录
```c++
// 通过 env 创建 wal 目录
s = impl->env_->CreateDirIfMissing(impl->immutable_db_options_.wal_dir);
if (s.ok()) {
  // 通过 env 创建 sst 文件路径
  std::vector<std::string> paths;
  for (auto& db_path : impl->immutable_db_options_.db_paths) {
    paths.emplace_back(db_path.path);
  }
  for (auto& cf : column_families) {
    for (auto& cf_path : cf.options.cf_paths) {
      paths.emplace_back(cf_path.path);
    }
  }
  for (auto& path : paths) {
    s = impl->env_->CreateDirIfMissing(path);
    if (!s.ok()) {
      break;
    }
  }
}
```
默认情况下， wal 的目录 `wal_dir` 就是当前 DB 目录

在创建完所需要的目录后，就可以进行 DB 数据的恢复了。即通过重放 wal (代码里有时也叫 log file) 来恢复 DB 的状态。这部分的逻辑主要在 `DBImpl::Recover()` 中。整个过程中 DB 都需要上 `mutex_` 锁
```c++
// 恢复前需要先对 db 进行加锁
impl->mutex_.Lock();
s = impl->Recover(column_families);
if (s.ok()) {
  // 获取 log number (log file 就是 WAL)
  uint64_t new_log_number = impl->versions_->NewFileNumber();
  unique_ptr<WritableFile> lfile;
  EnvOptions soptions(db_options);
  EnvOptions opt_env_options =
      impl->immutable_db_options_.env->OptimizeForLogWrite(
          soptions, BuildDBOptions(impl->immutable_db_options_,
                                   impl->mutable_db_options_));
  // 根据 log number 生成 log 文件名
  std::string log_fname =
      LogFileName(impl->immutable_db_options_.wal_dir, new_log_number);
  // 创建新的 log file
  s = NewWritableFile(impl->immutable_db_options_.env, log_fname, &lfile,
                      opt_env_options);
}
// ...
```

在完成 DB 的恢复后，会创建一个新的 log file (即 wal)。`LogFileName()` 会为 wal 生成 `{number}.log` 格式的文件。我们看到的以 `.log` 为结尾的文件就是 DB 的 wal 了

随后，会更新 `DBImpl::logfile_number_` 和 `DBImpl::logs_`。这个过程需要对 `DBImpl::log_write_mutex_` 加锁
```c++
{
  // 上 DBImpl::log_write_mutex_ 锁
  InstrumentedMutexLock wl(&impl->log_write_mutex_);
  // 设置 DBImpl::logfile_number_ 为新的 log number
  impl->logfile_number_ = new_log_number;
  unique_ptr<WritableFileWriter> file_writer(new WritableFileWriter(
      std::move(lfile), log_fname, opt_env_options));
  // 构造 LogWriterNumber 并插入 DBImpl::logs_
  impl->logs_.emplace_back(
      new_log_number,
      new log::Writer(
          std::move(file_writer), new_log_number,
          impl->immutable_db_options_.recycle_log_file_num > 0,
          impl->immutable_db_options_.manual_wal_flush));
}
```

之前在恢复 DB 时， DB 中的 cf (column family) 对应的对象 `ColumnFamilyData` 都已经创建完成。这时可以设置用户传入的 `handles` 参数
```c++
for (auto cf : column_families) {
  // 尝试获取 ColumnFamilyData
  auto cfd =
      impl->versions_->GetColumnFamilySet()->GetColumnFamily(cf.name);
  if (cfd != nullptr) {
    handles->push_back(
        new ColumnFamilyHandleImpl(cfd, impl, &impl->mutex_));
    impl->NewThreadStatusCfInfo(cfd);
  } else {
    if (db_options.create_missing_column_families) {
      // missing column family, create it
      // 如果设置 create_missing_column_families 为 true 则创建不存在的 cf
      ColumnFamilyHandle* handle;
      // 解锁 DBImpl::mutex_
      impl->mutex_.Unlock();
      // 调用 DBImpl::CreateColumnFamily()
      s = impl->CreateColumnFamily(cf.options, cf.name, &handle);
      // 加锁 DBImpl::mutex_
      impl->mutex_.Lock();
      if (s.ok()) {
        handles->push_back(handle);
      } else {
        break;
      }
    } else {
      s = Status::InvalidArgument("Column family not found: ", cf.name);
      break;
    }
  }
}
```
上面的代码遍历用户传入的 `column_families` ，根据 cf 的名字到内存中查找对应的 `ColumnFamilyData` 对象。如果找到了，则可以插入 `handles` 。否则，说明这是原来 DB 里面没有的 cf 。这时需要检查 `DBOptions::create_missing_column_families` 参数。如果参数为 true ，则可以通过 `DBImpl::CreateColumnFamily()` 创建不存在的 cf ，然后插入 `handles` 。如果参数为 false ，则 DB 打开会失败

随后，对于每一个 cf ，都会调用 `DBImpl::InstallSuperVersionAndScheduleWork()` 函数。该函数会为 cf 生成一个新的 `SuperVersion` ，然后会尝试触发 flush 和 compaction 操作。`SuperVersion` 可以看作是 cf 当前 memtable 和 sst 的一个版本
```c++
SuperVersionContext sv_context(/* create_superversion */ true);
// 对于每个 cf 调用 DBImpl::InstallSuperVersionAndScheduleWork()
for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
  impl->InstallSuperVersionAndScheduleWork(
      cfd, &sv_context, *cfd->GetLatestMutableCFOptions());
}
sv_context.Clean();
```

然后，就可以更新 `DBImpl::alive_log_files_` 、清除无用的文件并进行目录的 sync 操作。
```c++
// DBImpl::two_write_queues_ 默认为 false
if (impl->two_write_queues_) {
  impl->log_write_mutex_.Lock();
}
// 更新 DBImpl::alive_log_files_
impl->alive_log_files_.push_back(
    DBImpl::LogFileNumberSize(impl->logfile_number_));
if (impl->two_write_queues_) {
  impl->log_write_mutex_.Unlock();
}
// 调用 DBImpl::DeleteObsoleteFiles() 删除无用文件
impl->DeleteObsoleteFiles();
// 调用 Fsync()
s = impl->directories_.GetDbDir()->Fsync();
```

对于一些设置了 merge operator 的 cf ，会检查对应的 cf 是否支持 merge operator
```c++
for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
  // ...
  // 检查是否支持 merge operator
  if (cfd->ioptions()->merge_operator != nullptr &&
      !cfd->mem()->IsMergeOperatorSupported()) {
    s = Status::InvalidArgument(
        "The memtable of column family %s does not support merge operator "
        "its options.merge_operator is non-null", cfd->GetName().c_str());
  }
  if (!s.ok()) {
    break;
  }
}
```

DB 会尝试将当前的 RocksDB Options 持久化，方便用户下次打开 DB 时可以复用 Options 。这些 Options 会被持久化到 `OPTIONS-{number}` 文件中。持久化完成后，会设置用户传入的 dbptr ，并标记 `DBImpl::opened_successfully_` 为 true 。然后调用 `DBImpl::MaybeScheduleFlushOrCompaction()` ，该函数会根据需要进行 flush 和 compaction 操作
```c++
// 将当前的选项写入到文件
persist_options_status = impl->WriteOptionsFile(
    false /*need_mutex_lock*/, false /*need_enter_write_thread*/);

// 打开操作基本完成，设置 *dbptr
*dbptr = impl;
// 设置 DBImpl::opened_successfully_ 为 true
impl->opened_successfully_ = true;
// 调用 DBImpl::MaybeScheduleFlushOrCompaction()
impl->MaybeScheduleFlushOrCompaction();
// ...
impl->mutex_.Unlock();
```

前面在恢复 DB 前会对 `DBImpl::mutex_` 进行加锁。到这之后，就可以加锁该锁了。至此， DB 的打开基本完成

## DBImpl
`DBImpl` 的构造函数主要是对 `DBImpl` 的成员进行一些初始化。在这个过程中，会调用 `SanitizeOptions()` 函数，其实现在 `db_impl_open.cc` 中
```c++
DBOptions SanitizeOptions(const std::string& dbname, const DBOptions& src) {
  // ...
  if (result.info_log == nullptr) {
    // 这里会创建日志文件，在此之前还会根据 dbname 创建目录
    Status s = CreateLoggerFromOptions(dbname, result, &result.info_log);
    if (!s.ok()) {
      // No place suitable for logging
      result.info_log = nullptr;
    }
  }
  // ...
}
```

`SanitizeOptions()` 会调用 `CreateLoggerFromOptions()` 来创建日志文件。如果发现日志文件所在的目录不存在，会根据 `dbname` 创建对应名字的目录。这时， DB 的目录就创建好了

## DBImpl::Recover()
恢复 DB 状态的主要逻辑都在 `DBImpl::Recover` 当中。这里主要考虑非 read only 模式，即读写模式下打开 DB 时的行为

为了保证 DB 不会被重复打开，打开 DB 时会先尝试打开 `LOCK` 文件，并上文件锁。如果上文件锁失败，则说明 DB 已经被打开了
```c++
// 根据 DBImpl::dbname_ 获取 lock file `LOCK` 并上文件锁
s = env_->LockFile(LockFileName(dbname_), &db_lock_);
if (!s.ok()) {
  return s;
}
```

通过判断 `CURRENT` 文件是否存在，可以判断 DB 是否已经存在了。如果 DB 不存在，则会通过调用 `DBImpl::NewDB()` 来创建一个新的 DB
```c++
// 判断 `CURRENT` 文件是否存在
s = env_->FileExists(CurrentFileName(dbname_));
if (s.IsNotFound()) {
  if (immutable_db_options_.create_if_missing) {
    // 如果不存在说明 db 不存在
    // 如果 create_if_missing 为 true 则调用 DBImpl::NewDB() 创建一个新 db
    s = NewDB();
    // 设置 is_new_db 标记为 true
    is_new_db = true;
    if (!s.ok()) {
      return s;
    }
  } else {
    return Status::InvalidArgument(
        dbname_, "does not exist (create_if_missing is false)");
  }
}
```

随后会为 DB 生成一个 `IDENTITY` 文件，里面会包含一个字符串作为 DB 的唯一 ID ，可以标识 DB 的身份
```c++
// 检查 `IDENTITY` 文件，如果不存在，则创建一个，作为 db 的身份标识
s = env_->FileExists(IdentityFileName(dbname_));
if (s.IsNotFound()) {
  // 不存在，调用 SetIdentityFile() 创建
  s = SetIdentityFile(env_, dbname_);
  if (!s.ok()) {
    return s;
  }
}
```

调用 `VersionSet::Recover()` ，这会读取 `MANIFEST` 文件并重放里面的 VersionEdit
```c++
Status s = versions_->Recover(column_families, read_only);
```

获取 wal 目录下的所有 log 文件
```c++
std::vector<std::string> filenames;
s = env_->GetChildren(immutable_db_options_.wal_dir, &filenames);
if (!s.ok()) {
  return s;
}

// 将 WAL 目录下的所有 log 文件记录到 logs
std::vector<uint64_t> logs;
for (size_t i = 0; i < filenames.size(); i++) {
  uint64_t number;
  FileType type;
  if (ParseFileName(filenames[i], &number, &type) && type == kLogFile) {
    if (is_new_db) {
      return Status::Corruption(
          "While creating a new Db, wal_dir contains "
          "existing log file: ",
          filenames[i]);
    } else {
      logs.push_back(number);
    }
  }
}
```

对 log 文件排序后，就可以调用 `DBImpl::RecoverLogFiles()` 重放 log 文件来恢复数据
```c++
// Recover in the order in which the logs were generated
// 将 logs 排序
std::sort(logs.begin(), logs.end());
// 调用 DBImpl::RecoverLogFiles() 恢复上次可能丢失的数据
s = RecoverLogFiles(logs, &next_sequence, read_only);
```

## DBImpl::NewDB()
`DBImpl::NewDB()` 会创建一个新的 `MANIFEST` 文件，文件名格式为 `MANIFEST-{number}` ，然后将文件名写入 `CURRENT` 文件。 `MANIFEST` 记录了对 DB 信息的修改，主要涉及文件的变更。创建新的 DB 也可以看作一项修改，也会生成 VersionEdit 并记录到 `MANIFEST` 文件中
```c++
// 创建一个新的 db
Status DBImpl::NewDB() {
  // 生成新建 db 的 VersionEdit new_db
  VersionEdit new_db;
  new_db.SetLogNumber(0);
  new_db.SetNextFile(2);
  new_db.SetLastSequence(0);

  Status s;

  ROCKS_LOG_INFO(immutable_db_options_.info_log, "Creating manifest 1 \n");
  // 生成 manifest 文件名 `/MANIFEST-000001`
  const std::string manifest = DescriptorFileName(dbname_, 1);
  {
    unique_ptr<WritableFile> file;
    EnvOptions env_options = env_->OptimizeForManifestWrite(env_options_);
    s = NewWritableFile(env_, manifest, &file, env_options);
    if (!s.ok()) {
      return s;
    }
    file->SetPreallocationBlockSize(
        immutable_db_options_.manifest_preallocation_size);
    unique_ptr<WritableFileWriter> file_writer(
        new WritableFileWriter(std::move(file), manifest, env_options));
    log::Writer log(std::move(file_writer), 0, false);
    std::string record;
    // VersionEdit new_db 写入 manifest
    new_db.EncodeTo(&record);
    s = log.AddRecord(record);
    // 调用 SyncManifest() 同步 manifest
    if (s.ok()) {
      s = SyncManifest(env_, &immutable_db_options_, log.file());
    }
  }
  if (s.ok()) {
    // Make "CURRENT" file that points to the new manifest file.
    // 将 `CURRENT` 文件指向当前 manifest
    s = SetCurrentFile(env_, dbname_, 1, directories_.GetDbDir());
  } else {
    env_->DeleteFile(manifest);
  }
  return s;
}
```

## VersionSet::Recover()
`VersionSet::Recover()` 的实现在 `db/version_set.cc` 中

首先，进行一些变量的初始化
```c++
Status VersionSet::Recover(
    const std::vector<ColumnFamilyDescriptor>& column_families,
    bool read_only) {
  std::unordered_map<std::string, ColumnFamilyOptions> cf_name_to_options;
  for (auto cf : column_families) {
    cf_name_to_options.insert({cf.name, cf.options});
  }
  // keeps track of column families in manifest that were not found in
  // column families parameters. if those column families are not dropped
  // by subsequent manifest records, Recover() will return failure status
  std::unordered_map<int, std::string> column_families_not_found;
  // ...
}
```

从 `CURRENT` 文件中读取 `MANIFEST` 文件的名字。会通过文件末尾是否包含 `\n` 来判断文件是否完整
```c++
std::string manifest_filename;
Status s = ReadFileToString(
    env_, CurrentFileName(dbname_), &manifest_filename
);
// ...
// 通过 manifest 文件是否以 `\n` 结尾判断文件名是否完整
if (manifest_filename.empty() ||
    manifest_filename.back() != '\n') {
  return Status::Corruption("CURRENT file does not end with newline");
}
// remove the trailing '\n'
manifest_filename.resize(manifest_filename.size() - 1);
```

创建 `SequentialFileReader` 来读取 `MANIFEST` 文件
```c++
manifest_filename = dbname_ + "/" + manifest_filename;
unique_ptr<SequentialFileReader> manifest_file_reader;
{
  unique_ptr<SequentialFile> manifest_file;
  s = env_->NewSequentialFile(manifest_filename, &manifest_file,
                              env_->OptimizeForManifestRead(env_options_));
  if (!s.ok()) {
    return s;
  }
  manifest_file_reader.reset(
      new SequentialFileReader(std::move(manifest_file), manifest_filename));
}
```

DB 必然包含一个 `default` cf ，因此恢复时会直接先创建一个 `default` cf
```c++
std::unordered_map<uint32_t, BaseReferencedVersionBuilder*> builders;
// add default column family
auto default_cf_iter = cf_name_to_options.find(kDefaultColumnFamilyName);
if (default_cf_iter == cf_name_to_options.end()) {
  return Status::InvalidArgument("Default column family not specified");
}
VersionEdit default_cf_edit;
default_cf_edit.AddColumnFamily(kDefaultColumnFamilyName);
default_cf_edit.SetColumnFamily(0);
ColumnFamilyData* default_cfd =
    CreateColumnFamily(default_cf_iter->second, &default_cf_edit);
// In recovery, nobody else can access it, so it's fine to set it to be
// initialized earlier.
default_cfd->set_initialized();
// 为 default cf 创建 builders
builders.insert({0, new BaseReferencedVersionBuilder(default_cfd)});
```

从 `MANIFEST` 文件中一次读取出所有的 record (记录) 并解码成 `VersionEdit` ，然后调用 `VersionSet::ApplyOneVersionEdit()` 将 `VersionEdit` 的修改重新应用到 DB
```c++
log::Reader reader(nullptr, std::move(manifest_file_reader), &reporter,
                   true /* checksum */, 0 /* log_number */);
Slice record;
std::string scratch;
// ...
// 从 manifest 文件中依次读取每个 record
while (reader.ReadRecord(&record, &scratch) && s.ok()) {
  // 从 record 中通过 VersionEdit::DecodeFrom 解码出对应的 VersionEdit
  VersionEdit edit;
  s = edit.DecodeFrom(record);
  if (!s.ok()) {
    break;
  }

  // ...

  // 调用 VersionSet::ApplyOneVersionEdit() 校验并应用 VersionEdit
  s = ApplyOneVersionEdit(
      edit, cf_name_to_options, column_families_not_found, builders,
      &have_log_number, &log_number, &have_prev_log_number,
      &previous_log_number, &have_next_file, &next_file,
      &have_last_sequence, &last_sequence, &min_log_number_to_keep,
      &max_column_family);
  if (!s.ok()) {
    break;
  }
}
```

处理完 `MANIFEST` 文件的数据后，就可以更新 DB 的一些信息
```c++
if (s.ok()) {
  // ...
  if (!have_prev_log_number) {
    previous_log_number = 0;
  }

  column_family_set_->UpdateMaxColumnFamily(max_column_family);

  // When reading DB generated using old release, min_log_number_to_keep=0.
  // All log files will be scanned for potential prepare entries.
  MarkMinLogNumberToKeep2PC(min_log_number_to_keep);
  MarkFileNumberUsed(previous_log_number);
  MarkFileNumberUsed(log_number);
}
```

如果发现用户没有打开所有的 cf ，则在非 read only 模式下需要报错
```c++
// there were some column families in the MANIFEST that weren't specified
// in the argument. This is OK in read_only mode
if (read_only == false && !column_families_not_found.empty()) {
  std::string list_of_not_found;
  for (const auto& cf : column_families_not_found) {
    list_of_not_found += ", " + cf.second;
  }
  list_of_not_found = list_of_not_found.substr(2);
  s = Status::InvalidArgument(
      "You have to open all column families. Column families not opened: " +
      list_of_not_found);
}
```

然后就可以为每个 cf 都生成新的 `Version` 。之前每个 cf 都有一个对应的 `VersionBuilder` ，该 cf 的 `VersionEdit` 都会先应用到 `VersionBuilder` 中，最后统一生成一个 `Version`
```c++
for (auto cfd : *column_family_set_) {
  if (cfd->IsDropped()) {
    continue;
  }
  if (read_only) {
    cfd->table_cache()->SetTablesAreImmortal();
  }
  assert(cfd->initialized());
  auto builders_iter = builders.find(cfd->GetID());
  assert(builders_iter != builders.end());
  // 获取对应的 VersionBuilder
  auto* builder = builders_iter->second->version_builder();

  // ...

  // 创建并加载新的 Version
  Version* v = new Version(cfd, this, env_options_,
                           *cfd->GetLatestMutableCFOptions(),
                           current_version_number_++);
  builder->SaveTo(v->storage_info());

  // Install recovered version
  // 将 Version 加入链表
  v->PrepareApply(*cfd->GetLatestMutableCFOptions(),
      !(db_options_->skip_stats_update_on_db_open));
  AppendVersion(cfd, v);
}
```

最后是一些信息更新
```c++
manifest_file_size_ = current_manifest_file_size;
next_file_number_.store(next_file + 1);
last_allocated_sequence_ = last_sequence;
last_published_sequence_ = last_sequence;
last_sequence_ = last_sequence;
prev_log_number_ = previous_log_number;
```

## VersionSet::ApplyOneVersionEdit()
该函数的参数比较多，不用太在意每个参数的含义。这些参数主要是为了从 `VersionEdit` 中提取对应的信息
```c++
Status VersionSet::ApplyOneVersionEdit(
    VersionEdit& edit,
    const std::unordered_map<std::string, ColumnFamilyOptions>& name_to_options,
    std::unordered_map<int, std::string>& column_families_not_found,
    std::unordered_map<uint32_t, BaseReferencedVersionBuilder*>& builders,
    bool* have_log_number, uint64_t* /* log_number */,
    bool* have_prev_log_number, uint64_t* previous_log_number,
    bool* have_next_file, uint64_t* next_file, bool* have_last_sequence,
    SequenceNumber* last_sequence, uint64_t* min_log_number_to_keep,
    uint32_t* max_column_family) {
  // Not found means that user didn't supply that column
  // family option AND we encountered column family add
  // record. Once we encounter column family drop record,
  // we will delete the column family from
  // column_families_not_found.
  // cf 在 column_families_not_found 中，表明用户没有提供该 cf 的 option
  bool cf_in_not_found = (column_families_not_found.find(edit.column_family_) !=
                          column_families_not_found.end());
  // in builders means that user supplied that column family
  // option AND that we encountered column family add record
  // cf 在 builders 中，表明用户提供了该 cf 的 option
  bool cf_in_builders = builders.find(edit.column_family_) != builders.end();
  // ...
}
```

然后就可以根据 `VersionEdit` 的类型进行相应的恢复
```c++
if (edit.is_column_family_add_) {
  // 如果是新增 cf 操作，则不能是已有的 cf
  if (cf_in_builders || cf_in_not_found) {
    return Status::Corruption(
        "Manifest adding the same column family twice: " +
        edit.column_family_name_);
  }
  // 从 name_to_options 查找对应的 option
  auto cf_options = name_to_options.find(edit.column_family_name_);
  if (cf_options == name_to_options.end()) {
    // 如果找不到，插入 column_families_not_found
    column_families_not_found.insert(
        {edit.column_family_, edit.column_family_name_});
  } else {
    // 找到了，调用 VersionSet::CreateColumnFamily() 创建 cf
    cfd = CreateColumnFamily(cf_options->second, &edit);
    cfd->set_initialized();
    builders.insert(
        {edit.column_family_, new BaseReferencedVersionBuilder(cfd)});
  }
} else if (edit.is_column_family_drop_) {
  // 如果是删除 cf
  if (cf_in_builders) {
    // 如果 cf_in_builders 为 true ，删除对应的 bulder
    auto builder = builders.find(edit.column_family_);
    assert(builder != builders.end());
    delete builder->second;
    builders.erase(builder);
    cfd = column_family_set_->GetColumnFamily(edit.column_family_);
    assert(cfd != nullptr);
    // 如果 cfd 不再被引用，清理 cfd
    if (cfd->Unref()) {
      delete cfd;
      cfd = nullptr;
    } else {
      // who else can have reference to cfd!?
      assert(false);
    }
  } else if (cf_in_not_found) {
    // 如果 cf_in_not_found 为 true ，从 cf_in_not_found 删除 cf
    column_families_not_found.erase(edit.column_family_);
  } else {
    return Status::Corruption(
        "Manifest - dropping non-existing column family");
  }
} else if (!cf_in_not_found) {
  if (!cf_in_builders) {
    return Status::Corruption(
        "Manifest record referencing unknown column family");
  }

  cfd = column_family_set_->GetColumnFamily(edit.column_family_);
  // this should never happen since cf_in_builders is true
  assert(cfd != nullptr);

  // if it is not column family add or column family drop,
  // then it's a file add/delete, which should be forwarded
  // to builder
  // 其他操作(文件的增删)，调用 VersionBuilder::Apply() 处理
  auto builder = builders.find(edit.column_family_);
  assert(builder != builders.end());
  // 主要负责创建或删除文件对应的 FileMetaData 对象
  builder->second->version_builder()->Apply(&edit);
}
```
每个 cf 都有一个对应的 `ColumnFamilyData` 对象。在 cf 的 `Version` 中，还维护了当前的所有 sst 文件列表。在恢复时就需要根据 `MANIFEST` 中的 `VersionEdit` 来重建这些信息
* 如果发现创建 cf 的 `VersionEdit` ，则会调用 `VersionSet::CreateColumnFamily()` 来创建一个新的 `ColumnFamilyData` 对象
* 如果发现删除 cf 的 `VersionEdit` ，则会删除对应的 `ColumnFamilyData` 对象
* 否则，则说明是其他操作，即文件的增删。该 `VersionEdit` 会交由 `VersionBuilder::Apply()` 做处理，以更新文件列表的信息

## DBImpl::RecoverLogFiles()
`DBImpl::RecoverLogFiles()` 负责从 wal 中恢复数据。函数的实现在 `db/db_impl_open.cc` 中。传入该函数的 log number 需要是排好序的。这样，就可以按顺序对数据进行恢复

首先需要初始化一些变量
```c++
// REQUIRES: log_numbers are sorted in ascending order
Status DBImpl::RecoverLogFiles(const std::vector<uint64_t>& log_numbers,
                               SequenceNumber* next_sequence, bool read_only) {
  // ...
  mutex_.AssertHeld();
  Status status;
  std::unordered_map<int, VersionEdit> version_edits;
  // no need to refcount because iteration is under mutex
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    VersionEdit edit;
    edit.SetColumnFamily(cfd->GetID());
    version_edits.insert({cfd->GetID(), edit});
  }
  int job_id = next_job_id_.fetch_add(1);
  // ...
  bool stop_replay_by_wal_filter = false;
  bool stop_replay_for_corruption = false;
  bool flushed = false;
  uint64_t corrupted_log_number = kMaxSequenceNumber;
  // ...
}
```

按照顺序遍历每一个 log number 对应的 wal 。对每一个 wal 都会打开 `log::Reader` 并准备从里面读取 record
```c++
// 遍历每个 log number
for (auto log_number : log_numbers) {
  // ...
  // The previous incarnation may not have written any MANIFEST
  // records after allocating this log number.  So we manually
  // update the file number allocation counter in VersionSet.
  versions_->MarkFileNumberUsed(log_number);
  // Open the log file
  // 生成 log 文件名
  std::string fname = LogFileName(immutable_db_options_.wal_dir, log_number);
  // ...

  // 打开 log 文件
  unique_ptr<SequentialFileReader> file_reader;
  {
    unique_ptr<SequentialFile> file;
    status = env_->NewSequentialFile(fname, &file,
                                     env_->OptimizeForLogRead(env_options_));
    // ...
    file_reader.reset(new SequentialFileReader(std::move(file), fname));
  }

  // We intentially make log::Reader do checksumming even if
  // paranoid_checks==false so that corruptions cause entire commits
  // to be skipped instead of propagating bad information (like overly
  // large sequence numbers).
  log::Reader reader(immutable_db_options_.info_log, std::move(file_reader),
                     &reporter, true /*checksum*/, log_number);
  // ...
}
```

一个 wal 的数据处理过程如下
```c++
// Determine if we should tolerate incomplete records at the tail end of the
// Read all the records and add to a memtable
std::string scratch;
Slice record;
WriteBatch batch;

// 调用 Reader::ReadRecord() 从文件中读取 record
while (!stop_replay_by_wal_filter &&
       reader.ReadRecord(&record, &scratch,
                         immutable_db_options_.wal_recovery_mode) &&
       status.ok()) {
  if (record.size() < WriteBatchInternal::kHeader) {
    reporter.Corruption(record.size(),
                        Status::Corruption("log record too small"));
    continue;
  }
  // 用 record 的数据设置 WriteBatch
  WriteBatchInternal::SetContents(&batch, record);
  
  // ...

  // If column family was not found, it might mean that the WAL write
  // batch references to the column family that was dropped after the
  // insert. We don't want to fail the whole write batch in that case --
  // we just ignore the update.
  // That's why we set ignore missing column families to true
  // 调用 WriteBatchInternal::InsertInto() 将 WriteBatch 插入 memtable
  bool has_valid_writes = false;
  status = WriteBatchInternal::InsertInto(
      &batch, column_family_memtables_.get(), &flush_scheduler_, true,
      log_number, this, false /* concurrent_memtable_writes */,
      next_sequence, &has_valid_writes, seq_per_batch_, batch_per_txn_);

  // ...

  if (has_valid_writes && !read_only) {
    // we can do this because this is called before client has access to the
    // DB and there is only a single thread operating on DB
    ColumnFamilyData* cfd;

    // 如果 FlushScheduler::TakeNextColumnFamily() 能够返回下一个 cfd
    while ((cfd = flush_scheduler_.TakeNextColumnFamily()) != nullptr) {
      cfd->Unref();
      // If this asserts, it means that InsertInto failed in
      // filtering updates to already-flushed column families
      assert(cfd->GetLogNumber() <= log_number);
      auto iter = version_edits.find(cfd->GetID());
      assert(iter != version_edits.end());
      // 找到 cfd 对应的 VersionEdit
      VersionEdit* edit = &iter->second;
      // 调用 DBImpl::WriteLevel0TableForRecovery() 将 memtable 刷盘
      status = WriteLevel0TableForRecovery(job_id, cfd, cfd->mem(), edit);
      if (!status.ok()) {
        // Reflect errors immediately so that conditions like full
        // file-systems cause the DB::Open() to fail.
        return status;
      }
      flushed = true;

      // 为 cfd 创建新的 memtable
      cfd->CreateNewMemtable(*cfd->GetLatestMutableCFOptions(),
                             *next_sequence);
    }
  }
}

// ...
// 调用 FlushScheduler::Clear()
flush_scheduler_.Clear();
// 用 record 中的 sequence number 修正 DBImpl::versions_ 的当前 sequence
auto last_sequence = *next_sequence - 1;
if ((*next_sequence != kMaxSequenceNumber) &&
    (versions_->LastSequence() <= last_sequence)) {
  versions_->SetLastAllocatedSequence(last_sequence);
  versions_->SetLastPublishedSequence(last_sequence);
  versions_->SetLastSequence(last_sequence);
}
```

处理时会遍历 wal 中的每一个 record ，用 record 的数据填充一个 `WriteBatch` 的内容。然后通过调用 `WriteBatchInternal::InsertInto()` 将 `WriteBatch` 插入 memtable 。通过这个方法，就可以将 wal 的数据重新恢复到 memtable 中

调用 `WriteBatchInternal::InsertInto()` 时会传入 `log_number` 作为 `MemTableInserter` 的 `recovering_log_number_` 成员。如果发现 `recovering_log_number_` 小于 memtable 的 log number ，则插入时会自动忽略要插入的数据，因为对于 log 的数据上次已经恢复了

在恢复 wal 的数据的过程中，会检查是否需要将 memtable 刷盘。如果需要，则会调用 `DBImpl::WriteLevel0TableForRecovery()` 将 memtable 刷盘，产生 sst ，随后创建新的 memtable

每处理完一个 wal ，都可以用 record 中的 sequence number 修正当前 VersionSet 的 sequence number

处理完所有 wal 后，则 wal 中的数据都已恢复。随后会遍历所有的 cf
* 如果 cf 的 log number 已经大于当前的 log number ，则说明当前的 log 里已经没有 cf 需要的 log 了，可以不用处理该 cf
* 如果 cf 的 memtable 中有数据且恢复时已经刷过盘，则直接将 memtable 刷盘，这样该 cf 的数据要么都没刷盘，要么都刷盘了
* cf 对应的 `VersionEdit` 记录最大的 log number ，表示在这之前的 log 下次恢复时都可以忽略了。然后通过调用 `VersionSet::LogAndApply()` 将 `VersionEdit` 写入到 `MANIFEST` 文件。这应该是在前面提到的 `WriteBatchInternal::InsertInto()` 时处理的
```c++
// True if there's any data in the WALs; if not, we can skip re-processing
// them later
bool data_seen = false;
if (!read_only) {
  // no need to refcount since client still doesn't have access
  // to the DB and can not drop column families while we iterate
  auto max_log_number = log_numbers.back();
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    auto iter = version_edits.find(cfd->GetID());
    assert(iter != version_edits.end());
    VersionEdit* edit = &iter->second;

    // 如果 cf 已经 flush 了 log 中的所有数据到 sst 中
    if (cfd->GetLogNumber() > max_log_number) {
      // Column family cfd has already flushed the data
      // from all logs. Memtable has to be empty because
      // we filter the updates based on log_number
      // (in WriteBatch::InsertInto)
      assert(cfd->mem()->GetFirstSequenceNumber() == 0);
      assert(edit->NumEntries() == 0);
      continue;
    }

    // flush the final memtable (if non-empty)
    // flush 最后一个 memtable
    if (cfd->mem()->GetFirstSequenceNumber() != 0) {
      // If flush happened in the middle of recovery (e.g. due to memtable
      // being full), we flush at the end. Otherwise we'll need to record
      // where we were on last flush, which make the logic complicated.
      if (flushed || !immutable_db_options_.avoid_flush_during_recovery) {
        status = WriteLevel0TableForRecovery(job_id, cfd, cfd->mem(), edit);
        if (!status.ok()) {
          // Recovery failed
          break;
        }
        flushed = true;

        cfd->CreateNewMemtable(*cfd->GetLatestMutableCFOptions(),
                               versions_->LastSequence());
      }
      data_seen = true;
    }

    // write MANIFEST with update
    // writing log_number in the manifest means that any log file
    // with number strongly less than (log_number + 1) is already
    // recovered and should be ignored on next reincarnation.
    // Since we already recovered max_log_number, we want all logs
    // with numbers `<= max_log_number` (includes this one) to be ignored
    if (flushed || cfd->mem()->GetFirstSequenceNumber() == 0) {
      edit->SetLogNumber(max_log_number + 1);
    }
    // we must mark the next log number as used, even though it's
    // not actually used. that is because VersionSet assumes
    // VersionSet::next_file_number_ always to be strictly greater than any
    // log number
    versions_->MarkFileNumberUsed(max_log_number + 1);
    // 调用 VersionSet::LogAndApply() 更新 manifest 文件
    status = versions_->LogAndApply(
        cfd, *cfd->GetLatestMutableCFOptions(), edit, &mutex_);
    if (!status.ok()) {
      // Recovery failed
      break;
    }
  }
}
```

TODO 后续有时间可以介绍下 `VersionSet::LogAndApply()` 的实现

最后，如果发现从 wal 中恢复了数据，并且这些数据没有刷盘，则仍然需要保留这些 wal 文件。调用 `DBImpl::RestoreAliveLogFiles()` 标记这些文件为 alive 以避免后面被 `DBImpl::FindObsoleteFiles()` 标记为删除
```c++
if (status.ok() && data_seen && !flushed) {
  // 如果有 log 数据且没有 flush 过，则这些 log 文件肯定都还有用
  // 调用 DBImpl::RestoreAliveLogFiles() 用 log_numbers
  // 恢复 DBImpl::alive_log_files_ 和 DBImpl::total_log_size_
  status = RestoreAliveLogFiles(log_numbers);
}
```
