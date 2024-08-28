## 与split 有关的一些 worker

先来看看这几个 woker:
```go
bs.workers = &workers{
    splitCheckWorker: worker.NewWorker("split-check", wg),
    regionWorker:     worker.NewWorker("snapshot-worker", wg),
    raftLogGCWorker:  worker.NewWorker("raft-gc-worker", wg),
    schedulerWorker:  worker.NewWorker("scheduler-worker", wg),
    wg:               wg,
}
```

**scheduler worker**
- 负责接收 peer 发送过来的 SchedulerRegionHeartbeatTask ，发送 region heartbeat 给 scheduler，报告 region 的 leader、region size 等信息。
- 负责接收 store worker 发送过来的 SchedulerStoreHeartbeatTask ，发送 store heartbeat 给 scheduler，报告 store 的 capacity, used size 等信息。
- 负责接收 peer 发送过来的 SchedulerAskSplitTask，发送 ask split 给 scheduler，请求 scheduler 为分裂出的 region 分配 peers。为了 fault tolerance，一个 region 最少需要有三个 region replicas，且它们- 需要分布在不同的 store 上。每个 peer 都需要有一个 unique peer id，如此才能进行 routing。这样的分配任务只能由具有全局信息的 scheduler 完成。

**split check worker**

- 负责接收 peer 发送过来的 SplitCheckTask ，检查是否需要对 region 进行 split。如果需要，则把 split key 塞进一个 MsgTypeSplitRegion msg 中，发送给 peer。peer 收到以后，则会向 scheduler 发送一个 SchedulerAskSplitTask。


**region worker**
- 主要和快照相关。接收 peer 发送过来的 RegionTaskGen、 RegionTaskApply、 RegionTaskDestroy，负责generate snapshot、ingest snapshot、和清理 region data。



## split的流程
`func (d *peerMsgHandler) onTick()`函数中有`PeerTickSchedulerHeartbeat`时钟，定期向 PD 模块汇报 Region 的信息；还有`PeerTickSplitRegionCheck`时钟，它会先做一些检查，如果符合 split 的条件，就生成一个`SplitCheckTask`任务。
```go
if d.ticker.isOnTick(PeerTickSplitRegionCheck) {
    d.onSplitRegionCheckTick()
}
```

```go
func (d *peerMsgHandler) onSplitRegionCheckTick() {
	d.ticker.schedule(PeerTickSplitRegionCheck)
	....
	if d.ApproximateSize != nil && d.SizeDiffHint < d.ctx.cfg.RegionSplitSize/8 {
		return
	}
	d.ctx.splitCheckTaskSender <- &runner.SplitCheckTask{
		Region: d.Region(),
	}
	d.SizeDiffHint = 0
}
```
这个 `SplitCheckTask` 会被 `Handle`处理，我们看到会调用 `r.router.Send()`路由到指定的 Peer
```go
/// run checks a region with split checkers to produce split keys and generates split admin command.
func (r *splitCheckHandler) Handle(t worker.Task) {
	spCheckTask, ok := t.(*SplitCheckTask)
	...
	region := spCheckTask.Region
	regionId := region.Id
	...
	key := r.splitCheck(regionId, region.StartKey, region.EndKey)
	if key != nil {
		_, userKey, err := codec.DecodeBytes(key)
		if err == nil {
			key = codec.EncodeBytes(userKey)
		}
		msg := message.Msg{
			Type:     message.MsgTypeSplitRegion,
			RegionID: regionId,
			Data: &message.MsgSplitRegion{
				RegionEpoch: region.GetRegionEpoch(),
				SplitKey:    key,
			},
		}
		err = r.router.Send(regionId, msg)
		if err != nil {
			log.Warnf("failed to send check result: [regionId: %d, err: %v]", regionId, err)
		}
	}
}
```

在 peer_msg_handler.go 中的 `HandleMsg()` 方法中调用 `onPrepareSplitRegion()`，发送 `SchedulerAskSplitTask` 请求到 scheduler_task.go 中，申请其分配新的 region id 和 peer id。申请成功后其会发起一个 `AdminCmdType_Split` 的 AdminRequest 到 region 中。

