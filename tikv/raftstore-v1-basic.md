# TiKV v1.0 raftstore

## 基本模块和入口
### tikv-server
tikv-server.rs run_raft_server()
```rust
fn run_raft_server(pd_client: RpcClient, cfg: &TiKvConfig) {
    // ... snipped ...
    // Initialize raftstore channels.
    let mut event_loop = store::create_event_loop(&cfg.raft_store)
        .unwrap_or_else(|e| fatal!("failed to create event loop: {:?}", e));
    let store_sendch = SendCh::new(event_loop.channel(), "raftstore");
    let (significant_msg_sender, significant_msg_receiver) = mpsc::channel();
    let raft_router = ServerRaftStoreRouter::new(store_sendch.clone(), significant_msg_sender);

    // Create kv engine, storage.
    let kv_db_opts = cfg.rocksdb.build_opt();
    let kv_cfs_opts = cfg.rocksdb.build_cf_opts();
    let kv_engine = Arc::new(
        rocksdb_util::new_engine_opt(db_path.to_str().unwrap(), kv_db_opts, kv_cfs_opts)
            .unwrap_or_else(|s| fatal!("failed to create kv engine: {:?}", s)),
    );
    let mut storage = create_raft_storage(raft_router.clone(), kv_engine.clone(), &cfg.storage)
        .unwrap_or_else(|e| fatal!("failed to create raft stroage: {:?}", e));

    // Create raft engine.
    let raft_db_opts = cfg.raftdb.build_opt();
    let raft_db_cf_opts = cfg.raftdb.build_cf_opts();
    let raft_engine = Arc::new(
        rocksdb_util::new_engine_opt(
            raft_db_path.to_str().unwrap(),
            raft_db_opts,
            raft_db_cf_opts,
        ).unwrap_or_else(|s| fatal!("failed to create raft engine: {:?}", s)),
    );
    let engines = Engines::new(kv_engine.clone(), raft_engine.clone());

    // Create pd client and pd work, snapshot manager, server.
    let pd_client = Arc::new(pd_client);
    let pd_worker = FutureWorker::new("pd worker");
    let (mut worker, resolver) = resolve::new_resolver(pd_client.clone())
        .unwrap_or_else(|e| fatal!("failed to start address resolver: {:?}", e));
    let snap_mgr = SnapManager::new(
        snap_path.as_path().to_str().unwrap().to_owned(),
        Some(store_sendch),
    );

    // Create server
    let mut server = Server::new(
        &cfg.server,
        cfg.raft_store.region_split_size.0 as usize,
        storage.clone(),
        raft_router,
        resolver,
        snap_mgr.clone(),
        pd_worker.scheduler(),
        Some(engines.clone()),
    ).unwrap_or_else(|e| fatal!("failed to create server: {:?}", e));
    let trans = server.transport();

    // Create node.
    let mut node = Node::new(&mut event_loop, &cfg.server, &cfg.raft_store, pd_client);
    node.start(
        event_loop,
        engines.clone(),
        trans,
        snap_mgr,
        significant_msg_receiver,
        pd_worker,
    ).unwrap_or_else(|e| fatal!("failed to start node: {:?}", e));
    initial_metric(&cfg.metric, Some(node.id()));

    // Start storage.
    info!("start storage");
    if let Err(e) = storage.start(&cfg.storage) {
        fatal!("failed to start storage, error: {:?}", e);
    }

    // ... snipped ...
    // Run server.
    server
        .start(&cfg.server)
        .unwrap_or_else(|e| fatal!("failed to start server: {:?}", e));
    // ... snipped ...
}
```

