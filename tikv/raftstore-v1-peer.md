# TiKV v1.0 raftstore Peer

## Peer
Peer 大部分是对 RawNode 和 PeerStorage 一些操作的封装

```rust
pub struct Peer {
    kv_engine: Arc<DB>,
    raft_engine: Arc<DB>,
    cfg: Rc<Config>,
    peer_cache: RefCell<FlatMap<u64, metapb::Peer>>,
    pub peer: metapb::Peer,
    region_id: u64,
    pub raft_group: RawNode<PeerStorage>,
    proposals: ProposalQueue,
    apply_proposals: Vec<Proposal>,
    pending_reads: ReadIndexQueue,
    // Record the last instant of each peer's heartbeat response.
    pub peer_heartbeats: FlatMap<u64, Instant>,
    coprocessor_host: Arc<CoprocessorHost>,
    /// an inaccurate difference in region size since last reset.
    pub size_diff_hint: u64,
    /// delete keys' count since last reset.
    pub delete_keys_hint: u64,
    /// approximate region size.
    pub approximate_size: Option<u64>,

    pub consistency_state: ConsistencyState,

    pub tag: String,

    // Index of last scheduled committed raft log.
    pub last_applying_idx: u64,
    pub last_compacted_idx: u64,
    // Approximate size of logs that is applied but not compacted yet.
    pub raft_log_size_hint: u64,
    // When entry exceed max size, reject to propose the entry.
    pub raft_entry_max_size: u64,

    apply_scheduler: Scheduler<ApplyTask>,

    pub pending_remove: bool,

    marked_to_be_checked: bool,

    leader_missing_time: Option<Instant>,

    // `leader_lease_expired_time` contains either timestamps of
    //   1. Either::Left<Timespec>
    //      A safe leader lease expired time, which marks the leader holds the lease for now.
    //      The lease is safe until the clock time goes over this timestamp.
    //      It would increase when raft log entries are applied in current term.
    //   2. Either::Right<Timespec>
    //      An unsafe leader lease expired time, which marks the leader may still hold or lose
    //      its lease until the clock time goes over this timestamp.
    //      It would be set after the message MsgTimeoutNow is sent by current peer.
    //      The message MsgTimeoutNow starts a leader transfer procedure. During this procedure,
    //      current peer as an old leader may still hold its lease or lose it.
    //      It's possible there is a new leader elected and current peer as an old leader
    //      doesn't step down due to network partition from the new leader. In that case,
    //      current peer lose its leader lease.
    //      Within this unsafe leader lease expire time, read requests could not be performed
    //      locally.
    leader_lease_expired_time: Option<Either<Timespec, Timespec>>,

    pub peer_stat: PeerStat,
}
```

## Peer::new()

```rust
fn new<T, C>(store: &mut Store<T, C>, region: &metapb::Region, peer_id: u64) -> Result<Peer> {
    if peer_id == raft::INVALID_ID {
        return Err(box_err!("invalid peer id"));
    }

    let cfg = store.config();

    let store_id = store.store_id();
    let sched = store.snap_scheduler();
    let peer_cache = FlatMap::default();
    let tag = format!("[region {}] {}", region.get_id(), peer_id);

    let ps = PeerStorage::new(
        store.kv_engine(),
        store.raft_engine(),
        region,
        sched,
        tag.clone(),
        store.entry_cache_metries.clone(),
    )?;

    let applied_index = ps.applied_index();

    let raft_cfg = raft::Config {
        id: peer_id,
        peers: vec![],
        election_tick: cfg.raft_election_timeout_ticks,
        heartbeat_tick: cfg.raft_heartbeat_ticks,
        max_size_per_msg: cfg.raft_max_size_per_msg.0,
        max_inflight_msgs: cfg.raft_max_inflight_msgs,
        applied: applied_index,
        check_quorum: true,
        tag: tag.clone(),
        skip_bcast_commit: true,
        ..Default::default()
    };

    let raft_group = RawNode::new(&raft_cfg, ps, &[])?;

    let mut peer = Peer {
        kv_engine: store.kv_engine(),
        raft_engine: store.raft_engine(),
        peer: util::new_peer(store_id, peer_id),
        region_id: region.get_id(),
        raft_group: raft_group,
        // ...snipped...
        leader_missing_time: Some(Instant::now()),
        tag: tag,
        last_applying_idx: applied_index,
        last_compacted_idx: 0,
        // ...snipped...
    };

    // If this region has only one peer and I am the one, campaign directly.
    if region.get_peers().len() == 1 && region.get_peers()[0].get_store_id() == store_id {
        peer.raft_group.campaign()?;
    }

    Ok(peer)
}
```

Peer 的创建主要通过 Peer::create() 或者 Peer::replicate() 触发，最终都会走到 Peer::new() ，可以看到在 Peer::new() 里会
- PeerStorage::new() 创建 PeerStorage
- 调用 PeerStorage::applied_index() 获取 applied index
- 创建 RawNode
- 创建 Peer

## Peer::step()
这个函数主要是对 RawNode::step() 的封装
```rust
pub fn step(&mut self, m: eraftpb::Message) -> Result<()> {
    if self.is_leader() && m.get_from() != INVALID_ID {
        self.peer_heartbeats.insert(m.get_from(), Instant::now());
    }
    self.raft_group.step(m)?;
    Ok(())
}
```

流程
- 如果 Peer::is_leader() 且 m 的发送方是合法的
    - 将发送方和当前时间加入到 Peer::peer_heartbeats 里
    - 可见 Peer::peer_heartbeats 记录了和该 peer 进行心跳的 peer 和上次心跳的时间
