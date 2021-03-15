# TiKV v1.0 raftstore Apply

## Runner
Store::run() 时会启动 ApplyRunner
```rust
pub fn run(&mut self, event_loop: &mut EventLoop<Self>) -> Result<()> {
    // ...snipped...
    let (tx, rx) = mpsc::channel();
    let apply_runner = ApplyRunner::new(self, tx, self.cfg.sync_log);
    self.apply_res_receiver = Some(rx);
    box_try!(self.apply_worker.start(apply_runner));

    event_loop.run(self)?;
    Ok(())
}
```

而 ApplyRunner 实际上就是 `store/worker/mod.rs` 里 export 的
```rust
pub use self::apply::{Apply, ApplyMetrics, ApplyRes, Proposal, RegionProposal, Registration,
                      Runner as ApplyRunner, Task as ApplyTask, TaskRes as ApplyTaskRes};
```

实现上就是 `store/worker/apply.rs` 里的 Runner

```rust
// TODO: use threadpool to do task concurrently
pub struct Runner {
    db: Arc<DB>,
    host: Arc<CoprocessorHost>,
    delegates: HashMap<u64, ApplyDelegate>,
    notifier: Sender<TaskRes>,
    sync_log: bool,
    tag: String,
}
```
delegates 里的 u64 就是 region_id

## Runner::run()
Runner 实现了 Runnable 的 trait ，每次处理一个 task，可以看到对不同的 Task 执行不同的函数
```rust
impl Runnable<Task> for Runner {
    fn run(&mut self, task: Task) {
        match task {
            Task::Applies(a) => self.handle_applies(a),
            Task::Proposals(props) => self.handle_proposals(props),
            Task::Registration(s) => self.handle_registration(s),
            Task::Destroy(d) => self.handle_destroy(d),
        }
    }

    fn shutdown(&mut self) {
        self.handle_shutdown();
    }
}
```

## Runner::handle_proposals()
主要是接收 proposals ，将其放到 region 对应的 ApplyDelegate::pending_cmds
```rust
fn handle_proposals(&mut self, proposals: Vec<RegionProposal>) {
    let mut propose_num = 0;
    for region_proposal in proposals {
        propose_num += region_proposal.props.len();
        let delegate = match self.delegates.get_mut(&region_proposal.region_id) {
            Some(d) => d,
            None => {
                for p in region_proposal.props {
                    let cmd = PendingCmd::new(p.index, p.term, p.cb);
                    notify_region_removed(region_proposal.region_id, region_proposal.id, cmd);
                }
                continue;
            }
        };
        assert_eq!(delegate.id, region_proposal.id);
        for p in region_proposal.props {
            let cmd = PendingCmd::new(p.index, p.term, p.cb);
            if p.is_conf_change {
                if let Some(cmd) = delegate.pending_cmds.take_conf_change() {
                    // if it loses leadership before conf change is replicated, there may be
                    // a stale pending conf change before next conf change is applied. If it
                    // becomes leader again with the stale pending conf change, will enter
                    // this block, so we notify leadership may have been changed.
                    notify_stale_command(&delegate.tag, delegate.term, cmd);
                }
                delegate.pending_cmds.set_conf_change(cmd);
            } else {
                delegate.pending_cmds.append_normal(cmd);
            }
        }
    }
    APPLY_PROPOSAL.observe(propose_num as f64);
}
```
主要流程
- 遍历 proposals 拿到 region_proposal
    - Runner::delegates.get_mut(&region_proposal.region_id) 拿到 ApplyDelegate
        - 如果 region id 对应的 ApplyDelegate 不存在
            - 遍历 RegionProposal::props
                - 创建 PendingCmd::new(p.index, p.term, p.cb)
                - 执行 notify_region_removed()
                    - 调用 cb() ，传入 Error::RegionNotFound(region_id)
            - continue
    - 遍历 RegionProposal::props
        - 创建 PendingCmd::new(p.index, p.term, p.cb)
        - 如果 RegionProposal::is_conf_change
            - 如果 ApplyDelegate::pending_cmds.take_conf_change() 为 Some(cmd)
                - notify_stale_command(&delegate.tag, delegate.term, cmd)
                    - 调用 cb() ，传入 Error::StaleCommand
                - 主要是通知之前的 conf_change cmd 过期了
            - ApplyDelegate::pending_cmds.set_conf_change(cmd) 覆盖掉之前的 conf_change
        - 否则
            - ApplyDelegate::pending_cmds.append_normal(cmd)

