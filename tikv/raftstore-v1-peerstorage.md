# TiKV v1.0 raftstore PeerStorage

## PeerStorage
PeerStorage 负责 Peer 的状态存储，同时也实现了 raft-rs 的 Storage 接口，每个 Peer 实例都有一个对应的 PeerStorage 实例
```rust
pub struct PeerStorage {
    pub kv_engine: Arc<DB>,
    pub raft_engine: Arc<DB>,

    pub region: metapb::Region,
    pub raft_state: RaftLocalState,
    pub apply_state: RaftApplyState,
    pub applied_index_term: u64,
    pub last_term: u64,

    snap_state: RefCell<SnapState>,
    region_sched: Scheduler<RegionTask>,
    snap_tried_cnt: RefCell<usize>,

    cache: EntryCache,
    stats: Rc<RefCell<CacheQueryStats>>,

    pub tag: String,
}
```

## PeerStorage::new()
PeerStorage::new() 会创建一个 PeerStorage
```rust
pub fn new(
    kv_engine: Arc<DB>,
    raft_engine: Arc<DB>,
    region: &metapb::Region,
    region_sched: Scheduler<RegionTask>,
    tag: String,
    stats: Rc<RefCell<CacheQueryStats>>,
) -> Result<PeerStorage> {
    debug!("creating storage on {} for {:?}", kv_engine.path(), region);
    let raft_state = init_raft_state(&raft_engine, region)?;
    let apply_state = init_apply_state(&kv_engine, region)?;
    if raft_state.get_last_index() < apply_state.get_applied_index() {
        panic!(
            "{} unexpected raft log index: last_index {} < applied_index {}",
            tag,
            raft_state.get_last_index(),
            apply_state.get_applied_index()
        );
    }
    let last_term = init_last_term(&raft_engine, region, &raft_state, &apply_state)?;

    Ok(PeerStorage {
        kv_engine: kv_engine,
        raft_engine: raft_engine,
        region: region.clone(),
        raft_state: raft_state,
        apply_state: apply_state,
        snap_state: RefCell::new(SnapState::Relax),
        region_sched: region_sched,
        snap_tried_cnt: RefCell::new(0),
        tag: tag,
        applied_index_term: RAFT_INIT_LOG_TERM,
        last_term: last_term,
        cache: EntryCache::default(),
        stats: stats,
    })
}
```

主要流程
- init_raft_state() 得到 raft_state
- init_apply_state() 得到 apply_state
- 保证 RaftLocalState::get_last_index() >= RaftApplyState::get_applied_index()
- init_last_term() 得到 last_term
- 创建 PeerStorage

### init_raft_state()
```rust
fn init_raft_state(raft_engine: &DB, region: &Region) -> Result<RaftLocalState> {
    let state_key = keys::raft_state_key(region.get_id());
    Ok(match raft_engine.get_msg(&state_key)? {
        Some(s) => s,
        None => {
            let mut raft_state = RaftLocalState::new();
            if !region.get_peers().is_empty() {
                // new split region
                raft_state.set_last_index(RAFT_INIT_LOG_INDEX);
                raft_state.mut_hard_state().set_term(RAFT_INIT_LOG_TERM);
                raft_state.mut_hard_state().set_commit(RAFT_INIT_LOG_INDEX);
                raft_engine.put_msg(&state_key, &raft_state)?;
            }
            raft_state
        }
    })
}
```
init_raft_state() 主要从存储中恢复 raft state
- 通过 keys::raft_state_key(region.get_id()) 从 raft_engine 里 get 到对应 region 的状态
- 如果没有则初始化状态
    - 如果 region 里的 peer 不为空，则
        - 设置 last_index 为 RAFT_INIT_LOG_INDEX
        - 设置 term 为 RAFT_INIT_LOG_TERM
        - 设置 commit 为 RAFT_INIT_LOG_INDEX
        - 持久化状态

