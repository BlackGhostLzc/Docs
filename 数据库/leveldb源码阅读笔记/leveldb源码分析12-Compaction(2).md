## **Compaction** 

上一篇介绍了Compaction的一些基本概念和原理，这一篇，我们开始研究Compaction中的代码，力求清楚代码中的细节。



## **Minor Compaction**

Minor Compaction主要通过`DBImpl::CompactionMemTable`方法实现：

`CompactionMemTable`方法首先调用`DBImpl::WriteLevel0Table`方法将Immutable MemTable转储为SSTable，由于该方法需要使用当前的Version信息，因此在调用前后增减了当前Version的引用计数以避免其被回收。接着，通过`VersionSet::LogAndApply`方法将增量的版本更新VersionEdit写入Manifest（其中prev log number已被弃用，不需要再关注）。如果上述操作都成功完成，则可以释放对Immutable MemTable的引用，并通过`RemoveObsoleteFiles`方法回收不再需要保留的文件。

```c
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // Commit to the new state
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
    RemoveObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
}

```

接下来看一下`WriteLevel0Table` 的代码。

实现也比较简单，获取了需要转储的MemTable的迭代器，并传给`BuildTable`方法。`BuildTable`方法会通过`TableBuilder`来构造SSTable文件然后写入。 

```c
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}

```

在LST-Tree的基本概念中，Minor Compaction只需要将Immutable MemTable全量转储为SSTable，并将其推至level-0即可。而LevelDB对这一步骤进行了优化，在`DBImpl::WriteLevel0Table` 函数中，调用了`PickLevelForMemTableOutput` ,也就是Immutable MemTable全量转储为SSTable不一定处于level 0。

我们来看`PickLevelForMemTableOutput` 的代码：

1. 目标level不能超过配置`config::kMaxMemCompactLevel`中限制的最大高度（默认为2）
2. 目标level的文件不能与该sst相重叠
3. 该sst不能与level + 1中的sst重叠得过多，计算方式`GetOverlappingInputs` 

```c
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key,
                                        const Slice& largest_user_key) {
  int level = 0;
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    // Push to next level if there is no overlap in next level,
    // and the #bytes overlapping in the level after that are limited.
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      if (level + 2 < config::kNumLevels) {
        // Check that file does not overlap too many grandparent bytes.
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
}

```



## **Size Compaction**

在`DBImpl::BackgroundCompaction()` 中，如果要执行Size Compaction(或者Seek Compaction)，首先要执行`versions_->PickCompaction()` 函数，找到level层需要被合并的sst文件(1个)。

> 注意，在Size Compaction中，对于level 0层级的合并(level 0 sst之间可能会有重叠)，输入文件可能不止1个sst文件，要添加所有与c->inputs_[0]重叠的sst作为输入文件。

这里的文件是如何选取的呢？(什么依据，什么是`compact_pointer_` )

```c
icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0)
```

> `compact_pointer_` 后的第一个文件作为 Compaction 对象，即本层上一次 Compaction 区间之后的文件.

`PickCompaction` 函数最后调用了`SetupOtherInputs` 函数，这个函数主要是找到level + 1 层级中与input的sst文件相重叠的所有sst文件。

```c
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != nullptr);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level + 1 < config::kNumLevels);
    c = new Compaction(options_, level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(options_, level);
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return nullptr;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // Files in level 0 may overlap each other, so pick up all overlapping ones
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }

  SetupOtherInputs(c);

  return c;
}
```

在完成了`PickCompaction` 后，首先判断`inputs_[1]` 文件列表是否为空，如果为空就直接把文件移动到下一层，不需要进行多路归并的合并操作。(这段代码体现在`BackgroundCompaction()  c->c->IsTrivialMove()` 中）

随后调用`DoCompactionWork` 函数。

首先来看调用的是:

```c
Iterator* input = versions_->MakeInputIterator(compact->compaction);
```

```c
Iterator* VersionSet::MakeInputIterator(Compaction* c) {
  ReadOptions options;
  options.verify_checksums = options_->paranoid_checks;
  options.fill_cache = false;

  // Level-0 files have to be merged together.  For other levels,
  // we will make a concatenating iterator per level.
  // TODO(opt): use concatenating iterator for level-0 if there is no overlap
  const int space = (c->level() == 0 ? c->inputs_[0].size() + 1 : 2);
  Iterator** list = new Iterator*[space];
  int num = 0;
  for (int which = 0; which < 2; which++) {
    if (!c->inputs_[which].empty()) {
      if (c->level() + which == 0) {
        const std::vector<FileMetaData*>& files = c->inputs_[which];
        for (size_t i = 0; i < files.size(); i++) {
          list[num++] = table_cache_->NewIterator(options, files[i]->number,
                                                  files[i]->file_size);
        }
      } else {
        // Create concatenating iterator for the files from this level
        list[num++] = NewTwoLevelIterator(
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
      }
    }
  }
  assert(num <= space);
  Iterator* result = NewMergingIterator(&icmp_, list, num);
  delete[] list;
  return result;
}
```

`MakeInputIterator` 返回的是MergeIterator，包含了对`c->inputs_`两层文件的迭代器。
它的实现也相当值的研究，首先，如果是level 0层级的sstable，由于不同sst之间可能存在着重叠，所以每一个sst都需要一个迭代器；而对于level > 0层级的sst，由于整体文件是有序的，所以只需要一个迭代器。关于迭代器的结构下面会进行讲解。

最后返回一个`MergingIterator` ,到此，level n和 level n+1的sst文件就已经可以使用一个**统一的、有序的** 迭代器进行访问了。

再回到`DoCompactionWork` 的代码：

1. 判断当前是否有需要Minor Compaction的Immutable MemTable，如果有则让出任务，先进行Minor Compaction（该过程需要加锁）。

```c
while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
    // Prioritize immutable compaction work
    if (has_imm_.load(std::memory_order_relaxed)) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != nullptr) {
        CompactMemTable();
        // Wake up MakeRoomForWrite() if necessary.
        background_work_finished_signal_.SignalAll();
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }
    .................
```

2. 通过`ShouldStopBefore`方法估算当前SSTable大小，并判断其是否超过了`max_file_size`的限制，如果超过了则通过`FinishCompactionOutputFile`完整对当前SSTable的写入。

```c
//接上
 Slice key = input->key();
 if (compact->compaction->ShouldStopBefore(key) && compact->builder != nullptr) {
     status = FinishCompactionOutputFile(compact, input);
     if (!status.ok()) {
        break;
     }
 }
```



3. 接下来我们就开始遍历所有需要进行合并的所有`InternalKey` ，并判断是否丢弃这个`InternalKey` ，什么情况下需要丢弃这个`InternalKey` 呢?

   3.1 对于多次出现的user key，我们只关心最后写入的值 or >snapshot的值

   > 我们需要保留一个小于等于 snapshot 的版本(如果有的话)。也就是对于重复的key，我们只需要保存最新的(遍历到的第1个)。大于compact->smallest_snapshot的版本都需要保存。这里也就是把snapshot过低的版本(snapshot number < compact->smallest_snapshot的版本)全部给删除掉。

   3.2 如果是删除key && <= snapshot 并且 更高层没有该key，那么也可以忽略

   > 对于`kTypeDeletion`类型，虽然可以丢弃在smallest_snapshot前的key，但是还需要保证在更高的level中没有该UserKey，否则在查询时，在当前level失配后会在下层中找到该UserKey的更旧的版本。

```c
bool drop = false;
if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
} else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) !=
              0) {
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }

      if (last_sequence_for_key <= compact->smallest_snapshot) {
        drop = true;  // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        drop = true;
 }
```

