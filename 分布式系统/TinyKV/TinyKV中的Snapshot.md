## raft节点如何自动compact自己的entries日志
对于一个长期运行的服务器来说，永远记住完整的Raft日志是不现实的。相反，服务器会检查Raft日志的数量，并不时地丢弃超过阈值的日志。
一般来说，Snapshot 只是一个像 AppendEntries 一样的 Raft 消息，用来复制数据给 Follower，不同的是它的大小，Snapshot 包含了某个时间点的整个状态机数据，一次性建立和发送这么大的消息会消耗很多资源和时间，可能会阻碍其他 Raft 消息的处理，为了避免这个问题，Snapshot 消息会使用独立的连接，把数据分成几块来传输。这就是为什么TinyKV 服务有一个快照 RPC API 的原因。在TinyKV中，这个独立的连接叫snapRunner。

在`HandleMsg`中有时钟msg：
```go
case message.MsgTypeTick:
    d.onTick()
```
这个`onTick()`函数有几种类型的时钟,`onRaftBaseTick()`驱动的是raft模块的选举超时和心跳的时钟，`onRaftGCLogTick()`驱动是进行Log GC的时钟，`onSchedulerHeartbeatTick()`是驱动raft模块向PD模块报告心跳信息的时钟，`onSplitRegionCheckTick()`会生成一个`SplitCheckTask`任务，检查时如果发现满足 split 要求，则生成一个`MsgTypeSplitRegion`请求。
```go
func (d *peerMsgHandler) onTick() {
	if d.stopped {
		return
	}
	d.ticker.tickClock()
	if d.ticker.isOnTick(PeerTickRaft) {
		d.onRaftBaseTick()
	}
	if d.ticker.isOnTick(PeerTickRaftLogGC) {
		d.onRaftGCLogTick()
	}
	if d.ticker.isOnTick(PeerTickSchedulerHeartbeat) {
		d.onSchedulerHeartbeatTick()
	}
	if d.ticker.isOnTick(PeerTickSplitRegionCheck) {
		d.onSplitRegionCheckTick()
	}
	d.ctx.tickDriverSender <- d.regionId
}
```
这里我们关注的是`onRaftGCLogTick()`这个时钟驱动，我们看到Compact需要满足一定条件(日志过长)`appliedIdx > firstIdx && appliedIdx-firstIdx >= d.ctx.cfg.RaftLogGcCountLimit`，这里把`compactIdx`设置成`commitIdx`，接着调用函数`proposeRaftCommand`进行Log GC，这是一个`AdminRequest`。
```go
func (d *peerMsgHandler) onRaftGCLogTick() {
	d.ticker.schedule(PeerTickRaftLogGC)
	if !d.IsLeader() {
		return
	}

	appliedIdx := d.peerStorage.AppliedIndex()
	firstIdx, _ := d.peerStorage.FirstIndex()
	var compactIdx uint64
	if appliedIdx > firstIdx && appliedIdx-firstIdx >= d.ctx.cfg.RaftLogGcCountLimit {
		compactIdx = appliedIdx
	} else {
		return
	}

	y.Assert(compactIdx > 0)
	compactIdx -= 1
	if compactIdx < firstIdx {
		// In case compact_idx == first_idx before subtraction.
		return
	}

	term, err := d.RaftGroup.Raft.RaftLog.Term(compactIdx)
	if err != nil {
		log.Fatalf("appliedIdx: %d, firstIdx: %d, compactIdx: %d", appliedIdx, firstIdx, compactIdx)
		panic(err)
	}

	// Create a compact log request and notify directly.
	regionID := d.regionId
	request := newCompactLogRequest(regionID, d.Meta, compactIdx, term)
	d.proposeRaftCommand(request, nil)
}
```
`proposeRaftCommand()`接着调用 `RawNode.Propose()`函数，`Propose()`调用了`Raft.Step`驱动raft模块。像普通请求一样，将`CompactLogRequest`请求 `propose` 到 Raft 中，等待 Raft Group 确认。
当这条Log GC的 command 成功复制到集群中大多数的节点上(成功commit)后，在`HandleRaftReady()`中会执行已被提交的日志，其中包括这条Log GC的command(在`processAdminRequest`中执行)。
因为生成快照的时候compactIdx前面的日志可以被截断了，所以需要记录最后一条被截断的日志(快照中的最后一条日志)的索引和任期。
随后调用`d.ScheduleCompactLog(adminReq.CompactLog.CompactIndex)`函数。
```go
func (d *peerMsgHandler) ScheduleCompactLog(truncatedIndex uint64) {
	raftLogGCTask := &runner.RaftLogGCTask{
		RaftEngine: d.ctx.engine.Raft,
		RegionID:   d.regionId,
		StartIdx:   d.LastCompactedIdx,
		EndIdx:     truncatedIndex + 1,
	}
	d.LastCompactedIdx = raftLogGCTask.EndIdx
	d.ctx.raftLogGCTaskSender <- raftLogGCTask
}
```
这里的`d.LastCompactedIdx`定义在`peer`结构体中：
```go
// Index of last scheduled compacted raft log.
// (Used in 2C)
LastCompactedIdx uint64
```

