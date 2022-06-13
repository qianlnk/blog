# SDS简单动态字符串

![](/uploads/upload_3657ee016b9b8969d97ceb3cc63571d0.png)

redis给出4种不同长度字符串的结构

```c
/*不同长度的header 8 16 32 64共4种 都给出了四个成员
len：当前使用的空间大小；alloc去掉header和结尾空字符的最大空间大小
flags:8位的标记 下面关于SDS_TYPE_x的宏定义只有5种 3bit足够了 5bit没有用
buf:这个跟C语言中的字符数组是一样的，从typedef char* sds可以知道就是这样的。
buf的最大长度是2^n 其中n为sdshdr的类型，如当选择sdshdr16，buf_max=2^16。
*/

struct sdshdr8 {
 uint8_t len;
 uint8_t alloc;
 unsigned char flags;
 char buf[];
}
```

**优势**

- **获取字符串长度O(1)**，c语言中获取长度需要遍历字符串。

- **防止缓冲溢出**，bufferoverflow。当sds需要对字符串进行修改时，首先借助于len和alloc检查空间是否满足修改所需的要求，如果空间不够的话，SDS会自动扩展空间，避免了像C字符串操作中的覆盖情况；

- **有效降低内存分配次数**；C字符串在涉及增加或者清除操作时会改变底层数组的大小造成重新分配、sds使用了空间预分配和惰性空间释放机制，说白了就是每次在扩展时是成倍的多分配的，在缩容是也是先留着并不正式归还给OS；

- **二进制安全**：C语言字符串只能保存ascii码，对于图片、音频等信息无法保存，sds是二进制安全的，写入什么读取就是什么，不做任何过滤和限制；

![](/uploads/upload_b295cc4a120fdd7105ad70b0ebd6e333.png)