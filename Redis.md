## 基本数据类型
## SDS 动态字符串
### 结构
![sds-内存结构](./images/Redis-sds-1.png)

```c
typedef char *sds;
// __attribute__ 设置内存对齐
// __packed__ 紧凑型，相当于取消了内存对齐
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
// len: buf中已用的内存空间
// alloc: buf总共的空间
// flags: 低3位表示sds类型
// buf: 存储字符的空间
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    * uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

## 特点
- 二进制安全
  可以存储\0，C字符串中以\0为结束符，所以统计字符串时遇到\0就结束，这样导致C字符串无法存储一些二进制数据，sds通过buf数组用来存储字符串，并且在头部利用len变量记录了字符串的实际长度，这样就可以支持存储二进制数据
- 快速遍历字符串
  因为在头部记录了字符串的长度，所以在进行字符串长度统计、分割等操作时都可以快速遍历字符串，或者直接访问len变量就可以得到实际长度
- 避免内存越界
  通过在头部存储了buf数组的总空间大小和已用空间大小，可以保证在做字符串拼接、截取等操作时能有效判断空间是否够用，避免内存越界。c中内存越界例子: 两个连续内存空间存储了两个字符数组进行拼接之后，第二个字符数组的内容被改变了。
- 兼容部分C函数
  sds的buf为字节数组，并且最后一位都会被设置为\0，这样就兼容了C中的一些函数
- 动态扩展内存和预分配内存
  比如sds扩容的策略时，如果需要的内存大小小于1M，则会2倍扩容，否则会多扩容1M，采用这样的预分配策略可以减少很多的内存分配操作,如果内存空间够用，在多次append字符串时可以直接在内存空间中存放数据。
- 惰性释放
  sdsclear()函数用于置空sds字符串，该函数只是将头部len变量设置为0，将buf第一字节设置为\0，而不会立即回收内存空间, 并通过相关api将内存管理交给上层决定。