随后`raftlog_gc.go` 里面收到 `RaftLogGCTask` 并在 `gcRaftLog()` 函数中处理。它会清除 raftDB 中已经被持久化的 entries，因为已经被 compact 了，所以不需要了。
```go
// gcRaftLog does the GC job and returns the count of logs collected.
func (r *raftLogGCTaskHandler) gcRaftLog(raftDb *badger.DB, regionId, startIdx, endIdx uint64) (uint64, error) {

	raftWb := engine_util.WriteBatch{}
	for idx := firstIdx; idx < endIdx; idx += 1 {
		key := meta.RaftLogKey(regionId, idx)
		raftWb.DeleteMeta(key)
	}

	return endIdx - firstIdx, nil
}
```

至此所有节点已经完成了日志的截断，以此确保 raftDB 中的 entries 不会无限制的占用空间(毕竟已经被 apply 了，留着也没多大作用)。

在DB中已经完成了日志的压缩，那么在内存中，也就可以丢弃被压缩过的日志了
```go
// Advance通知RawNode，应用程序已经应用并保存了最后一个Ready结果中的进度。
func (rn *RawNode) Advance(rd Ready) {
    // 2C
	rn.Raft.RaftLog.maybeCompact()        //丢弃被压缩的暂
}
```



## 快照发送

当 Leader append 日志给落后 node 节点时，发现对方所需要的 entry 已经被 compact。此时 Leader 会发送 Snapshot 过去。
```go
func (r *Raft) sendAppend(to uint64) bool {
    ...
    // 2C
    r.sendSnapshot(to)
    ...
}
```
当 Leader 需要发送 Snapshot 时，调用 `r.RaftLog.storage.Snapshot()` 生成 Snapshot。因为 Snapshot 很大，不会马上生成，这里为了避免阻塞，如果 Snapshot 还没有生成好，Snapshot 会先返回 `raft.ErrSnapshotTemporarilyUnavailable` 错误，Leader 就应该放弃本次 Snapshot，等待下一次再次请求 Snapshot。
```go
// sendSnapshot 发送快照给别的节点
func (r *Raft) sendSnapshot(to uint64) {
	var snapshot pb.Snapshot
	var err error
	if !IsEmptySnap(r.RaftLog.pendingSnapshot) {
		snapshot = *r.RaftLog.pendingSnapshot  // 挂起的还未处理的快照
	} else {
		snapshot, err = r.RaftLog.storage.Snapshot()  // 生成一份快照
	}

	if err != nil {
		return
	}
	r.msgs = append(r.msgs, pb.Message{
		MsgType:  pb.MessageType_MsgSnapshot,
		From:     r.id,
		To:       to,
		Term:     r.Term,
		Snapshot: &snapshot,
	})
	r.Prs[to].Next = snapshot.Metadata.Index + 1
}
```
而生成 snapshot 是异步的,调用的是这个`PeerStorage.Snapshot()`函数：
```go
func (ps *PeerStorage) Snapshot() (eraftpb.Snapshot, error) {
	var snapshot eraftpb.Snapshot
	if ps.snapState.StateType == snap.SnapState_Generating {
		select {
		case s := <-ps.snapState.Receiver:
			if s != nil {
				snapshot = *s
			}
		default:
			return snapshot, raft.ErrSnapshotTemporarilyUnavailable
		}
		ps.snapState.StateType = snap.SnapState_Relax
		if snapshot.GetMetadata() != nil {
			ps.snapTriedCnt = 0
			if ps.validateSnap(&snapshot) {
				return snapshot, nil
			}
		} else {
			log.Warnf("%s failed to try generating snapshot, times: %d", ps.Tag, ps.snapTriedCnt)
		}
	}

	if ps.snapTriedCnt >= 5 {
		err := errors.Errorf("failed to get snapshot after %d times", ps.snapTriedCnt)
		ps.snapTriedCnt = 0
		return snapshot, err
	}

	log.Infof("%s requesting snapshot", ps.Tag)
	ps.snapTriedCnt++
	ch := make(chan *eraftpb.Snapshot, 1)
	ps.snapState = snap.SnapState{
		StateType: snap.SnapState_Generating,
		Receiver:  ch,
	}
	// schedule snapshot generate task
	ps.regionSched <- &runner.RegionTaskGen{
		RegionId: ps.region.GetId(),
		Notifier: ch,
	}
	return snapshot, raft.ErrSnapshotTemporarilyUnavailable
}
```
下一次 Leader 请求 `Snapshot()` 时，因为已经创建完成，直接拿出来，包装为 `pb.MessageType_MsgSnapshot` 发送至目标节点。
```go
// HandleRaftReady函数
func (d *peerMsgHandler) HandleRaftReady() {
	//3. 调用 d.Send() 方法将 Ready 中的 Msg 发送出去；
	d.Send(d.ctx.trans, ready.Messages)
}
```
如果是 snapshot 的话调用的 `SendSnapshotSock()` 函数，普通 msg 是调用 `raftClient.Send()`函数发送
```go
func (t *ServerTransport) WriteData(storeID uint64, addr string, msg *raft_serverpb.RaftMessage) {
	if msg.GetMessage().GetSnapshot() != nil {
		t.SendSnapshotSock(addr, msg)
		return
	}
	if err := t.raftClient.Send(storeID, addr, msg); err != nil {
		log.Errorf("send raft msg err. err: %v", err)
	}
}
```