- 调用 RawNode::step()
    - 这里驱动 raft-rs 去处理 m

## Peer::propose()

```rust
/// Propose a request.
///
/// Return true means the request has been proposed successfully.
pub fn propose(
    &mut self,
    cb: Callback,
    req: RaftCmdRequest,
    mut err_resp: RaftCmdResponse,
    metrics: &mut RaftProposeMetrics,
) -> bool {
    if self.pending_remove {
        return false;
    }

    metrics.all += 1;

    let mut is_conf_change = false;

    let res = match self.get_handle_policy(&req) {
        Ok(RequestPolicy::ReadLocal) => {
            self.read_local(req, cb, metrics);
            return false;
        }
        Ok(RequestPolicy::ReadIndex) => return self.read_index(req, cb, metrics),
        Ok(RequestPolicy::ProposeNormal) => self.propose_normal(req, metrics),
        Ok(RequestPolicy::ProposeTransferLeader) => {
            return self.propose_transfer_leader(req, cb, metrics)
        }
        Ok(RequestPolicy::ProposeConfChange) => {
            is_conf_change = true;
            self.propose_conf_change(req, metrics)
        }
        Err(e) => Err(e),
    };

    match res {
        Err(e) => {
            cmd_resp::bind_error(&mut err_resp, e);
            cb(err_resp);
            false
        }
        Ok(idx) => {
            let meta = ProposalMeta {
                index: idx,
                term: self.term(),
                renew_lease_time: None,
            };
            self.post_propose(meta, is_conf_change, cb);
            true
        }
    }
}
```

主要流程
- 如果 Peer::pending_remove
    - 返回 false
- match Peer::get_handle_policy(&req)
    - RequestPolicy::ReadLocal
        - 调用 Peer::read_local()
        - 返回 false
    - RequestPolicy::ReadIndex
        - 返回 Peer::read_index()
    - RequestPolicy::ProposeNormal 则调用 Peer::propose_normal()
    - RequestPolicy::ProposeTransferLeader
        - 返回 Peer::propose_transfer_leader()
    - RequestPolicy::ProposeConfChange
        - is_conf_change 设置为 true
        - 调用 Peer::propose_conf_change()
- 对于上面的处理结果 res 进行 match
    - Err 则直接调用 cb 并返回 false
    - Ok 则调用 Peer::post_propose() 并返回 true

### Peer::propose_normal()
大多数操作走的都是 propose_normal() 的流程
```rust
fn propose_normal(
    &mut self,
    mut req: RaftCmdRequest,
    metrics: &mut RaftProposeMetrics,
) -> Result<u64> {
    metrics.normal += 1;

    // TODO: validate request for unexpected changes.
    self.coprocessor_host.pre_propose(self.region(), &mut req)?;
    let data = req.write_to_bytes()?;

    // TODO: use local histogram metrics
    PEER_PROPOSE_LOG_SIZE_HISTOGRAM.observe(data.len() as f64);

    if data.len() as u64 > self.raft_entry_max_size {
        error!("entry is too large, entry size {}", data.len());
        return Err(Error::RaftEntryTooLarge(self.region_id, data.len() as u64));
    }

    let sync_log = get_sync_log_from_request(&req);
    let propose_index = self.next_proposal_index();
    self.raft_group.propose(data, sync_log)?;
    if self.next_proposal_index() == propose_index {
        // The message is dropped silently, this usually due to leader absence
        // or transferring leader. Both cases can be considered as NotLeader error.
        return Err(Error::NotLeader(self.region_id, None));
    }

    Ok(propose_index)
}
```

主要流程
- 序列化 RaftCmdRequest 为 data
- 如果 data 的大小超过了 Peer::raft_entry_max_size 则报错
- 调用 get_sync_log_from_request(&req) 来获取是否需要 sync 日志的 flag
- 调用 Peer::next_proposal_index() 获取 propose_index
    - Peer::raft_group.raft.raft_log.last_index() + 1
    - 实际上就是 raft log 的 last index + 1
- 执行 Peer::raft_group 也就是 RawNode::propose()
- 如果 Peer::next_proposal_index() == propose_index
    - 说明 propose 没有被处理，主要是因为 leader 变了，返回 NotLeader 报错
- 返回 propose_index

### Peer::post_propose()
执行完 Peer::propose_normal() 之后还会执行 post_propose()
```rust
let meta = ProposalMeta {
    index: idx,
    term: self.term(),
    renew_lease_time: None,
};
self.post_propose(meta, is_conf_change, cb);
```
这里传给 post_propose() 的 meta 的信息包含
- 前面 propose 阶段产生的 propose_index
- 通过 Peer::term() 获取的当前的 term

```rust
fn post_propose(&mut self, mut meta: ProposalMeta, is_conf_change: bool, cb: Callback) {
    // Try to renew leader lease on every consistent read/write request.
    meta.renew_lease_time = Some(monotonic_raw_now());

    let p = Proposal::new(is_conf_change, meta.index, meta.term, cb);
    self.apply_proposals.push(p);

    self.proposals.push(meta);
}
```
post_propose() 主要流程包括
- meta.renew_lease_time 设置为 monotonic_raw_now()
- Proposal::new() 创建 Proposal
- Proposal 放入 Peer::apply_proposals
    - Peer::apply_proposals 会在 Peer::take_apply_proposals() 时被取出，这个是在 Store::on_raft_ready() 时调用的
- meta 放入 Peer::proposals