## Runner::handle_applies()
主要是执行 Apply 中的 entries
```rust
fn handle_applies(&mut self, applys: Vec<Apply>) {
    let t = SlowTimer::new();

    let mut applys_res = Vec::with_capacity(applys.len());
    let mut apply_ctx = ApplyContext::new(self.host.as_ref());
    let mut committed_count = 0;
    for apply in applys {
        if apply.entries.is_empty() {
            continue;
        }
        let mut e = match self.delegates.entry(apply.region_id) {
            MapEntry::Vacant(_) => {
                error!("[region {}] is missing", apply.region_id);
                continue;
            }
            MapEntry::Occupied(e) => e,
        };
        {
            let delegate = e.get_mut();
            delegate.metrics = ApplyMetrics::default();
            delegate.term = apply.term;
            committed_count += apply.entries.len();
            let results = delegate.handle_raft_committed_entries(&mut apply_ctx, apply.entries);

            if delegate.pending_remove {
                delegate.destroy();
            }

            applys_res.push(ApplyRes {
                region_id: apply.region_id,
                apply_state: delegate.apply_state.clone(),
                exec_res: results,
                metrics: delegate.metrics.clone(),
                applied_index_term: delegate.applied_index_term,
            });
        }
        if e.get().pending_remove {
            e.remove();
        }
    }

    // Write to engine
    // raftsotre.sync-log = true means we need prevent data loss when power failure.
    // take raft log gc for example, we write kv WAL first, then write raft WAL,
    // if power failure happen, raft WAL may synced to disk, but kv WAL may not.
    // so we use sync-log flag here.
    let mut write_opts = WriteOptions::new();
    write_opts.set_sync(self.sync_log && apply_ctx.sync_log);
    self.db
        .write_opt(apply_ctx.wb.take().unwrap(), &write_opts)
        .unwrap_or_else(|e| panic!("failed to write to engine, error: {:?}", e));

    // Call callbacks
    for (cb, resp) in apply_ctx.cbs.drain(..) {
        cb(resp);
    }

    if !applys_res.is_empty() {
        self.notifier.send(TaskRes::Applys(applys_res)).unwrap();
    }

    STORE_APPLY_LOG_HISTOGRAM.observe(duration_to_sec(t.elapsed()) as f64);

    slow_log!(
        t,
        "{} handle ready {} committed entries",
        self.tag,
        committed_count
    );
}
```
主要流程
- committed_count 设置为 0
- 通过 ApplyContext::new(Runner::host.as_ref()) 创建 apply_ctx
- 遍历 applys 里的 apply
    - 如果 Apply::entries.is_empty() ，则跳过该 apply
    - Runner::delegates.entry(apply.region_id) 获取 region id 对应的 entry
        - 不存在则跳过
    - 获取 entry 的 ApplyDelegate
    - ApplyDelegate::term 设置为 Apply::term
    - committed_count += apply.entries.len()
    - 执行 ApplyDelegate::handle_raft_committed_entries(&mut apply_ctx, apply.entries)
    - 如果 ApplyDelegate::pending_remove
        - 执行 ApplyDelegate::destroy()
    - 构造 ApplyRes 并 push 到 applys_res
    - 如果 ApplyDelegate::pending_remove
        - 将 entry 删除
- Runner::db 调用 DB::write_opt 将 apply_ctx.wb 写入 engine
- 遍历 apply_ctx.cbs 并执行回调
- 如果 !applys_res.is_empty()
    - 执行 notifier.send(TaskRes::Applys(applys_res))