### init_apply_state()
```rust
fn init_apply_state(kv_engine: &DB, region: &Region) -> Result<RaftApplyState> {
    Ok(match try!(
        kv_engine.get_msg_cf(CF_RAFT, &keys::apply_state_key(region.get_id()))
    ) {
        Some(s) => s,
        None => {
            let mut apply_state = RaftApplyState::new();
            if !region.get_peers().is_empty() {
                apply_state.set_applied_index(RAFT_INIT_LOG_INDEX);
                let state = apply_state.mut_truncated_state();
                state.set_index(RAFT_INIT_LOG_INDEX);
                state.set_term(RAFT_INIT_LOG_TERM);
            }
            apply_state
        }
    })
}
```
init_apply_state() 同理
- 通过 keys::apply_state_key(region.get_id()) 从 kv_engine 里 get 对应 region 的状态
- 没有则初始化状态
    - 如果 region 里的 peer 不为空，则
        - applied_index 初始化为 RAFT_INIT_LOG_INDEX
        - truncated_state 初始化 index 为 RAFT_INIT_LOG_INDEX
        - truncated_state 初始化 term 为 RAFT_INIT_LOG_TERM

### init_last_term()

```rust
fn init_last_term(
    raft_engine: &DB,
    region: &Region,
    raft_state: &RaftLocalState,
    apply_state: &RaftApplyState,
) -> Result<u64> {
    let last_idx = raft_state.get_last_index();
    if last_idx == 0 {
        return Ok(0);
    } else if last_idx == RAFT_INIT_LOG_INDEX {
        return Ok(RAFT_INIT_LOG_TERM);
    } else if last_idx == apply_state.get_truncated_state().get_index() {
        return Ok(apply_state.get_truncated_state().get_term());
    } else {
        assert!(last_idx > RAFT_INIT_LOG_INDEX);
    }
    let last_log_key = keys::raft_log_key(region.get_id(), last_idx);
    Ok(match raft_engine.get_msg::<Entry>(&last_log_key)? {
        None => {
            return Err(box_err!(
                "[region {}] entry at {} doesn't exist, may lose data.",
                region.get_id(),
                last_idx
            ))
        }
        Some(e) => e.get_term(),
    })
}
```

主要流程
- 从 raft_state 里获取 last_idx
    - 如果 last_idx == 0 ，则返回 0
    - 如果 last_idx == RAFT_INIT_LOG_INDEX 则返回 RAFT_INIT_LOG_TERM
    - 如果 last_idx == apply_state.get_truncated_state().get_index() 则返回 apply_state.get_truncated_state().get_term()
    - 否则 last_idx 必须大于 RAFT_INIT_LOG_INDEX
- keys::raft_log_key(region.get_id(), last_idx) 得到 last_log 在存储里的 key last_log_key
- 从 raft_engine 里 get 对应的 Entry
- 如果没有，报错
- 有，则返回 entry 的 term

## prepare_bootstrap()
Tikv 在启动时，如果之前没有初始化过，会调用 prepare_bootstrap() 进行初始化
```rust
// Prepare bootstrap.
pub fn prepare_bootstrap(
    engines: &Engines,
    store_id: u64,
    region_id: u64,
    peer_id: u64,
) -> Result<metapb::Region> {
    let mut region = metapb::Region::new();
    region.set_id(region_id);
    region.set_start_key(keys::EMPTY_KEY.to_vec());
    region.set_end_key(keys::EMPTY_KEY.to_vec());
    region.mut_region_epoch().set_version(INIT_EPOCH_VER);
    region.mut_region_epoch().set_conf_ver(INIT_EPOCH_CONF_VER);

    let mut peer = metapb::Peer::new();
    peer.set_store_id(store_id);
    peer.set_id(peer_id);
    region.mut_peers().push(peer);

    write_prepare_bootstrap(engines, &region)?;

    Ok(region)
}
```