### Peer::read_local()
可以看到， Peer::read_local() 直接调用 Peer::handle_read() ，而 Peer::handle_read() 直接调用了 Peer::exec_read()
```rust
fn read_local(&mut self, req: RaftCmdRequest, cb: Callback, metrics: &mut RaftProposeMetrics) {
    metrics.local_read += 1;
    cb(self.handle_read(req));
}

fn handle_read(&mut self, req: RaftCmdRequest) -> RaftCmdResponse {
    let mut resp = self.exec_read(&req).unwrap_or_else(|e| {
        match e {
            Error::StaleEpoch(..) => info!("{} stale epoch err: {:?}", self.tag, e),
            _ => error!("{} execute raft command err: {:?}", self.tag, e),
        }
        cmd_resp::new_error(e)
    });

    cmd_resp::bind_term(&mut resp, self.term());
    resp
}
```

而 Peer::exec_read() 就是直接去查询 Peer::kv_engine 了
```rust
fn exec_read(&mut self, req: &RaftCmdRequest) -> Result<RaftCmdResponse> {
        check_epoch(self.region(), req)?;
        let mut snap = None;
        let requests = req.get_requests();
        let mut responses = Vec::with_capacity(requests.len());

        for req in requests {
            let cmd_type = req.get_cmd_type();
            let mut resp = match cmd_type {
                CmdType::Get => {
                    if snap.is_none() {
                        snap = Some(Snapshot::new(self.kv_engine.clone()));
                    }
                    apply::do_get(&self.tag, self.region(), snap.as_ref().unwrap(), req)?
                }
                CmdType::Snap => apply::do_snap(self.region().to_owned())?,
                CmdType::Prewrite |
                CmdType::Put |
                CmdType::Delete |
                CmdType::DeleteRange |
                CmdType::Invalid => unreachable!(),
            };

            resp.set_cmd_type(cmd_type);

            responses.push(resp);
        }

        let mut resp = RaftCmdResponse::new();
        resp.set_responses(protobuf::RepeatedField::from_vec(responses));
        Ok(resp)
    }
```

### Peer::read_index()
```rust
fn read_index(
    &mut self,
    req: RaftCmdRequest,
    cb: Callback,
    metrics: &mut RaftProposeMetrics,
) -> bool {
    metrics.read_index += 1;

    let renew_lease_time = monotonic_raw_now();
    if let Some(read) = self.pending_reads.reads.back_mut() {
        if read.renew_lease_time + self.cfg.raft_store_max_leader_lease() > renew_lease_time {
            read.cmds.push((req, cb));
            return false;
        }
    }

    // Should we call pre_propose here?
    let last_pending_read_count = self.raft_group.raft.pending_read_count();
    let last_ready_read_count = self.raft_group.raft.ready_read_count();

    let id = self.pending_reads.next_id();
    let ctx: [u8; 8] = unsafe { mem::transmute(id) };
    self.raft_group.read_index(ctx.to_vec());

    let pending_read_count = self.raft_group.raft.pending_read_count();
    let ready_read_count = self.raft_group.raft.ready_read_count();

    if pending_read_count == last_pending_read_count &&
        ready_read_count == last_ready_read_count
    {
        // The message gets dropped silently, can't be handled anymore.
        apply::notify_stale_req(self.term(), cb);
        return false;
    }

    self.pending_reads.reads.push_back(ReadIndexRequest {
        id: id,
        cmds: vec![(req, cb)],
        renew_lease_time: renew_lease_time,
    });

    match self.leader_lease_expired_time {
        Some(Either::Right(_)) => {}
        _ => return true,
    };

    // TimeoutNow has been sent out, so we need to propose explicitly to
    // update leader lease.

    let req = RaftCmdRequest::new();
    if let Ok(index) = self.propose_normal(req, metrics) {
        let meta = ProposalMeta {
            index: index,
            term: self.term(),
            renew_lease_time: Some(renew_lease_time),
        };
        self.post_propose(meta, false, box |_| {});
    }

    true
}
```

主要流程
- 获取当前时间 renew_lease_time
- Peer::pending_reads ，即 ReadIndexQueue::reads.back_mut() 有数据
    - 如果最后的元素 read 的 renew_lease_time + raft_store_max_leader_lease() 大于当前时间
        - 说明仍然可以走 read index 并且直接复用上次的 read index 请求
        - read.cmds.push((req, cb))
        - 返回 false
- 获取 raft group 的 last_pending_read_count 和 last_ready_read_count
- Peer::pending_reads.next_id() 得到 id 并转为 ctx (Vec<u8>)
- 使用 ctx 去调用 RawNode::read_index()
- 再次获取 raft group 的 pending_read_count 和 ready_read_count
- 如果 pending_read_count 和 ready_read_count 没有变化
    - 说明请求被 drop 了
    - 执行 apply::notify_stale_req(self.term(), cb)
- 创建 ReadIndexRequest
    - cmds 设置为 vec![(req, cb)]
- 插入 Peer::pending_reads.reads
- match Peer::leader_lease_expired_time
    - 如果为 Some(Either::Right(_)) ，说明当前 lease 不安全，会继续往下走
    - 其他情况，当前 lease 安全，返回 true
- 调用 RaftCmdRequest::new() 创建 req
- 执行 Peer::propose_normal(req) 和 Peer::post_propose()
    - 这样可以更新 lease

### Peer::propose_transfer_leader()

