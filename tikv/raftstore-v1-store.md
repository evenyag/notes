# TiKV v1.0 raftstore Store

## Handler
回顾下 Handler
```rust
impl<T: Transport, C: PdClient> mio::Handler for Store<T, C> {
    type Timeout = Tick;
    type Message = Msg;

    fn notify(&mut self, event_loop: &mut EventLoop<Self>, msg: Msg) {
        match msg {
            Msg::RaftMessage(data) => if let Err(e) = self.on_raft_message(data) {
                error!("{} handle raft message err: {:?}", self.tag, e);
            },
            Msg::RaftCmd {
                send_time,
                request,
                callback,
            } => {
                self.raft_metrics
                    .propose
                    .request_wait_time
                    .observe(duration_to_sec(send_time.elapsed()) as f64);
                self.propose_raft_command(request, callback)
            }
            // For now, it is only called by batch snapshot.
            Msg::BatchRaftSnapCmds {
                send_time,
                batch,
                on_finished,
            } => {
                self.raft_metrics
                    .propose
                    .request_wait_time
                    .observe(duration_to_sec(send_time.elapsed()) as f64);
                self.propose_batch_raft_snapshot_command(batch, on_finished);
            }
            Msg::Quit => {
                info!("{} receive quit message", self.tag);
                event_loop.shutdown();
            }
            Msg::SnapshotStats => self.store_heartbeat_pd(),
            Msg::ComputeHashResult {
                region_id,
                index,
                hash,
            } => {
                self.on_hash_computed(region_id, index, hash);
            }
            Msg::SplitRegion {
                region_id,
                region_epoch,
                split_key,
                callback,
            } => {
                info!(
                    "[region {}] on split region at key {:?}.",
                    region_id,
                    split_key
                );
                self.on_prepare_split_region(region_id, region_epoch, split_key, callback);
            }
            Msg::ApproximateRegionSize {
                region_id,
                region_size,
            } => self.on_approximate_region_size(region_id, region_size),
        }
    }

    fn timeout(&mut self, event_loop: &mut EventLoop<Self>, timeout: Tick) {
        let t = SlowTimer::new();
        match timeout {
            Tick::Raft => self.on_raft_base_tick(event_loop),
            Tick::RaftLogGc => self.on_raft_gc_log_tick(event_loop),
            Tick::SplitRegionCheck => self.on_split_region_check_tick(event_loop),
            Tick::CompactCheck => self.on_compact_check_tick(event_loop),
            Tick::PdHeartbeat => self.on_pd_heartbeat_tick(event_loop),
            Tick::PdStoreHeartbeat => self.on_pd_store_heartbeat_tick(event_loop),
            Tick::SnapGc => self.on_snap_mgr_gc(event_loop),
            Tick::CompactLockCf => self.on_compact_lock_cf(event_loop),
            Tick::ConsistencyCheck => self.on_consistency_check_tick(event_loop),
        }
        slow_log!(t, "{} handle timeout {:?}", self.tag, timeout);
    }

    // This method is invoked very frequently, should avoid time consuming operation.
    fn tick(&mut self, event_loop: &mut EventLoop<Self>) {
        if !event_loop.is_running() {
            self.stop();
            return;
        }

        // We handle raft ready in event loop.
        if !self.pending_raft_groups.is_empty() {
            self.on_raft_ready();
        }

        self.poll_apply();

        self.pending_snapshot_regions.clear();
    }
}
```

后面逐步跟着相关入口了解实现

## notify
### Store::on_raft_message()
RaftMessage 主要是其他机器发给该 Store 的消息，handler 中的处理如下
```rust
Msg::RaftMessage(data) => if let Err(e) = self.on_raft_message(data) {
    error!("{} handle raft message err: {:?}", self.tag, e);
}
```

raft_serverpb.proto 里定义了 RaftMessage
```
message RaftMessage {
    uint64 region_id = 1;
    metapb.Peer from_peer = 2;
    metapb.Peer to_peer = 3;
    eraftpb.Message message = 4;
    metapb.RegionEpoch region_epoch = 5;
    // true means to_peer is a tombstone peer and it should remove itself.
    bool is_tombstone = 6;
    // Region key range [start_key, end_key).
    bytes start_key = 7;
    bytes end_key = 8;
    // If it has value, to_peer should be removed if merge is never going to complete.
    metapb.Region merge_target = 9;
    ExtraMessage extra_msg = 10;
}
```

