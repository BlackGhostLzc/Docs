## TinyKV如何增加删除节点(region)

remove 和 add 在`HandleMsg`中是一种 admin request:
```go
case message.MsgTypeRaftCmd:
    raftCMD := msg.Data.(*message.MsgRaftCmd)
    d.proposeRaftCommand(raftCMD.Request, raftCMD.Callback)
```

```go
confChange := eraftpb.ConfChange{
    ChangeType: req.ChangePeer.ChangeType,
    NodeId:     req.ChangePeer.Peer.Id, 
    Context:    data,
}
err = d.RaftGroup.ProposeConfChange(confChange)
```
这个`ProposeConfChange()`会调用`Step()`函数，把日志塞进Raft层进行同步。
1. 调用 `d.RaftGroup.ApplyConfChange()` 方法，修改 raft 内部的 peers 信息，其会调用 3A 中实现的 Add 和 Remove；
2. 修改 region.Peers，是删除就删除，是增加就增加一个 peer。如果删除的目标节点正好是自己本身，那么直接调用 `d.destroyPeer()` 方法销毁自己，并直接返回；
3. 令 `region.RegionEpoch.ConfVer ++` ;
4. 持久化修改后的 Region，写到 kvDB 里面。使用 `meta.WriteRegionState()` 方法。其中 state 要使用 `rspb.PeerState_Normal`，因为其要正常服务请求的；
5. 调用 `d.insertPeerCache()` 或 `d.removePeerCache()` 方法，更新缓存；
6. 更新 scheduler 那里的 region 缓存;

### 删除节点
就假如说，1个集群(只有1个region)总共有5个节点，这些 peerId 依次是 A,B,C,D,E。当上层想要删除节点 E 时，会下发一个 RaftCmdRequest 给集群来同步，在 entry 完成同步之前，集群中仍然是这个 5 个节点，没少。接下来，5 个节点同步完成，相继通过 handleRaftReady 来执行 Remove。对于节点 A B C D 而言，它们会从下到上更改自己认知的 peers 信息，包括 3A 中写的 raft 层、更上层的 Region() 信息、GlobalContext 的 storeMeta 等等。而对于节点 E 而言，当发现要删的节点就是自己时，它就会调用 destroyPeer 来销毁自己。至此，节点 E 脱离集群，同时其余节点认知中也不再存在 E，Remove 执行完毕。

### 增加节点
还是以上面的这个情况为例，假设想在原来集群的基础上加入F,和 Remove 一样，Add 请求被封装为 entry 在 A B C D 中同步，然后在 handleRaftReady 中解封装为 RaftCmdRequest 执行。直到此刻，集群啊还是 A B C D，没有 F。梳理 Add 流程的关键在于，F 何时会出现。
和 Remove 类似，A B C D 会从下至上更改自己认知的 peers 信息，也即，它们认为有 F 了。但不同于 Remove 在 handleRaftReady 完成之后也就完成了，Add 在 handleRaftReady 之后还有一段路要走，因为此时只是现有节点认为有个新节点为 F，但实际上，F 根本就没有创建。
接下来探讨 F 何时创建，我们将视角转到 Leader（假设为 A） 。此时，A 的 peers 中有 F，那么它就会给 F 发送心跳 Msg。当目标收到心跳时，是通过`Raft()`函数进行接收的，并把 Raft Message 交给`SendRaftMessage()`进行路由，交给目标 Raft 模块，可这时，F 根本不存在，所以 `r.router.send()` 会失败，然后把这个任务交给了 Store Worker 进一步处理。
```go
func (rs *RaftStorage) Raft(stream tinykvpb.TinyKv_RaftServer) error {
    for {
        msg, err := stream.Recv()
        if err != nil {
            return err
        }
        rs.raftRouter.SendRaftMessage(msg)
    }
}
func (r *RaftstoreRouter) SendRaftMessage(msg *raft_serverpb.RaftMessage) error {
	regionID := msg.RegionId
	if r.router.send(regionID, message.NewPeerMsg(message.MsgTypeRaftMessage, regionID, msg)) != nil {
		r.router.sendStore(message.NewPeerMsg(message.MsgTypeStoreRaftMessage, regionID, msg))
	}
	return nil
}
```
Store Worker 会把消息交给一个名为 `onRaftMessage()` 的函数，该函数的主要作用是检查。要检查的内容很多， 其中一项就是检查 Msg.To 是否存在。各种检查之后，会调用 `maybeCreatePeer()` 函数，该函数才是真正创造 F 的入口。

```go
func (d *storeWorker) onRaftMessage(msg *rspb.RaftMessage) error {
    regionID := msg.RegionId
    if err := d.ctx.router.send(regionID, message.Msg{Type: message MsgTypeRaftMessage, Data: msg}); err == nil {
        return nil
    }
    ....
    created, err := d.maybeCreatePeer(regionID, msg)
    ....
    _ = d.ctx.router.send(regionID, message.Msg{Type: message.MsgTypeRaftMessage, Data: msg})
    return nil
}
```
接下来看 `maybeCreatePeer()`，直接看最核心的部分：
```go
func (d *storeWorker) maybeCreatePeer(regionID uint64, msg *rspb.RaftMessage) (bool, error) {
    // ...
    peer, err := replicatePeer(
    d.ctx.store.Id, d.ctx.cfg, d.ctx.regionTaskSender, d.ctx.engine, regionID, msg.ToPeer)
    if err != nil {
        return false, err
    }
    // following snapshot may overlap, should insert into regionRanges after
    // snapshot is applied.
    meta.regions[regionID] = peer.Region()
    d.ctx.router.register(peer)
    _ = d.ctx.router.send(regionID, message.Msg{Type: message.MsgTypeStart})
    return true, nil
    // ...
}
```
上述代码才是真正创建 F 并将其注册进集群的，此时，集群才成为 A B C D F。

Add 操作到此位置，F 加入完毕。但还有一个重要问题没有解决，那就是 F 的信息同步问题了。F 刚加进来，什么都不知道，就连 Region() 都是空的（这点可以看 replicatePeer），也就是说 F 所认知的 region 信息和其余四个相比完全落后。因此，我们要更改 F 对 region 的认知，使它和其余四个一致，或者说，和 Leader 一致。
>The leader then will know this Follower has no data (there exists a Log gap from 0 to 5) and it will directly send a snapshot to this Follower

也即，A 即刻会发现 F 日志落后了，随机给它发送 snapshot。当 F 收到 snapshot 之后，会递交给 `HandleRaftReady()` 中应用，而 F.Region() 的更新，就是在 snapshot 的应用中完成的。F 的 `HandleRaftReady()` 收到 snapshot 之后，会调用 `SaveReadyState()` 来应用，而 `SaveReadyState()` 实际是调用 `ApplySnapshot()` 来应用。

