# TiKV 笔记

## 启动
### 入口
```
bin/tikv-server.rs
-> binutil/server.rs:run_tikv()
    -> run_raft_server()
```

这里产生的就是 `server/server.rs:Server`
```rust
/// The TiKV server
///
/// It hosts various internal components, including gRPC, the raftstore router
/// and a snapshot worker.
pub struct Server<T: RaftStoreRouter + 'static, S: StoreAddrResolver + 'static> {
    env: Arc<Environment>,
    /// A GrpcServer builder or a GrpcServer.
    ///
    /// If the listening port is configured, the server will be started lazily.
    builder_or_server: Option<Either<ServerBuilder, GrpcServer>>,
    local_addr: SocketAddr,
    // Transport.
    trans: ServerTransport<T, S>,
    raft_router: T,
    // For sending/receiving snapshots.
    snap_mgr: SnapManager,
    snap_worker: Worker<SnapTask>,

    // Currently load statistics is done in the thread.
    stats_pool: Option<ThreadPool>,
    thread_load: Arc<ThreadLoad>,
    timer: Handle,
}
```

这里依赖两个 trait `RaftStoreRouter` 和 `StoreAddrResolver`

### run_raft_server()
几个重点函数调用
```rust
fn run_raft_server(pd_client: RpcClient, cfg: &TiKvConfig, security_mgr: Arc<SecurityManager>) {
    // Initialize raftstore channels.
    let (router, system) = fsm::create_raft_batch_system(&cfg.raft_store);

    // Create pd client and pd worker
    let pd_client = Arc::new(pd_client);
    let pd_worker = FutureWorker::new("pd-worker");
    let (mut worker, resolver) = resolve::new_resolver(Arc::clone(&pd_client))
        .unwrap_or_else(|e| fatal!("failed to start address resolver: {}", e));

    // Create kv engine, storage.
    let kv_cfs_opts = cfg.rocksdb.build_cf_opts(&cache);
    let kv_engine = rocks::util::new_engine_opt(db_path.to_str().unwrap(), kv_db_opts, kv_cfs_opts)
        .unwrap_or_else(|s| fatal!("failed to create kv engine: {}", s));

    let engines = Engines::new(Arc::new(kv_engine), Arc::new(raft_engine), cache.is_some());
    let store_meta = Arc::new(Mutex::new(StoreMeta::new(PENDING_VOTES_CAP)));
    let local_reader = LocalReader::new(engines.kv.clone(), store_meta.clone(), router.clone());
    let raft_router = ServerRaftStoreRouter::new(router.clone(), local_reader);

    let engine = RaftKv::new(raft_router.clone());

    let storage = create_raft_storage(
        engine.clone(),
        &cfg.storage,
        &cfg.gc,
        storage_read_pool,
        Some(engines.kv.clone()),
        Some(raft_router.clone()),
        lock_mgr.clone(),
    )
    .unwrap_or_else(|e| fatal!("failed to create raft storage: {}", e));

    let mut server = Server::new(
        &server_cfg,
        &security_mgr,
        storage.clone(),
        cop,
        raft_router,
        resolver.clone(),
        snap_mgr.clone(),
        Some(engines.clone()),
        Some(import_service),
        deadlock_service,
    )
    .unwrap_or_else(|e| fatal!("failed to create server: {}", e));

    // Create node.
    let mut node = Node::new(system, &server_cfg, &cfg.raft_store, pd_client.clone());

    node.start(
        engines.clone(),
        trans,
        snap_mgr,
        pd_worker,
        store_meta,
        coprocessor_host,
        importer,
    )
    .unwrap_or_else(|e| fatal!("failed to start node: {}", e));

    // Run server.
    server
        .start(server_cfg, security_mgr)
        .unwrap_or_else(|e| fatal!("failed to start server: {}", e));

}
```

这里记录里面用到的 trait/struct 及其对应的实现位置
- router
    - RaftRouter
        - `raftstore/store/fsm/store.rs`
        - 内部的 router 为 BatchRouter
            - `components/batch-system/src/batch.rs`
- system
    - RaftBatchSystem
- resolver
    - PdStoreAddrResolver
        - `server/resolver.rs`
- kv_engine
    - rocksdb 的 DB
- local_reader
    - LocalReader
        - `raftstore/store/worker/read.rs`
- raft_router
    - ServerRaftStoreRouter
        - `server/transport.rs`
- engine
    - RaftKv<ServerRaftStoreRouter>
        - trait Engine
            - `storage/kv/mod.rs`
        - `storage/kv/raftkv.rs`
- storage
    - trait Storage
        - `Storage<RaftKv<S>, LockManager>`
        - `storage/mod.rs`
- server
    - Server
        - `server/server.rs`
- node
    - Node
        - `server/node.rs`
- trans
    - ServerTransport
        - `server/transport.rs`


另外，在 `Server::new()` 中还会创建 `KvService` 并启动 grpc 服务
```rust
let kv_service = KvService::new(
    storage,
    cop,
    raft_router.clone(),
    snap_worker.scheduler(),
    Arc::clone(&thread_load),
);

let mut sb = ServerBuilder::new(Arc::clone(&env))
    .channel_args(channel_args)
    .register_service(create_tikv(kv_service));
```

`KvService` 实际上是 `Service` 的 reimport ，定义在 `server/service/kv.rs`
```rust
/// Service handles the RPC messages for the `Tikv` service.
#[derive(Clone)]
pub struct Service<T: RaftStoreRouter + 'static, E: Engine, L: LockMgr> {
    // For handling KV requests.
    storage: Storage<E, L>,
    // For handling coprocessor requests.
    cop: Endpoint<E>,
    // For handling raft messages.
    ch: T,
    // For handling snapshot.
    snap_scheduler: Scheduler<SnapTask>,

    thread_load: Arc<ThreadLoad>,
}
```

而 `Node` 会使用 `Server::trans` 进行 start , `Node` 的结构如下
```rust
/// A wrapper for the raftstore which runs Multi-Raft.
// TODO: we will rename another better name like RaftStore later.
pub struct Node<C: PdClient + 'static> {
    cluster_id: u64,
    store: metapb::Store,
    store_cfg: StoreConfig,
    store_handle: Option<thread::JoinHandle<()>>,
    system: RaftBatchSystem,

    pd_client: Arc<C>,
}
```