而 prepare_bootstrap() 则会调用 write_prepare_bootstrap()
```rust
// Write first region meta and prepare state.
pub fn write_prepare_bootstrap(engines: &Engines, region: &metapb::Region) -> Result<()> {
    let mut state = RegionLocalState::new();
    state.set_region(region.clone());

    let wb = WriteBatch::new();
    wb.put_msg(&keys::prepare_bootstrap_key(), region)?;
    let handle = rocksdb::get_cf_handle(&engines.kv_engine, CF_RAFT)?;
    wb.put_msg_cf(handle, &keys::region_state_key(region.get_id()), &state)?;
    write_initial_apply_state(&engines.kv_engine, &wb, region.get_id())?;
    engines.kv_engine.write(wb)?;
    engines.kv_engine.sync_wal()?;

    let raft_wb = WriteBatch::new();
    write_initial_raft_state(&raft_wb, region.get_id())?;
    engines.raft_engine.write(raft_wb)?;
    engines.raft_engine.sync_wal()?;
    Ok(())
}
```

write_prepare_bootstrap() 会
- keys::prepare_bootstrap_key() 和 region 信息（初始化时分配到的第一个 region）写入 wb
- keys::region_state_key(region.get_id()) 和 RegionLocalState 写入 wb
- 调用 write_initial_apply_state(&engines.kv_engine, &wb, region.get_id())
- wb 写入 kv_engine 并 sync wal
- 调用 write_initial_raft_state(&raft_wb, region.get_id())
- raft_wb 写入 raft_engine 并 sync wal

### write_initial_apply_state()
init_apply_state() 中的状态就是在 write_initial_apply_state() 时写入的
```rust
// When we bootstrap the region or handling split new region, we must
// call this to initialize region apply state first.
pub fn write_initial_apply_state<T: Mutable>(
    kv_engine: &DB,
    kv_wb: &T,
    region_id: u64,
) -> Result<()> {
    let mut apply_state = RaftApplyState::new();
    apply_state.set_applied_index(RAFT_INIT_LOG_INDEX);
    apply_state
        .mut_truncated_state()
        .set_index(RAFT_INIT_LOG_INDEX);
    apply_state
        .mut_truncated_state()
        .set_term(RAFT_INIT_LOG_TERM);

    let handle = rocksdb::get_cf_handle(kv_engine, CF_RAFT)?;
    kv_wb
        .put_msg_cf(handle, &keys::apply_state_key(region_id), &apply_state)?;
    Ok(())
}
```

### write_initial_raft_state()
同理 init_raft_state() 中得到的状态也是通过 write_initial_raft_state() 写入的
```rust
// When we bootstrap the region we must call this to initialize region local state first.
pub fn write_initial_raft_state<T: Mutable>(raft_wb: &T, region_id: u64) -> Result<()> {
    let mut raft_state = RaftLocalState::new();
    raft_state.set_last_index(RAFT_INIT_LOG_INDEX);
    raft_state.mut_hard_state().set_term(RAFT_INIT_LOG_TERM);
    raft_state.mut_hard_state().set_commit(RAFT_INIT_LOG_INDEX);

    raft_wb
        .put_msg(&keys::raft_state_key(region_id), &raft_state)?;
    Ok(())
}
```

## PeerStorage::check_applying_snap()
PeerStorage::check_applying_snap() 会判断 snapshot 目前进行的状态
- 根据 snapshot 的状态更新 PeerStorage::snap_state
- 返回是否正在 applying snapshot

```rust
pub const JOB_STATUS_PENDING: usize = 0;
pub const JOB_STATUS_RUNNING: usize = 1;
pub const JOB_STATUS_CANCELLING: usize = 2;
pub const JOB_STATUS_CANCELLED: usize = 3;
pub const JOB_STATUS_FINISHED: usize = 4;
pub const JOB_STATUS_FAILED: usize = 5;


#[derive(Debug)]
pub enum SnapState {
    Relax,
    Generating(Receiver<Snapshot>),
    Applying(Arc<AtomicUsize>),
    ApplyAborted,
}

/// Check if the storage is applying a snapshot.
#[inline]
pub fn check_applying_snap(&mut self) -> bool {
    let new_state = match *self.snap_state.borrow() {
        SnapState::Applying(ref status) => {
            let s = status.load(Ordering::Relaxed);
            if s == JOB_STATUS_FINISHED {
                SnapState::Relax
            } else if s == JOB_STATUS_CANCELLED {
                SnapState::ApplyAborted
            } else if s == JOB_STATUS_FAILED {
                // TODO: cleanup region and treat it as tombstone.
                panic!("{} applying snapshot failed", self.tag);
            } else {
                return true;
            }
        }
        _ => return false,
    };
    *self.snap_state.borrow_mut() = new_state;
    false
}
```

