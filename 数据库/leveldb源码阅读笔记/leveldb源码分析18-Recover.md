## **Manifest文件**

在介绍Recover机制前，我们先要看一下leveldb中的`Manifest` 文件。

我们知道，每次进行Compaction生成或删除SSTable文件时，都会生成一个新的版本。LevelDB 采用了增量式的存储方式，用 VersionEdit 记录每一个 Version 对上一个 Version 的变化情况。

Manifest 文件专用于记录版本信息。一个 Manifest 文件中，包含了多条Session Record。一个 Session Record 记录了从上一个版本至该版本的变化情况。每一个 Session Record 对应一个 VersionEdit，不断调用`Apply` 函数最终可以得到最新的版本信息。

问题：那么这样是不是会出现Manifest文件过大的问题呢？

> 其中第一条 Session Record 包含当时LevelDB的全量版本信息。这是压缩Manifest文件的方式。

`descriptor_log_` 就是Manifest文件。

```c
descriptor_log_ = new log::Writer(descriptor_file_, manifest_size);
```



在第一次执行`LogAndApply` 函数（这个函数接受一个 VersionEdit，产生一个新的 Version）的时候，如果没有`descriptor_log_` 这个文件，那么就需要把当前版本的全量版本信息写入Manifest文件(这是上面替代的压缩Mainifest的方式)

> 有一个参数`save_manifest` 就是判断这一方面逻辑的。

```c
// LogAndApply 函数
if (descriptor_log_ == nullptr) {
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == nullptr);
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
      descriptor_log_ = new log::Writer(descriptor_file_);
      s = WriteSnapshot(descriptor_log_);
    }
  }
```

其中`WriteSnapshot` 函数就是把当前版本`version_` 以 log 形式写入Manifest文件。



那么既然需要Manifest文件才可以恢复数据库，那么刚`DB::Open` 的时候，如何知道Manifest文件的路径呢？

> 有一个文件叫做CURRENT，里面存放了Manifest文件的路径。





## **Recover机制**

当我们打开一个数据库的时候，我们需要考虑一些什么，怎么样才能恢复到数据库原来的状态呢？

1. 首先需要恢复到原来的`version` 版本  (Manifest文件)
2. 然后需要恢复没写入磁盘的内存中的`memtable` (WAL的log文件)



理清了Recover的逻辑，我们先看如何恢复到最新的Version版本。

`DBImpl::Recover -> VersionSet::Recover`  ，主要逻辑在`VersionSet::Recover` 中，主要逻辑就是不断获取Manifest中的`VersionEdit` ，然后不断`Apply` 到`builder` 中得到最新的version。

```c
log::Reader reader(file, &reporter, true /*checksum*/,
                       0 /*initial_offset*/);
    Slice record;
    std::string scratch;
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      ++read_records;
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (s.ok()) {
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      if (s.ok()) {
        builder.Apply(&edit);
      }

      if (edit.has_log_number_) {
        log_number = edit.log_number_;
        have_log_number = true;
      }

      if (edit.has_prev_log_number_) {
        prev_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
```



然后就开始恢复`memtable` ，要恢复哪一些日志文件呢？

1. 首先获取该版本的所有文件，筛选出所有的Log文件(number >= min_log)
2. 把所有Log文件进行排序，再对所有文件进行`RecoverLogFile` 恢复日志

```c
for (size_t i = 0; i < filenames.size(); i++) {
    if (ParseFileName(filenames[i], &number, &type)) {
      expected.erase(number);
      if (type == kLogFile && ((number >= min_log) || (number == prev_log)))
        logs.push_back(number);
    }
  }
  if (!expected.empty()) {
    char buf[50];
    std::snprintf(buf, sizeof(buf), "%d missing files; e.g.",
                  static_cast<int>(expected.size()));
    return Status::Corruption(buf, TableFileName(dbname_, *(expected.begin())));
  }

  // Recover in the order in which the logs were generated
  std::sort(logs.begin(), logs.end());
  for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], (i == logs.size() - 1), save_manifest, edit,
                       &max_sequence);
    if (!s.ok()) {
      return s;
    }
    versions_->MarkFileNumberUsed(logs[i]);
  }

```