```rust
// Return true to if the transfer leader request is accepted.
fn propose_transfer_leader(
    &mut self,
    req: RaftCmdRequest,
    cb: Callback,
    metrics: &mut RaftProposeMetrics,
) -> bool {
    metrics.transfer_leader += 1;

    let transfer_leader = get_transfer_leader_cmd(&req).unwrap();
    let peer = transfer_leader.get_peer();

    let transfered = if self.is_transfer_leader_allowed(peer) {
        self.transfer_leader(peer);
        true
    } else {
        info!(
            "{} transfer leader message {:?} ignored directly",
            self.tag,
            req
        );
        false
    };

    // transfer leader command doesn't need to replicate log and apply, so we
    // return immediately. Note that this command may fail, we can view it just as an advice
    cb(make_transfer_leader_response());

    transfered
}
```

主要流程
- 获取需要 transfer 到的 peer
- Peer::is_transfer_leader_allowed(peer)
    - 如果为 true ，执行 Peer::transfer_leader(peer)
        - transfered 设为 true
    - 否则 transfered 设为 false
- 直接调用回调
- 返回 transfered

```rust
fn is_transfer_leader_allowed(&self, peer: &metapb::Peer) -> bool {
    let peer_id = peer.get_id();
    let status = self.raft_group.status();

    if !status.progress.contains_key(&peer_id) {
        return false;
    }

    for progress in status.progress.values() {
        if progress.state == ProgressState::Snapshot {
            return false;
        }
    }

    let last_index = self.get_store().last_index();
    last_index <= status.progress[&peer_id].matched + TRANSFER_LEADER_ALLOW_LOG_LAG
}
```
Peer::is_transfer_leader_allowed() 主要流程为
- 获取 Peer::raft_group RawNode::status()
- 如果 status.progress 不包含 peer_id
    - 返回 false ，不允许切主到 peer
- 遍历 status.progress
    - 如果 progress.state 为 ProgressState::Snapshot
        - 返回 false ，不允许切主到 peer
- 从 Peer::get_store().last_index() 获取 last_index
- 如果 `last_index <= progress[&peer_id].matched + TRANSFER_LEADER_ALLOW_LOG_LAG`
    - 返回 true
    - 即对应的 slave 没有落后太多才允许切


Peer::transfer_leader() 则是直接调用了 RawNode::transfer_leader()
```rust
fn transfer_leader(&mut self, peer: &metapb::Peer) {
    info!("{} transfer leader to {:?}", self.tag, peer);

    self.raft_group.transfer_leader(peer.get_id());
}
```

### Peer::propose_conf_change()
Peer::propose_conf_change() 其实也是封装了 RawNode::propose_conf_change()
```rust
fn propose_conf_change(
    &mut self,
    req: RaftCmdRequest,
    metrics: &mut RaftProposeMetrics,
) -> Result<u64> {
    if self.raft_group.raft.pending_conf {
        info!("{} there is a pending conf change, try later", self.tag);
        return Err(box_err!(
            "{} there is a pending conf change, try later",
            self.tag
        ));
    }

    self.check_conf_change(&req)?;

    metrics.conf_change += 1;

    let data = req.write_to_bytes()?;

    // TODO: use local histogram metrics
    PEER_PROPOSE_LOG_SIZE_HISTOGRAM.observe(data.len() as f64);

    let change_peer = apply::get_change_peer_cmd(&req).unwrap();

    let mut cc = eraftpb::ConfChange::new();
    cc.set_change_type(change_peer.get_change_type());
    cc.set_node_id(change_peer.get_peer().get_id());
    cc.set_context(data);

    info!(
        "{} propose conf change {:?} peer {:?}",
        self.tag,
        cc.get_change_type(),
        cc.get_node_id()
    );

    let propose_index = self.next_proposal_index();
    self.raft_group.propose_conf_change(cc)?;
    if self.next_proposal_index() == propose_index {
        // The message is dropped silently, this usually due to leader absence
        // or transferring leader. Both cases can be considered as NotLeader error.
        return Err(Error::NotLeader(self.region_id, None));
    }

    Ok(propose_index)
}
```

主要流程如下
- 如果 Peer::raft_group.raft.pending_conf ，即有 pending 的 conf_change 请求，报错
- Peer::check_conf_change()
    - 主要检查 conf change 之后仍然保证存在多数派
- req 序列化为 data
- apply::get_change_peer_cmd(&req) 得到 change_peer 命令
- eraftpb::ConfChange::new() 创建 cc
- Peer::next_proposal_index() 得到 propose_index
- Peer::raft_group 调用 RawNode::propose_conf_change(cc)
- 如果 Peer::next_proposal_index() 没变
    - 说明请求被 drop 了，返回 NotLeader 报错
- 返回 propose_index

### Peer::get_handle_policy()
主要是根据请求类型和当前的 term ， lease 等情况判断走哪种处理方式

## Peer::propose_snapshot()

```rust
/// Propose a snapshot request. Note that the `None` response means
/// it requires the peer to perform a read-index. The request never
/// be actual proposed to other nodes.
pub fn propose_snapshot(
    &mut self,
    req: RaftCmdRequest,
    metrics: &mut RaftProposeMetrics,
) -> Option<RaftCmdResponse> {
    if self.pending_remove {
        let mut resp = RaftCmdResponse::new();
        cmd_resp::bind_error(&mut resp, box_err!("peer is pending remove"));
        return Some(resp);
    }
    metrics.all += 1;

    // TODO: deny non-snapshot request.

    match self.get_handle_policy(&req) {
        Ok(RequestPolicy::ReadLocal) => {
            metrics.local_read += 1;
            Some(self.handle_read(req))
        }
        // require to propose again, and use the `propose` above.
        Ok(RequestPolicy::ReadIndex) => None,
        Ok(_) => unreachable!(),
        Err(e) => {
            let mut resp = cmd_resp::new_error(e);
            cmd_resp::bind_term(&mut resp, self.term());
            Some(resp)
        }
    }
}
```
主要流程
- match Peer::get_handle_policy(&req)
    - 如果是 Ok(RequestPolicy::ReadLocal)
        - 调用 Peer::handle_read()
        - 返回 Some
    - 如果是 Ok(RequestPolicy::ReadIndex)
        - 返回 None
    - 如果是 Err
        - 返回具体错误