## 写入请求处理
先从负责处理 rpc 请求的 `Service` 入手，这里先只关注较简单的写入接口
```rust
impl<T: RaftStoreRouter + 'static, E: Engine, L: LockMgr> tikvpb_grpc::Tikv for Service<T, E, L> {
    fn raw_batch_put(
        &mut self,
        ctx: RpcContext<'_>,
        req: RawBatchPutRequest,
        sink: UnarySink<RawBatchPutResponse>,
    ) {
        let timer = GRPC_MSG_HISTOGRAM_VEC.raw_batch_put.start_coarse_timer();

        let future = future_raw_batch_put(&self.storage, req)
            .and_then(|res| sink.success(res).map_err(Error::from))
            .map(|_| timer.observe_duration())
            .map_err(move |e| {
                debug!("kv rpc failed";
                    "request" => "raw_batch_put",
                    "err" => ?e
                );
                GRPC_MSG_FAIL_COUNTER.raw_batch_put.inc();
            });

        ctx.spawn(future);
    }
}
```

重点是调用了 `future_raw_batch_put()`
```rust
fn future_raw_batch_put<E: Engine, L: LockMgr>(
    storage: &Storage<E, L>,
    mut req: RawBatchPutRequest,
) -> impl Future<Item = RawBatchPutResponse, Error = Error> {
    let cf = req.take_cf();
    let pairs = req
        .take_pairs()
        .into_iter()
        .map(|mut x| (x.take_key(), x.take_value()))
        .collect();

    let (cb, f) = paired_future_callback();
    let res = storage.async_raw_batch_put(req.take_context(), cf, pairs, cb);

    AndThenWith::new(res, f.map_err(Error::from)).map(|v| {
        let mut resp = RawBatchPutResponse::new();
        if let Some(err) = extract_region_error(&v) {
            resp.set_region_error(err);
        } else if let Err(e) = v {
            resp.set_error(format!("{}", e));
        }
        resp
    })
}
```

随后调用 `storage/mod.rs:Storage::async_raw_batch_put()`
```rust
/// Write some keys to the storage in a batch.
pub fn async_raw_batch_put(
    &self,
    ctx: Context,
    cf: String,
    pairs: Vec<KvPair>,
    callback: Callback<()>,
) -> Result<()> {
    let cf = Self::rawkv_cf(&cf)?;
    for &(ref key, _) in &pairs {
        if key.len() > self.max_key_size {
            callback(Err(Error::KeyTooLarge(key.len(), self.max_key_size)));
            return Ok(());
        }
    }
    let requests = pairs
        .into_iter()
        .map(|(k, v)| Modify::Put(cf, Key::from_encoded(k), v))
        .collect();
    self.engine.async_write(
        &ctx,
        requests,
        Box::new(|(_, res): (_, kv::Result<_>)| callback(res.map_err(Error::from))),
    )?;
    KV_COMMAND_COUNTER_VEC_STATIC.raw_batch_put.inc();
    Ok(())
}
```

可以看到会调用 `Engine` 的 `async_write()` ，这个 trait 定义为
```rust
pub trait Engine: Send + Clone + 'static {
    type Snap: Snapshot;

    fn async_write(&self, ctx: &Context, batch: Vec<Modify>, callback: Callback<()>) -> Result<()>;
    fn async_snapshot(&self, ctx: &Context, callback: Callback<Self::Snap>) -> Result<()>;

    fn write(&self, ctx: &Context, batch: Vec<Modify>) -> Result<()> {
        let timeout = Duration::from_secs(DEFAULT_TIMEOUT_SECS);
        match wait_op!(|cb| self.async_write(ctx, batch, cb), timeout) {
            Some((_, res)) => res,
            None => Err(Error::Timeout(timeout)),
        }
    }

    fn snapshot(&self, ctx: &Context) -> Result<Self::Snap> {
        let timeout = Duration::from_secs(DEFAULT_TIMEOUT_SECS);
        match wait_op!(|cb| self.async_snapshot(ctx, cb), timeout) {
            Some((_, res)) => res,
            None => Err(Error::Timeout(timeout)),
        }
    }

    fn put(&self, ctx: &Context, key: Key, value: Value) -> Result<()> {
        self.put_cf(ctx, CF_DEFAULT, key, value)
    }

    fn put_cf(&self, ctx: &Context, cf: CfName, key: Key, value: Value) -> Result<()> {
        self.write(ctx, vec![Modify::Put(cf, key, value)])
    }

    fn delete(&self, ctx: &Context, key: Key) -> Result<()> {
        self.delete_cf(ctx, CF_DEFAULT, key)
    }

    fn delete_cf(&self, ctx: &Context, cf: CfName, key: Key) -> Result<()> {
        self.write(ctx, vec![Modify::Delete(cf, key)])
    }
}
```

而在启动的时候，已经设置了其实现为 `RaftKv` ，因此会调用 `RaftKv::async_write()`
```rust
fn async_write(
    &self,
    ctx: &Context,
    modifies: Vec<Modify>,
    cb: Callback<()>,
) -> kv::Result<()> {
    fail_point!("raftkv_async_write");
    if modifies.is_empty() {
        return Err(kv::Error::EmptyRequest);
    }

    let mut reqs = Vec::with_capacity(modifies.len());
    for m in modifies {
        let mut req = Request::new();
        match m {
            Modify::Delete(cf, k) => {
                let mut delete = DeleteRequest::new();
                delete.set_key(k.into_encoded());
                if cf != CF_DEFAULT {
                    delete.set_cf(cf.to_string());
                }
                req.set_cmd_type(CmdType::Delete);
                req.set_delete(delete);
            }
            Modify::Put(cf, k, v) => {
                let mut put = PutRequest::new();
                put.set_key(k.into_encoded());
                put.set_value(v);
                if cf != CF_DEFAULT {
                    put.set_cf(cf.to_string());
                }
                req.set_cmd_type(CmdType::Put);
                req.set_put(put);
            }
            Modify::DeleteRange(cf, start_key, end_key, notify_only) => {
                let mut delete_range = DeleteRangeRequest::new();
                delete_range.set_cf(cf.to_string());
                delete_range.set_start_key(start_key.into_encoded());
                delete_range.set_end_key(end_key.into_encoded());
                delete_range.set_notify_only(notify_only);
                req.set_cmd_type(CmdType::DeleteRange);
                req.set_delete_range(delete_range);
            }
        }
        reqs.push(req);
    }

    ASYNC_REQUESTS_COUNTER_VEC.write.all.inc();
    let req_timer = ASYNC_REQUESTS_DURATIONS_VEC.write.start_coarse_timer();

    self.exec_write_requests(
        ctx,
        reqs,
        Box::new(move |(cb_ctx, res)| match res {
            Ok(CmdRes::Resp(_)) => {
                req_timer.observe_duration();
                ASYNC_REQUESTS_COUNTER_VEC.write.success.inc();
                fail_point!("raftkv_async_write_finish");
                cb((cb_ctx, Ok(())))
            }
            Ok(CmdRes::Snap(_)) => cb((
                cb_ctx,
                Err(box_err!("unexpect snapshot, should mutate instead.")),
            )),
            Err(e) => {
                let status_kind = get_status_kind_from_engine_error(&e);
                ASYNC_REQUESTS_COUNTER_VEC.write.get(status_kind).inc();
                cb((cb_ctx, Err(e)))
            }
        }),
    )
    .map_err(|e| {
        let status_kind = get_status_kind_from_error(&e);
        ASYNC_REQUESTS_COUNTER_VEC.write.get(status_kind).inc();
        e.into()
    })
}
```