on_raft_message() 会处理收到的 RaftMessage
```rust
fn on_raft_message(&mut self, mut msg: RaftMessage) -> Result<()> {
    let region_id = msg.get_region_id();
    if !self.validate_raft_msg(&msg) {
        return Ok(());
    }

    if msg.get_is_tombstone() {
        // we receive a message tells us to remove ourself.
        self.handle_gc_peer_msg(&msg);
        return Ok(());
    }

    if self.check_msg(&msg)? {
        return Ok(());
    }

    if !self.maybe_create_peer(region_id, &msg)? {
        return Ok(());
    }

    if !self.check_snapshot(&msg)? {
        return Ok(());
    }

    let peer = self.region_peers.get_mut(&region_id).unwrap();
    peer.insert_peer_cache(msg.take_from_peer());
    peer.step(msg.take_message())?;

    // Add into pending raft groups for later handling ready.
    peer.mark_to_be_checked(&mut self.pending_raft_groups);

    Ok(())
}
```

主要流程包括
- Store::validate_raft_msg() 校验 RaftMessage 是否合法，包括
    - 目标 store id 是否跟当前 store 相同
    - msg 是否带 region_epoch
- 如果是 tombstone
    - 说明目标 peer 需要被删除掉
    - 调用 Store::handle_gc_peer_msg() 删除对应的 region/peer
- Store::check_msg() 会做一些更偏业务上的检查
- Store::maybe_create_peer() 对于部分 msg ，如果 peer 不存在，则还需要创建相应的 peer
- Store::check_snapshot() 如果 msg 是一个 snapshot
    - 执行相应的检查
    - 如果检查通过，会将相应的信息 push 到 Store::pending_snapshot_regions
- 根据 region_id 从 Store::region_peers 获取 peer
- Peer::insert_peer_cache() 将 msg 的 from_peer 插入 Peer 的 cache 里
- Peer::step() 对 raft group 执行 step
    - 这里会调用 raft-rs 的 RawNode::step()
- Peer::mark_to_be_checked() 设置 Peer::marked_to_be_checked 为 true 并将该 region 加入 Store::pending_raft_groups

### Store::propose_raft_command()
RaftCmd 主要是交由该 Store 发起的 proposal ，处理如下
```rust

Msg::RaftCmd {
    send_time,
    request,
    callback,
} => {
    self.raft_metrics
        .propose
        .request_wait_time
        .observe(duration_to_sec(send_time.elapsed()) as f64);
    self.propose_raft_command(request, callback)
}
```

RaftCmd 结构如下
```rust
pub type Callback = Box<FnBox(RaftCmdResponse) + Send>;

RaftCmd {
    send_time: Instant,
    request: RaftCmdRequest,
    callback: Callback,
},
```

raft_cmdpb.proto 中定义了 RaftCmdRequest 和 RaftCmdResponse
```rust
message RaftCmdRequest {
    RaftRequestHeader header = 1;
    // We can't enclose normal requests and administrator request
    // at same time. 
    repeated Request requests = 2;
    AdminRequest admin_request = 3;
    StatusRequest status_request = 4;
}

message RaftCmdResponse {
    RaftResponseHeader header = 1;
    repeated Response responses = 2;
    AdminResponse admin_response = 3;
    StatusResponse status_response = 4;
}
```

其中 Request 和 Response 是各种 kv 操作的请求和响应
```rust
message Request {
    CmdType cmd_type = 1;
    GetRequest get = 2;
    PutRequest put = 4;
    DeleteRequest delete = 5;
    SnapRequest snap = 6;
    PrewriteRequest prewrite = 7;
    DeleteRangeRequest delete_range = 8;
    IngestSSTRequest ingest_sst = 9;
    ReadIndexRequest read_index = 10;
}

message Response {
    CmdType cmd_type = 1;
    GetResponse get = 2;
    PutResponse put = 4;
    DeleteResponse delete = 5;
    SnapResponse snap = 6;
    PrewriteResponse prewrite = 7;
    DeleteRangeResponse delte_range = 8;
    IngestSSTResponse ingest_sst = 9;
    ReadIndexResponse read_index = 10;
}
```