### ServerRaftStoreRouter
外界对 server 的 raft 命令实际上是交由 ServerRaftStoreRouter 来做转发处理的，代码位于 server/transport.rs
```rust
#[derive(Clone)]
pub struct ServerRaftStoreRouter {
    pub ch: SendCh<StoreMsg>,
    pub significant_msg_sender: Sender<SignificantMsg>,
}

impl RaftStoreRouter for ServerRaftStoreRouter {
    fn try_send(&self, msg: StoreMsg) -> RaftStoreResult<()> {
        self.ch.try_send(msg)?;
        Ok(())
    }

    fn send(&self, msg: StoreMsg) -> RaftStoreResult<()> {
        self.ch.send(msg)?;
        Ok(())
    }

    fn send_raft_msg(&self, msg: RaftMessage) -> RaftStoreResult<()> {
        self.try_send(StoreMsg::RaftMessage(msg))
    }

    fn send_command(&self, req: RaftCmdRequest, cb: Callback) -> RaftStoreResult<()> {
        self.try_send(StoreMsg::new_raft_cmd(req, cb))
    }

    fn send_batch_commands(
        &self,
        batch: Vec<RaftCmdRequest>,
        on_finished: BatchCallback,
    ) -> RaftStoreResult<()> {
        self.try_send(StoreMsg::new_batch_raft_snapshot_cmd(batch, on_finished))
    }

    fn significant_send(&self, msg: SignificantMsg) -> RaftStoreResult<()> {
        if let Err(e) = self.significant_msg_sender.send(msg) {
            return Err(box_err!("failed to sendsignificant msg {:?}", e));
        }

        Ok(())
    }
}

```

注意在 run_raft_server 里，会创建 store_sendch 并基于该 channel 创建  ServerRaftStoreRouter
- store_sendch 的消息会通过 event_loop 的 channel 进行发送，最终被 Node 处理
- significant_msg_sender 同样也是一个 channel ，里面的消息同样也会被 Node 处理
- raft 的消息最终都会通过 channel 以事件的形式交由 Node 处理，后面只需要关注 Node 如何处理这些事件即可

### Node
Node 实际上应该是 raftstore 的一个封装，见 server/node.rs
```rust
// Node is a wrapper for raft store.
// TODO: we will rename another better name like RaftStore later.
pub struct Node<C: PdClient + 'static> {
    cluster_id: u64,
    store: metapb::Store,
    store_cfg: StoreConfig,
    store_handle: Option<thread::JoinHandle<()>>,
    ch: SendCh<Msg>,

    pd_client: Arc<C>,
}
```

和启动相关的主要函数
- Node::new() 创建新的 Node
- Node::start() 启动 Node ，核心逻辑在 start

```rust
impl<C> Node<C>
where
    C: PdClient,
{
    pub fn new<T>(
        event_loop: &mut EventLoop<Store<T, C>>,
        cfg: &ServerConfig,
        store_cfg: &StoreConfig,
        pd_client: Arc<C>,
    ) -> Node<C>
    where
        T: Transport + 'static,
    {
        let mut store = metapb::Store::new();
        store.set_id(INVALID_ID);
        if cfg.advertise_addr.is_empty() {
            store.set_address(cfg.addr.clone());
        } else {
            store.set_address(cfg.advertise_addr.clone())
        }

        let mut labels = Vec::new();
        for (k, v) in &cfg.labels {
            let mut label = metapb::StoreLabel::new();
            label.set_key(k.to_owned());
            label.set_value(v.to_owned());
            labels.push(label);
        }
        store.set_labels(RepeatedField::from_vec(labels));

        let ch = SendCh::new(event_loop.channel(), "raftstore");
        Node {
            cluster_id: cfg.cluster_id,
            store: store,
            store_cfg: store_cfg.clone(),
            store_handle: None,
            pd_client: pd_client,
            ch: ch,
        }
    }

    pub fn start<T>(
        &mut self,
        event_loop: EventLoop<Store<T, C>>,
        engines: Engines,
        trans: T,
        snap_mgr: SnapManager,
        significant_msg_receiver: Receiver<SignificantMsg>,
        pd_worker: FutureWorker<PdTask>,
    ) -> Result<()>
    where
        T: Transport + 'static,
    {
        let bootstrapped = self.check_cluster_bootstrapped()?;
        let mut store_id = self.check_store(&engines)?;
        if store_id == INVALID_ID {
            store_id = self.bootstrap_store(&engines)?;
        } else if !bootstrapped {
            // We have saved data before, and the cluster must be bootstrapped.
            return Err(box_err!(
                "store {} is not empty, but cluster {} is not bootstrapped, \
                 maybe you connected a wrong PD or need to remove the TiKV data \
                 and start again",
                store_id,
                self.cluster_id
            ));
        }

        self.store.set_id(store_id);
        self.check_prepare_bootstrap_cluster(&engines)?;
        if !bootstrapped {
            // cluster is not bootstrapped, and we choose first store to bootstrap
            // prepare bootstrap.
            let region = self.prepare_bootstrap_cluster(&engines, store_id)?;
            self.bootstrap_cluster(&engines, region)?;
        }

        // inform pd.
        self.pd_client.put_store(self.store.clone())?;
        self.start_store(
            event_loop,
            store_id,
            engines,
            trans,
            snap_mgr,
            significant_msg_receiver,
            pd_worker,
        )?;
        Ok(())
    }
}
```

