## TinyKV的启动流程
我们先看`./kv/main.go`的`main`函数。
```go
var storage storage.Storage
if conf.Raft {
	storage = raft_storage.NewRaftStorage(conf)
}
...
if err := storage.Start(); err != nil {
	log.Fatal(err)
}
server := server.NewServer(storage)
```
`server`的结构很简单
```go
type Server struct {
	storage storage.Storage
	// (Used in 4A/4B)
	Latches *latches.Latches        // 用于实现并发控制
	// coprocessor API handler, out of course scope
	copHandler *coprocessor.CopHandler
}
```
下面是`RaftStorage`的结构：
```go
type RaftStorage struct {
	engines *engine_util.Engines
	config  *config.Config

	node          *raftstore.Node
	snapManager   *snap.SnapManager
	raftRouter    *raftstore.RaftstoreRouter
	raftSystem    *raftstore.Raftstore
	resolveWorker *worker.Worker
	snapWorker    *worker.Worker

	wg sync.WaitGroup
}
```
随着调用了`RaftStorage`的`Start()`函数。
```go
func (rs *RaftStorage) Start() error {
	cfg := rs.config
    // 创建一个与PD模块进行交互的客户端
	schedulerClient, err := scheduler_client.NewClient(strings.Split(cfg.SchedulerAddr, ","), "")  
	if err != nil {
		return err
	}
    
    // raftRouter是用来接受信息进行路由处理的，比如客户端的命令和raft间的信息可以通过这个router路由到该TinyKV节点上的具体的raft(region)实例。
    // 这个RaftstoreRouter有两个函数用来接受信息，一个是SendRaftMessage，一个是SendRaftCommand，它们都调用了函数router.Send来进行路由。
    // 这个router.Send函数只是简单地把信息传递给router.peerSender的一个channel
    // 同时raftWorker的raftCh指向了router.peerSender这个channel
    // raftWorker调用run函数不断判断raftCh中有没有需要处理的msg，如果有就调用HanleMsg处理所有msg，处理完后再调用HandleRaftReady推进raft的状态
	rs.raftRouter, rs.raftSystem = raftstore.CreateRaftstore(cfg)

	rs.resolveWorker = worker.NewWorker("resolver", &rs.wg)
	resolveSender := rs.resolveWorker.Sender()
	resolveRunner := newResolverRunner(schedulerClient)
	rs.resolveWorker.Start(resolveRunner)

	rs.snapManager = snap.NewSnapManager(filepath.Join(cfg.DBPath, "snap"))
	rs.snapWorker = worker.NewWorker("snap-worker", &rs.wg)
	snapSender := rs.snapWorker.Sender()
	snapRunner := newSnapRunner(rs.snapManager, rs.config, rs.raftRouter)
	rs.snapWorker.Start(snapRunner)

	raftClient := newRaftClient(cfg)

    // 这个trans用来转发 raft 间的内部消息，比如心跳，追加日志，选举投票这些
	trans := NewServerTransport(raftClient, snapSender, rs.raftRouter, resolveSender)

	rs.node = raftstore.NewNode(rs.raftSystem, rs.config, schedulerClient)
	err = rs.node.Start(context.TODO(), rs.engines, trans, rs.snapManager)
	if err != nil {
		return err
	}

	return nil
}
```
接下来调用`Node.Start()`：
```go
func (n *Node) Start(ctx context.Context, engines *engine_util.Engines, trans Transport, snapMgr *snap.SnapManager) error {
    // 首先判断有没有被分配 storeId(注意区分storeId和regionId以及clusterId，storeId可以看做一个tinykv实例的标识)
	storeID, err := n.checkStore(engines)
	if err != nil {
		return err
	}
	if storeID == util.InvalidID {
        // 如果没有被分配 storeId 就要像PD模块申请一个storeId并向其注册自己的存在
		storeID, err = n.bootstrapStore(ctx, engines)
	}
	if err != nil {
		return err
	}
	n.store.Id = storeID

    // 需要有至少1个region，如果没有region，再向PD申请region
	firstRegion, err := n.checkOrPrepareBootstrapCluster(ctx, engines, storeID)
	if err != nil {
		return err
	}
	newCluster := firstRegion != nil
	if newCluster {
		log.Infof("try bootstrap cluster, storeID: %d, region: %s", storeID, firstRegion)
		newCluster, err = n.BootstrapCluster(ctx, engines, firstRegion)
		if err != nil {
			return err
		}
	}

    // 负责接收 peer 发送过来的 SchedulerRegionHeartbeatTask，向PD模块报告自己 region 的 leader、region size等信息
    // 负责接收 store worker 发送过来的 SchedulerStoreHeartbeatTask,发送给PD，报告store的capacity、used size等信息
    // 负责接收 peer 发送过来的 SchedulerAskSplitTask，发送 ask split 给 scheduler，请求 scheduler 为分裂出的 region 分配 peers。
	err = n.schedulerClient.PutStore(ctx, n.store)
	if err != nil {
		return err
	}
	if err = n.startNode(engines, trans, snapMgr); err != nil {
		return err
	}

	return nil
}
```
然后调用`Node.StartNode()`函数,这个函数主要调用的是`Raftstore.start()`：
```go
func (bs *Raftstore) start(
	meta *metapb.Store,
	cfg *config.Config,
	engines *engine_util.Engines,
	trans Transport,
	schedulerClient scheduler_client.Client,
	snapMgr *snap.SnapManager) error {

	y.Assert(bs.workers == nil)
	// TODO: we can get cluster meta regularly too later.
	if err := cfg.Validate(); err != nil {
		return err
	}
	err := snapMgr.Init()
	if err != nil {
		return err
	}
	wg := new(sync.WaitGroup)
	bs.workers = &workers{
		splitCheckWorker: worker.NewWorker("split-check", wg),
		regionWorker:     worker.NewWorker("snapshot-worker", wg),
		raftLogGCWorker:  worker.NewWorker("raft-gc-worker", wg),
		schedulerWorker:  worker.NewWorker("scheduler-worker", wg),
		wg:               wg,
	}
	bs.ctx = &GlobalContext{
		cfg:                  cfg,
		engine:               engines,
		store:                meta,
		storeMeta:            newStoreMeta(),
		snapMgr:              snapMgr,
		router:               bs.router,
		trans:                trans,
		schedulerTaskSender:  bs.workers.schedulerWorker.Sender(),
		regionTaskSender:     bs.workers.regionWorker.Sender(),
		splitCheckTaskSender: bs.workers.splitCheckWorker.Sender(),
		raftLogGCTaskSender:  bs.workers.raftLogGCWorker.Sender(),
		schedulerClient:      schedulerClient,
		tickDriverSender:     bs.tickDriver.newRegionCh,
	}
	regionPeers, err := bs.loadPeers()
	if err != nil {
		return err
	}

	for _, peer := range regionPeers {
		bs.router.register(peer)
	}
	bs.startWorkers(regionPeers)
	return nil
}
```
我们看到调用了`loadPeers()`,随后也向`router`模块注册了自己的存在，也就可以接收来自外部的msg/cmd和内部的msg了。