propose_raft_command() 实现如下
```rust
fn propose_raft_command(&mut self, msg: RaftCmdRequest, cb: Callback) {
    match self.pre_propose_raft_command(&msg) {
        Ok(Some(resp)) => {
            cb.call_box((resp,));
            return;
        }
        Err(e) => {
            cb.call_box((new_error(e),));
            return;
        }
        _ => (),
    }

    // Note:
    // The peer that is being checked is a leader. It might step down to be a follower later. It
    // doesn't matter whether the peer is a leader or not. If it's not a leader, the proposing
    // command log entry can't be committed.

    let mut resp = RaftCmdResponse::new();
    let region_id = msg.get_header().get_region_id();
    let peer = self.region_peers.get_mut(&region_id).unwrap();
    let term = peer.term();
    bind_term(&mut resp, term);
    if peer.propose(cb, msg, resp, &mut self.raft_metrics.propose) {
        peer.mark_to_be_checked(&mut self.pending_raft_groups);
    }

    // TODO: add timeout, if the command is not applied after timeout,
    // we will call the callback with timeout error.
}
```

主要流程包括
- Store::pre_propose_raft_command()
    - Store::validate_store_id() 校验 store id 是否相同
    - 如果 RaftCmdRequest::has_status_request()
        - 直接执行 Store::execute_status_command()
    - 否则，执行 Store::validate_region()
        - 从 Store::region_peers 获取 peer ，不存在则报错
        - Peer::is_leader() 判断 peer 是否 leader ，不是则报错
        - Peer::peer_id() 是否和目标 peer 相同，不同则报错
        - 如果 header.get_term() > 0 && peer.term() > header.get_term() + 1
            - 说明请求的 term 落后太多，报错
        - res = peer::check_epoch(peer.region(), msg)
            - 校验 epoch 的 version ，如果 msg version 比 peer 的小，返回 Error::StaleEpoch
        - 如果 res 是 Err(Error::StaleEpoch(msg, mut new_regions))
            - 通过 Store::find_sibling_region() 找到相邻的 region 并记录到 Error::StaleEpoch 的 new_regions 中
            - 返回 Error::StaleEpoch 报错
    - 返回 None
- 根据 region id 从 Store::region_peers 获取 peer
- cmd_resp::bind_term() 设置 RaftCmdResponse 的 term
- Peer::propose() propose 该请求
    - 这里会调用 raft-rs 的 RawNode::propose()
- 同样执行 Peer::mark_to_be_checked()

### Store::propose_batch_raft_snapshot_command()
主要用于执行 batch snapshot
```rust
fn propose_batch_raft_snapshot_command(
    &mut self,
    batch: Vec<RaftCmdRequest>,
    on_finished: BatchCallback,
) {
    let size = batch.len();
    BATCH_SNAPSHOT_COMMANDS.observe(size as f64);
    let mut ret = Vec::with_capacity(size);
    for msg in batch {
        match self.pre_propose_raft_command(&msg) {
            Ok(Some(resp)) => {
                ret.push(Some(resp));
                continue;
            }
            Err(e) => {
                ret.push(Some(new_error(e)));
                continue;
            }
            _ => (),
        }

        let region_id = msg.get_header().get_region_id();
        let peer = self.region_peers.get_mut(&region_id).unwrap();
        ret.push(peer.propose_snapshot(msg, &mut self.raft_metrics.propose));
    }
    on_finished.call_box((ret,));
}
```

主要流程
- 遍历 batch 里的每个 msg
    - 同样调用 Store::pre_propose_raft_command()
    - 获取 region id
    - 根据 region id 从 Store::region_peers 中找到对应的 peer
    - Peer::propose_snapshot()
    - 结果收集到 ret 中
- 结果回调 on_finished