而 Node 会在 start_store() 里把 raftstore 模块的核心 Store 启动起来
```rust
#[allow(too_many_arguments)]
fn start_store<T>(
    &mut self,
    mut event_loop: EventLoop<Store<T, C>>,
    store_id: u64,
    engines: Engines,
    trans: T,
    snap_mgr: SnapManager,
    significant_msg_receiver: Receiver<SignificantMsg>,
    pd_worker: FutureWorker<PdTask>,
) -> Result<()>
where
    T: Transport + 'static,
{
    info!("start raft store {} thread", store_id);

    if self.store_handle.is_some() {
        return Err(box_err!("{} is already started", store_id));
    }

    let cfg = self.store_cfg.clone();
    let pd_client = self.pd_client.clone();
    let store = self.store.clone();
    let sender = event_loop.channel();

    let (tx, rx) = mpsc::channel();
    let builder = thread::Builder::new().name(thd_name!(format!("raftstore-{}", store_id)));
    let h = builder.spawn(move || {
        let ch = StoreChannel {
            sender,
            significant_msg_receiver,
        };
        let mut store = match Store::new(
            ch,
            store,
            cfg,
            engines,
            trans,
            pd_client,
            snap_mgr,
            pd_worker,
        ) {
            Err(e) => panic!("construct store {} err {:?}", store_id, e),
            Ok(s) => s,
        };
        tx.send(0).unwrap();
        if let Err(e) = store.run(&mut event_loop) {
            error!("store {} run err {:?}", store_id, e);
        };
    })?;
    // wait for store to be initialized
    rx.recv().unwrap();

    self.store_handle = Some(h);
    Ok(())
}
```

这里主要做的事情有
- 调用 Store::new() 创建 Store
- 创建一个线程 `raftstore-{store_id}` 并调用 Store::run() 来驱动 Store ，在 v1.0 版本里 tikv 只有一个 raftstore 线程
- 父线程会等到 Store::new() 创建出 Store 后返回

### Store
Store 位于 raftstore/store/store.rs ，是 raft 存储模块的入口，里面管理了节点上的所有 region (raft group) ，而 tikv 会把一个 region 或者说 raft node 封装成一个 Peer
```rust
pub struct Store<T, C: 'static> {
    cfg: Rc<Config>,
    kv_engine: Arc<DB>,
    raft_engine: Arc<DB>,
    store: metapb::Store,
    sendch: SendCh<Msg>,

    significant_msg_receiver: StdReceiver<SignificantMsg>,

    // region_id -> peers
    region_peers: HashMap<u64, Peer>,
    pending_raft_groups: HashSet<u64>,
    // region end key -> region id
    region_ranges: BTreeMap<Key, u64>,
    // the regions with pending snapshots between two mio ticks.
    pending_snapshot_regions: Vec<metapb::Region>,
    split_check_worker: Worker<SplitCheckTask>,
    region_worker: Worker<RegionTask>,
    raftlog_gc_worker: Worker<RaftlogGcTask>,
    compact_worker: Worker<CompactTask>,
    pd_worker: FutureWorker<PdTask>,
    consistency_check_worker: Worker<ConsistencyCheckTask>,
    pub apply_worker: Worker<ApplyTask>,
    apply_res_receiver: Option<StdReceiver<ApplyTaskRes>>,

    trans: T,
    pd_client: Arc<C>,

    pub coprocessor_host: Arc<CoprocessorHost>,

    snap_mgr: SnapManager,

    raft_metrics: RaftMetrics,
    pub entry_cache_metries: Rc<RefCell<CacheQueryStats>>,

    tag: String,

    start_time: Timespec,
    is_busy: bool,

    pending_votes: RingQueue<RaftMessage>,

    store_stat: StoreStat,
}
```