这个`SendSnapshotSock()`函数如下：

```go
t.snapScheduler <- &sendSnapTask{
	addr:     addr,
	msg:      msg,
	callback: callback,
}
```

然后会有个 snap-worker 执行发送快照的任务：

```go
func (r *snapRunner) Handle(t worker.Task) {
	switch t.(type) {
	case *sendSnapTask:
		r.send(t.(*sendSnapTask))
	case *recvSnapTask:
		r.recv(t.(*recvSnapTask))
	}
}
```

看这个`sendSnap`函数：

```go
	...
	buf := make([]byte, snapChunkLen)
	for remain := snap.TotalSize(); remain > 0; remain -= uint64(len(buf)) {
		if remain < uint64(len(buf)) {
			buf = buf[:remain]
		}
		_, err := io.ReadFull(snap, buf)
		if err != nil {
			return errors.Errorf("failed to read snapshot chunk: %v", err)
		}
		err = stream.Send(&raft_serverpb.SnapshotChunk{Data: buf})
		if err != nil {
			return err
		}
	}
	_, err = stream.CloseAndRecv()
	...
```

如果你不懂Go语法，看到`io.ReadFull(snap, buf)`一定有这样一个疑问，`snap`怎么可以作为`io.ReadFull`的参数呢，实际上，`snap` 它实际上是一个 `io.Reader` 实现。虽然快照对象可能是自定义类型，但它实现了 `io.Reader` 接口，允许数据以流的形式被读取。`snap`的`Read`接口如下，实际上就是遍历所有的CF文件，其中维护一个`cfIndex`表示的是当前遍历文件的位置，而我们知道，文件描述符会维护文件内部的一个`offset`，`read`系统调用后会自动进行`offset`的更新，所以我们只需要维护这个`cfIndex`就行。

```go
func (s *Snap) Read(b []byte) (int, error) {
	if len(b) == 0 {
		return 0, nil
	}
	for s.cfIndex < len(s.CFFiles) {
		cfFile := s.CFFiles[s.cfIndex]
		if cfFile.Size == 0 {
			s.cfIndex++
			continue
		}
		n, err := cfFile.File.Read(b)
		if n > 0 {
			return n, nil
		}
		if err != nil {
			if err == io.EOF {
				s.cfIndex++
				continue
			}
			return 0, errors.WithStack(err)
		}
	}
	return 0, io.EOF
}
```



## 快照接收与应用

快照的接收如下，它先调用`recvSnap`函数接收，并把snapshot写入本地文件中，然后再对 raft 模块发送一个 msg ，需要raft 调用 `handleSnapshot()`函数。

```go
func (r *snapRunner) recv(t *recvSnapTask) {
	// 这个 recvSnap 函数和 sendSnap 函数逻辑差不多
    msg, err := r.recvSnap(t.stream)
	if err == nil {
		r.router.SendRaftMessage(msg)
	}
	t.callback(err)
}
```