### ApplyDelegate::handle_raft_committed_entries()
这里 commited_entries 就是 Apply::entries
```rust
fn handle_raft_committed_entries(
    &mut self,
    apply_ctx: &mut ApplyContext,
    committed_entries: Vec<Entry>,
) -> Vec<ExecResult> {
    if committed_entries.is_empty() {
        return vec![];
    }
    // If we send multiple ConfChange commands, only first one will be proposed correctly,
    // others will be saved as a normal entry with no data, so we must re-propose these
    // commands again.
    let mut results = vec![];
    for entry in committed_entries {
        if self.pending_remove {
            // This peer is about to be destroyed, skip everything.
            break;
        }

        let expect_index = self.apply_state.get_applied_index() + 1;
        if expect_index != entry.get_index() {
            panic!(
                "{} expect index {}, but got {}",
                self.tag,
                expect_index,
                entry.get_index()
            );
        }

        let res = match entry.get_entry_type() {
            EntryType::EntryNormal => self.handle_raft_entry_normal(apply_ctx, entry),
            EntryType::EntryConfChange => self.handle_raft_entry_conf_change(apply_ctx, entry),
        };

        if let Some(res) = res {
            results.push(res);
        }
    }

    if !self.pending_remove {
        self.write_apply_state(apply_ctx.wb_mut());
    }

    self.update_metrics(apply_ctx);
    apply_ctx.mark_last_bytes_and_keys();

    results
}
```
主要流程
- 遍历 committed_entries
    - 如果 ApplyDelegate::pending_remove ，说明这个 peer 要被删除了，直接 break 循环
    - 如果 expect_index 不等于 ApplyDelegate::apply_state.get_applied_index() + 1
        - panic
    - match Entry::get_entry_type()
        - EntryType::EntryNormal ，执行 ApplyDelegate::handle_raft_entry_normal(apply_ctx, entry)
        - EntryType::EntryConfChange ，执行 ApplyDelegate::handle_raft_entry_conf_change(apply_ctx, entry)
        - 拿到执行结果 res
    - results.push(res)
- 如果 !ApplyDelegate::pending_remove
    - 执行 ApplyDelegate::write_apply_state(apply_ctx.wb_mut())
- 执行 ApplyDelegate::update_metrics() 更新 apply_ctx 里的统计
- 执行 ApplyContext::mark_last_bytes_and_keys()

### ApplyDelegate::handle_raft_entry_normal()
```rust
fn handle_raft_entry_normal(
    &mut self,
    apply_ctx: &mut ApplyContext,
    entry: Entry,
) -> Option<ExecResult> {
    let index = entry.get_index();
    let term = entry.get_term();
    let data = entry.get_data();

    if !data.is_empty() {
        let cmd = parse_data_at(data, index, &self.tag);

        if should_flush_to_engine(&cmd, apply_ctx.wb_ref().count()) {
            self.write_apply_state(apply_ctx.wb_mut());

            self.update_metrics(apply_ctx);

            // flush to engine
            self.engine
                .write(apply_ctx.wb.take().unwrap())
                .unwrap_or_else(|e| {
                    panic!("{} failed to write to engine, error: {:?}", self.tag, e)
                });

            // call callback
            for (cb, resp) in apply_ctx.cbs.drain(..) {
                cb(resp);
            }
            apply_ctx.wb = Some(WriteBatch::with_capacity(DEFAULT_APPLY_WB_SIZE));
            apply_ctx.mark_last_bytes_and_keys();
        }

        return self.process_raft_cmd(apply_ctx, index, term, cmd);
    }

    // when a peer become leader, it will send an empty entry.
    let mut state = self.apply_state.clone();
    state.set_applied_index(index);
    self.apply_state = state;
    self.applied_index_term = term;
    assert!(term > 0);
    while let Some(mut cmd) = self.pending_cmds.pop_normal(term - 1) {
        // apprently, all the callbacks whose term is less than entry's term are stale.
        apply_ctx.cbs.push((
            cmd.cb.take().unwrap(),
            cmd_resp::err_resp(Error::StaleCommand, term),
        ));
    }
    None
}
```
主要流程
- 如果 Entry::get_data() 不为空
    - parse_data_at(data, index, &self.tag) 得到 cmd
        - 就是 pb 反序列化
    - 如果 should_flush_to_engine(&cmd, apply_ctx.wb_ref().count())
        - should_flush_to_engine() 满足以下情况之一会返回 true
            - cmd.has_admin_request() 且 cmd.get_admin_request().get_cmd_type() == AdminCmdType::ComputeHash
            - wb_keys >= WRITE_BATCH_MAX_KEYS ，即一定批次写一次 engine
            - cmd 的 req 里有 req.has_delete_range()
        - 执行 ApplyDelegate::write_apply_state(apply_ctx.wb_mut())
        - 更新 metrics
        - ApplyDelegate::engine 执行 DB::write() 写入 apply_ctx.wb
        - 遍历 apply_ctx.cbs 执行 cb
        - 重新初始化 apply_ctx.wb 为 Some(WriteBatch::with_capacity(DEFAULT_APPLY_WB_SIZE))
        - 执行 ApplyContext::mark_last_bytes_and_keys
    - 返回 ApplyDelegate::process_raft_cmd(apply_ctx, index, term, cmd)