`loadPeers()`函数就是不断从KV数据库中读出所有region的元信息，为了更快速找到对应的元数据信息，tinykv在这里为其Key增加了一个统一的前缀，比如prefix + regionId + suffix。这样就能够快速定位到所有的region元数据信息。
此外对应的value是一个`RegionLocalState`的结构体，所以还需要`Unmarshal()`函数进行反序列化。
```go
/// loadPeers loads peers in this store. It scans the db engine, loads all regions and their peers from it
/// WARN: This store should not be used before initialized.
func (bs *Raftstore) loadPeers() ([]*peer, error) {
	// Scan region meta to get saved regions.
	startKey := meta.RegionMetaMinKey
	endKey := meta.RegionMetaMaxKey
	ctx := bs.ctx
	kvEngine := ctx.engine.Kv
	storeID := ctx.store.Id

	var totalCount, tombStoneCount int
	var regionPeers []*peer

	t := time.Now()
	kvWB := new(engine_util.WriteBatch)
	raftWB := new(engine_util.WriteBatch)
	err := kvEngine.View(func(txn *badger.Txn) error {
		// get all regions from RegionLocalState
		it := txn.NewIterator(badger.DefaultIteratorOptions)
		defer it.Close()
		for it.Seek(startKey); it.Valid(); it.Next() {
			item := it.Item()
			if bytes.Compare(item.Key(), endKey) >= 0 {
				break
			}
			regionID, suffix, err := meta.DecodeRegionMetaKey(item.Key())
			if err != nil {
				return err
			}
			if suffix != meta.RegionStateSuffix {
				continue
			}
			val, err := item.Value()
			if err != nil {
				return errors.WithStack(err)
			}
			totalCount++
			localState := new(rspb.RegionLocalState)
			err = localState.Unmarshal(val)
			if err != nil {
				return errors.WithStack(err)
			}
			region := localState.Region
			if localState.State == rspb.PeerState_Tombstone {
				tombStoneCount++
				bs.clearStaleMeta(kvWB, raftWB, localState)
				continue
			}

			peer, err := createPeer(storeID, ctx.cfg, ctx.regionTaskSender, ctx.engine, region)
			if err != nil {
				return err
			}
			ctx.storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: region})
			ctx.storeMeta.regions[regionID] = region
			// No need to check duplicated here, because we use region id as the key
			// in DB.
			regionPeers = append(regionPeers, peer)
		}
		return nil
	})
	if err != nil {
		return nil, err
	}
	kvWB.MustWriteToDB(ctx.engine.Kv)
	raftWB.MustWriteToDB(ctx.engine.Raft)

	log.Infof("start store %d, region_count %d, tombstone_count %d, takes %v",
		storeID, totalCount, tombStoneCount, time.Since(t))
	return regionPeers, nil
}
```