## Peer::handle_raft_ready_append()
Store::on_raft_ready() 会调用这个函数，目前调用方式还是通过 tick() 触发，主要是处理 RawNode::ready()
```rust
pub fn handle_raft_ready_append<T: Transport>(
    &mut self,
    ctx: &mut ReadyContext<T>,
    worker: &FutureWorker<PdTask>,
) {
    self.marked_to_be_checked = false;
    if self.pending_remove {
        return;
    }
    if self.mut_store().check_applying_snap() {
        // If we continue to handle all the messages, it may cause too many messages because
        // leader will send all the remaining messages to this follower, which can lead
        // to full message queue under high load.
        debug!(
            "{} still applying snapshot, skip further handling.",
            self.tag
        );
        return;
    }

    if self.has_pending_snapshot() && !self.ready_to_handle_pending_snap() {
        debug!(
            "{} [apply_idx: {}, last_applying_idx: {}] is not ready to apply snapshot.",
            self.tag,
            self.get_store().applied_index(),
            self.last_applying_idx
        );
        return;
    }

    if !self.raft_group
        .has_ready_since(Some(self.last_applying_idx))
    {
        return;
    }

    debug!("{} handle raft ready", self.tag);

    let mut ready = self.raft_group.ready_since(self.last_applying_idx);

    self.on_role_changed(&ready, worker);

    self.add_ready_metric(&ready, &mut ctx.metrics.ready);

    // The leader can write to disk and replicate to the followers concurrently
    // For more details, check raft thesis 10.2.1.
    if self.is_leader() {
        fail_point!("raft_before_leader_send");
        let msgs = ready.messages.drain(..);
        self.send(ctx.trans, msgs, &mut ctx.metrics.message)
            .unwrap_or_else(|e| {
                // We don't care that the message is sent failed, so here just log this error.
                warn!("{} leader send messages err {:?}", self.tag, e);
            });
    }

    let invoke_ctx = match self.mut_store().handle_raft_ready(ctx, &ready) {
        Ok(r) => r,
        Err(e) => {
            // We may have written something to writebatch and it can't be reverted, so has
            // to panic here.
            panic!("{} failed to handle raft ready: {:?}", self.tag, e);
        }
    };

    ctx.ready_res.push((ready, invoke_ctx));
}
```

主要流程
- Peer::marked_to_be_checked 设置为 false
- 如果 Peer::pending_remove 则返回
- Peer::mut_store() 执行 PeerStorage::check_applying_snap()
    - 判断是否正在 apply snapshot
    - 如果在 applying snapshot ，会先返回，防止 leader 一直向这个机器发送 message
- 如果 Peer::has_pending_snapshot() 且 Peer::ready_to_handle_pending_snap() 为 false
    - 说明现在无法处理 snapshot ，返回
    - Peer::ready_to_handle_pending_snap() 主要判断 Peer::last_applying_idx == Peer::get_store().applied_index()
- 如果 Peer::has_ready_since(Peer::last_applying_idx) 为 false
    - 直接返回
- 执行 Peer::raft_group RawNode::ready_since(self.last_applying_idx) 拿到 ready
- 执行 Peer::on_role_changed()
- Peer::add_ready_metric() ，主要是维护一些统计
- 如果 Peer::is_leader()
    - ready.messages.drain() 取出所有 msg
    - Peer::send(ctx.trans, msgs, &mut ctx.metrics.message) 发送消息
    - 发送失败会打 warn 日志
- Peer::mut_store() 会执行 PeerStorage::handle_raft_ready() 得到 invoke_ctx
    - 失败则 panic
- ctx.ready_res.push((ready, invoke_ctx))

### Peer::on_role_changed()

```rust
fn on_role_changed(&mut self, ready: &Ready, worker: &FutureWorker<PdTask>) {
    // Update leader lease when the Raft state changes.
    if let Some(ref ss) = ready.ss {
        match ss.raft_state {
            StateRole::Leader => {
                // The local read can only be performed after a new leader has applied
                // the first empty entry on its term. After that the lease expiring time
                // should be updated to
                //   send_to_quorum_ts + max_lease
                // as the comments in `next_lease_expired_time` function explain.
                // It is recommended to update the lease expiring time right after
                // this peer becomes leader because it's more convenient to do it here and
                // it has no impact on the correctness.
                let next_expired_time = self.next_lease_expired_time(monotonic_raw_now());
                self.leader_lease_expired_time = Some(Either::Left(next_expired_time));
                debug!(
                    "{} becomes leader and lease expired time is {:?}",
                    self.tag,
                    next_expired_time
                );
                self.heartbeat_pd(worker)
            }
            StateRole::Follower => {
                self.leader_lease_expired_time = None;
            }
            _ => {}
        }
    }
}
```

主要流程
- 如果 Ready::ss 为 Some
    - match ss.raft_state
        - 如果是 StateRole::Leader
            - 更新 Peer::leader_lease_expired_time 为 Either::Left(next_lease_expired_time(monotonic_raw_now()))
            - 执行 Peer::heartbeat_pd()
        - 如果 StateRole::Follower
            - Peer::leader_lease_expired_time 设置为 None

### Peer::send()