- 否则说明是一个空 Entry
- 更新 ApplyDelegate::apply_state 的 applied_index 以及 ApplyDelegate::applied_index_term
- pop 掉 ApplyDelegate::pending_cmds 里 term 小于 term - 1 的 cmd

ApplyDelegate::write_apply_state() 会构造 CF_RAFT cf 的 write batch 到 ApplyContext::wb 中
- key 为 keys::apply_state_key(self.region.get_id())
- value 为 ApplyDelegate::apply_state
```rust
fn write_apply_state(&self, wb: &WriteBatch) {
    rocksdb::get_cf_handle(&self.engine, CF_RAFT)
        .map_err(From::from)
        .and_then(|handle| {
            wb.put_msg_cf(
                handle,
                &keys::apply_state_key(self.region.get_id()),
                &self.apply_state,
            )
        })
        .unwrap_or_else(|e| {
            panic!(
                "{} failed to save apply state to write batch, error: {:?}",
                self.tag,
                e
            );
        });
}
```

### ApplyDelegate::handle_raft_entry_conf_change()
和 ApplyDelegate::handle_raft_entry_normal() 类似，会调用 ApplyDelegate::process_raft_cmd()
```rust
fn handle_raft_entry_conf_change(
    &mut self,
    apply_ctx: &mut ApplyContext,
    entry: Entry,
) -> Option<ExecResult> {
    let index = entry.get_index();
    let term = entry.get_term();
    let conf_change: ConfChange = parse_data_at(entry.get_data(), index, &self.tag);
    let cmd = parse_data_at(conf_change.get_context(), index, &self.tag);
    Some(
        self.process_raft_cmd(apply_ctx, index, term, cmd)
            .map_or_else(
                || {
                    // If failed, tell raft that the config change was aborted.
                    ExecResult::ChangePeer(Default::default())
                },
                |mut res| {
                    if let ExecResult::ChangePeer(ref mut cp) = res {
                        cp.conf_change = conf_change;
                    } else {
                        panic!(
                            "{} unexpected result {:?} for conf change {:?} at {}",
                            self.tag,
                            res,
                            conf_change,
                            index
                        );
                    }
                    res
                },
            ),
    )
}
```

### ApplyDelegate::process_raft_cmd()
```rust
fn process_raft_cmd(
    &mut self,
    apply_ctx: &mut ApplyContext,
    index: u64,
    term: u64,
    mut cmd: RaftCmdRequest,
) -> Option<ExecResult> {
    if index == 0 {
        panic!(
            "{} processing raft command needs a none zero index",
            self.tag
        );
    }

    if cmd.has_admin_request() {
        apply_ctx.sync_log = true;
    }

    let cmd_cb = self.find_cb(index, term, &cmd);
    apply_ctx.host.pre_apply(&self.region, &mut cmd);
    let (mut resp, exec_result) = self.apply_raft_cmd(apply_ctx.wb_mut(), index, term, &cmd);

    debug!("{} applied command at log index {}", self.tag, index);

    let cb = match cmd_cb {
        None => return exec_result,
        Some(cb) => cb,
    };

    // TODO: Involve post apply hook.
    // TODO: if we have exec_result, maybe we should return this callback too. Outer
    // store will call it after handing exec result.
    cmd_resp::bind_term(&mut resp, self.term);
    apply_ctx.cbs.push((cb, resp));

    exec_result
}
```
主要流程
- 如果 RaftCmdRequest::has_admin_request()
    - apply_ctx.sync_log = true
- ApplyDelegate::find_cb(index, term, &cmd) 获取 cmd_cb
- 执行 ApplyContext::host CoprocessorHost::pre_apply(&self.region, &mut cmd) ，主要是通知 RegionObserver
- 执行 ApplyDelegate::apply_raft_cmd(apply_ctx.wb_mut(), index, term, &cmd)
- 如果有 cmd_cb ，还会执行 apply_ctx.cbs.push((cb, resp))