## PeerStorage::handle_raft_ready()
Peer::handle_raft_ready_append() 会调用这个函数来将 Ready 中的数据写入到 WriteBatch 中
```rust
/// Save memory states to disk.
///
/// This function only write data to `ready_ctx`'s `WriteBatch`. It's caller's duty to write
/// it explictly to disk. If it's flushed to disk successfully, `post_ready` should be called
/// to update the memory states properly.
// Using `&Ready` here to make sure `Ready` struct is not modified in this function. This is
// a requirement to advance the ready object properly later.
pub fn handle_raft_ready<T>(
    &mut self,
    ready_ctx: &mut ReadyContext<T>,
    ready: &Ready,
) -> Result<InvokeContext> {
    let mut ctx = InvokeContext::new(self);
    let snapshot_index = if raft::is_empty_snap(&ready.snapshot) {
        0
    } else {
        fail_point!("raft_before_apply_snap");
        self.apply_snapshot(
            &mut ctx,
            &ready.snapshot,
            &ready_ctx.kv_wb,
            &ready_ctx.raft_wb,
        )?;
        fail_point!("raft_after_apply_snap");

        last_index(&ctx.raft_state)
    };

    if ready.must_sync {
        ready_ctx.sync_log = true;
    }

    if !ready.entries.is_empty() {
        self.append(&mut ctx, &ready.entries, ready_ctx)?;
    }

    // Last index is 0 means the peer is created from raft message
    // and has not applied snapshot yet, so skip persistent hard state.
    if ctx.raft_state.get_last_index() > 0 {
        if let Some(ref hs) = ready.hs {
            ctx.raft_state.set_hard_state(hs.clone());
        }
    }

    if ctx.raft_state != self.raft_state {
        ctx.save_raft_state_to(&mut ready_ctx.raft_wb)?;
        if snapshot_index > 0 {
            // in case of restart happen when we just write region state to Applying,
            // but not write raft_local_state to raft rocksdb in time.
            // we write raft state to default rocksdb, with last index set to snap index,
            // in case of recv raft log after snapshot.
            ctx.save_snapshot_raft_state_to(
                snapshot_index,
                &self.kv_engine,
                &mut ready_ctx.kv_wb,
            )?;
        }
    }

    // only when apply snapshot
    if ctx.apply_state != self.apply_state {
        ctx.save_apply_state_to(&self.kv_engine, &mut ready_ctx.kv_wb)?;
    }

    Ok(ctx)
}

pub struct InvokeContext {
    pub region_id: u64,
    pub raft_state: RaftLocalState,
    pub apply_state: RaftApplyState,
    last_term: u64,
    pub snap_region: Option<Region>,
}

impl InvokeContext {
    pub fn new(store: &PeerStorage) -> InvokeContext {
        InvokeContext {
            region_id: store.get_region_id(),
            raft_state: store.raft_state.clone(),
            apply_state: store.apply_state.clone(),
            last_term: store.last_term,
            snap_region: None,
        }
    }
}
```

主要流程
- InvokeContext::new(self) 创建 ctx
- 判断 ready 里有没有 snapshot
    - 如果没有， snapshot_index 设置为 0
    - 否则
        - 执行 PeerStorage::apply_snapshot()
        - 通过 last_index(&ctx.raft_state) 获取 InvokeContext::raft_state 中的 last_index 作为 snapshot_index
- 如果 Ready::must_sync
    - ReadyContext::sync_log 设置为 true