这里会构造请求，然后调用 `RaftKv::exec_write_requests()` ，这个函数做的操作也十分简单，就是调用 `ServerRaftStoreRouter::send_command()` 将 `RaftCmdRequest` 发送出去
```rust
fn exec_write_requests(
    &self,
    ctx: &Context,
    reqs: Vec<Request>,
    cb: Callback<CmdRes>,
) -> Result<()> {
    fail_point!("raftkv_early_error_report", |_| Err(
        RaftServerError::RegionNotFound(ctx.get_region_id()).into()
    ));
    let len = reqs.len();
    let header = self.new_request_header(ctx);
    let mut cmd = RaftCmdRequest::new();
    cmd.set_header(header);
    cmd.set_requests(RepeatedField::from_vec(reqs));

    self.router
        .send_command(
            cmd,
            StoreCallback::Write(Box::new(move |resp| {
                let (cb_ctx, res) = on_write_result(resp, len);
                cb((cb_ctx, res.map_err(Error::into)));
            })),
        )
        .map_err(From::from)
}
```

`ServerRaftStoreRouter` 定义如下
```rust
/// A router that routes messages to the raftstore
#[derive(Clone)]
pub struct ServerRaftStoreRouter {
    router: RaftRouter,
    local_reader: LocalReader<RaftRouter>,
}
```

`ServerRaftStoreRouter::send_command()` 实现如下，实际上会交由 `RaftRouter` 转发
```rust
fn send_command(&self, req: RaftCmdRequest, cb: Callback) -> RaftStoreResult<()> {
    let cmd = RaftCommand::new(req, cb);
    if LocalReader::<RaftRouter>::acceptable(&cmd.request) {
        self.local_reader.execute_raft_command(cmd);
        Ok(())
    } else {
        let region_id = cmd.request.get_header().get_region_id();
        self.router
            .send_raft_command(cmd)
            .map_err(|e| handle_error(region_id, e))
    }
}
```

`RaftRouter` 定义如下
```rust
#[derive(Clone)]
pub struct RaftRouter {
    pub router: BatchRouter<PeerFsm, StoreFsm>,
}
```

而 `RaftRouter::send_raft_command()` 会调用 `BatchRouter<PeerFsm, StoreFsm>::send()`
```rust
#[inline]
pub fn send_raft_command(
    &self,
    cmd: RaftCommand,
) -> std::result::Result<(), TrySendError<RaftCommand>> {
    let region_id = cmd.request.get_header().get_region_id();
    match self.send(region_id, PeerMsg::RaftCommand(cmd)) {
        Ok(()) => Ok(()),
        Err(TrySendError::Full(PeerMsg::RaftCommand(cmd))) => Err(TrySendError::Full(cmd)),
        Err(TrySendError::Disconnected(PeerMsg::RaftCommand(cmd))) => {
            Err(TrySendError::Disconnected(cmd))
        }
        _ => unreachable!(),
    }
}
```

`BatchRouter` 是 `components/batch-system/src/router.rs::Router` 的 typedef
```rust
pub type BatchRouter<N, C> = Router<N, C, NormalScheduler<N, C>, ControlScheduler<N, C>>;
```

而 `Router` 定义为，实际上就是负责将 message 转发到 mailbox
```rust
/// Router route messages to its target mailbox.
///
/// Every fsm has a mailbox, hence it's necessary to have an address book
/// that can deliver messages to specified fsm, which is exact router.
///
/// In our abstract model, every batch system has two different kind of
/// fsms. First is normal fsm, which does the common work like peers in a
/// raftstore model or apply delegate in apply model. Second is control fsm,
/// which does some work that requires a global view of resources or creates
/// missing fsm for specified address. Normal fsm and control fsm can have
/// different scheduler, but this is not required.
pub struct Router<N: Fsm, C: Fsm, Ns, Cs> {
    normals: Arc<Mutex<HashMap<u64, BasicMailbox<N>>>>,
    caches: Cell<HashMap<u64, BasicMailbox<N>>>,
    pub(super) control_box: BasicMailbox<C>,
    // TODO: These two schedulers should be unified as single one. However
    // it's not possible to write FsmScheduler<Fsm=C> + FsmScheduler<Fsm=N>
    // for now.
    pub(crate) normal_scheduler: Ns,
    control_scheduler: Cs,
}
```

`Router::send` 实现如下
```rust
/// Send the message to specified address.
#[inline]
pub fn send(&self, addr: u64, msg: N::Message) -> Result<(), TrySendError<N::Message>> {
    match self.try_send(addr, msg) {
        Either::Left(res) => res,
        Either::Right(m) => Err(TrySendError::Disconnected(m)),
    }
}

/// Try to send a message to specified address.
///
/// If Either::Left is returned, then the message is sent. Otherwise,
/// it indicates mailbox is not found.
#[inline]
pub fn try_send(
    &self,
    addr: u64,
    msg: N::Message,
) -> Either<Result<(), TrySendError<N::Message>>, N::Message> {
    let mut msg = Some(msg);
    let res = self.check_do(addr, |mailbox| {
        let m = msg.take().unwrap();
        match mailbox.try_send(m, &self.normal_scheduler) {
            Ok(()) => Some(Ok(())),
            r @ Err(TrySendError::Full(_)) => {
                // TODO: report channel full
                Some(r)
            }
            Err(TrySendError::Disconnected(m)) => {
                msg = Some(m);
                None
            }
        }
    });
    match res {
        CheckDoResult::Valid(r) => Either::Left(r),
        CheckDoResult::Invalid => Either::Left(Err(TrySendError::Disconnected(msg.unwrap()))),
        CheckDoResult::NotExist => Either::Right(msg.unwrap()),
    }
}
```

随后调用 `components/batch-system/src/mailbox.rs:BasicMailbox::try_send()`
```rust
/// A basic mailbox.
///
/// Every mailbox should have one and only one owner, who will receive all
/// messages sent to this mailbox.
///
/// When a message is sent to a mailbox, its owner will be checked whether it's
/// idle. An idle owner will be scheduled via `FsmScheduler` immediately, which
/// will drive the fsm to poll for messages.
pub struct BasicMailbox<Owner: Fsm> {
    sender: mpsc::LooseBoundedSender<Owner::Message>,
    state: Arc<FsmState<Owner>>,
}

/// Try to send a message to the mailbox.
///
/// If there are too many pending messages, function may fail.
#[inline]
pub fn try_send<S: FsmScheduler<Fsm = Owner>>(
    &self,
    msg: Owner::Message,
    scheduler: &S,
) -> Result<(), TrySendError<Owner::Message>> {
    self.sender.try_send(msg)?;
    self.state.notify(scheduler, Cow::Borrowed(self));
    Ok(())
}
```