### ApplyDelegate::apply_raft_cmd()
```rust
// apply operation can fail as following situation:
//   1. encouter an error that will occur on all store, it can continue
// applying next entry safely, like stale epoch for example;
//   2. encouter an error that may not occur on all store, in this case
// we should try to apply the entry again or panic. Considering that this
// usually due to disk operation fail, which is rare, so just panic is ok.
fn apply_raft_cmd(
    &mut self,
    wb: &mut WriteBatch,
    index: u64,
    term: u64,
    req: &RaftCmdRequest,
) -> (RaftCmdResponse, Option<ExecResult>) {
    // if pending remove, apply should be aborted already.
    assert!(!self.pending_remove);

    let mut ctx = self.new_ctx(wb, index, term, req);
    ctx.wb.set_save_point();
    let (resp, exec_result) = self.exec_raft_cmd(&mut ctx).unwrap_or_else(|e| {
        // clear dirty values.
        ctx.wb.rollback_to_save_point().unwrap();
        match e {
            Error::StaleEpoch(..) => info!("{} stale epoch err: {:?}", self.tag, e),
            _ => error!("{} execute raft command err: {:?}", self.tag, e),
        }
        (cmd_resp::new_error(e), None)
    });

    ctx.apply_state.set_applied_index(index);

    self.apply_state = ctx.apply_state;
    self.applied_index_term = term;

    if let Some(ref exec_result) = exec_result {
        match *exec_result {
            ExecResult::ChangePeer(ref cp) => {
                self.region = cp.region.clone();
            }
            ExecResult::ComputeHash { .. } |
            ExecResult::VerifyHash { .. } |
            ExecResult::CompactLog { .. } |
            ExecResult::DeleteRange { .. } => {}
            ExecResult::SplitRegion {
                ref left,
                ref right,
                right_derive,
            } => {
                if right_derive {
                    self.region = right.clone();
                } else {
                    self.region = left.clone();
                }
                self.metrics.size_diff_hint = 0;
                self.metrics.delete_keys_hint = 0;
            }
        }
    }

    (resp, exec_result)
}
```
主要流程
- ApplyDelegate::new_ctx(wb, index, term, req) 得到 ExecContext ctx
- ExecContext::wb.set_save_point()
- 执行 ApplyDelegate::exec_raft_cmd(&mut ctx) 拿到 resp 和 exec_result
    - 如果失败，执行 ExecContext::wb.rollback_to_save_point()
- ExecContext::apply_state.set_applied_index(index)
- ApplyDelegate::apply_state = ctx.apply_state
- ApplyDelegate::applied_index_term = term
- match exec_result ，如果是 ExecResult::ChangePeer 或者 ExecResult::SplitRegion 则更新 ApplyDelegate::region
- 返回 resp 和 exec_result

### ApplyDelegate::exec_raft_cmd()
ApplyDelegate::exec_raft_cmd() 会执行对应的 request ，需要注意这里产生的数据都在记录在 WriteBatch wb 内，还没有真正刷到 rocksdb
```rust
// Only errors that will also occur on all other stores should be returned.
fn exec_raft_cmd(
    &mut self,
    ctx: &mut ExecContext,
) -> Result<(RaftCmdResponse, Option<ExecResult>)> {
    check_epoch(&self.region, ctx.req)?;
    if ctx.req.has_admin_request() {
        self.exec_admin_cmd(ctx)
    } else {
        self.exec_write_cmd(ctx)
    }
}
```
主要流程
- 执行 check_epoch()
- 如果是 admin request
    - ApplyDelegate::exec_admin_cmd(ctx)
- 否则 ApplyDelegate::exec_write_cmd(ctx)