`handleSnapshot`函数先做一些判断，然后会设置`pendingSnapshot`，待应用快照。

```go
case pb.MessageType_MsgSnapshot:
	r.handleSnapshot(m)
```
`handleSnapshot()`函数会做一些检查，然后设置`RaftLog.pendingSnapshot`
```go
r.RaftLog.pendingSnapshot = m.Snapshot
```
然后再由`HandleRaftReady()`对 snapshot 进行回放，调用函数`ApplySnapshot()`(这里流程是`HandleRaftReady()` -> `SaveReadyState()` -> `ApplySnapshot()`)，`ready`中的`snapshot`就是`pendingSnapshot`。

```go
func (ps *PeerStorage) SaveReadyState(ready *raft.Ready) (*ApplySnapResult, error) {
	.....
	if !raft.IsEmptySnap(&ready.Snapshot) {
		var err error
        // ready.Snapshot 来自于 RaftLog.pendingSnapshot
		applySnap, err = ps.ApplySnapshot(&ready.Snapshot, kvWB, raftWB)
		if err != nil {
			return nil, err
		}
	}
    .....
}
```
这个`ApplySnapshot`函数主要的操作如下：

```go
	ps.regionSched <- &runner.RegionTaskApply{
		RegionId: snapData.Region.GetId(),
		Notifier: ch,
		SnapMeta: snapshot.Metadata,
		StartKey: snapData.Region.GetStartKey(),
		EndKey:   snapData.Region.GetEndKey(),
	}
```

随后会被这个`regionTaskHandler`的`Handle`函数处理：

```go
func (r *regionTaskHandler) Handle(t worker.Task) {
	switch t.(type) {
	case *RegionTaskGen:
		task := t.(*RegionTaskGen)
		// It is safe for now to handle generating and applying snapshot concurrently,
		// but it may not when merge is implemented.
		r.ctx.handleGen(task.RegionId, task.Notifier)
	case *RegionTaskApply:
		task := t.(*RegionTaskApply)
		r.ctx.handleApply(task.RegionId, task.Notifier, task.StartKey, task.EndKey, task.SnapMeta)
	case *RegionTaskDestroy:
		task := t.(*RegionTaskDestroy)
		r.ctx.cleanUpRange(task.RegionId, task.StartKey, task.EndKey)
	}
}
```

`handleApply`函数会调用`applySnap`函数进行快照的应用。

```go
// applySnap applies snapshot data of the Region.
func (snapCtx *snapContext) applySnap(regionId uint64, startKey, endKey []byte, snapMeta *eraftpb.SnapshotMetadata) error {
	log.Infof("begin apply snap data. [regionId: %d]", regionId)

	// cleanUpOriginData clear up the region data before applying snapshot
	snapCtx.cleanUpRange(regionId, startKey, endKey)

	snapKey := snap.SnapKey{RegionID: regionId, Index: snapMeta.Index, Term: snapMeta.Term}
	snapCtx.mgr.Register(snapKey, snap.SnapEntryApplying)
	defer snapCtx.mgr.Deregister(snapKey, snap.SnapEntryApplying)

	snapshot, err := snapCtx.mgr.GetSnapshotForApplying(snapKey)
	if err != nil {
		return errors.New(fmt.Sprintf("missing snapshot file %s", err))
	}

	t := time.Now()
	applyOptions := snap.NewApplyOptions(snapCtx.engines.Kv, &metapb.Region{
		Id:       regionId,
		StartKey: startKey,
		EndKey:   endKey,
	})
	if err := snapshot.Apply(*applyOptions); err != nil {
		return err
	}

	log.Infof("applying new data. [regionId: %d, timeTakes: %v]", regionId, time.Now().Sub(t))
	return nil
}
```



最后在 `Advance()` 的时候，清除 `pendingSnapshot`。至此，整个 Snapshot 接收流程结束。
可以看到 Snapshot 的 msg 发送不像传统的 msg，因为 Snapshot 通常很大，如果和普通方法一样发送，会占用大量的内网宽带。同时如果你不切片发送，中间一旦一部分失败了，就全功尽气。这个迅雷的切片下载一个道理。



## 快照生成