这里在往 `sender` 发送数据之后，会调用 `state` 的 `notify()` 函数。其中 `sender` 发送的数据最终会由 `Fsm` 接收
```rust
/// Notify fsm via a `FsmScheduler`.
#[inline]
pub fn notify<S: FsmScheduler<Fsm = N>>(
    &self,
    scheduler: &S,
    mailbox: Cow<'_, BasicMailbox<N>>,
) {
    match self.take_fsm() {
        None => {}
        Some(mut n) => {
            n.set_mailbox(mailbox);
            scheduler.schedule(n);
        }
    }
}
```

然后会调度 fsm 的执行
```rust
/// `FsmScheduler` schedules `Fsm` for later handles.
pub trait FsmScheduler {
    type Fsm: Fsm;

    /// Schedule a Fsm for later handles.
    fn schedule(&self, fsm: Box<Self::Fsm>);
    /// Shutdown the scheduler, which indicates that resources like
    /// background thread pool should be released.
    fn shutdown(&self);
}
```

而这里具体的 `FsmScheduler` 就是 `NormalScheduler` 了

## BatchSystem
前面提到的 `create_raft_batch_system()` 会调用 `batch_system::create_system()` 来创建 BatchSystem
```rust
pub fn create_raft_batch_system(cfg: &Config) -> (RaftRouter, RaftBatchSystem) {
    let (store_tx, store_fsm) = StoreFsm::new(cfg);
    let (apply_router, apply_system) = create_apply_batch_system(&cfg);
    let (router, system) = batch_system::create_system(
        cfg.store_pool_size,
        cfg.store_max_batch_size,
        store_tx,
        store_fsm,
    );
    let raft_router = RaftRouter { router };
    let system = RaftBatchSystem {
        system,
        workers: None,
        apply_router,
        apply_system,
        router: raft_router.clone(),
    };
    (raft_router, system)
}
```

这里就会负责创建对应的 `NormalScheduler` ，而 `NormalScheduler` 最终会将被 notify 的 `Fsm` 发送到 `BatchSystem::receiver` 中
```rust
/// Create a batch system with the given thread name prefix and pool size.
///
/// `sender` and `controller` should be paired.
pub fn create_system<N: Fsm, C: Fsm>(
    pool_size: usize,
    max_batch_size: usize,
    sender: mpsc::LooseBoundedSender<C::Message>,
    controller: Box<C>,
) -> (BatchRouter<N, C>, BatchSystem<N, C>) {
    let control_box = BasicMailbox::new(sender, controller);
    let (tx, rx) = channel::unbounded();
    let normal_scheduler = NormalScheduler { sender: tx.clone() };
    let control_scheduler = ControlScheduler { sender: tx };
    let router = Router::new(control_box, normal_scheduler, control_scheduler);
    let system = BatchSystem {
        name_prefix: None,
        router: router.clone(),
        receiver: rx,
        pool_size,
        max_batch_size,
        workers: vec![],
    };
    (router, system)
}
```

而 `create_raft_batch_system()` 创建的 `RaftBatchSystem` 会用来创建 `Node`
```rust
/// A wrapper for the raftstore which runs Multi-Raft.
// TODO: we will rename another better name like RaftStore later.
pub struct Node<C: PdClient + 'static> {
    cluster_id: u64,
    store: metapb::Store,
    store_cfg: StoreConfig,
    store_handle: Option<thread::JoinHandle<()>>,
    system: RaftBatchSystem,

    pd_client: Arc<C>,
}
```

`Node` 在 start 时会调用 `Node::start_store()`
```rust
/// Starts the Node. It tries to bootstrap cluster if the cluster is not
/// bootstrapped yet. Then it spawns a thread to run the raftstore in
/// background.
#[allow(clippy::too_many_arguments)]
pub fn start<T>(
    &mut self,
    engines: Engines,
    trans: T,
    snap_mgr: SnapManager,
    pd_worker: FutureWorker<PdTask>,
    store_meta: Arc<Mutex<StoreMeta>>,
    coprocessor_host: CoprocessorHost,
    importer: Arc<SSTImporter>,
) -> Result<()>
where
    T: Transport + 'static,
{
    let mut store_id = self.check_store(&engines)?;
    if store_id == INVALID_ID {
        store_id = self.bootstrap_store(&engines)?;
        fail_point!("node_after_bootstrap_store", |_| Err(box_err!(
            "injected error: node_after_bootstrap_store"
        )));
    }
    self.store.set_id(store_id);
    {
        let mut meta = store_meta.lock().unwrap();
        meta.store_id = Some(store_id);
    }
    if let Some(first_region) = self.check_or_prepare_bootstrap_cluster(&engines, store_id)? {
        info!("try bootstrap cluster"; "store_id" => store_id, "region" => ?first_region);
        // cluster is not bootstrapped, and we choose first store to bootstrap
        fail_point!("node_after_prepare_bootstrap_cluster", |_| Err(box_err!(
            "injected error: node_after_prepare_bootstrap_cluster"
        )));
        self.bootstrap_cluster(&engines, first_region)?;
    }

    self.start_store(
        store_id,
        engines,
        trans,
        snap_mgr,
        pd_worker,
        store_meta,
        coprocessor_host,
        importer,
    )?;

    // Put store only if the cluster is bootstrapped.
    info!("put store to PD"; "store" => ?&self.store);
    self.pd_client.put_store(self.store.clone())?;

    Ok(())
}
```

随后调用 `RaftBatchSystem::spawn()`
```rust
#[allow(clippy::too_many_arguments)]
fn start_store<T>(
    &mut self,
    store_id: u64,
    engines: Engines,
    trans: T,
    snap_mgr: SnapManager,
    pd_worker: FutureWorker<PdTask>,
    store_meta: Arc<Mutex<StoreMeta>>,
    coprocessor_host: CoprocessorHost,
    importer: Arc<SSTImporter>,
) -> Result<()>
where
    T: Transport + 'static,
{
    info!("start raft store thread"; "store_id" => store_id);

    if self.store_handle.is_some() {
        return Err(box_err!("{} is already started", store_id));
    }

    let cfg = self.store_cfg.clone();
    let pd_client = Arc::clone(&self.pd_client);
    let store = self.store.clone();
    self.system.spawn(
        store,
        cfg,
        engines,
        trans,
        pd_client,
        snap_mgr,
        pd_worker,
        store_meta,
        coprocessor_host,
        importer,
    )?;
    Ok(())
}
```