### Store::store_heartbeat_pd()
主要用于收集 Store 的信息并发送心跳上报给 pd
```rust
fn store_heartbeat_pd(&mut self) {
    let mut stats = StoreStats::new();

    let used_size = self.snap_mgr.get_total_snap_size();
    stats.set_used_size(used_size);
    stats.set_store_id(self.store_id());
    stats.set_region_count(self.region_peers.len() as u32);

    let snap_stats = self.snap_mgr.stats();
    stats.set_sending_snap_count(snap_stats.sending_count as u32);
    stats.set_receiving_snap_count(snap_stats.receiving_count as u32);
    // ... snipped...

    let mut apply_snapshot_count = 0;
    for peer in self.region_peers.values_mut() {
        if peer.mut_store().check_applying_snap() {
            apply_snapshot_count += 1;
        }
    }

    stats.set_applying_snap_count(apply_snapshot_count as u32);
    // ...snipped...

    let store_info = StoreInfo {
        engine: self.kv_engine.clone(),
        capacity: self.cfg.capacity.0,
    };

    let task = PdTask::StoreHeartbeat {
        stats: stats,
        store_info: store_info,
    };
    if let Err(e) = self.pd_worker.schedule(task) {
        error!("{} failed to notify pd: {}", self.tag, e);
    }
}
```

### Store::on_hash_computed()
主要用于 Consistency Check ，这里不展开来看

### Store::on_approximate_region_size()
比较简单，就是更新下 peer 的 approximate_size
```rust
fn on_approximate_region_size(&mut self, region_id: u64, region_size: u64) {
    let peer = match self.region_peers.get_mut(&region_id) {
        Some(peer) => peer,
        None => {
            warn!(
                "[region {}] receive stale approximate size {}",
                region_id,
                region_size,
            );
            return;
        }
    };
    peer.approximate_size = Some(region_size);
}
```

### Store::on_prepare_split_region()
收到 split region 请求时调用
```rust
fn on_prepare_split_region(
    &mut self,
    region_id: u64,
    region_epoch: metapb::RegionEpoch,
    split_key: Vec<u8>, // `split_key` is a encoded key.
    cb: Option<Callback>,
) {
    if let Err(e) = self.validate_split_region(region_id, &region_epoch, &split_key) {
        cb.map(|cb| cb(new_error(e)));
        return;
    }
    let peer = &self.region_peers[&region_id];
    let region = peer.region();
    let task = PdTask::AskSplit {
        region: region.clone(),
        split_key: split_key,
        peer: peer.peer.clone(),
        right_derive: self.cfg.right_derive_when_split,
        callback: cb,
    };
    if let Err(Stopped(t)) = self.pd_worker.schedule(task) {
        error!("{} failed to notify pd to split: Stopped", peer.tag);
        match t {
            PdTask::AskSplit { callback, .. } => {
                callback.map(|cb| cb(new_error(box_err!("failed to split: Stopped"))));
            }
            _ => unreachable!(),
        }
    }
}
```

主要流程
- Store::validate_split_region()
    - 校验 split_key 非空
    - 从 Store::region_peers 查询出 region_id 对应的 peer
        - 不存在则报错
        - 存在，则通过 Peer::is_leader() 校验是否是 leader
    - 校验 region 的 epoch 和请求的 epoch 相等
- 根据 region id 从 Store::region_peers 里获取 peer
- 创建 PdTask::AskSplit 并发送给 pd

## tick
这里先看下 tick 部分调用的函数

```rust
// This method is invoked very frequently, should avoid time consuming operation.
fn tick(&mut self, event_loop: &mut EventLoop<Self>) {
    if !event_loop.is_running() {
        self.stop();
        return;
    }

    // We handle raft ready in event loop.
    if !self.pending_raft_groups.is_empty() {
        self.on_raft_ready();
    }

    self.poll_apply();

    self.pending_snapshot_regions.clear();
}
```

主要流程
- 如果 event_loop 不再运行
    - Store::stop()
- 如果 Store::pending_raft_groups 不为空
    - Store::on_raft_ready()
- Store::poll_apply()
- 清空 Store::pending_snapshot_regions

### Store::on_raft_ready()