- 如果 Ready::entries 非空
    - 执行 PeerStorage::append()
- 如果 InvokeContext::raft_state.get_last_index() > 0
    - InvokeContext::raft_state.set_hard_state(Ready::hs.clone())
    - 将 InvokeContext::raft_state 的 hard state 设置为 Ready 的
- 如果 InvokeContext::raft_state != PeerStorage::raft_state
    - 执行 InvokeContext::save_raft_state_to(&mut ready_ctx.raft_wb)
    - 如果 snapshot_index > 0
        - 执行 InvokeContext::save_snapshot_raft_state_to()
            - 在 Store::on_raft_ready() 时
                - kv_wb 先持久化，并且会做 sync (RegionLocalState, ApplyState)
                - raft_wb 后持久化，不一定做 sync (RaftLocalState, Raft Log Entry)
            - 这里对于 applying snapshot 的场景会调用一次 InvokeContext::save_snapshot_raft_state_to() 将 RaftLocalState 也写一份到 kv_batch 中，防止 region 的状态为 Applying 的时候没来得及持久化 RaftLocalState
            - 不过这里可以看到，在极端情况下（例如掉电），是由可能造成 raft log 丢失，但是对应数据已经 apply 掉的问题，启动时会因为 apply index 大于 last index 而 panic
            - https://github.com/pingcap/tidb/issues/4731
- 如果 InvokeContext::apply_state != PeerStorage::apply_state
- 执行 InvokeContext::save_apply_state_to(&self.kv_engine, &mut ready_ctx.kv_wb)
- 返回 InvokeContext

### PeerStorage::apply_snapshot()

```rust
// Apply the peer with given snapshot.
pub fn apply_snapshot(
    &mut self,
    ctx: &mut InvokeContext,
    snap: &Snapshot,
    kv_wb: &WriteBatch,
    raft_wb: &WriteBatch,
) -> Result<()> {
    info!("{} begin to apply snapshot", self.tag);

    let mut snap_data = RaftSnapshotData::new();
    snap_data.merge_from_bytes(snap.get_data())?;

    let region_id = self.get_region_id();

    let region = snap_data.take_region();
    if region.get_id() != region_id {
        return Err(box_err!(
            "mismatch region id {} != {}",
            region_id,
            region.get_id()
        ));
    }

    if self.is_initialized() {
        // we can only delete the old data when the peer is initialized.
        self.clear_meta(kv_wb, raft_wb)?;
    }

    write_peer_state(&self.kv_engine, kv_wb, &region, PeerState::Applying)?;

    let last_index = snap.get_metadata().get_index();

    ctx.raft_state.set_last_index(last_index);
    ctx.last_term = snap.get_metadata().get_term();
    ctx.apply_state.set_applied_index(last_index);

    // The snapshot only contains log which index > applied index, so
    // here the truncate state's (index, term) is in snapshot metadata.
    ctx.apply_state.mut_truncated_state().set_index(last_index);
    ctx.apply_state
        .mut_truncated_state()
        .set_term(snap.get_metadata().get_term());

    info!(
        "{} apply snapshot for region {:?} with state {:?} ok",
        self.tag,
        region,
        ctx.apply_state
    );

    ctx.snap_region = Some(region);
    Ok(())
}
```

主要流程
- 从 snap.get_data() 中反序列化出 RaftSnapshotData
- 检查 region_id 和当前 peer 相同
- 如果 PeerStorage::is_initialized()
    - 就是判断 PeerStorage::region 的 peers 是否为空，不为空则说明已经初始化了
    - 执行 PeerStorage::clear_meta(kv_wb, raft_wb)
- 执行 PeerStorage::write_peer_state(&self.kv_engine, kv_wb, &region, PeerState::Applying)
- 用 Snapshot::get_metadata() 的数据更新 InvokeContext
    - 更新 raft_state 的 last_index
    - 更新 last_term
    - 更新 apply_state 的 applied_index (设置为 last_index)
    - 更新 apply_state 的 truncated_state 的 index 为 last_index ， term 为 Snapshot::get_metadata() 的 term
