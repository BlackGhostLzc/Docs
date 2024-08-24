## TinyKV中的Raft

TinyKV中的 Raft 模块是和 Log 模块、RawNode 模块一起配合工作的。



## log模块

```
snapshot/first.....applied....committed....stabled.....last
--------|------------------------------------------------|
	                          log entries
```

TinyKV中的 log 是由 `RaftLog`进行管理的，它可以看成是两部分，其中一部分是 snapshot 部分，这部分已经在磁盘中被删除了；第二部分的话是所有没有被删除的日志的部分，也就是上面的 log entries。其中这部分有一部分是已经持久化在磁盘上，有一部分还驻留在内存上(等待被持久化)，这个`stabled`就是用来区分这两种的。

```go
type RaftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage
	committed uint64   // 需要被持久化，storage.InitialState()
	applied uint64     // 初始时赋值为 firstIndex - 1
	stabled uint64     
	entries []pb.Entry
	pendingSnapshot *pb.Snapshot
}

// RaftLog 的相关接口
func newLog(storage Storage) *RaftLog
func (l *RaftLog) unstableEntries() []pb.Entry
func (l *RaftLog) nextEnts() (ents []pb.Entry)
func (l *RaftLog) LastIndex() uint64
// 调用 storage.FirstIndex() 可以知道当前截断的 index
func (l *RaftLog) FirsIndex() uint64
func (l *RaftLog) Term(i uint64) (uint64, error)
func (l *RaftLog) maybeCompact()
```



## RawNode模块

RawNode 作为一个信息传递的模块，主要就是上层信息的下传和下层信息的上传。

```go
func (rn *RawNode) Tick()   // 驱动Raft的计时
func (rn *RawNode) Propose(data []byte) error  // Leader接收到来的 entry,并把它放入日志,向其他成员同步
func (rn *RawNode) Step(m pb.Message) error    // 驱动raft去处理消息
func (rn *RawNode) HasReady() bool
func (rn *RawNode) Advance(rd Ready)
```

主要的是一个`Ready`结构体：

```go
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// 需要被持久化的日志
	Entries []pb.Entry
	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot
	// 待应用的日志
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MessageType_MsgSnapshot message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	Messages []pb.Message
}
```