```rust
pub struct ReadyContext<'a, T: 'a> {
    pub kv_wb: WriteBatch,
    pub raft_wb: WriteBatch,
    pub sync_log: bool,
    pub metrics: &'a mut RaftMetrics,
    pub trans: &'a T,
    pub ready_res: Vec<(Ready, InvokeContext)>,
}

fn on_raft_ready(&mut self) {
    let t = SlowTimer::new();
    let pending_count = self.pending_raft_groups.len();
    let previous_ready_metrics = self.raft_metrics.ready.clone();

    self.raft_metrics.ready.pending_region += pending_count as u64;

    let mut region_proposals = Vec::with_capacity(pending_count);
    let (kv_wb, raft_wb, append_res, sync_log) = {
        let mut ctx = ReadyContext::new(&mut self.raft_metrics, &self.trans, pending_count);
        for region_id in self.pending_raft_groups.drain() {
            if let Some(peer) = self.region_peers.get_mut(&region_id) {
                if let Some(region_proposal) = peer.take_apply_proposals() {
                    region_proposals.push(region_proposal);
                }
                peer.handle_raft_ready_append(&mut ctx, &self.pd_worker);
            }
        }
        (ctx.kv_wb, ctx.raft_wb, ctx.ready_res, ctx.sync_log)
    };

    if !region_proposals.is_empty() {
        self.apply_worker
            .schedule(ApplyTask::Proposals(region_proposals))
            .unwrap();

        // In most cases, if the leader proposes a message, it will also
        // broadcast the message to other followers, so we should flush the
        // messages ASAP.
        self.trans.flush();
    }

    self.raft_metrics.ready.has_ready_region += append_res.len() as u64;

    // apply_snapshot, peer_destroy will clear_meta, so we need write region state first.
    // otherwise, if program restart between two write, raft log will be removed,
    // but region state may not changed in disk.
    fail_point!("raft_before_save");
    if !kv_wb.is_empty() {
        // RegionLocalState, ApplyState
        let mut write_opts = WriteOptions::new();
        write_opts.set_sync(true);
        self.kv_engine
            .write_opt(kv_wb, &write_opts)
            .unwrap_or_else(|e| {
                panic!("{} failed to save append state result: {:?}", self.tag, e);
            });
    }
    fail_point!("raft_between_save");

    if !raft_wb.is_empty() {
        // RaftLocalState, Raft Log Entry
        let mut write_opts = WriteOptions::new();
        write_opts.set_sync(self.cfg.sync_log || sync_log);
        self.raft_engine
            .write_opt(raft_wb, &write_opts)
            .unwrap_or_else(|e| {
                panic!("{} failed to save raft append result: {:?}", self.tag, e);
            });
    }
    fail_point!("raft_after_save");

    let mut ready_results = Vec::with_capacity(append_res.len());
    for (mut ready, invoke_ctx) in append_res {
        let region_id = invoke_ctx.region_id;
        let res =
            self.region_peers
                .get_mut(&region_id)
                .unwrap()
                .post_raft_ready_append(
                    &mut self.raft_metrics,
                    &self.trans,
                    &mut ready,
                    invoke_ctx,
                );
        ready_results.push((region_id, ready, res));
    }

    self.raft_metrics
        .append_log
        .observe(duration_to_sec(t.elapsed()) as f64);

    slow_log!(
        t,
        "{} handle {} pending peers include {} ready, {} entries, {} messages and {} \
            snapshots",
        self.tag,
        pending_count,
        ready_results.capacity(),
        self.raft_metrics.ready.append - previous_ready_metrics.append,
        self.raft_metrics.ready.message - previous_ready_metrics.message,
        self.raft_metrics.ready.snapshot - previous_ready_metrics.snapshot
    );

    let mut apply_tasks = Vec::with_capacity(ready_results.len());
    for (region_id, ready, res) in ready_results {
        self.region_peers
            .get_mut(&region_id)
            .unwrap()
            .handle_raft_ready_apply(ready, &mut apply_tasks);
        if let Some(apply_result) = res {
            self.on_ready_apply_snapshot(apply_result);
        }
    }
    self.apply_worker
        .schedule(ApplyTask::applies(apply_tasks))
        .unwrap();

    let dur = t.elapsed();
    if !self.is_busy {
        let election_timeout = Duration::from_millis(
            self.cfg.raft_base_tick_interval.as_millis() *
                self.cfg.raft_election_timeout_ticks as u64,
        );
        if dur >= election_timeout {
            self.is_busy = true;
        }
    }

    self.raft_metrics
        .process_ready
        .observe(duration_to_sec(dur) as f64);

    self.trans.flush();

    slow_log!(t, "{} on {} regions raft ready", self.tag, pending_count);
}
```

