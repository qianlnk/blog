# 哈希 hash，rehash过程

## 哈希底层实现

![](/uploads/upload_2b3e9ef06a8e12ef35446ebf9602f15c.png)


![](/uploads/upload_e995b8467e9f50e3de757c88083bb130.png)

```c
//哈希节点结构
typedef struct dictEntry {
    void *key;  
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; //下一个节点，哈希冲突时使用开链表
}dictEntry;

//封装的是字典的操作函数指针
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

//哈希结构
typedef struct dictgt {
    dictEntry **table;
    unsigned long size;     //哈希表table的大小
    unsigned long sizemask; //size-1，和哈希值计算一个键在table数组的索引
    unsigned long used;     //已有节点数量
}dictht;

//字典结构
typedef struct dict {
    dictType *type;
    void *prevdata;
    dictht ht[2];
    long rehashidx;
    unsigned long iterators;
```

- 哈希结构

![](/uploads/upload_65dba7de104bae2b8860b4b826cdae2d.png)



如上图展示了一个大小为4的table中的哈希节点情况，其中k1和k0在index=2发生了哈希冲突，进行开链表存在，本质上是先存储的k0，k1放置是发生冲突为了保证效率直接放在冲突链表的最前面，因为该链表没有尾指针。

- 字典结构

![](/uploads/upload_360fe21c4c486f1338451794f79afd89.png)


从源码中看到dict结构体就是字典的定义，包含的成员有type，privdata、ht、rehashidx。其中dictType指针类型的type指向了操作字典的api，理解为函数指针即可，ht是包含2个dictht的数组，也就是字典包含了2个哈希表，rehashidx进行rehash时使用的变量，privdata配合。

## rehash过程

### rehash 条件

**负载因子 = ht[0].used / ht[0].size**

- 扩容

    - 当redis没有执行`BGSAVE`和`BGREWRITEAOF`命令时，且负载因子大于等于1；

    - 当redis在执行`BGSAVE`或`BGREWRITEAOF`命令时，且负载因子大于等于5；

- 缩容
    
    -当负载因子小于0.1时。
    
### 普通rehash过程

1. 为字典ht[1]分配空间

    扩容时ht[1].size = ht[0].used * 2 的第一个2^n 次方。
    缩容时ht[1].size = ht[0].used 的第一个2^n 次方。
![](/uploads/upload_e5f32c3df149d4ee014b78cb3568a0fb.png)


2. 将ht[0]中的数据转移到ht[1]中，转移过程中重新计算键的哈希值和索引值，然后对应放置在ht[1]指定的位置。
![](/uploads/upload_d678eee8b45cbd864278911cb0a42cbd.png)


3. 当ht[0]的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），将ht[0]释放，然后将ht[1]设置成ht[0]，最后为ht[1]分配一个空白哈希表。

![](/uploads/upload_916a2b1275e94ff822fc49cd01093e3c.png)


### 渐进rehash

1. 为ht[1] 分配空间，让字典同时持有ht[0]和ht[1]两个哈希表

2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash 开始。

3. 在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一

4. 当ht[0]中所有数据转移到ht[1]中时，将rehashidx 设置成-1，表示rehash 结束


**在渐进式 rehash 操作过程中，因为同时存在两个哈希表，所以字典的删除，查找，更新操作会在两个哈希表上进行。程序会先尝试在 ht[0] 中寻找目标键值对，如果没有找到则会在 ht[1] 再次进行寻找，然后进行具体操作。但是新增操作只会在 ht[1] 上进行，这保证了 ht[0] 中的已经被清空的单向链表不会新增元素**