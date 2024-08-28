





## TinyKV中的 wroker

### Raft Worker

* 它持有 `peerSender`的接收端，负责处理输入到 Raft 中的 msg(TinyKV中的所有 Raft 共用一个 Raft Worker)
* Raft Worker会执行一个内嵌 select 的循环，等待接收 `raftCh` 中的 msg，如果收到就调用`HandleMsg`处理消息，处理完毕后，调用`HandleRaftReady`函数

### Store Worker

它持有`storeSender`，接收与 store 有关的信息。它的基本任务有：

* 定期向调度器发送心跳，汇报 store 内的信息，自身的状态
* 定期清理断线 Peer 留下的 Snapshot
* 同时处理 Raft Worker 没有发送成功的 Raft Msg