主要流程
- 取出（调用了 drain()） Store::pending_raft_groups 并遍历
    - 根据 region_id 从 Store::region_peers 获取 peer
        - 创建 ReadyContext ctx ，里面也会记录需要写入到 raft_engine 和 kv_engine 的 write batch
        - Peer::take_apply_proposals() 取出 RegionProposal ，这里的 proposal 是 Peer::post_propose() 时插入的，而 Peer::propose() 会调用 Peer::post_propose()
        - 插入 region_proposals
        - 执行 Peer::handle_raft_ready_append(&mut ctx, &self.pd_worker) ，这个函数会
            - 调用 raft-rs 的 RawNode::ready_since(Peer::last_applying_idx) ，类似 RawNode::ready()
            - 对 ready 进行处理
                - 处理 ready.snapshot
                - 处理 ready.messages
                - ready 的各种 state 和 ready.entries 写入到对应的 raft_wb 和 kv_wb 以准备在后面持久化
    - 取出 ReadyContext 的 kv_wb ， raft_wb ， append_res ， sync_log
- 如果刚刚的 region_proposals 非空
    - Store::apply_worker 调用 schedule 执行 ApplyTask::Proposals(region_proposals)
    - Store::trans 调用 flush() 尽快将消息发送出去
- 如果 kv_wb 非空
    - 用于持久化 RegionLocalState, ApplyState
    - WriteOptions::set_sync(true)
    - Store::kv_engine 写入 kv_wb ，失败直接 panic
- 如果 raft_wb 非空
    - 用于持久化 RaftLocalState, Raft Log Entry
    - WriteOptions::set_sync(self.cfg.sync_log || sync_log)
    - Store::raft_engine 写入 raft_wb ，失败直接 panic
- 遍历 append_res
    - 根据 peer_id 从 region_peers 获取 peer
    - Peer::post_raft_ready_append()
    - 执行结果插入 ready_results
- 遍历 ready_results
    - Peer::handle_raft_ready_apply()
    - 如果有 apply_result
        - 这个实际上叫 apply_snapshot_result 更清晰些
        - Store::on_ready_apply_snapshot()
- Store::apply_worker 执行 ApplyTask::applies(apply_tasks)
- 执行 Store::trans flush()


### Store::poll_apply()
这个函数主要接收 apply 的结果并进行处理
```rust
fn poll_apply(&mut self) {
    loop {
        match self.apply_res_receiver.as_ref().unwrap().try_recv() {
            Ok(ApplyTaskRes::Applys(multi_res)) => for res in multi_res {
                if let Some(p) = self.region_peers.get_mut(&res.region_id) {
                    debug!("{} async apply finish: {:?}", p.tag, res);
                    p.post_apply(&res, &mut self.pending_raft_groups, &mut self.store_stat);
                }
                self.store_stat.lock_cf_bytes_written += res.metrics.lock_cf_written_bytes;
                self.on_ready_result(res.region_id, res.exec_res);
            },
            Ok(ApplyTaskRes::Destroy(p)) => {
                let store_id = self.store_id();
                self.destroy_peer(p.region_id(), util::new_peer(store_id, p.id()));
            }
            Err(TryRecvError::Empty) => break,
            Err(e) => panic!("unexpected error {:?}", e),
        }
    }
}
```

