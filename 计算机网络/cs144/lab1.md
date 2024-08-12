## 实验目的
- 背景介绍
在这个实验中和下一个实验中，我们需要实现TCP receiver。它接收数据报，把它们转成可靠的字节流，以供socket读取。
TCP sender把字节流分成不同的segments，每个segment都封装成数据报。但是网络传送可能会丢失，重复或失序，这就需要TCP receiver重新组合，还原最初的字节流。
- 实验目的
完成一个数据结构`StreamReassembler`。它接收一些子串（字节流），以及每个子串第一个字节所在字节流的索引。
`StreamReassembler`有一个成员`ByteStream`，只要`StreamReassembler`知道了字节流的下一个字节，就会把数据写入`ByteStream`中。




## 实现思路

<img src="https://s1.ax1x.com/2022/10/19/xsG8nP.png" alt="capacity.png" style="zoom:67%;" />
如上图所示，`capacity`是整个缓存的大小，也就是`ByteStream`的大小，蓝色部分表示已经写入并被`ByteStream`读出来的部分；绿色部分表示已经写入`ByteStream`还没有被读的部分；红色部分是需要我们在一个buffer中缓存起来的部分，就是接收到的字节流，但这一部分字节流只是已经被缓存，但还没有重组，等到连续时一并写入`ByteSream`中。
我们还记得，也就是`ByteStream`中有下面几个成员：

```c
size_t capacity_;
size_t read_size_;     //总共读取的字节数
size_t write_size_;    //总共写入的字节数
bool end_input_;       //能否再写
```

所以就是说：

* 上图中的first unread是`read_size_`
* 上图的first unassembled是`write_size_`，表示第一个可以写到`ByteStream`中的字节
* 上图的first unacceptable是`read_size_ + capacity_`

所以有以下几种可能性：

1. 如果子字节流的`index`落在first unacceptable之后，那么这个substring应该被丢弃。
2. 如果子字节流全部落在[0, first unassembled - 1]中，那么这个substring已经写入了`ByteStream`中，也应该丢弃
3. 除此之外，应该截断子字节流，使之完全落在[first unassembled - 1, first unacceptable - 1]中

完成了上述操作后，应该处理字节流区间重复的问题，必须要保证`set<Segment> _buffer`中每个`Segment`都必须彼此不重复，具体思路见代码。



## 代码实现
首先定义`Segment`结构体。
```c
// stream_reassembler.hh
struct Segment{
  public:
    size_t idx_;
    std::string data_;
  
    Segment():idx_(0),data_(""){}
    Segment(size_t idx, std::string data):idx_(idx), data_(data){}

    bool operator<(const Segment& seg) const {
      return this->idx_ < seg.idx_;
    }
};
```
为`StreamReassembler`添加成员：
```c
std::set<Segment> _buffer;
bool _eof;
size_t _eof_idx;
size_t _unassembled_bytes; 


void handle_substring(const std::string &data, const uint64_t index);
void handle_overlap(Segment& seg);
void adjustment(Segment& seg, const std::set<Segment>::iterator &it);
void buffer_erase(const std::set<Segment>::iterator &it);
void buffer_insert(Segment& seg);
```

>  上面的 `_buffer` 只存储位于上图中红色的区间`Segment`，红色区间的大小就为`_unassembled_bytes`。
>
>  `_eof_idx`就是当最后一个结束符号的索引，当 `_output`写的字节数(`bytes_written()`)等于 `_eof_idx` 时，应该结束输入，调用 `_output.end_input()`函数。


