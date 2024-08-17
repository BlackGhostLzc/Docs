## 1.简介
获取实验指导书：[Lab Checkpoint 0: networking warmup](https://vixbob.github.io/cs144-web-page/assignments/lab0.pdf)
个人CS144实验地址：[github](https://github.com/BlackGhostLzc/CS144.git)



## 2.telnet手动访问网页

Telnet协议是TCP/IP协议族中的一员，是Internet远程登陆服务的标准协议和主要方式。它为用户提供了在本地计算机上完成远程主机工作的 能力。在终端使用者的电脑上使用`telnet`程序，用它连接到服务器。终端使用者可以在`telnet`程序中输入命令，这些命令会在服务器上运行，就像直接在服务器的控制台上输入一样。

使用 `telnet cs144.keithw.org http`命令以连接远程网页服务器，之后在终端键入以下内容:
```
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close
```
之后看到远程服务器会返回网页的内容并展示再终端上。




## 3. 实现webget.cc
实验指导书上也有提示，需要借助`libsponge`库中的`TCPSocket`和`Address`两个类来实现。

注意：

1. HTTP头部每一行都以`\r\n`结尾，而不是`\n`
2. 需要包含`Connection: close`
3. 借助`eof`函数接收服务器的发送数据

其余部分按照上面的`telnet`命令仿写即可。
```c
void get_URL(const string &host, const string &path) {
    // Your code here.
    
    Address addr(host, "http");
    TCPSocket http_socket;
    
    http_socket.connect(addr);
    http_socket.write("GET " + path + " HTTP/1.1\r\n");
    http_socket.write("Host: " + host + "\r\n");
    http_socket.write("Connection: close\r\n");
    http_socket.write("\r\n");

    while(!http_socket.eof()){
        cout<<http_socket.read();
    }

    http_socket.close();
    // You will need to connect to the "http" service on
    // the computer whose name is in the "host" string,
    // then request the URL path given in the "path" string.

    // Then you'll need to print out everything the server sends back,
    // (not just one call to read() -- everything) until you reach
    // the "eof" (end of file).
}
```



## 实现ByteStream

要求实现一个在内存中的有序可靠字节流。

注意：

* 字节流可以从写入端写入，并以相同的顺序，从读取端读取
* 字节流是有限的，写入者可以终止写入
* 缓冲区是有大小限制的，缓冲区满的时候，写入者不允许再写入
* ByteStream流的字节可能比缓冲区要大的多
* 单线程环境，不考虑竞态条件

我们考虑用`deque`数据结构来实现这个字节流。
需要为ByteStream添加数据成员：
```c
// byte_stream.hh文件
std::deque<char> dq_;
size_t capacity_;
size_t read_size_;     //总共读取的字节数
size_t write_size_;    //总共写入的字节数
bool end_input_;       //能否再写
```


具体函数也不难实现：
```c
// byte_stream.cc文件
#include "byte_stream.hh"

// Dummy implementation of a flow-controlled in-memory byte stream.

// For Lab 0, please replace with a real implementation that passes the
// automated checks run by `make check_lab0`.

// You will need to add private members to the class declaration in `byte_stream.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

ByteStream::ByteStream(const size_t capacity):dq_(), capacity_(capacity), read_size_(0), write_size_(0),end_input_(0) { 
    
}

size_t ByteStream::write(const string &data) {
    if(end_input_){
        return 0;
    }

    // 把 data 写进 dq_中
    size_t write_sz = std::min(data.size(), capacity_ - dq_.size());
    write_size_ += write_sz;

    for(size_t i = 0; i < write_sz; i++){
        dq_.push_back(data[i]);
    }

    return write_sz;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    // dq_中的内容不会被清除
    size_t pop_size = min(len, dq_.size());
    return string(dq_.begin(), dq_.begin() + pop_size);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) { 
    //dq_中的内容会被清除
    size_t pop_size = min(len, dq_.size());
    read_size_ += pop_size;
    for(size_t i = 0; i < pop_size; i++){
        dq_.pop_front();
    }
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    std::string ret = peek_output(len);
    pop_output(len);
    return ret;
}

void ByteStream::end_input() {
    end_input_ = true;
}

bool ByteStream::input_ended() const { 
    return end_input_;    
}

size_t ByteStream::buffer_size() const { 
    return dq_.size();
}

bool ByteStream::buffer_empty() const { 
    return dq_.size() == 0;
}

bool ByteStream::eof() const { 
    return dq_.size() == 0 && input_ended();    
}

size_t ByteStream::bytes_written() const { 
    return write_size_;
}

size_t ByteStream::bytes_read() const { 
    return read_size_;
}

size_t ByteStream::remaining_capacity() const { 
    return capacity_ - dq_.size();    
}

```