主要流程
- 处理 Store::apply_res_receiver 中所有的 ApplyTaskRes
    - 如果是 ApplyTaskRes::Applys(multi_res)
        - 遍历 multi_res
            - 从 Store::region_peers 中获取对应的 peer
            - 执行 Peer::post_apply(&res, &mut self.pending_raft_groups, &mut self.store_stat)
            - Store::on_ready_result(res.region_id, res.exec_res)
                - 遍历 exec_res 并 match
                    - 如果是 ExecResult::ChangePeer(cp) ，执行 Store::on_ready_change_peer()
                    - 如果是 ExecResult::CompactLog ，执行 Store::on_ready_compact_log()
                    - 如果是 ExecResult::SplitRegion ，执行 Store::on_ready_split_region()
                    - 如果是 ExecResult::ComputeHash ，执行 Store::on_ready_compute_hash()
                    - 如果是 ExecResult::VerifyHash ，执行 Store::on_ready_verify_hash()
    - 如果是 ApplyTaskRes::Destroy(p)
        - 说明可以删除 peer 了，执行 Store::destroy_peer()

## timeout

### Store::on_raft_base_tick()
这个函数一个重要功能是执行 RawNode::tick()
```rust
fn on_raft_base_tick(&mut self, event_loop: &mut EventLoop<Self>) {
    let timer = self.raft_metrics.process_tick.start_coarse_timer();
    for peer in &mut self.region_peers.values_mut() {
        if peer.pending_remove {
            continue;
        }
        // When having pending snapshot, if election timeout is met, it can't pass
        // the pending conf change check because first index has been updated to
        // a value that is larger than last index.
        if peer.is_applying_snapshot() || peer.has_pending_snapshot() {
            // need to check if snapshot is applied.
            peer.mark_to_be_checked(&mut self.pending_raft_groups);
            continue;
        }

        if peer.raft_group.tick() {
            peer.mark_to_be_checked(&mut self.pending_raft_groups);
        }

        // If this peer detects the leader is missing for a long long time,
        // it should consider itself as a stale peer which is removed from
        // the original cluster.
        // This most likely happens in the following scenario:
        // At first, there are three peer A, B, C in the cluster, and A is leader.
        // Peer B gets down. And then A adds D, E, F into the cluster.
        // Peer D becomes leader of the new cluster, and then removes peer A, B, C.
        // After all these peer in and out, now the cluster has peer D, E, F.
        // If peer B goes up at this moment, it still thinks it is one of the cluster
        // and has peers A, C. However, it could not reach A, C since they are removed
        // from the cluster or probably destroyed.
        // Meantime, D, E, F would not reach B, since it's not in the cluster anymore.
        // In this case, peer B would notice that the leader is missing for a long time,
        // and it would check with pd to confirm whether it's still a member of the cluster.
        // If not, it destroys itself as a stale peer which is removed out already.
        let max_missing_duration = self.cfg.max_leader_missing_duration.0;
        if let StaleState::ToValidate = peer.check_stale_state(max_missing_duration) {
            // for peer B in case 1 above
            info!(
                "{} detects leader missing for a long time. To check with pd \
                    whether it's still valid",
                peer.tag
            );
            let task = PdTask::ValidatePeer {
                peer: peer.peer.clone(),
                region: peer.region().clone(),
            };
            if let Err(e) = self.pd_worker.schedule(task) {
                error!("{} failed to notify pd: {}", peer.tag, e)
            }
        }
    }

    self.poll_significant_msg();

    timer.observe_duration();

    self.raft_metrics.flush();
    self.entry_cache_metries.borrow_mut().flush();

    self.register_raft_base_tick(event_loop);
}
```

主要流程
- 遍历 Store::region_peers 里所有的 Peer
    - 如果 Peer::pending_remove 则跳过该 peer
    - 如果 peer.is_applying_snapshot() || peer.has_pending_snapshot()
        - Peer::mark_to_be_checked(&mut self.pending_raft_groups)
            - 设置 Peer::marked_to_be_checked 为 true
            - 将 Peer::region_id 插入 pending_raft_groups
        - continue
    - 执行 Peer::raft_group.tick() ，即调用 raft-rs 的 RawNode::tick()
        - 如果返回 true ，执行 Peer::mark_to_be_checked(&mut self.pending_raft_groups)
        - 可见 raft-rs 定期的 tick 就是在这里被触发的
    - 如果 Peer::check_stale_state(max_missing_duration) 返回 StaleState::ToValidate
        - 调度 Store::pd_worker 执行 PdTask::ValidatePeer