## Store 启动
接下来关注下 Store 的创建和启动，首先需要知道 Store 依赖的 EventLoop 在创建的时候会设置上一些 tick 周期等配置，详见 create_event_loop()
```rust
const MIO_TICK_RATIO: u64 = 10;

pub fn create_event_loop<T, C>(cfg: &Config) -> Result<EventLoop<Store<T, C>>>
where
    T: Transport,
    C: PdClient,
{
    let mut config = EventLoopConfig::new();
    // To make raft base tick more accurate, timer tick should be small enough.
    config.timer_tick_ms(cfg.raft_base_tick_interval.as_millis() / MIO_TICK_RATIO);
    config.notify_capacity(cfg.notify_capacity);
    config.messages_per_tick(cfg.messages_per_tick);
    let event_loop = EventLoop::configured(config)?;
    Ok(event_loop)
}
```

创建 Store::new() 实现如下
```rust
impl<T, C> Store<T, C> {
    #[allow(too_many_arguments)]
    pub fn new(
        ch: StoreChannel,
        meta: metapb::Store,
        cfg: Config,
        engines: Engines,
        trans: T,
        pd_client: Arc<C>,
        mgr: SnapManager,
        pd_worker: FutureWorker<PdTask>,
    ) -> Result<Store<T, C>> {
        // TODO: we can get cluster meta regularly too later.
        cfg.validate()?;

        let sendch = SendCh::new(ch.sender, "raftstore");
        let tag = format!("[store {}]", meta.get_id());

        let mut coprocessor_host = CoprocessorHost::new();
        // TODO load coprocessors from configuration
        coprocessor_host
            .registry
            .register_observer(100, box SplitObserver);

        let mut s = Store {
            cfg: Rc::new(cfg),
            store: meta,
            kv_engine: engines.kv_engine,
            raft_engine: engines.raft_engine,
            sendch: sendch,
            significant_msg_receiver: ch.significant_msg_receiver,
            region_peers: HashMap::default(),
            pending_raft_groups: HashSet::default(),
            split_check_worker: Worker::new("split check worker"),
            // ... snipped ...
        };
        s.init()?;
        Ok(s)
    }
}
```