`RaftBatchSystem::spawn()` 会调用 `raftstore/store/fsm/store.rs:RaftPollerBuilder::init()` 和 `RaftBatchSystem::start_system()` ，这两个函数就是启动 raftstore 的核心
```rust
pub fn spawn<T: Transport + 'static, C: PdClient + 'static>(
    &mut self,
    meta: metapb::Store,
    mut cfg: Config,
    engines: Engines,
    trans: T,
    pd_client: Arc<C>,
    mgr: SnapManager,
    pd_worker: FutureWorker<PdTask>,
    store_meta: Arc<Mutex<StoreMeta>>,
    mut coprocessor_host: CoprocessorHost,
    importer: Arc<SSTImporter>,
) -> Result<()> {
    assert!(self.workers.is_none());
    // TODO: we can get cluster meta regularly too later.
    cfg.validate()?;

    // TODO load coprocessors from configuration
    coprocessor_host
        .registry
        .register_admin_observer(100, Box::new(SplitObserver));

    let workers = Workers {
        split_check_worker: Worker::new("split-check"),
        region_worker: Worker::new("snapshot-worker"),
        raftlog_gc_worker: Worker::new("raft-gc-worker"),
        compact_worker: Worker::new("compact-worker"),
        pd_worker,
        consistency_check_worker: Worker::new("consistency-check"),
        cleanup_sst_worker: Worker::new("cleanup-sst"),
        coprocessor_host: Arc::new(coprocessor_host),
        future_poller: tokio_threadpool::Builder::new()
            .name_prefix("future-poller")
            .pool_size(cfg.future_poll_size)
            .build(),
    };
    let mut builder = RaftPollerBuilder {
        cfg: Arc::new(cfg),
        store: meta,
        engines,
        router: self.router.clone(),
        split_check_scheduler: workers.split_check_worker.scheduler(),
        region_scheduler: workers.region_worker.scheduler(),
        raftlog_gc_scheduler: workers.raftlog_gc_worker.scheduler(),
        compact_scheduler: workers.compact_worker.scheduler(),
        pd_scheduler: workers.pd_worker.scheduler(),
        consistency_check_scheduler: workers.consistency_check_worker.scheduler(),
        cleanup_sst_scheduler: workers.cleanup_sst_worker.scheduler(),
        apply_router: self.apply_router.clone(),
        trans,
        pd_client,
        coprocessor_host: workers.coprocessor_host.clone(),
        importer,
        snap_mgr: mgr,
        global_stat: GlobalStoreStat::default(),
        store_meta,
        applying_snap_count: Arc::new(AtomicUsize::new(0)),
        future_poller: workers.future_poller.sender().clone(),
    };
    let region_peers = builder.init()?;
    self.start_system(workers, region_peers, builder)?;
    Ok(())
}
```
`RaftPollerBuilder::init()` 会从 `Engines::kv` 中恢复各个 region 的状态

这里我们重点关注 `RaftBatchSystem::start_system()` ，这里会
- 准备好各个 region 的 address ， mailboxes 和 router
- 启动各个 runner
- spawn `RaftBatchSystem::system`
- spawn `RaftBatchSystem::apply_system`

```rust
fn start_system<T: Transport + 'static, C: PdClient + 'static>(
    &mut self,
    mut workers: Workers,
    region_peers: Vec<(LooseBoundedSender<PeerMsg>, Box<PeerFsm>)>,
    builder: RaftPollerBuilder<T, C>,
) -> Result<()> {
    builder.snap_mgr.init()?;

    let engines = builder.engines.clone();
    let snap_mgr = builder.snap_mgr.clone();
    let cfg = builder.cfg.clone();
    let store = builder.store.clone();
    let pd_client = builder.pd_client.clone();
    let importer = builder.importer.clone();

    let apply_poller_builder = ApplyPollerBuilder::new(
        &builder,
        ApplyNotifier::Router(self.router.clone()),
        self.apply_router.clone(),
    );
    self.apply_system
        .schedule_all(region_peers.iter().map(|pair| pair.1.get_peer()));

    {
        let mut meta = builder.store_meta.lock().unwrap();
        for (_, peer_fsm) in &region_peers {
            let peer = peer_fsm.get_peer();
            meta.readers
                .insert(peer_fsm.region_id(), ReadDelegate::from_peer(peer));
        }
    }

    let router = Mutex::new(self.router.clone());
    pd_client.handle_reconnect(move || {
        router
            .lock()
            .unwrap()
            .broadcast_normal(|| PeerMsg::HeartbeatPd);
    });

    let tag = format!("raftstore-{}", store.get_id());
    self.system.spawn(tag, builder);
    let mut mailboxes = Vec::with_capacity(region_peers.len());
    let mut address = Vec::with_capacity(region_peers.len());
    for (tx, fsm) in region_peers {
        address.push(fsm.region_id());
        mailboxes.push((fsm.region_id(), BasicMailbox::new(tx, fsm)));
    }
    self.router.register_all(mailboxes);

    // Make sure Msg::Start is the first message each FSM received.
    for addr in address {
        self.router.force_send(addr, PeerMsg::Start).unwrap();
    }
    self.router
        .send_control(StoreMsg::Start {
            store: store.clone(),
        })
        .unwrap();

    self.apply_system
        .spawn("apply".to_owned(), apply_poller_builder);

    let split_check_runner = SplitCheckRunner::new(
        Arc::clone(&engines.kv),
        self.router.clone(),
        Arc::clone(&workers.coprocessor_host),
    );
    box_try!(workers.split_check_worker.start(split_check_runner));

    let region_runner = RegionRunner::new(
        engines.clone(),
        snap_mgr,
        cfg.snap_apply_batch_size.0 as usize,
        cfg.use_delete_range,
        cfg.clean_stale_peer_delay.0,
        self.router(),
    );
    let timer = region_runner.new_timer();
    box_try!(workers.region_worker.start_with_timer(region_runner, timer));

    let raftlog_gc_runner = RaftlogGcRunner::new(None);
    box_try!(workers.raftlog_gc_worker.start(raftlog_gc_runner));

    let compact_runner = CompactRunner::new(Arc::clone(&engines.kv));
    box_try!(workers.compact_worker.start(compact_runner));

    let pd_runner = PdRunner::new(
        store.get_id(),
        Arc::clone(&pd_client),
        self.router.clone(),
        Arc::clone(&engines.kv),
        workers.pd_worker.scheduler(),
    );
    box_try!(workers.pd_worker.start(pd_runner));

    let consistency_check_runner = ConsistencyCheckRunner::new(self.router.clone());
    box_try!(workers
        .consistency_check_worker
        .start(consistency_check_runner));

    let cleanup_sst_runner = CleanupSSTRunner::new(
        store.get_id(),
        self.router.clone(),
        Arc::clone(&importer),
        Arc::clone(&pd_client),
    );
    box_try!(workers.cleanup_sst_worker.start(cleanup_sst_runner));

    if let Err(e) = sys_util::thread::set_priority(sys_util::HIGH_PRI) {
        warn!("set thread priority for raftstore failed"; "error" => ?e);
    }
    self.workers = Some(workers);
    Ok(())
}
```