- Store::poll_significant_msg()
    - 遍历 channel Store::significant_msg_receiver 中的所有消息
        - 如果是 SignificantMsg::SnapshotStatus
            - 执行 Store::report_snapshot_status(region_id, to_peer_id, status)
                - 根据 region_id 找到 peer
                - 根据 to_peer_id 从 Peer::get_peer_from_cache(to_peer_id) 找到 to_peer
                - 调用 to_peer.raft_group 即 raft-rs RawNode::report_snapshot()
        - 如果是 SignificantMsg::Unreachable
            - 根据 region_id 找到 peer
            - 调用 peer.raft_group 即 raft-rs RawNode::report_unreachable()
- Store::register_raft_base_tick() 注册下一次 timeout 调用

### Store::on_raft_gc_log_tick()
主要用于定期触发 raft log 的 gc
```rust
fn on_raft_gc_log_tick(&mut self, event_loop: &mut EventLoop<Self>) {
    let mut total_gc_logs = 0;

    for (&region_id, peer) in &mut self.region_peers {
        if !peer.is_leader() {
            continue;
        }

        // Leader will replicate the compact log command to followers,
        // If we use current replicated_index (like 10) as the compact index,
        // when we replicate this log, the newest replicated_index will be 11,
        // but we only compact the log to 10, not 11, at that time,
        // the first index is 10, and replicated_index is 11, with an extra log,
        // and we will do compact again with compact index 11, in cycles...
        // So we introduce a threshold, if replicated index - first index > threshold,
        // we will try to compact log.
        // raft log entries[..............................................]
        //                  ^                                       ^
        //                  |-----------------threshold------------ |
        //              first_index                         replicated_index
        let replicated_idx = peer.raft_group
            .status()
            .progress
            .values()
            .map(|p| p.matched)
            .min()
            .unwrap();
        // When an election happened or a new peer is added, replicated_idx can be 0.
        if replicated_idx > 0 {
            let last_idx = peer.raft_group.raft.raft_log.last_index();
            assert!(
                last_idx >= replicated_idx,
                "expect last index {} >= replicated index {}",
                last_idx,
                replicated_idx
            );
            REGION_MAX_LOG_LAG.observe((last_idx - replicated_idx) as f64);
        }
        let applied_idx = peer.get_store().applied_index();
        let first_idx = peer.get_store().first_index();
        let mut compact_idx;
        if applied_idx > first_idx &&
            applied_idx - first_idx >= self.cfg.raft_log_gc_count_limit
        {
            compact_idx = applied_idx;
        } else if peer.raft_log_size_hint >= self.cfg.raft_log_gc_size_limit.0 {
            compact_idx = applied_idx;
        } else if replicated_idx < first_idx ||
            replicated_idx - first_idx <= self.cfg.raft_log_gc_threshold
        {
            continue;
        } else {
            compact_idx = replicated_idx;
        }

        // Have no idea why subtract 1 here, but original code did this by magic.
        assert!(compact_idx > 0);
        compact_idx -= 1;
        if compact_idx < first_idx {
            // In case compact_idx == first_idx before subtraction.
            continue;
        }

        total_gc_logs += compact_idx - first_idx;

        let term = peer.raft_group.raft.raft_log.term(compact_idx).unwrap();

        // Create a compact log request and notify directly.
        let request = new_compact_log_request(region_id, peer.peer.clone(), compact_idx, term);

        if let Err(e) = self.sendch
            .try_send(Msg::new_raft_cmd(request, Box::new(|_| {})))
        {
            error!("{} send compact log {} err {:?}", peer.tag, compact_idx, e);
        }
    }

    PEER_GC_RAFT_LOG_COUNTER
        .inc_by(total_gc_logs as f64)
        .unwrap();
    self.register_raft_gc_log_tick(event_loop);
}
```

主要流程
- 遍历 Store::region_peers 的 peer
    - 如果 !Peer::is_leader() 则 continue ，即只对 leader 作用
    - 计算 compact_idx
    - 通过 new_compact_log_request() 创建 request
    - 通过 Msg::new_raft_cmd 创建一个新的 raft command 并通过 Store::sendch 发送给自己处理
- 调用 Store::register_raft_gc_log_tick() 再次注册 timeout 任务

### TODO
其他任务暂时先不做分析