### ApplyDelegate::exec_write_cmd()
可以看到基本流程是遍历 requests ，根据 cmd_type 执行相应的 ApplyDelegate::handle_xxx() 函数，然后返回 resp
```rust
fn exec_write_cmd(
    &mut self,
    ctx: &ExecContext,
) -> Result<(RaftCmdResponse, Option<ExecResult>)> {
    let requests = ctx.req.get_requests();
    let mut responses = Vec::with_capacity(requests.len());

    let mut ranges = vec![];
    for req in requests {
        let cmd_type = req.get_cmd_type();
        let mut resp = match cmd_type {
            CmdType::Put => self.handle_put(ctx, req),
            CmdType::Delete => self.handle_delete(ctx, req),
            CmdType::DeleteRange => self.handle_delete_range(req, &mut ranges),
            // Readonly commands are handled in raftstore directly.
            // Don't panic here in case there are old entries need to be applied.
            // It's also safe to skip them here, because a restart must have happened,
            // hence there is no callback to be called.
            CmdType::Snap | CmdType::Get => {
                warn!("{} skip readonly command: {:?}", self.tag, req);
                continue;
            }
            CmdType::Prewrite | CmdType::Invalid => {
                Err(box_err!("invalid cmd type, message maybe currupted"))
            }
        }?;

        resp.set_cmd_type(cmd_type);

        responses.push(resp);
    }

    let mut resp = RaftCmdResponse::new();
    resp.set_responses(RepeatedField::from_vec(responses));

    let exec_res = if ranges.is_empty() {
        None
    } else {
        Some(ExecResult::DeleteRange { ranges: ranges })
    };

    Ok((resp, exec_res))
}
```

### ApplyDelegate::handle_put()
以 handle_put() 为例子，可以看到就是将数据 put 到 WriteBatch 中
```rust
fn handle_put(&mut self, ctx: &ExecContext, req: &Request) -> Result<Response> {
    let (key, value) = (req.get_put().get_key(), req.get_put().get_value());
    check_data_key(key, &self.region)?;

    let resp = Response::new();
    let key = keys::data_key(key);
    self.metrics.size_diff_hint += key.len() as i64;
    self.metrics.size_diff_hint += value.len() as i64;
    if req.get_put().has_cf() {
        let cf = req.get_put().get_cf();
        // TODO: don't allow write preseved cfs.
        if cf == CF_LOCK {
            self.metrics.lock_cf_written_bytes += key.len() as u64;
            self.metrics.lock_cf_written_bytes += value.len() as u64;
        }
        // TODO: check whether cf exists or not.
        rocksdb::get_cf_handle(&self.engine, cf)
            .and_then(|handle| ctx.wb.put_cf(handle, &key, value))
            .unwrap_or_else(|e| {
                panic!(
                    "{} failed to write ({}, {}) to cf {}: {:?}",
                    self.tag,
                    escape(&key),
                    escape(value),
                    cf,
                    e
                )
            });
    } else {
        ctx.wb.put(&key, value).unwrap_or_else(|e| {
            panic!(
                "{} failed to write ({}, {}): {:?}",
                self.tag,
                escape(&key),
                escape(value),
                e
            );
        });
    }
    Ok(resp)
}
```

### ApplyDelegate::exec_admin_cmd()
同理，该函数同样根据 cmd_type 执行 相应的 ApplyDelegate::exec_xxx() 函数，并返回 resp
```rust
fn exec_admin_cmd(
    &mut self,
    ctx: &mut ExecContext,
) -> Result<(RaftCmdResponse, Option<ExecResult>)> {
    let request = ctx.req.get_admin_request();
    let cmd_type = request.get_cmd_type();
    info!(
        "{} execute admin command {:?} at [term: {}, index: {}]",
        self.tag,
        request,
        ctx.term,
        ctx.index
    );

    let (mut response, exec_result) = match cmd_type {
        AdminCmdType::ChangePeer => self.exec_change_peer(ctx, request),
        AdminCmdType::Split => self.exec_split(ctx, request),
        AdminCmdType::CompactLog => self.exec_compact_log(ctx, request),
        AdminCmdType::TransferLeader => Err(box_err!("transfer leader won't exec")),
        AdminCmdType::ComputeHash => self.exec_compute_hash(ctx, request),
        AdminCmdType::VerifyHash => self.exec_verify_hash(ctx, request),
        AdminCmdType::InvalidAdmin => Err(box_err!("unsupported admin command type")),
    }?;
    response.set_cmd_type(cmd_type);

    let mut resp = RaftCmdResponse::new();
    resp.set_admin_response(response);
    Ok((resp, exec_result))
}
```
不过 admin cmd 执行的逻辑则相对复杂些，因为有些 cmd 还涉及到 region 状态的改变

