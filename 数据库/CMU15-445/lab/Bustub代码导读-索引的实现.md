## **Bustub中的索引机制**

在`catalog`类中，有这样一个成员`index_names_`，它记录了某个表所有的索引。

```c
std::unordered_map<std::string, std::unordered_map<std::string, index_oid_t>> index_names_;
```

我们先从函数`CreateIndex`出发，了解一下Bustub中索引的实现。

`schema`：表中Tuple的格式。

`key_schema`：索引的格式，也就是哪几个字段合在一起建立索引，组成一条Tuple。

`key_attrs`：索引字段在table中的下标。

`keysize`：key的大小

```c
auto CreateIndex(Transaction *txn, const std::string &index_name, const std::string &table_name, const Schema &schema,const Schema &key_schema, const std::vector<uint32_t> &key_attrs, std::size_t keysize, HashFunction<KeyType> hash_function) -> IndexInfo *;
```



首先是一些相关的检查，不必多言。

```c
    if (table_names_.find(table_name) == table_names_.end()) {
      return NULL_INDEX_INFO;
    }

    BUSTUB_ASSERT((index_names_.find(table_name) != index_names_.end()), "Broken Invariant");

    auto &table_indexes = index_names_.find(table_name)->second;
    if (table_indexes.find(index_name) != table_indexes.end()) {
      return NULL_INDEX_INFO;
    }

    auto meta = std::make_unique<IndexMetadata>(index_name, table_name, &schema, key_attr
```



然后就是建立一个B+树(我们在P2实现的)，这个B+树存取索引，其中的Key也是Tuple(由`key_schema`和`key_attars`定义)，**Value就是每一条数据的`RID`，`RID`记录的是每一条Tuple的物理页面的和页面内的编号**，所以索引就建好了。但这里有一个要求，在Bustub中如果要对某个表在字段x1,x2....xn中建立索引，那么这个表中，任意两条的tuple的x1,x2...xn不能同时相同。

```c
    auto index = std::make_unique<BPlusTreeIndex<KeyType, ValueType, KeyComparator>>(std::move(meta), bpm_);

    // Populate the index with all tuples in table heap
    auto *table_meta = GetTable(table_name);
    auto *heap = table_meta->table_.get();
    for (auto tuple = heap->Begin(txn); tuple != heap->End(); ++tuple) {
      index->InsertEntry(tuple->KeyFromTuple(schema, key_schema, key_attrs), tuple->GetRid(), txn);
    }
```



接下来就是把索引添加到`catalog_`中了。

```c
    // Get the next OID for the new index
    const auto index_oid = next_index_oid_.fetch_add(1);

    // Construct index information; IndexInfo takes ownership of the Index itself
    auto index_info =
        std::make_unique<IndexInfo>(key_schema, index_name, std::move(index), index_oid, table_name, keysize);
    auto *tmp = index_info.get();

    // Update internal tracking
    indexes_.emplace(index_oid, std::move(index_info));
    table_indexes.emplace(index_name, index_oid);

    return tmp;
```



索引中保存了整个表的所有数据，所以当某个表有Insert或者删除某个Tuple时，必须也对索引进行Insert或者Delete。

索引建好了，就可以利用索引进行查询了。索引查询也很简单，我们对索引建有迭代器(B+树的迭代器)，我们就可以获取每一条Tuple的`RID`信息，有了这个信息，就可以调用`GetTuple`函数得到相应的tuple了。

```
auto TableHeap::GetTuple(const RID &rid, Tuple *tuple, Transaction *txn, bool acquire_read_lock) -> bool
```



但是我发现，Bustub里面似乎不能够把字符串类型的字段做索引。我们之前在实现B+树的时候，有一个柔型数组：

```c
MappingType array_[1];
```

可见每一个的Key的长度都是固定的。所以貌似来看，Bustub必须保证索引是建立在定长字段上的。

