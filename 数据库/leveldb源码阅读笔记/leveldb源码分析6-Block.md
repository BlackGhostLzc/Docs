## Block
上一讲的BlockBuilder是在构建Block用到的结构体，它不提供访问元素的接口，只提供添加entry的一些接口。当我们在SST中构建好Block后，我们需要一些接口，或者是迭代器访问Block中的元素。

```c
class Iter;
const char* data_;
size_t size_;
uint32_t restart_offset_;  // Offset in data_ of restart array
bool owned_;               // Block owns data_[]
```

其中`Iter` 是Block的迭代器，我们来看看`Iter` 是如何实现的。

```c
class Block::Iter : public Iterator {
 private:
  const Comparator* const comparator_;
  const char* const data_;       // underlying block contents, block的内容
  uint32_t const restarts_;      // Offset of restart array (list of fixed32)
  uint32_t const num_restarts_;  // Number of uint32_t entries in restart array

  uint32_t current_;        // current_ is offset in data_ of current entry.  >= restarts_ if !Valid
  uint32_t restart_index_;  // Index of restart block in which current_ falls
  std::string key_;
  Slice value_;
  Status status_;
....
....
```



## 迭代器访问的接口

### Next方法

```c
void Next() override {
    assert(Valid());
    ParseNextKey();
}
```

```c
bool ParseNextKey() {
    current_ = NextEntryOffset();
    const char* p = data_ + current_;
    const char* limit = data_ + restarts_;  // Restarts come right after data
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }

    // Decode next entry
    uint32_t shared, non_shared, value_length;
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
    if (p == nullptr || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
      key_.resize(shared);
      key_.append(p, non_shared);
      value_ = Slice(p + non_shared, value_length);
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {
        ++restart_index_;
      }
      return true;
    }
  }
```
首先来看`DecodeEntry`函数，根据上面的结构图，我们看到，首先解析出`shared` 、`non_shared` 、`value_length` 的长度，然后`p` 指针来到了`non_shared` 的 key 的部分。然后`key_.resize(shared);key_.append(p, non_shared);` 由于key是上一个key的值，`resize` 和`append` 后就变成了当前的key的值。

然后我们就可以移动重启点`restart_index_` 的位置了。

这一部分的代码在清楚了Block的结构之后还是很显而易见的。

```c
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared, uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;
  *shared = reinterpret_cast<const uint8_t*>(p)[0];
  *non_shared = reinterpret_cast<const uint8_t*>(p)[1];
  *value_length = reinterpret_cast<const uint8_t*>(p)[2];
  if ((*shared | *non_shared | *value_length) < 128) {
    // Fast path: all three values are encoded in one byte each
    p += 3;
  } else {
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  return p;
}
```



### Prev方法

如何进行向后遍历呢？Prev操作分为两步：

1. 首先回到current_之前的重启点(这个重启点的offset要严格小于当前`current ` 的offset)
2. _然后再向后直到current_ ,(不断调用`ParseNextKey` 和 `NextEntryOffset` 函数)

```c
void Prev() override {
    assert(Valid());
    // Scan backwards to a restart point before current_
    const uint32_t original = current_;
    while (GetRestartPoint(restart_index_) >= original) {
      if (restart_index_ == 0) {
        // No more entries
        current_ = restarts_;
        restart_index_ = num_restarts_;
        return;
      }
      restart_index_--;
    }

    SeekToRestartPoint(restart_index_);
    do {
      // Loop until end of current entry hits the start of original entry
    } while (ParseNextKey() && NextEntryOffset() < original);
  }

```



### SeekToFirst/Last函数

这两个函数都很简单，借助于前面的SeekToResartPoint函数就可以完成。



### Seek函数

跳到指定的target(Slice)，这里主要是二分查找和线性查找的思路
1. 找到key < target的最后一个重启点，典型的二分查找算法，代码就不再贴了。
2. 找到后，跳转到重启点，其索引由left指定，这是前面二分查找到的结果。
3. 自重启点线性向下，直到遇到key>= target的k/v对。

```c
void Seek(const Slice& target) override {
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    int current_key_compare = 0;

    if (Valid()) {
      // If we're already scanning, use the current position as a starting
      // point. This is beneficial if the key we're seeking to is ahead of the
      // current position.
      current_key_compare = Compare(key_, target);
      if (current_key_compare < 0) {
        // key_ is smaller than target
        left = restart_index_;
      } else if (current_key_compare > 0) {
        right = restart_index_;
      } else {
        // We're seeking to the key we're already at.
        return;
      }
    }

    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      const char* key_ptr =
          DecodeEntry(data_ + region_offset, data_ + restarts_, &shared,
                      &non_shared, &value_length);
      if (key_ptr == nullptr || (shared != 0)) {
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }

    // We might be able to use our current position within the restart block.
    // This is true if we determined the key we desire is in the current block
    // and is after than the current key.
    assert(current_key_compare == 0 || Valid());
    bool skip_seek = left == restart_index_ && current_key_compare < 0;
    if (!skip_seek) {
      SeekToRestartPoint(left);
    }
    // Linear search (within restart block) for first key >= target
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }
```