在上面我们说过，在通过`SnapShot()`获得 snapshot 的时候，实际会有一个异步的处理，把任务放入`peerStorage.regionSched`中。
```go
// schedule snapshot generate task
ps.regionSched <- &runner.RegionTaskGen{
    RegionId: ps.region.GetId(),
    Notifier: ch,
}
```
`region_task.go`中的`Handle`函数：
```go
func (r *regionTaskHandler) Handle(t worker.Task) {
	switch t.(type) {
	case *RegionTaskGen:
		task := t.(*RegionTaskGen)
		// It is safe for now to handle generating and applying snapshot concurrently,
		// but it may not when merge is implemented.
		r.ctx.handleGen(task.RegionId, task.Notifier)
	case *RegionTaskApply:
		task := t.(*RegionTaskApply)
		r.ctx.handleApply(task.RegionId, task.Notifier, task.StartKey, task.EndKey, task.SnapMeta)
	case *RegionTaskDestroy:
		task := t.(*RegionTaskDestroy)
		r.ctx.cleanUpRange(task.RegionId, task.StartKey, task.EndKey)
	}
}
```

```go
// handleGen handles the task of generating snapshot of the Region.
func (snapCtx *snapContext) handleGen(regionId uint64, notifier chan<- *eraftpb.Snapshot) {
	snap, err := doSnapshot(snapCtx.engines, snapCtx.mgr, regionId)
	if err != nil {
		log.Errorf("failed to generate snapshot!!!, [regionId: %d, err : %v]", regionId, err)
		notifier <- nil
	} else {
		notifier <- snap
	}
}
```
```go
func doSnapshot(engines *engine_util.Engines, mgr *snap.SnapManager, regionId uint64) (*eraftpb.Snapshot, error) {
	txn := engines.Kv.NewTransaction(false)
	
    // 这里的 index 是 appliedIndex
	index, term, err := getAppliedIdxTermForSnapshot(engines.Raft, txn, regionId)
	if err != nil {
		return nil, err
	}
	
    // 这个 snapkey 可以标识一个快照，并且可以通过 snapkey 为 快照文件命名 
	key := snap.SnapKey{RegionID: regionId, Index: index, Term: term}
	mgr.Register(key, snap.SnapEntryGenerating)
	defer mgr.Deregister(key, snap.SnapEntryGenerating)

	regionState := new(rspb.RegionLocalState)
	err = engine_util.GetMetaFromTxn(txn, meta.RegionStateKey(regionId), regionState)
	if err != nil {
		panic(err)
	}
	if regionState.GetState() != rspb.PeerState_Normal {
		return nil, errors.Errorf("snap job %d seems stale, skip", regionId)
	}

	region := regionState.GetRegion()
	confState := util.ConfStateFromRegion(region)
	snapshot := &eraftpb.Snapshot{
        // Meta data 表示的是这个 snapshot 是 [appliedIndex, term] 日志前(包括这条)的全量 KV 键值对
		Metadata: &eraftpb.SnapshotMetadata{
			Index:     key.Index,
			Term:      key.Term,
			ConfState: &confState,
		},
	}
	s, err := mgr.GetSnapshotForBuilding(key)
	if err != nil {
		return nil, err
	}
	// Set snapshot data
	snapshotData := &rspb.RaftSnapshotData{Region: region}
	snapshotStatics := snap.SnapStatistics{}
	err = s.Build(txn, region, snapshotData, &snapshotStatics, mgr)
	if err != nil {
		return nil, err
	}
	snapshot.Data, err = snapshotData.Marshal()
	return snapshot, err
}
```
这个函数重点在`err = s.Build(txn, region, snapshotData, &snapshotStatics, mgr)`这行代码。
这个`Build()`函数后面调用了`build()`函数：

```go
func (b *snapBuilder) build() error {
	defer b.txn.Discard()
	startKey, endKey := b.region.StartKey, b.region.EndKey
    // 需要写三个文件(cf)，分别是default、write、lock
    for _, file := range b.cfFiles {
		cf := file.CF
		sstWriter := file.SstWriter

		it := engine_util.NewCFIterator(cf, b.txn)
		for it.Seek(startKey); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			if engine_util.ExceedEndKey(key, endKey) {
				break
			}
			value, err := item.Value()
			if err != nil {
				return err
			}
			cfKey := engine_util.KeyWithCF(cf, key)
			if err := sstWriter.Add(cfKey, y.ValueStruct{
				Value: value,
			}); err != nil {
				return err
			}
			file.KVCount++
			file.Size += uint64(len(cfKey) + len(value))
		}
		it.Close()
		b.kvCount += file.KVCount
		b.size += int(file.Size)
	}
	return nil
}
```