- 设置 InvokeContext::snap_region


```rust
/// Delete all meta belong to the region. Results are stored in `wb`.
pub fn clear_meta(
    kv_engine: &DB,
    raft_engine: &DB,
    kv_wb: &WriteBatch,
    raft_wb: &WriteBatch,
    region_id: u64,
    raft_state: &RaftLocalState,
) -> Result<()> {
    let t = Instant::now();
    let handle = rocksdb::get_cf_handle(kv_engine, CF_RAFT)?;
    kv_wb.delete_cf(handle, &keys::region_state_key(region_id))?;
    kv_wb.delete_cf(handle, &keys::apply_state_key(region_id))?;

    let last_index = last_index(raft_state);
    let mut first_index = last_index + 1;
    let begin_log_key = keys::raft_log_key(region_id, 0);
    let end_log_key = keys::raft_log_key(region_id, first_index);
    raft_engine.scan(
        &begin_log_key,
        &end_log_key,
        false,
        &mut |key, _| {
            first_index = keys::raft_log_index(key).unwrap();
            Ok(false)
        },
    )?;
    for id in first_index..last_index + 1 {
        raft_wb.delete(&keys::raft_log_key(region_id, id))?;
    }
    raft_wb.delete(&keys::raft_state_key(region_id))?;

    info!(
        "[region {}] clear peer 1 meta key, 1 apply key, 1 raft key and {} raft logs, takes {:?}",
        region_id,
        last_index + 1 - first_index,
        t.elapsed()
    );
    Ok(())
}
```
clear_meta() 清除 region 的元数据
- 删除 kv_engine CF_RAFT cf 下
    - keys::region_state_key(region_id)
    - keys::apply_state_key(region_id)
- 删除 raft_engine 下 region 所有的 raft log
    - keys::raft_log_index(key)
- 删除 raft_engine 下的 raft state
    - keys::raft_state_key(region_id)

```rust
pub fn write_peer_state<T: Mutable>(
    kv_engine: &DB,
    kv_wb: &T,
    region: &metapb::Region,
    state: PeerState,
) -> Result<()> {
    let region_id = region.get_id();
    let mut region_state = RegionLocalState::new();
    region_state.set_state(state);
    region_state.set_region(region.clone());
    let handle = rocksdb::get_cf_handle(kv_engine, CF_RAFT)?;
    kv_wb
        .put_msg_cf(handle, &keys::region_state_key(region_id), &region_state)?;
    Ok(())
}
```
write_peer_state() 将 PeerState 持久化
- 信息记录到 RegionLocalState
- 持久化到 kv_engine CF_RAFT 下
    - keys::region_state_key(region_id)

### PeerStorage::append()
主要用于将 entries 写入到 raft_wb 中用于后续的持久化 (raft log 的持久化)
```rust
// Append the given entries to the raft log using previous last index or self.last_index.
// Return the new last index for later update. After we commit in engine, we can set last_index
// to the return one.
pub fn append<T>(
    &mut self,
    invoke_ctx: &mut InvokeContext,
    entries: &[Entry],
    ready_ctx: &mut ReadyContext<T>,
) -> Result<u64> {
    debug!("{} append {} entries", self.tag, entries.len());
    let prev_last_index = invoke_ctx.raft_state.get_last_index();
    if entries.is_empty() {
        return Ok(prev_last_index);
    }

    let (last_index, last_term) = {
        let e = entries.last().unwrap();
        (e.get_index(), e.get_term())
    };

    for entry in entries {
        if entry.get_sync_log() {
            ready_ctx.sync_log = true;
        }
        ready_ctx.raft_wb.put_msg(
            &keys::raft_log_key(self.get_region_id(), entry.get_index()),
            entry,
        )?;
    }

    // Delete any previously appended log entries which never committed.
    for i in (last_index + 1)..(prev_last_index + 1) {
        ready_ctx
            .raft_wb
            .delete(&keys::raft_log_key(self.get_region_id(), i))?;
    }

    invoke_ctx.raft_state.set_last_index(last_index);
    invoke_ctx.last_term = last_term;

    // TODO: if the writebatch is failed to commit, the cache will be wrong.
    self.cache.append(&self.tag, entries);
    Ok(last_index)
}
```
主要流程
- InvokeContext::raft_state.get_last_index() 获取 prev_last_index
- 如果 entries 为空
    - 返回 prev_last_index