### init
会看到最后会调用 Store::init() 进行初始化，经过 init 之后，整个 store 的 region 信息就从存储恢复
```rust
/// Initialize this store. It scans the db engine, loads all regions
/// and their peers from it, and schedules snapshot worker if neccessary.
/// WARN: This store should not be used before initialized.
fn init(&mut self) -> Result<()> {
    // Scan region meta to get saved regions.
    let start_key = keys::REGION_META_MIN_KEY;
    let end_key = keys::REGION_META_MAX_KEY;
    let kv_engine = self.kv_engine.clone();
    let mut total_count = 0;
    let mut tomebstone_count = 0;
    let mut applying_count = 0;

    let t = Instant::now();
    let mut kv_wb = WriteBatch::new();
    let mut raft_wb = WriteBatch::new();
    let mut applying_regions = vec![];
    kv_engine.scan_cf(
        CF_RAFT,
        start_key,
        end_key,
        false,
        &mut |key, value| {
            let (region_id, suffix) = keys::decode_region_meta_key(key)?;
            if suffix != keys::REGION_STATE_SUFFIX {
                return Ok(true);
            }

            total_count += 1;

            let local_state = protobuf::parse_from_bytes::<RegionLocalState>(value)?;
            let region = local_state.get_region();
            if local_state.get_state() == PeerState::Tombstone {
                tomebstone_count += 1;
                debug!(
                    "region {:?} is tombstone in store {}",
                    region,
                    self.store_id()
                );
                self.clear_stale_meta(&mut kv_wb, &mut raft_wb, region);
                return Ok(true);
            }
            if local_state.get_state() == PeerState::Applying {
                // in case of restart happen when we just write region state to Applying,
                // but not write raft_local_state to raft rocksdb in time.
                peer_storage::recover_from_applying_state(
                    &self.kv_engine,
                    &self.raft_engine,
                    &raft_wb,
                    region_id,
                )?;
                applying_count += 1;
                applying_regions.push(region.clone());
                return Ok(true);
            }

            let peer = Peer::create(self, region)?;
            self.region_ranges.insert(enc_end_key(region), region_id);
            // No need to check duplicated here, because we use region id as the key
            // in DB.
            self.region_peers.insert(region_id, peer);
            Ok(true)
        },
    )?;

    if !kv_wb.is_empty() {
        self.kv_engine.write(kv_wb).unwrap();
        self.kv_engine.sync_wal().unwrap();
    }
    if !raft_wb.is_empty() {
        self.raft_engine.write(raft_wb).unwrap();
        self.raft_engine.sync_wal().unwrap();
    }

    // schedule applying snapshot after raft writebatch were written.
    for region in applying_regions {
        info!(
            "region {:?} is applying in store {}",
            region,
            self.store_id()
        );
        let mut peer = Peer::create(self, &region)?;
        peer.mut_store().schedule_applying_snapshot();
        self.region_ranges
            .insert(enc_end_key(&region), region.get_id());
        self.region_peers.insert(region.get_id(), peer);
    }

    info!(
        "{} starts with {} regions, including {} tombstones and {} applying \
            regions, takes {:?}",
        self.tag,
        total_count,
        tomebstone_count,
        applying_count,
        t.elapsed()
    );

    self.clear_stale_data()?;

    Ok(())
}
```

初始化的主要流程包括
- scan 整个 kv engine
    - column family: CF_RAFT
    - start_key: keys::REGION_META_MIN_KEY
    - end_key: keys::REGION_META_MAX_KEY;
    - 读取所有 suffix 是 keys::REGION_STATE_SUFFIX 的 kv 对
        - key 记录了 region id
        - value 为 RegionLocalState
    - 处理 RegionLocalState
        - state == PeerState::Tombstone
            - Store::clear_stale_meta()
        - state == PeerState::Applying
            - peer_storage::recover_from_applying_state()
            - applying_regions.push(region.clone());
        - 其他情况
            - Peer::create() 创建 Peer;
            - self.region_ranges.insert(enc_end_key(region), region_id) 记录下 region_ranges
            - self.region_peers.insert(region_id, peer) 记录下 region_peers
- 将初始化过程中 Tombstone 和 Applying 的 region 的信息持久化到 kv_engine 和 raft_engine
- 遍历 applying_regions
    - Peer::create() 创建 peer
    - 调用 peer.mut_store().schedule_applying_snapshot() 调度 apply snapshot 的任务
- Store::clear_stale_data() 清理掉存储里的一些无用信息

### run
前面看到， start_store() 专门启动了一个线程里执行 Store::run()
```rust
impl<T: Transport, C: PdClient> Store<T, C> {
    pub fn run(&mut self, event_loop: &mut EventLoop<Self>) -> Result<()> {
        self.snap_mgr.init()?;

        self.register_raft_base_tick(event_loop);
        self.register_raft_gc_log_tick(event_loop);
        self.register_split_region_check_tick(event_loop);
        self.register_compact_check_tick(event_loop);
        self.register_pd_store_heartbeat_tick(event_loop);
        self.register_pd_heartbeat_tick(event_loop);
        self.register_snap_mgr_gc_tick(event_loop);
        self.register_compact_lock_cf_tick(event_loop);
        self.register_consistency_check_tick(event_loop);

        let split_check_runner = SplitCheckRunner::new(
            self.kv_engine.clone(),
            self.sendch.clone(),
            self.cfg.region_max_size.0,
            self.cfg.region_split_size.0,
        );
        box_try!(self.split_check_worker.start(split_check_runner));

        let runner = RegionRunner::new(
            self.kv_engine.clone(),
            self.raft_engine.clone(),
            self.snap_mgr.clone(),
            self.cfg.snap_apply_batch_size.0 as usize,
        );
        box_try!(self.region_worker.start(runner));

        let raftlog_gc_runner = RaftlogGcRunner::new(None);
        box_try!(self.raftlog_gc_worker.start(raftlog_gc_runner));

        let compact_runner = CompactRunner::new(self.kv_engine.clone());
        box_try!(self.compact_worker.start(compact_runner));

        let pd_runner = PdRunner::new(
            self.store_id(),
            self.pd_client.clone(),
            self.sendch.clone(),
            self.kv_engine.clone(),
        );
        box_try!(self.pd_worker.start(pd_runner));

        let consistency_check_runner = ConsistencyCheckRunner::new(self.sendch.clone());
        box_try!(
            self.consistency_check_worker
                .start(consistency_check_runner)
        );

        let (tx, rx) = mpsc::channel();
        let apply_runner = ApplyRunner::new(self, tx, self.cfg.sync_log);
        self.apply_res_receiver = Some(rx);
        box_try!(self.apply_worker.start(apply_runner));

        event_loop.run(self)?;
        Ok(())
    }
}
```