`RaftBatchSystem` 实际上包含了两个 `BatchSystem`
```rust
pub struct RaftBatchSystem {
    system: BatchSystem<PeerFsm, StoreFsm>,
    apply_router: ApplyRouter,
    apply_system: ApplyBatchSystem,
    router: RaftRouter,
    workers: Option<Workers>,
}
```

而 `raftstore/store/fsm/apply.rs:ApplyBatchSystem` 实际上也是个 `BatchSystem` ，是在 `create_raft_batch_system()` 里通过 `create_apply_batch_system()` 创建的
```rust
pub struct ApplyBatchSystem {
    system: BatchSystem<ApplyFsm, ControlFsm>,
}
```

所以我们只需要关注下 `BatchSystem::spawn` 即可
```rust
/// Start the batch system.
pub fn spawn<B>(&mut self, name_prefix: String, mut builder: B)
where
    B: HandlerBuilder<N, C>,
    B::Handler: Send + 'static,
{
    for i in 0..self.pool_size {
        let handler = builder.build();
        let mut poller = Poller {
            router: self.router.clone(),
            fsm_receiver: self.receiver.clone(),
            handler,
            max_batch_size: self.max_batch_size,
        };
        let t = thread::Builder::new()
            .name(thd_name!(format!("{}-{}", name_prefix, i)))
            .spawn(move || poller.poll())
            .unwrap();
        self.workers.push(t);
    }
    self.name_prefix = Some(name_prefix);
}
```

这里会创建若干个线程来调用 `components/batch-system/src/batch.rs:Poller::poll()`
- 每个线程都有一个对应的 handler
- 对于 `RaftBatchSystem::system`
    - 其 `HandlerBuilder` 为 `RaftPollerBuilder`
    - 而 `Handler` 为 `RaftPoller`
    - 均位于 `raftstore/store/fsm/store.rs`
- 对于 `RaftBatchSystem::apply_system`
    - 其 `HandlerBuilder` 为 `ApplyPollerBuilder` ，实际上就是 `raftstore/store/fsm/apply.rs:Builder`
    - 而 `Handler` 为 `ApplyPoller`
    - 均位于 `raftstore/store/fsm/apply.rs`

`Poller` 结构如下，其中 `fsm_receiver` 会接收通过 `FsmScheduler::schedule()` 发送的消息
```rust
/// Internal poller that fetches batch and call handler hooks for readiness.
struct Poller<N: Fsm, C: Fsm, Handler> {
    router: Router<N, C, NormalScheduler<N, C>, ControlScheduler<N, C>>,
    fsm_receiver: channel::Receiver<FsmTypes<N, C>>,
    handler: Handler,
    max_batch_size: usize,
}
```

`Poller::poll()` 实现如下
```rust
// Poll for readiness and forward to handler. Remove stale peer if necessary.
fn poll(&mut self) {
    let mut batch = Batch::with_capacity(self.max_batch_size);
    let mut reschedule_fsms = Vec::with_capacity(self.max_batch_size);

    self.fetch_batch(&mut batch, self.max_batch_size);
    while !batch.is_empty() {
        let mut hot_fsm_count = 0;
        self.handler.begin(batch.len());
        if batch.control.is_some() {
            let len = self.handler.handle_control(batch.control.as_mut().unwrap());
            if batch.control.as_ref().unwrap().is_stopped() {
                batch.remove_control(&self.router.control_box);
            } else if let Some(len) = len {
                batch.release_control(&self.router.control_box, len);
            }
        }
        if !batch.normals.is_empty() {
            for (i, p) in batch.normals.iter_mut().enumerate() {
                let len = self.handler.handle_normal(p);
                batch.counters[i] += 1;
                if p.is_stopped() {
                    reschedule_fsms.push((i, ReschedulePolicy::Remove));
                } else {
                    if batch.counters[i] > 3 {
                        hot_fsm_count += 1;
                        // We should only reschedule a half of the hot regions, otherwise,
                        // it's possible all the hot regions are fetched in a batch the
                        // next time.
                        if hot_fsm_count % 2 == 0 {
                            reschedule_fsms.push((i, ReschedulePolicy::Schedule));
                            continue;
                        }
                    }
                    if let Some(l) = len {
                        reschedule_fsms.push((i, ReschedulePolicy::Release(l)));
                    }
                }
            }
        }
        self.handler.end(batch.normals_mut());
        // Because release use `swap_remove` internally, so using pop here
        // to remove the correct FSM.
        while let Some((r, mark)) = reschedule_fsms.pop() {
            match mark {
                ReschedulePolicy::Release(l) => batch.release(r, l),
                ReschedulePolicy::Remove => batch.remove(r),
                ReschedulePolicy::Schedule => batch.reschedule(&self.router, r),
            }
        }
        // Fetch batch after every round is finished. It's helpful to protect regions
        // from becoming hungry if some regions are hot points.
        self.fetch_batch(&mut batch, self.max_batch_size);
    }
}
```
基本流程为
- `Poller::fetch_batch()`
- 调用 `PollHandler::begin()`
- 调用 `PollHandler::handle_control()`
- 调用 `PollHandler::handle_normal()`
- 调用 `PollHandler::end()`

`PollHandler` 的定义如下
```rust
/// A handler that poll all FSM in ready.
///
/// A General process works like following:
/// ```text
/// loop {
///     begin
///     if control is ready:
///         handle_control
///     foreach ready normal:
///         handle_normal
///     end
/// }
/// ```
///
/// Note that, every poll thread has its own handler, which doesn't have to be
/// Sync.
pub trait PollHandler<N, C> {
    /// This function is called at the very beginning of every round.
    fn begin(&mut self, batch_size: usize);

    /// This function is called when handling readiness for control FSM.
    ///
    /// If returned value is Some, then it represents a length of channel. This
    /// function will only be called for the same fsm after channel's lengh is
    /// larger than the value. If it returns None, then this function will
    /// still be called for the same FSM in the next loop unless the FSM is
    /// stopped.
    fn handle_control(&mut self, control: &mut C) -> Option<usize>;

    /// This function is called when handling readiness for normal FSM.
    ///
    /// The returned value is handled in the same way as `handle_control`.
    fn handle_normal(&mut self, normal: &mut N) -> Option<usize>;

    /// This function is called at the end of every round.
    fn end(&mut self, batch: &mut [Box<N>]);

