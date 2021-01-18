# TiKV v1.0 raftstore Store

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
- 创建 PeerStorage
- 调用 PeerStorage::applied_index() 获取 applied index
- 创建 RawNode
- 创建 Peer

## Peer::destroy()
TODO

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