- entries 最后一个 Entry 的 index 和 term 作为 last_index, last_term
- 遍历 entries
    - 如果 Entry::get_sync_log() 则设置 ReadyContext::sync_log 为 true
    - keys::raft_log_key(self.get_region_id(), entry.get_index() 和 entry 写入 ReadyContext::raft_wb
- 删除 index 在 (last_index + 1)..(prev_last_index + 1) 之间的 raft log
    - raft_wb 记录 delete keys::raft_log_key(self.get_region_id(), i)
    - 可以看到以 append 的 entries 里的 index 为准
- 更新 InvokeContext::raft_state 的 last_index 和 last_term 为 entries 的 last_index 和 last_term
- PeerStorage::cache 插入 entries
- 返回 last_index

## PeerStorage::post_ready()
在 raft log append 之后，调用 Peer::post_raft_ready_append() 时会执行该函数
```rust
/// Update the memory state after ready changes are flushed to disk successfully.
pub fn post_ready(&mut self, ctx: InvokeContext) -> Option<ApplySnapResult> {
    self.raft_state = ctx.raft_state;
    self.apply_state = ctx.apply_state;
    self.last_term = ctx.last_term;
    // If we apply snapshot ok, we should update some infos like applied index too.
    let snap_region = match ctx.snap_region {
        Some(r) => r,
        None => return None,
    };
    // cleanup data before scheduling apply task
    if self.is_initialized() {
        if let Err(e) = self.clear_extra_data(&self.region) {
            // No need panic here, when applying snapshot, the deletion will be tried
            // again. But if the region range changes, like [a, c) -> [a, b) and [b, c),
            // [b, c) will be kept in rocksdb until a covered snapshot is applied or
            // store is restarted.
            error!(
                "{} cleanup data fail, may leave some dirty data: {:?}",
                self.tag,
                e
            );
        }
    }

    self.schedule_applying_snapshot();
    let prev_region = self.region.clone();
    self.region = snap_region;

    Some(ApplySnapResult {
        prev_region: prev_region,
        region: self.region.clone(),
    })
}
```

主要流程
- 用 InvokeContext 中的信息更新 PeerStorage 的 raft_state, apply_state, last_term
- 如果 InvokeContext::snap_region 为 none 则直接返回 None
- 否则取出 snap_region
    - 这时需要 apply snapshot
- 如果 PeerStorage::is_initialized()
    - 执行 PeerStorage::clear_extra_data(&PeerStorage::region)
        - 主要是删除不在新的 region 范围内的数据
        - 如果出错则打印日志
        - 不过这里看上去不会删除数据，因为新的 region 用的 self.region
- 执行 PeerStorage::schedule_applying_snapshot()
- PeerStorage::region 作为 prev_region
- PeerStorage::region 设置为 snap_region
- 返回 ApplySnapResult

### PeerStorage::schedule_applying_snapshot()
```rust
pub fn schedule_applying_snapshot(&mut self) {
    let status = Arc::new(AtomicUsize::new(JOB_STATUS_PENDING));
    self.set_snap_state(SnapState::Applying(status.clone()));
    let task = RegionTask::Apply {
        region_id: self.get_region_id(),
        status: status,
    };
    // TODO: gracefully remove region instead.
    self.region_sched
        .schedule(task)
        .expect("snap apply job should not fail");
}
```

主要流程
- 设置 status 为 AtomicUsize::new(JOB_STATUS_PENDING)
- PeerStorage::set_snap_state(SnapState::Applying(status.clone()))
- 创建 RegionTask::Apply
- PeerStorage::region_sched 调度执行 RegionTask