    /// This function is called when batch system is going to sleep.
    fn pause(&mut self) {}
}
```

### RaftPoller
重点关注 `RaftPoller::handle_control()` 和 `RaftPoller::handle_normal()`
```rust
impl<T: Transport, C: PdClient> PollHandler<PeerFsm, StoreFsm> for RaftPoller<T, C> {
    fn handle_control(&mut self, store: &mut StoreFsm) -> Option<usize> {
        let mut expected_msg_count = None;
        while self.store_msg_buf.len() < self.messages_per_tick {
            match store.receiver.try_recv() {
                Ok(msg) => self.store_msg_buf.push(msg),
                Err(TryRecvError::Empty) => {
                    expected_msg_count = Some(0);
                    break;
                }
                Err(TryRecvError::Disconnected) => {
                    store.store.stopped = true;
                    expected_msg_count = Some(0);
                    break;
                }
            }
        }
        let mut delegate = StoreFsmDelegate {
            fsm: store,
            ctx: &mut self.poll_ctx,
        };
        delegate.handle_msgs(&mut self.store_msg_buf);
        expected_msg_count
    }

    fn handle_normal(&mut self, peer: &mut PeerFsm) -> Option<usize> {
        let mut expected_msg_count = None;

        fail_point!(
            "pause_on_peer_collect_message",
            peer.peer_id() == 1,
            |_| unreachable!()
        );

        while self.peer_msg_buf.len() < self.messages_per_tick {
            match peer.receiver.try_recv() {
                // TODO: we may need a way to optimize the message copy.
                Ok(msg) => {
                    fail_point!(
                        "pause_on_peer_destroy_res",
                        peer.peer_id() == 1
                            && match msg {
                                PeerMsg::ApplyRes {
                                    res: ApplyTaskRes::Destroy { .. },
                                } => true,
                                _ => false,
                            },
                        |_| unreachable!()
                    );
                    self.peer_msg_buf.push(msg)
                }
                Err(TryRecvError::Empty) => {
                    expected_msg_count = Some(0);
                    break;
                }
                Err(TryRecvError::Disconnected) => {
                    peer.stop();
                    expected_msg_count = Some(0);
                    break;
                }
            }
        }
        let mut delegate = PeerFsmDelegate::new(peer, &mut self.poll_ctx);
        delegate.handle_msgs(&mut self.peer_msg_buf);
        delegate.collect_ready(&mut self.pending_proposals);
        expected_msg_count
    }
}
```
可以看到，具体的处理会分别交给 `StoreFsmDelegate` 和 `PeerFsmDelegate`

### ApplyPoller
重点关注 `ApplyPoller::handle_normal()`
```rust
impl PollHandler<ApplyFsm, ControlFsm> for ApplyPoller {
    /// There is no control fsm in apply poller.
    fn handle_control(&mut self, _: &mut ControlFsm) -> Option<usize> {
        unimplemented!()
    }

    fn handle_normal(&mut self, normal: &mut ApplyFsm) -> Option<usize> {
        let mut expected_msg_count = None;
        normal.delegate.written = false;
        if normal.delegate.yield_state.is_some() {
            if normal.delegate.wait_merge_state.is_some() {
                // We need to query the length first, otherwise there is a race
                // condition that new messages are queued after resuming and before
                // query the length.
                expected_msg_count = Some(normal.receiver.len());
            }
            if !normal.resume_pending(&mut self.apply_ctx) {
                return expected_msg_count;
            }
            expected_msg_count = None;
        }
        while self.msg_buf.len() < self.messages_per_tick {
            match normal.receiver.try_recv() {
                Ok(msg) => self.msg_buf.push(msg),
                Err(TryRecvError::Empty) => {
                    expected_msg_count = Some(0);
                    break;
                }
                Err(TryRecvError::Disconnected) => {
                    normal.delegate.stopped = true;
                    expected_msg_count = Some(0);
                    break;
                }
            }
        }
        normal.handle_tasks(&mut self.apply_ctx, &mut self.msg_buf);
        if normal.delegate.wait_merge_state.is_some() {
            // Check it again immediately as catching up logs can be very fast.
            expected_msg_count = Some(0);
        } else if normal.delegate.yield_state.is_some() {
            // Let it continue to run next time.
            expected_msg_count = None;
        }
        expected_msg_count
    }
}
```
可以看到，具体的处理交给 `ApplyFsm::handle_tasks()`

### StoreFsm 和 StoreFsmDelegate
主要实现在 `raftstore/store/fsm/store.rs`

`StoreFsm` 会不断接收 `StoreMsg`
```rust
struct Store {
    // store id, before start the id is 0.
    id: u64,
    last_compact_checked_key: Key,
    stopped: bool,
    start_time: Option<Timespec>,
    consistency_check_time: HashMap<u64, Instant>,
    last_unreachable_report: HashMap<u64, Instant>,
}

pub struct StoreFsm {
    store: Store,
    receiver: Receiver<StoreMsg>,
}
```

`StoreMsg` 如下
```rust
pub enum StoreMsg {
    RaftMessage(RaftMessage),
    // For snapshot stats.
    SnapshotStats,

    ValidateSSTResult {
        invalid_ssts: Vec<SSTMeta>,
    },

    // Clear region size and keys for all regions in the range, so we can force them to re-calculate
    // their size later.
    ClearRegionSizeInRange {
        start_key: Vec<u8>,
        end_key: Vec<u8>,
    },
    StoreUnreachable {
        store_id: u64,
    },