```rust
#[inline]
fn send<T, I>(&mut self, trans: &T, msgs: I, metrics: &mut RaftMessageMetrics) -> Result<()>
where
    T: Transport,
    I: IntoIterator<Item = eraftpb::Message>,
{
    for msg in msgs {
        let msg_type = msg.get_msg_type();

        self.send_raft_message(msg, trans)?;

        match msg_type {
            // ...snipped...
            MessageType::MsgTimeoutNow => {
                // After a leader transfer procedure is triggered, the lease for
                // the old leader may be expired earlier than usual, since a new leader
                // may be elected and the old leader doesn't step down due to
                // network partition from the new leader.
                // For lease safty during leader transfer, mark `leader_lease_expired_time`
                // to be unsafe until next_lease_expired_time from now
                self.leader_lease_expired_time = Some(Either::Right(
                    self.next_lease_expired_time(monotonic_raw_now()),
                ));

                metrics.timeout_now += 1;
            }
            _ => {}
        }
    }
    Ok(())
}
```

可以看到主要是
- 执行 Peer::send_raft_message()
- 如果是 MessageType::MsgTimeoutNow
    - 设置 Peer::leader_lease_expired_time 为 Either::Right(self.next_lease_expired_time(monotonic_raw_now()))
- 更新 metrics

核心的 send 逻辑在 Peer::send_raft_message()
```rust
fn send_raft_message<T: Transport>(&mut self, msg: eraftpb::Message, trans: &T) -> Result<()> {
    let mut send_msg = RaftMessage::new();
    send_msg.set_region_id(self.region_id);
    // set current epoch
    send_msg.set_region_epoch(self.region().get_region_epoch().clone());

    let from_peer = self.peer.clone();

    let to_peer = match self.get_peer_from_cache(msg.get_to()) {
        Some(p) => p,
        None => {
            return Err(box_err!(
                "failed to look up recipient peer {} in region {}",
                msg.get_to(),
                self.region_id
            ))
        }
    };

    let to_peer_id = to_peer.get_id();
    let to_store_id = to_peer.get_store_id();
    let msg_type = msg.get_msg_type();
    debug!(
        "{} send raft msg {:?}[size: {}] from {} to {}",
        self.tag,
        msg_type,
        msg.compute_size(),
        from_peer.get_id(),
        to_peer_id
    );

    send_msg.set_from_peer(from_peer);
    send_msg.set_to_peer(to_peer);

    // There could be two cases:
    // 1. Target peer already exists but has not established communication with leader yet
    // 2. Target peer is added newly due to member change or region split, but it's not
    //    created yet
    // For both cases the region start key and end key are attached in RequestVote and
    // Heartbeat message for the store of that peer to check whether to create a new peer
    // when receiving these messages, or just to wait for a pending region split to perform
    // later.
    if self.get_store().is_initialized() &&
        (msg_type == MessageType::MsgRequestVote ||
        // the peer has not been known to this leader, it may exist or not.
        (msg_type == MessageType::MsgHeartbeat && msg.get_commit() == INVALID_INDEX))
    {
        let region = self.region();
        send_msg.set_start_key(region.get_start_key().to_vec());
        send_msg.set_end_key(region.get_end_key().to_vec());
    }

    send_msg.set_message(msg);

    if let Err(e) = trans.send(send_msg) {
        warn!(
            "{} failed to send msg to {} in store {}, err: {:?}",
            self.tag,
            to_peer_id,
            to_store_id,
            e
        );

        // unreachable store
        self.raft_group.report_unreachable(to_peer_id);
        if msg_type == eraftpb::MessageType::MsgSnapshot {
            self.raft_group
                .report_snapshot(to_peer_id, SnapshotStatus::Failure);
        }
    }

    Ok(())
}
```