```go
func (d *peerMsgHandler) onPrepareSplitRegion(regionEpoch *metapb.RegionEpoch, splitKey []byte, cb *message.Callback) {
	if err := d.validateSplitRegion(regionEpoch, splitKey); err != nil {
		cb.Done(ErrResp(err))
		return
	}
	region := d.Region()
	d.ctx.schedulerTaskSender <- &runner.SchedulerAskSplitTask{
		Region:   region,
		SplitKey: splitKey,
		Peer:     d.Meta,
		Callback: cb,
	}
}
```
在`func (r *SchedulerTaskHandler) Handle`这个函数中会调用`onAskSplit`函数，申请其分配新的 region id 和 peer id。申请成功后其会发起一个 `AdminCmdType_Split` 的 AdminRequest 到 region 中。
```go
case *SchedulerAskSplitTask:
    r.onAskSplit(t.(*SchedulerAskSplitTask))
```
```go
func (r *SchedulerTaskHandler) onAskSplit(t *SchedulerAskSplitTask) {
	resp, err := r.SchedulerClient.AskSplit(context.TODO(), t.Region)
	if err != nil {
		log.Error(err)
		return
	}

	aq := &raft_cmdpb.AdminRequest{
		CmdType: raft_cmdpb.AdminCmdType_Split,
		Split: &raft_cmdpb.SplitRequest{
			SplitKey:    t.SplitKey,
			NewRegionId: resp.NewRegionId,
			NewPeerIds:  resp.NewPeerIds,
		},
	}
	r.sendAdminRequest(t.Region.GetId(), t.Region.GetRegionEpoch(), t.Peer, aq, t.Callback)
}
```
后就和接收普通 AdminRequest 一样，把这条 cmd 追加到日志中向其他节点进行同步直到日志被提交(因为 split 本身就是一个 Raft 集群中所有节点都需要的操作，所以需要各节点达成共识)，propose 等待 apply。然后会调用函数`execSplit()`函数来执行 split 操作。

首先 splitKey 是否在目标 region 中和 regionEpoch 是否为最新，因为目标 region 可能已经产生了分裂。

然后我们再来看看 `SplitRequest` 这个结构体，`split_key`是分区的临界 key，然后两个 region 一个是原来的 regionId，并且这个 region 中 的 peerIds 保持不变，一个是新申请的叫做 `new_region_id`，还要为新的分区中的每一个实例申请 peerId，这是一个数组叫做 `new_peer_ids`。这个都是通过上面的`func (r *SchedulerTaskHandler) onAskSplit`向 scheduler 申请到的。
```go
message SplitRequest {
    // This can be only called in internal Raftstore now.
    // The split_key has to exist in the splitting region.
    bytes split_key = 1;
    // We split the region into two. The first uses the origin 
    // parent region id, and the second uses the new_region_id.
    // We must guarantee that the new_region_id is global unique.
    uint64 new_region_id = 2;
    // The peer ids for the new split region.
    repeated uint64 new_peer_ids = 3;
}
```

然后创建新的 Peer,它需要知道它的伙伴(raft组中的成员)是哪些：
```go
cpPeers := make([]*metapb.Peer,0)
for i, pr := range d.Region().Peers {
	cpPeers = append(cpPeers, &metapb.Peer{
		Id: req.Split.NewPeerIds[i],
		StoreId: pr.StoreId,
	})
}
```

oldRegion 和 newRegion 的 RegionEpoch.Version 均自增。
```go
d.Region().RegionEpoch.Version ++
newRegion.RegionEpoch.Version ++
```

修改原来 region 的 `EndKey` 为 `split_key`。
修改 `storeMeta` 信息，把两个 region 重新添加进去(更新)，修改 region state。
```go
d.Region().EndKey = req.Split.SplitKey
d.ctx.storeMeta.regions[req.Split.NewRegionId] = newRegion
d.ctx.storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: d.Region()})
d.ctx.storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: newRegion})
meta.WriteRegionState(kvWB, newRegion, rspb.PeerState_Normal)
meta.WriteRegionState(kvWB, d.Region(), rspb.PeerState_Normal)
```

然后注册新的 peer，并向 router 模块注册自己的存在
```go
newPeer, err := createPeer(d.storeID(), d.ctx.cfg, d.ctx.regionTaskSender, d.ctx.engine, newRegion)
newPeer.peerStorage.SetRegion(newRegion)
d.ctx.router.register(newPeer)
startMsg := message.Msg{
	RegionID: req.Split.NewRegionId,
	Type: message.MsgTypeStart,
}
err = d.ctx.router.send(req.Split.NewRegionId,startMsg)
if err != nil {
	log.Panic(err)
}
```

然后需要刷新 scheduler 的 region 缓存：
```go
d.notifyHeartbeatScheduler(d.Region(), d.peer)
d.notifyHeartbeatScheduler(newRegion, newPeer)
```

最后别忘了处理 proposal 进行返回。split 并不需要修改 KvDB 数据库，主要就是更改 region 的信息，因为改了 region state 就可以视 split 分区完成，所有的 region 共用一个 KvDB 数据库。

