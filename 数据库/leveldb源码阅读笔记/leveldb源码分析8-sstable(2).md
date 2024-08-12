## sstable的遍历

这一讲主要介绍是如何遍历sstable的，主要介绍迭代器部分。



首先是如何打开一个sstable文件

对于一个sstable文件，我们先读取它的`Footer` 。然后根据它的`Footer` 知道这个sstable的`metaindex_handle_` 和 `index_handle_` 信息，进而就可以解析整个sstable文件。

```c
Status Table::Open(const Options& options, RandomAccessFile* file,
                   uint64_t size, Table** table) {
  *table = nullptr;
  if (size < Footer::kEncodedLength) {
    return Status::Corruption("file is too short to be an sstable");
  }

  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  Status s = file->Read(size - Footer::kEncodedLength, Footer::kEncodedLength,
                        &footer_input, footer_space);
  if (!s.ok()) return s;

  Footer footer;
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;

  // Read the index block
  BlockContents index_block_contents;
  ReadOptions opt;
  if (options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);

  if (s.ok()) {
    // We've successfully read the footer and the index block: we're
    // ready to serve requests.
    Block* index_block = new Block(index_block_contents);
    Rep* rep = new Table::Rep;
    rep->options = options;
    rep->file = file;
    rep->metaindex_handle = footer.metaindex_handle();
    rep->index_block = index_block;
    rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0);
    rep->filter_data = nullptr;
    rep->filter = nullptr;
    *table = new Table(rep);
    (*table)->ReadMeta(footer);
  }

  return s;
}
```



然后是如何创建一个遍历`table` 的迭代器。这里使用了`TwoLevelIterator` 。

```c
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader, const_cast<Table*>(this), options);
}
```

```c
Iterator* NewTwoLevelIterator(Iterator* index_iter, BlockFunction block_function, void* arg,
                              const ReadOptions& options) {
  return new TwoLevelIterator(index_iter, block_function, arg, options);
}

```

为什么要叫`TwoLevelIterator` 呢？

不仅可以迭代其中存储的对象，它还接受了一个函数**BlockFunction**，可以遍历存储的对象，可见它是专门为**Table定制**的。 我们已经知道各种Block的**存储格式**都是**相同**的，但是各自block data存储的**k/v**又**互不相同**，于是我们就需要一个途径，能够在使用同一个方式**遍历**不同的block时，又能**解析**这些k/v。



`TwoLevelIterator` 类的主要成员变量：

```c
BlockFunction block_function_; // block操作函数  
void* arg_;                    // BlockFunction的自定义参数  
const ReadOptions options_;    // BlockFunction的read option参数  
Status status_;                // 当前状态  
IteratorWrapper index_iter_;   // 遍历block的迭代器  
IteratorWrapper data_iter_;    // May be NULL-遍历block data的迭代器  
// 如果data_iter_ != NULL，data_block_handle_保存的是传递给  
// block_function_的index value，以用来创建data_iter_  
std::string data_block_handle_; 
```



**在分析Seek系函数之前**，有必要先了解下面这几个函数的用途。

```c
void InitDataBlock();  
void SetDataIterator(Iterator*data_iter); 
void SkipEmptyDataBlocksForward();  
void SkipEmptyDataBlocksBackward(); 
```

1. 首先是InitDataBlock()，它是根据`index_iter`来初始化`data_iter`，当定位到新的block时，需要更新data Iterator，指向该block中k/v对的合适位置。这里主要调用了`block_function_` 函数(`BlockReader` )，参数`args_` 是结构体`Table` 的指针，`handle` 是需要定位的`offset` 和 `size` ，`BlockReader` 函数就是根据`Table` (`arg_` )里面的`Rep` 中的`File` 信息以及`handle` 来读取一个完整的Block的。

```c
void TwoLevelIterator::InitDataBlock() {
  if (!index_iter_.Valid()) {
    SetDataIterator(nullptr);
  } else {
    Slice handle = index_iter_.value();
    if (data_iter_.iter() != nullptr &&
        handle.compare(data_block_handle_) == 0) {
      // data_iter_ is already constructed with this iterator, so
      // no need to change anything
    } else {
      Iterator* iter = (*block_function_)(arg_, options_, handle);
      data_block_handle_.assign(handle.data(), handle.size());
      SetDataIterator(iter);
    }
  }
}
```



2. SkipEmptyDataBlocksForward，向前跳过空的datablock。

   实现思路也就是用一个while循环不断判断该data block是否有效，如果无效，那就`index_iter_` 需要调用`Next` 方法，同时调用上面讲到的`InitDataBlock` 函数把`data_iter` 也指向新的data block对象。

```c
void TwoLevelIterator::SkipEmptyDataBlocksForward() {
  while (data_iter_.iter() == nullptr || !data_iter_.Valid()) {
    // Move to next block
    if (!index_iter_.Valid()) {
      SetDataIterator(nullptr);
      return;
    }
    index_iter_.Next();
    InitDataBlock();
    if (data_iter_.iter() != nullptr) data_iter_.SeekToFirst();
  }
}

```



3. SkipEmptyDataBlocksBackward，向后跳过空的datablock



### Seek函数

1. 首先把`index_iter` 定位到指定位置。

> index_iter和data_iter没有本质区别，都是一个个kv键值对，都使用了前缀压缩来节省空间。但不一样的是，index_iter对应的index block 不需要重启点的信息。

2. 然后再把`data_iter_` 跳转到指定的data block，并指向指定的kv键值对。

```c
void TwoLevelIterator::Seek(const Slice& target) {
  index_iter_.Seek(target);
  InitDataBlock();
  if (data_iter_.iter() != nullptr) data_iter_.Seek(target);
  SkipEmptyDataBlocksForward();
}
```



### SeekToFirst函数

思路和实现也和上面差不多，就是移动`index_iter` 和`data_iter` 来实现的。

```
void TwoLevelIterator::SeekToFirst() 
{  
    index_iter_.SeekToFirst();  
    InitDataBlock();              // 根据index iter设置data iter  
    if (data_iter_.iter() != NULL)data_iter_.SeekToFirst();  
    SkipEmptyDataBlocksForward(); // 调整iter，跳过空的block  
} 
```