主要流程包括
- RaftMessage::new() 创建 send_msg ，设置 region_id 和 term
- 通过 Peer::get_peer_from_cache() 获取 to_peer
- 设置 send_msg 的 from_peer/to_peer
- 如果 PeerStorage::is_initialized() 且 (是 MessageType::MsgRequestVote 消息或者来自未知节点的 MessageType::MsgHeartbeat 消息
    - 设置 send_msg 的 start/end key 为当前 region 的 start/end key
- 通过 trans 的 Transport::send() 发送 send_msg
    - 如果失败
    - 执行 Peer::raft_group RawNode::report_unreachable()
    - 如果 msg_type == eraftpb::MessageType::MsgSnapshot
        - 执行 Peer::raft_group RawNode::report_snapshot() 通知失败

## Peer::post_raft_ready_append()
该函数在 raft log append 后执行
```rust
pub fn post_raft_ready_append<T: Transport>(
    &mut self,
    metrics: &mut RaftMetrics,
    trans: &T,
    ready: &mut Ready,
    invoke_ctx: InvokeContext,
) -> Option<ApplySnapResult> {
    if invoke_ctx.has_snapshot() {
        // When apply snapshot, there is no log applied and not compacted yet.
        self.raft_log_size_hint = 0;
    }

    let apply_snap_result = self.mut_store().post_ready(invoke_ctx);

    if !self.is_leader() {
        fail_point!("raft_before_follower_send");
        self.send(trans, ready.messages.drain(..), &mut metrics.message)
            .unwrap_or_else(|e| {
                warn!("{} follower send messages err {:?}", self.tag, e);
            });
    }

    if apply_snap_result.is_some() {
        let reg = ApplyTask::register(self);
        self.apply_scheduler.schedule(reg).unwrap();
    }

    apply_snap_result
}
```

主要流程
- 如果 InvokeContext::has_snapshot()
    - Peer::raft_log_size_hint 清 0
- 执行 PeerStorage::post_ready() 拿到 apply_snap_result
- 如果 Peer 不是 leader
    - Peer::send() 发送消息
- 如果 apply_snap_result 为 Some
    - 将自己作为 ApplyTask ApplyTask::register(self)
    - 放到 Peer::apply_scheduler 执行
- 返回 apply_snap_result
    - 这个结果会在 Store::on_raft_ready() 时用来调用 Store::on_ready_apply_snapshot()

## Peer::handle_raft_ready_apply()
主要是从 ready 里收集 commited 的 task 到 apply_tasks 里
```rust
pub fn handle_raft_ready_apply(&mut self, mut ready: Ready, apply_tasks: &mut Vec<Apply>) {
    // Call `handle_raft_committed_entries` directly here may lead to inconsistency.
    // In some cases, there will be some pending committed entries when applying a
    // snapshot. If we call `handle_raft_committed_entries` directly, these updates
    // will be written to disk. Because we apply snapshot asynchronously, so these
    // updates will soon be removed. But the soft state of raft is still be updated
    // in memory. Hence when handle ready next time, these updates won't be included
    // in `ready.committed_entries` again, which will lead to inconsistency.
    if self.is_applying_snapshot() {
        // Snapshot's metadata has been applied.
        self.last_applying_idx = self.get_store().truncated_index();
    } else {
        let committed_entries = ready.committed_entries.take().unwrap();
        // leader needs to update lease.
        let mut to_be_updated = self.is_leader();
        if !to_be_updated {
            // It's not leader anymore, we are safe to clear proposals. If it becomes leader
            // again, the lease should be updated when election is finished, old proposals
            // have no effect.
            self.proposals.clear();
        }
        for entry in committed_entries.iter().rev() {
            // raft meta is very small, can be ignored.
            self.raft_log_size_hint += entry.get_data().len() as u64;
            if to_be_updated {
                to_be_updated = !self.maybe_update_lease(entry);
            }
        }
        if !committed_entries.is_empty() {
            self.last_applying_idx = committed_entries.last().unwrap().get_index();
            apply_tasks.push(Apply::new(self.region_id, self.term(), committed_entries));
        }
    }

    self.apply_reads(&ready);

    self.raft_group.advance_append(ready);
    if self.is_applying_snapshot() {
        // Because we only handle raft ready when not applying snapshot, so following
        // line won't be called twice for the same snapshot.
        self.raft_group.advance_apply(self.last_applying_idx);
    }
}
```

主要流程
- 如果 Peer::is_applying_snapshot()
    - 更新 Peer::last_applying_idx 为 PeerStorage::truncated_index()
- 否则
    - 取出 Ready::committed_entries
    - to_be_updated 设置为 Peer::is_leader()
    - 如果 !to_be_updated
        - 不是 leader 了，清空 Peer::proposals
    - 反向遍历 committed_entries
        - Peer::raft_log_size_hint 累加上 entry 的 data 大小
        - 如果 to_be_updated
            - to_be_updated 设置为 !Peer::maybe_update_lease(entry)
    - 如果 committed_entries 非空
        - Peer::last_applying_idx 设置为 committed_entries 最后一项的 index
        - apply_tasks.push(Apply::new(self.region_id, self.term(), committed_entries))
- 执行 Peer::apply_reads()
- 执行 Peer::raft_group 即 RawNode::advance_append()
- 如果 Peer::is_applying_snapshot()
    - 执行 RawNode::advance_apply(self.last_applying_idx)

### Peer::maybe_update_lease()
```rust
/// Try to update lease.
///
/// If the it can make sure that its lease is the latest lease, returns true.
fn maybe_update_lease(&mut self, entry: &eraftpb::Entry) -> bool {
    let propose_time = match self.find_propose_time(entry.get_index(), entry.get_term()) {
        Some(t) => t,
        _ => return false,
    };

    self.update_lease_with(propose_time);

    true
}

fn find_propose_time(&mut self, index: u64, term: u64) -> Option<Timespec> {
    while let Some(meta) = self.proposals.pop(term) {
        if meta.index == index && meta.term == term {
            return Some(meta.renew_lease_time.unwrap());
        }
    }
    None
}
```
主要流程
- 把 Peer::proposals 即 ProposalQueue 的 proposals 一直 pop 直到碰到符合的 index 和 term
- 将对应的 renew_lease_time 作为 propose_time
- 调用 Peer::update_lease_with(propose_time)

lease 的更新规则如下
```rust
fn update_lease_with(&mut self, propose_time: Timespec) {
    // Try to renew the leader lease as this command asks to.
    if self.leader_lease_expired_time.is_some() {
        let current_expired_time =
            match self.leader_lease_expired_time.as_ref().unwrap().as_ref() {
                Either::Left(safe_expired_time) => *safe_expired_time,
                Either::Right(unsafe_expired_time) => *unsafe_expired_time,
            };
        // This peer is leader and has recorded leader lease.
        // Calculate the renewed lease for this command. If the renewed lease lives longer
        // than the current leader lease, update the current leader lease to the renewed lease.
        let next_expired_time = self.next_lease_expired_time(propose_time);
        // Use the lease expired timestamp comparison here, so that these codes still
        // work no matter how the leader changes before applying this command.
        if current_expired_time < next_expired_time {
            debug!(
                "{} update leader lease expired time from {:?} to {:?}",
                self.tag,
                current_expired_time,
                next_expired_time
            );
            self.leader_lease_expired_time = Some(Either::Left(next_expired_time));
        }
    } else if self.is_leader() {
        // This peer is leader but its leader lease has expired.
        // Calculate the renewed lease for this command, and update the leader lease
        // for this peer.
        let next_expired_time = self.next_lease_expired_time(propose_time);
        debug!(
            "{} update leader lease expired time from None to {:?}",
            self.tag,
            next_expired_time
        );
        self.leader_lease_expired_time = Some(Either::Left(next_expired_time));
    }
}
```

### Peer::apply_reads()
```rust
fn apply_reads(&mut self, ready: &Ready) {
    let mut propose_time = None;
    if self.ready_to_handle_read() {
        for state in &ready.read_states {
            let mut read = self.pending_reads.reads.pop_front().unwrap();
            assert_eq!(state.request_ctx.as_slice(), read.binary_id());
            for (req, cb) in read.cmds.drain(..) {
                // TODO: we should add test case that a split happens before pending
                // read-index is handled. To do this we need to control async-apply
                // procedure precisely.
                cb(self.handle_read(req));
            }
            propose_time = Some(read.renew_lease_time);
        }
    } else {
        for state in &ready.read_states {
            let read = &self.pending_reads.reads[self.pending_reads.ready_cnt];
            assert_eq!(state.request_ctx.as_slice(), read.binary_id());
            self.pending_reads.ready_cnt += 1;
            propose_time = Some(read.renew_lease_time);
        }
    }

    // Note that only after handle read_states can we identify what requests are
    // actually stale.
    if ready.ss.is_some() {
        let term = self.term();
        // all uncommitted reads will be dropped silently in raft.
        self.pending_reads.clear_uncommitted(term);
    }

    if let Some(Either::Right(_)) = self.leader_lease_expired_time {
        return;
    }

    if let Some(propose_time) = propose_time {
        self.update_lease_with(propose_time);
    }
}
```

主要流程
- 如果 Peer::ready_to_handle_read()
    - 即 self.get_store().applied_index_term == self.term()
    - 遍历 Ready::read_states
        - Peer::pending_reads.reads.pop_front() 取出 read ，这里的 read 请求是在 Peer::read_index() 时放入的
        - 对于 read 里的所有 cmds
            - 执行 Peer::handle_read()
            - 结果回调 cb
        - 设置 propose_time 为 read.renew_lease_time
- 否则
    - 不执行 read 操作
    - 遍历 Ready::read_states
        - 获取 Peer::pending_reads 里对应的 read
        - 设置 propose_time 为 read.renew_lease_time
        - 增加 Peer::pending_reads 的 ready_cnt ，这些是已经 ready 但是还没法处理的 read 请求数
- 如果 Ready::ss 为 Some
    - 处理完已有的 read 请求后，可以判断还有哪些 read 请求是 stale 的
    - 执行 Peer::pending_reads.clear_uncommitted(Peer::term())
- 如果 Peer::leader_lease_expired_time 为 Some(Either::Right(_))
    - 返回
- 执行 Peer::update_lease_with(propose_time)

## Peer::post_apply()
```rust
pub fn post_apply(
    &mut self,
    res: &ApplyRes,
    groups: &mut HashSet<u64>,
    store_stat: &mut StoreStat,
) {
    if self.is_applying_snapshot() {
        panic!("{} should not applying snapshot.", self.tag);
    }

    let has_split = res.exec_res.iter().any(|e| match *e {
        ExecResult::SplitRegion { .. } => true,
        _ => false,
    });

    self.raft_group
        .advance_apply(res.apply_state.get_applied_index());
    self.mut_store().apply_state = res.apply_state.clone();
    self.mut_store().applied_index_term = res.applied_index_term;
    self.peer_stat.written_keys += res.metrics.written_keys;
    self.peer_stat.written_bytes += res.metrics.written_bytes;
    store_stat.engine_total_bytes_written += res.metrics.written_bytes;
    store_stat.engine_total_keys_written += res.metrics.written_keys;

    let diff = if has_split {
        self.delete_keys_hint = res.metrics.delete_keys_hint;
        res.metrics.size_diff_hint
    } else {
        self.delete_keys_hint += res.metrics.delete_keys_hint;
        self.size_diff_hint as i64 + res.metrics.size_diff_hint
    };
    self.size_diff_hint = cmp::max(diff, 0) as u64;

    if self.has_pending_snapshot() && self.ready_to_handle_pending_snap() {
        self.mark_to_be_checked(groups);
    }

    if self.pending_reads.ready_cnt > 0 && self.ready_to_handle_read() {
        for _ in 0..self.pending_reads.ready_cnt {
            let mut read = self.pending_reads.reads.pop_front().unwrap();
            for (req, cb) in read.cmds.drain(..) {
                cb(self.handle_read(req));
            }
        }
        self.pending_reads.ready_cnt = 0;
    }
}
```
主要流程
- 判断是否有 split ，设置 has_split
- 执行 Peer::raft_group 的 RawNode::advance_apply(res.apply_state.get_applied_index())
- 更新 Peer::mut_store() 的 PeerStorage::apply_state 为 ApplyRes::apply_state
- 更新 Peer::mut_store() 的 PeerStorage::applied_index_term 为 ApplyRes::applied_index_term
- 更新 Peer::delete_keys_hint 和 Peer::size_diff_hint
- 如果 Peer::has_pending_snapshot() 且 Peer::ready_to_handle_pending_snap()
    - 设置 Peer::mark_to_be_checked()
- 如果 Peer::pending_reads.ready_cnt > 0 && Peer::ready_to_handle_read()
    - 遍历 Peer::pending_reads.ready_cnt 次
        - 取出 Peer::pending_reads.reads ，处理 read 请求并做回调