主要包括
- 注册一系列定时任务到 event_loop
- 创建各个 Worker 并执行起来
- 调用 EventLoop::run()

### timer
可以进一步看看 Store 的 timer 是怎么注册的，以 register_raft_base_tick() 为例子，其他的也是一样的原理
```rust
fn register_raft_base_tick(&self, event_loop: &mut EventLoop<Self>) {
    // If we register raft base tick failed, the whole raft can't run correctly,
    // TODO: shutdown the store?
    if let Err(e) = register_timer(
        event_loop,
        Tick::Raft,
        self.cfg.raft_base_tick_interval.as_millis(),
    ) {
        error!("{} register raft base tick err: {:?}", self.tag, e);
    };
}
```

可以看到是会直接调用 register_timer() ，同时传递了 Tick::Raft 这个 enum variant 给函数，timeout 则用了参数中的 raft_base_tick_interval

而 register_timer() 实际上就是调用了 EventLoop::timeout_ms() ，经过相应 interval 后，对应的 Tick 消息就会被触发，需要注意这个只会触发一次，下次需要重新注册
```rust
fn register_timer<T: Transport, C: PdClient>(
    event_loop: &mut EventLoop<Store<T, C>>,
    tick: Tick,
    delay: u64,
) -> Result<()> {
    // TODO: now mio TimerError doesn't implement Error trait,
    // so we can't use `try!` directly.
    if delay == 0 {
        // 0 delay means turn off the timer.
        return Ok(());
    }
    if let Err(e) = event_loop.timeout_ms(tick, delay) {
        return Err(box_err!(
            "failed to register timeout [{:?}, delay: {:?}ms]: {:?}",
            tick,
            delay,
            e
        ));
    }
    Ok(())
}
```

而 Tick 有很多类型，定义在 raftstore/store/msg.rs
```rust
#[derive(Debug, Clone, Copy)]
pub enum Tick {
    Raft,
    RaftLogGc,
    SplitRegionCheck,
    CompactCheck,
    PdHeartbeat,
    PdStoreHeartbeat,
    SnapGc,
    CompactLockCf,
    ConsistencyCheck,
}
```

可以看到是和上面注册的时候一一对应的，这些函数的实现其实也是类似的
```rust
self.register_raft_base_tick(event_loop);
self.register_raft_gc_log_tick(event_loop);
self.register_split_region_check_tick(event_loop);
self.register_compact_check_tick(event_loop);
self.register_pd_store_heartbeat_tick(event_loop);
self.register_pd_heartbeat_tick(event_loop);
self.register_snap_mgr_gc_tick(event_loop);
self.register_compact_lock_cf_tick(event_loop);
self.register_consistency_check_tick(event_loop);
```

### handler
而 Store 处理 EventLoop 的消息实际上是通过实现了对应的 Handler trait 来做到的
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

可以看到，定时消息， channel 里的消息等都是在这个 Handler 里处理的
- channel 里的消息通过 Store::notify() 处理
- 定时消息 Tick 会在 Store::timeout() 里处理
- 周期性地调用 Store::tick()

