## TinyKV两阶段提交
这部分可以参考TinySql(这可以看成是TinyKV的客户端)的实现，入口函数是`tikvTxn.Commit`，函数在文件`store/tikv/txn.go`中。我们这里只讲解一下两阶段提交的大致流程，了解一下比较工业级的二阶段锁大致是如何用代码实现的，不会太深入到琐碎的细节中。


```go
func (txn *tikvTxn) Commit(ctx context.Context) error {
	.....
	err = committer.execute(ctx)
	return errors.Trace(err)
}
```

这里我们看到调用了`commiter.execute()`函数。这个函数比较关键，我把所有的代码都贴在了下面。

```go
// execute executes the two-phase commit protocol.
func (c *twoPhaseCommitter) execute(ctx context.Context) (err error) {
	// 做一些清理工作，execute 函数退出的时候要么成功提交，要么提交失败。(提交失败会做清理，比如回滚)
	defer func() {
		// Always clean up all written keys if the txn does not commit.
		c.mu.RLock()
		committed := c.mu.committed
		undetermined := c.mu.undeterminedErr != nil
		c.mu.RUnlock()
		// 没有成功提交
		if !committed && !undetermined {
			c.cleanWg.Add(1)
			go func() {
				cleanupKeysCtx := context.WithValue(context.Background(), txnStartKey, ctx.Value(txnStartKey))
				err := c.cleanupKeys(NewBackoffer(cleanupKeysCtx, cleanupMaxBackoff).WithVars(c.txn.vars), c.keys)
				if err != nil {
					logutil.Logger(ctx).Info("2PC cleanup failed",
						zap.Error(err),
						zap.Uint64("txnStartTS", c.startTS))
				} else {
					logutil.Logger(ctx).Info("2PC clean up done",
						zap.Uint64("txnStartTS", c.startTS))
				}
				c.cleanWg.Done()
			}()
		}
		c.txn.commitTS = c.commitTS
	}()

	prewriteBo := NewBackoffer(ctx, PrewriteMaxBackoff).WithVars(c.txn.vars)
	// 第一个阶段 prewrite
	err = c.prewriteKeys(prewriteBo, c.keys)
	if err != nil {
		logutil.Logger(ctx).Debug("2PC failed on prewrite",
			zap.Error(err),
			zap.Uint64("txnStartTS", c.startTS))
		return errors.Trace(err)
	}

	// prewrite 完成，获取 commitTs
	commitTS, err := c.store.getTimestampWithRetry(NewBackoffer(ctx, tsoMaxBackoff).WithVars(c.txn.vars))
	if err != nil {
		logutil.Logger(ctx).Warn("2PC get commitTS failed",
			zap.Error(err),
			zap.Uint64("txnStartTS", c.startTS))
		return errors.Trace(err)
	}

	// check commitTS
	if commitTS <= c.startTS {
		err = errors.Errorf("conn %d Invalid transaction tso with txnStartTS=%v while txnCommitTS=%v",
			c.connID, c.startTS, commitTS)
		logutil.BgLogger().Error("invalid transaction", zap.Error(err))
		return errors.Trace(err)
	}
	c.commitTS = commitTS
	if err = c.checkSchemaValid(); err != nil {
		return errors.Trace(err)
	}

	// 如果 prewrite 时间过长，直接回滚退出，提交失败
	if c.store.oracle.IsExpired(c.startTS, kv.MaxTxnTimeUse) {
		err = errors.Errorf("conn %d txn takes too much time, txnStartTS: %d, comm: %d",
			c.connID, c.startTS, c.commitTS)
		return err
	}

	commitBo := NewBackoffer(ctx, CommitMaxBackoff).WithVars(c.txn.vars)
	// 进行 commit 阶段
	err = c.commitKeys(commitBo, c.keys)
	if err != nil {
		if undeterminedErr := c.getUndeterminedErr(); undeterminedErr != nil {
			logutil.Logger(ctx).Error("2PC commit result undetermined",
				zap.Error(err),
				zap.NamedError("rpcErr", undeterminedErr),
				zap.Uint64("txnStartTS", c.startTS))
			err = errors.Trace(terror.ErrResultUndetermined)
		}
		if !c.mu.committed {
			logutil.Logger(ctx).Debug("2PC failed on commit",
				zap.Error(err),
				zap.Uint64("txnStartTS", c.startTS))
			return errors.Trace(err)
		}
		logutil.Logger(ctx).Debug("got some exceptions, but 2PC was still successful",
			zap.Error(err),
			zap.Uint64("txnStartTS", c.startTS))
	}
	return nil
}
```

## prewrite阶段
TODO()

## commit阶段
```go
func (c *twoPhaseCommitter) commitKeys(bo *Backoffer, keys [][]byte) error {
	return c.doActionOnKeys(bo, actionCommit{}, keys)
}
```
下面节选了`doActionOnKeys()`函数:
```go
	firstIsPrimary := bytes.Equal(keys[0], c.primary())
	_, actionIsCommit := action.(actionCommit)
	_, actionIsCleanup := action.(actionCleanup)
	if firstIsPrimary && (actionIsCommit || actionIsCleanup) {
		// primary should be committed/cleanup first
		err = c.doActionOnBatches(bo, action, batches[:1])
		if err != nil {
			return errors.Trace(err)
		}
		batches = batches[1:]
	}
	if actionIsCommit {
		// Commit secondary batches in background goroutine to reduce latency.
		// The backoffer instance is created outside of the goroutine to avoid
		// potential data race in unit test since `CommitMaxBackoff` will be updated
		// by test suites.
		secondaryBo := NewBackoffer(context.Background(), CommitMaxBackoff).WithVars(c.txn.vars)
		go func() {
			e := c.doActionOnBatches(secondaryBo, action, batches)
			if e != nil {
				logutil.BgLogger().Debug("2PC async doActionOnBatches",
					zap.Uint64("conn", c.connID),
					zap.Stringer("action type", action),
					zap.Error(e))
			}
		}()
	}
```
Percolator的二阶段的提交阶段是可以分为两个小阶段的：
1. 首先需要提交 Primary Key，其他 Secondary Key 的提交必须等待
2. 等到 Primary Key 成功提交了，那么其他 Secondary Keys 的提交可以并发执行

上面代码也体现了这个特点。
`if firstIsPrimary && (actionIsCommit || actionIsCleanup){}`这部分是 Primary Key 的提交逻辑。等到执行完了，然后代码会执行到`if actionIsCommit {}`，这是 Secondary Keys 的提交，而这个提交可以交给一个协程去执行，甚至失败了也没有关系。