```c
#include "stream_reassembler.hh"

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}
using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity) : 
    _output(capacity), 
    _capacity(capacity), 
    _buffer(),
    _eof(false),
    _eof_idx(0),
    _unassembled_bytes(0)
{

}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.


void StreamReassembler::handle_substring(const string &data, const size_t index){
    
    auto seg = Segment{index, data};

    // 范围是[read_size_ , read_size_ + _capacity - 1],第一个不可以被写入的索引是 read_size_ + _capacity
    //      [index, index + data.length() - 1]

    // seg落在可写区域的外面 ok
    if(seg.idx_ >= _output.bytes_read() + _capacity){
        return;
    }

    // seg.data_右端如果超出了右边界
    if(seg.idx_ < _output.bytes_read() + _capacity &&
       seg.idx_ + seg.data_.length() - 1 >= _output.bytes_read() + _capacity){
        // [seg.idx_, read_size_ + _capacity - 1]
        seg.data_ = seg.data_.substr(0, _output.bytes_read() + _capacity - seg.idx_);
    }

    // 如果seg.data_都已经写入了ByteSream中 ok
    if(seg.idx_ + seg.data_.length() - 1 < _output.bytes_written()){
        return;
    }

    // 如果seg.data_有一部分已被写入，另一部分没有被写入
    if(seg.idx_ < _output.bytes_written() && 
       seg.idx_ + seg.data_.length() - 1 >= _output.bytes_written()){
        seg.data_ = seg.data_.substr(_output.bytes_written() - seg.idx_);
        seg.idx_ = _output.bytes_written();
    }

    if(_buffer.empty()){
        buffer_insert(seg);
        return;
    }

    handle_overlap(seg);
}



void StreamReassembler::adjustment(Segment& seg, const std::set<Segment>::iterator& it){
    // [it_l , it_r]
    size_t it_l = it->idx_;
    size_t it_r = it->idx_ + it->data_.length() - 1;
    // [seg_l, seg_r]
    size_t seg_l = seg.idx_;
    size_t seg_r = seg.idx_ + seg.data_.length() - 1;

    // 1. 如果 seg 全包含 it 的范围     
    // _buffer去除 it
    if(seg_l <= it_l && seg_r >= it_r){
        return;
    }

    // 2. 如果it全包含 seg 的范围
    // seg 变为 it, _buffer删除 it 
    if(it_l <= seg_l && it_r >= seg_r){
        seg.idx_ = it_l;
        seg.data_ = it->data_;
        return;
    }

    // 3. 如图下：
    /*
        seg:           _________
        it:        _______
    */
   
    if(seg_l >= it_l && seg_r > it_r){
        seg.data_ = it->data_ + seg.data_.substr(it->idx_ + it->data_.length() - seg.idx_);
        seg.idx_ = it->idx_;
        return;
    }
    


    // 4. 如下图：
     /*
        seg:      _________
        it:            ________
    */
    
    if(it_l > seg_l && it_r >= seg_r){
        seg.data_ = seg.data_.substr(0, it->idx_ - seg.idx_) + it->data_;
        return;
    }

}


void StreamReassembler::handle_overlap(Segment& seg){
    // 保证插入的和原本存在的没有重叠部分
    auto it = _buffer.begin();
    for(; it != _buffer.end();){
        size_t it_l = it->idx_;
        size_t it_r = it->idx_ + it->data_.length() - 1;

        size_t seg_l = seg.idx_;
        size_t seg_r = seg.idx_ + seg.data_.length() - 1;

        // 两条线段有重叠
        if((it_l >= seg_l && it_l <= seg_r) || (seg_l >= it_l && seg_l <= it_r)){
            adjustment(seg, it);
            buffer_erase(it++);
        }else{
            it++;
        }
    }

    buffer_insert(seg);
}

void StreamReassembler::buffer_erase(const std::set<Segment>::iterator& it){
    // 如果删除了，那么 _unassembled_bytes 就要减
    _unassembled_bytes -= it->data_.length();
    _buffer.erase(it);
}

void StreamReassembler::buffer_insert(Segment& seg){
    _unassembled_bytes += seg.data_.length();
    _buffer.insert(seg);
}


void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {

    if(!data.empty()){
        handle_substring(data, index);
    }

    // 检查 _buffer 中是否有连续的可写入 ByteStream 的部分
    // 第一个可以写入的字节是 written_size_
    while(!_buffer.empty() && _buffer.begin()->idx_ == _output.bytes_written()){
        auto it = _buffer.begin();
        _output.write(it->data_);
        buffer_erase(it);
    }

    if(eof){
        _eof = true;
        _eof_idx = index + data.length();
    }

    if(_eof && _output.bytes_written() == _eof_idx){
        _output.end_input();
    }

}


size_t StreamReassembler::unassembled_bytes() const { 
    return _unassembled_bytes;
}


bool StreamReassembler::empty() const { 
    return _buffer.empty();
}


```