    // Compaction finished event
    CompactedEvent(CompactedEvent),
    Tick(StoreTick),
    Start {
        store: metapb::Store,
    },
}
```

`StoreFsmDelegate` 定义如下
```rust
struct StoreFsmDelegate<'a, T: 'static, C: 'static> {
    fsm: &'a mut StoreFsm,
    ctx: &'a mut PollContext<T, C>,
}
```

处理的入口为 `StoreFsmDelegate::handle_msgs`
```rust
fn handle_msgs(&mut self, msgs: &mut Vec<StoreMsg>) {
    for m in msgs.drain(..) {
        match m {
            StoreMsg::Tick(tick) => self.on_tick(tick),
            StoreMsg::RaftMessage(msg) => {
                if let Err(e) = self.on_raft_message(msg) {
                    error!(
                        "handle raft message failed";
                        "store_id" => self.fsm.store.id,
                        "err" => ?e
                    );
                }
            }
            StoreMsg::CompactedEvent(event) => self.on_compaction_finished(event),
            StoreMsg::ValidateSSTResult { invalid_ssts } => {
                self.on_validate_sst_result(invalid_ssts)
            }
            StoreMsg::ClearRegionSizeInRange { start_key, end_key } => {
                self.clear_region_size_in_range(&start_key, &end_key)
            }
            StoreMsg::SnapshotStats => self.store_heartbeat_pd(),
            StoreMsg::StoreUnreachable { store_id } => {
                self.on_store_unreachable(store_id);
            }
            StoreMsg::Start { store } => self.start(store),
        }
    }
}
```

### PeerFsm 和 PeerFsmDelegate
主要实现在 `raftstore/store/fsm/peer.rs`

`PeerFsm` 定义如下
```rust
pub struct PeerFsm {
    peer: Peer,
    /// A registry for all scheduled ticks. This can avoid scheduling ticks twice accidentally.
    tick_registry: PeerTicks,
    /// Ticks for speed up campaign in chaos state.
    ///
    /// Followers will keep ticking in Idle mode to measure how many ticks have been skipped.
    /// Once it becomes chaos, those skipped ticks will be ticked so that it can campaign
    /// quickly instead of waiting an election timeout.
    ///
    /// This will be reset to 0 once it receives any messages from leader.
    missing_ticks: usize,
    group_state: GroupState,
    stopped: bool,
    has_ready: bool,
    mailbox: Option<BasicMailbox<PeerFsm>>,
    pub receiver: Receiver<PeerMsg>,
}
```

`PeerMsg` 定义如下
```rust
/// Message that can be sent to a peer.
pub enum PeerMsg {
    /// Raft message is the message sent between raft nodes in the same
    /// raft group. Messages need to be redirected to raftstore if target
    /// peer doesn't exist.
    RaftMessage(RaftMessage),
    /// Raft command is the command that is expected to be proposed by the
    /// leader of the target raft group. If it's failed to be sent, callback
    /// usually needs to be called before dropping in case of resource leak.
    RaftCommand(RaftCommand),
    /// Tick is periodical task. If target peer doesn't exist there is a potential
    /// that the raft node will not work anymore.
    Tick(PeerTicks),
    /// Result of applying committed entries. The message can't be lost.
    ApplyRes { res: ApplyTaskRes },
    /// Message that can't be lost but rarely created. If they are lost, real bad
    /// things happen like some peers will be considered dead in the group.
    SignificantMsg(SignificantMsg),
    /// Start the FSM.
    Start,
    /// A message only used to notify a peer.
    Noop,
    /// Message that is not important and can be dropped occasionally.
    CasualMessage(CasualMessage),
    /// Ask region to report a heartbeat to PD.
    HeartbeatPd,
}
```

在写入的时候 `RaftKv::exec_write_requests()` 就会发送一个 `RaftCommand`


`PeerFsmDelegate` 定义如下
```rust
pub struct PeerFsmDelegate<'a, T: 'static, C: 'static> {
    fsm: &'a mut PeerFsm,
    ctx: &'a mut PollContext<T, C>,
}
```

主要调用 `PeerFsmDelegate::handle_msgs()` 和 `PeerFsmDelegate::collect_ready()`
```rust
pub fn handle_msgs(&mut self, msgs: &mut Vec<PeerMsg>) {
    for m in msgs.drain(..) {
        match m {
            PeerMsg::RaftMessage(msg) => {
                if let Err(e) = self.on_raft_message(msg) {
                    error!(
                        "handle raft message err";
                        "region_id" => self.fsm.region_id(),
                        "peer_id" => self.fsm.peer_id(),
                        "err" => %e,
                    );
                }
            }
            PeerMsg::RaftCommand(cmd) => {
                self.ctx
                    .raft_metrics
                    .propose
                    .request_wait_time
                    .observe(duration_to_sec(cmd.send_time.elapsed()) as f64);
                self.propose_raft_command(cmd.request, cmd.callback)
            }
            PeerMsg::Tick(tick) => self.on_tick(tick),
            PeerMsg::ApplyRes { res } => {
                self.on_apply_res(res);
            }
            PeerMsg::SignificantMsg(msg) => self.on_significant_msg(msg),
            PeerMsg::CasualMessage(msg) => self.on_casual_msg(msg),
            PeerMsg::Start => self.start(),
            PeerMsg::HeartbeatPd => {
                if self.fsm.peer.is_leader() {
                    self.register_pd_heartbeat_tick()
                }
            }
            PeerMsg::Noop => {}
        }
    }
}

pub fn collect_ready(&mut self, proposals: &mut Vec<RegionProposal>) {
    let has_ready = self.fsm.has_ready;
    self.fsm.has_ready = false;
    if !has_ready || self.fsm.stopped {
        return;
    }
    self.ctx.pending_count += 1;
    self.ctx.has_ready = true;
    if let Some(p) = self.fsm.peer.take_apply_proposals() {
        proposals.push(p);
    }
    let res = self.fsm.peer.handle_raft_ready_append(self.ctx);
    if let Some(r) = res {
        self.on_role_changed(&r.0);
        if !r.0.entries.is_empty() {
            self.register_raft_gc_log_tick();
            self.register_split_region_check_tick();
        }
        self.ctx.ready_res.push(r);
    }
}
```

### ApplyFsm
主要实现在 `raftstore/store/fsm/apply.rs`

`ApplyFsm` 定义
```rust
pub struct ApplyFsm {
    delegate: ApplyDelegate,
    receiver: Receiver<Msg>,
    mailbox: Option<BasicMailbox<ApplyFsm>>,
}
```

`Msg` 定义
```rust
pub enum Msg {
    Apply {
        start: Instant,
        apply: Apply,
    },
    Registration(Registration),
    Proposal(RegionProposal),
    LogsUpToDate(CatchUpLogs),
    Noop,
    Destroy(Destroy),
    Snapshot(GenSnapTask),
    #[cfg(test)]
    Validate(u64, Box<dyn FnOnce(&ApplyDelegate) + Send>),
}
```

`ApplyFsm::handle_tasks()` 实现如下
```rust
fn handle_tasks(&mut self, apply_ctx: &mut ApplyContext, msgs: &mut Vec<Msg>) {
    let mut channel_timer = None;
    let mut drainer = msgs.drain(..);
    loop {
        match drainer.next() {
            Some(Msg::Apply { start, apply }) => {
                if channel_timer.is_none() {
                    channel_timer = Some(start);
                }
                self.handle_apply(apply_ctx, apply);
                if let Some(ref mut state) = self.delegate.yield_state {
                    state.pending_msgs = drainer.collect();
                    break;
                }
            }
            Some(Msg::Proposal(prop)) => self.handle_proposal(prop),
            Some(Msg::Registration(reg)) => self.handle_registration(reg),
            Some(Msg::Destroy(d)) => self.handle_destroy(apply_ctx, d),
            Some(Msg::LogsUpToDate(cul)) => self.logs_up_to_date_for_merge(apply_ctx, cul),
            Some(Msg::Noop) => {}
            Some(Msg::Snapshot(snap_task)) => self.handle_snapshot(apply_ctx, snap_task),
            #[cfg(test)]
            Some(Msg::Validate(_, f)) => f(&self.delegate),
            None => break,
        }
    }
    if let Some(timer) = channel_timer {
        let elapsed = duration_to_sec(timer.elapsed());
        APPLY_TASK_WAIT_TIME_HISTOGRAM.observe(elapsed);
    }
}
```

## TODO
- region_id 哪里来
