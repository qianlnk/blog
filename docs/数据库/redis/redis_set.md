# 集合 set

set使用intset和hashtable存储的。

使用intset存储的条件

- 集合中存储元素都是整型
- 集合元素数量不超过512个

## intset

```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

![](/uploads/upload_edff2390cf3c5488f480cbad27512fe1.png)


redis set存储过程

- 检查set是否存在不存在则创建一个set结合。
- 根据传入的set集合一个个进行添加，添加的时候需要进行内存压缩。
- setTypeAdd执行set添加过程中会判断是否进行编码转换。
    - 如果能够转成int的对象（isObjectRepresentableAsLongLong），那么就用intset保存。
    - 如果用intset保存的时候，如果长度超过512（REDIS_SET_MAX_INTSET_ENTRIES）就转为hashtable编码。
    - 其他情况统一用hashtable进行存储。