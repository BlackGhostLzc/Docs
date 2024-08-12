## 编码
Leveldb使用了很多VarInt型编码，典型的如后面将涉及到的各种key。其中的编码、解码函数分为VarInt和FixedInt两种。int32和int64操作都是类似的。

### Encode
1. FixedInt编码：直接把32位整数存储在内存中。
>小端：低位放到低字节地址中，高位放在高字节地址中
```c
void EncodeFixed32(char* buf, uint32_t value)
{
  if (port::kLittleEndian) {
    memcpy(buf, &value,sizeof(value));
  } else {
    buf[0] = value & 0xff;
    buf[1] = (value >> 8)& 0xff;
    buf[2] = (value >> 16)& 0xff;
    buf[3] = (value >> 24)& 0xff;
  }
}
```

2. VarInt编码：
首先看代码：
```c
char* EncodeVarint32(char* dst, uint32_t v)
{
  unsigned char* ptr =reinterpret_cast<unsigned char*>(dst);
  static const int B = 128;
  if (v < (1<<7)) {
    *(ptr++) = v;
  } else if (v < (1<<14)){
    *(ptr++) = v | B;
    *(ptr++) = v>>7;
  } else if (v < (1<<21)){
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)){
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}

// 对于uint64，直接循环
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  unsigned char* ptr =reinterpret_cast<unsigned char*>(dst);
  while (v >= B) {
    *(ptr++) = (v & (B-1)) |B;
    v >>= 7;
  }
  *(ptr++) =static_cast<unsigned char>(v);
  returnreinterpret_cast<char*>(ptr);
}
```
对于32位整数，当用VarInt编码时，可能需要1、2、3、4、5个字节进行编码。
如当如果整数 v 小于 128（1<<7），则使用一个字节直接存储该整数。
如果整数 v 大于等于 128 且小于 16384（1<<14），则使用两个字节存储。第一个字节的最高位设置为1（| B），表示后面还有一个字节，然后将 v 的低7位存储在第一个字节的低7位中，将 v 的高7位存储在第二个字节中。
如果整数 v 大于等于 16384 且小于 2097152（1<<21），则使用三个字节存储。依次类推，直到五个字节。
64位的整数编码处理思路也是一样的。


### Decode
1. Fixed Int的Decode、
```c
inline uint32_t DecodeFixed32(const char* ptr)
{
  if (port::kLittleEndian) {
    uint32_t result;
    // gcc optimizes this to a plain load
    memcpy(&result, ptr,sizeof(result));
    return result;
  } else {
    return((static_cast<uint32_t>(static_cast<unsigned char>(ptr[0])))
        |(static_cast<uint32_t>(static_cast<unsigned char>(ptr[1])) <<8)
        | (static_cast<uint32_t>(static_cast<unsignedchar>(ptr[2])) << 16)
        |(static_cast<uint32_t>(static_cast<unsigned char>(ptr[3])) <<24));
  }
```

再来看看VarInt的解码，很简单，依次读取1byte，直到最高位为0的byte结束，取低7bit，作(<<7)移位操作组合成Int。看代码.
```c
const char* GetVarint32Ptr(const char* p,
                           const char* limit, 
                           uint32_t* value)
{
  if (p < limit) {
    uint32_t result =*(reinterpret_cast<const unsigned char*>(p));
    if ((result & 128) == 0) {
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p,limit, value);
}

const char* GetVarint32PtrFallback(const char* p,
                                   const char* limit,
                                   uint32_t* value)
{
  uint32_t result = 0;
  for (uint32_t shift = 0; shift<= 28 && p < limit; shift += 7) {
    uint32_t byte =*(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) { // More bytes are present
      result |= ((byte & 127)<< shift);
    } else {
      result |= (byte <<shift);
      *value = result;
      returnreinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```



## 总结

整个的编码解码部分并不是leveldb的重点，但可以看出，leveldb为了提升效率，节约空间还是下了很多功夫的。

