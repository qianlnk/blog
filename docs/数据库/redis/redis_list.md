# 列表 list

![](/uploads/upload_14ff786ffce23417328d33faea646825.png)

在版本3.2之前，Redis 列表list使用两种数据结构作为底层实现：

- 压缩列表ziplist
- 双向链表linkedlist
    因为双向链表占用的内存比压缩列表要多， 所以当创建新的列表键时， 列表会优先考虑使用压缩列表， 并且在有需要的时候， 才从压缩列表实现转换到双向链表实现。
    
**压缩列表转化成双向链表条件**
创建新列表时 redis 默认使用 redis_encoding_ziplist 编码， 当以下任意一个条件被满足时， 列表会被转换成 redis_encoding_linkedlist 编码：

- 试图往列表新添加一个字符串值，且这个字符串的长度超过 server.list_max_ziplist_value （默认值为 64 ）。
- ziplist 包含的节点超过 server.list_max_ziplist_entries （默认值为 512 ）。

注意：这两个条件是可以修改的，在 redis.conf 中：
    
    list-max-ziplist-value 64 
    list-max-ziplist-entries 512
    
## linkedList

标准双向链表，Node节点包含prev和next指针，可以进行双向遍历；
还保存了 head 和 tail 两个指针，因此，对链表的表头和表尾进行插入的复杂度都为 (1) —— 这是高效实现 LPUSH 、 RPOP、 RPOPLPUSH 等命令的关键。

## ziplist

**ziplist 是一个特殊的双向链表**

特殊之处在于：没有维护双向指针:prev next；而是存储上一个 entry的长度和 当前entry的长度，通过长度推算下一个元素在什么地方。

牺牲读取的性能，获得高效的存储空间，因为(简短字符串的情况)存储指针比存储entry长度 更费内存。这是典型的“时间换空间”。


```
area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                         ZIPLIST_ENTRY_TAIL
```

![](/uploads/upload_ac1a147ba5a47ebe6b5f474f224ae36e.png)


```c
typedef struct zlentry {    // 压缩列表节点
    unsigned int prevrawlensize, prevrawlen;    // prevrawlen是前一个节点的长度，prevrawlensize是指prevrawlen的大小，有1字节和5字节两种
    unsigned int lensize, len;  // len为当前节点长度 lensize为编码len所需的字节大小
    unsigned int headersize;    // 当前节点的header大小
    unsigned char encoding; // 节点的编码方式
    unsigned char *p;   // 指向节点的指针
} zlentry;

void zipEntry(unsigned char *p, zlentry *e) {   // 根据节点指针返回一个enrty
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);    // 获取prevlen的值和长度
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);  // 获取当前节点的编码方式、长度等
    e->headersize = e->prevrawlensize + e->lensize; // 头大小
    e->p = p;
}
```

**ziplist连锁更新问题**

因为在ziplist中，每个zlentry都存储着前一个节点所占的字节数，而这个数值又是变长编码的。假设存在一个压缩列表，其包含e1、e2、e3、e4…..，e1节点的大小为253字节，那么e2.prevrawlen的大小为1字节，如果此时在e2与e1之间插入了一个新节点e_new，e_new编码后的整体长度（包含e1的长度）为254字节，此时e2.prevrawlen就需要扩充为5字节；如果e2的整体长度变化又引起了e3.prevrawlen的存储长度变化，那么e3也需要扩…….如此递归直到表尾节点或者某一个节点的prevrawlen本身长度可以容纳前一个节点的变化。其中每一次扩充都需要进行空间再分配操作。删除节点亦是如此，只要引起了操作节点之后的节点的prevrawlen的变化，都可能引起连锁更新。

连锁更新在最坏情况下需要进行N次空间再分配，而每次空间再分配的最坏时间复杂度为O(N)，因此连锁更新的总体时间复杂度是O(N^2)。
即使涉及连锁更新的时间复杂度这么高，但它能引起的性能问题的概率是极低的：需要列表中存在大量的节点长度接近254的zlentry。

**ziplist节点结构**

![](/uploads/upload_6e3f0f1ed14f999c29057571956b4b6b.png)


## quicklist

3.2版本之后

![](/uploads/upload_6b65c2c463dc8525357d1f7ed4c8dd2f.png)

大致结构，非源码：

```c
// 快速列表
struct quicklist {
    quicklistNode* head;
    quicklistNode* tail;
    long count; // 元素总数
    int nodes; // ziplist 节点的个数
    int compressDepth; // LZF 算法压缩深度
    ...
}
// 快速列表节点
struct quicklistNode {
    quicklistNode* prev;
    quicklistNode* next;
    ziplist* zl; // 指向压缩列表
    int32 size; // ziplist 的字节总数
    int16 count; // ziplist 中的元素数量
    int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
    ...
}

struct ziplist_compressed {
    int32 size;
    byte[] compressed_data;
}

struct ziplist {
    ...
}
```

源码在这：

```c
/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporarry decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 12 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev; //上一个node节点
    struct quicklistNode *next; //下一个node
    unsigned char *zl;            //保存的数据 压缩前ziplist 压缩后压缩的数据
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

/* quicklist is a 32 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor. */
typedef struct quicklist {
    quicklistNode *head; //头结点
    quicklistNode *tail; //尾节点
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes *///负数代表级别，正数代表个数
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off *///压缩级别
} quicklist;
```

**quicklistNode结构代表quicklist的一个节点，其中各个字段的含义如下：**

- prev: 指向链表前一个节点的指针。
- next: 指向链表后一个节点的指针。
- zl: 数据指针。如果当前节点的数据没有压缩，那么它指向一个ziplist结构；否则，它指向一个quicklistLZF结构。
- sz: 表示zl指向的ziplist的总大小（包括zlbytes, zltail, zllen, zlend和各个数据项）。需要注意的是：如果ziplist被压缩了，那么这个sz的值仍然是压缩前的ziplist大小。
- count: 表示ziplist里面包含的数据项个数。这个字段只有16bit。稍后我们会一起计算一下这16bit是否够用。
- encoding: 表示ziplist是否压缩了（以及用了哪个压缩算法）。目前只有两种取值：2表示被压缩了（而且用的是LZF压缩算法），1表示没有压缩。
- container: 是一个预留字段。本来设计是用来表明一个quicklist节点下面是直接存数据，还是使用ziplist存数据，或者用其它的结构来存数据（用作一个数据容器，所以叫container）。但是，在目前的实现中，这个值是一个固定的值2，表示使用ziplist作为数据容器。
- recompress: 当我们使用类似lindex这样的命令查看了某一项本来压缩的数据时，需要把数据暂时解压，这时就设置recompress=1做一个标记，等有机会再把数据重新压缩。
- attempted_compress: 这个值只对Redis的自动化测试程序有用。我们不用管它。
- extra: 其它扩展字段。目前Redis的实现里也没用上。

**quicklistLZF结构表示一个被压缩过的ziplist。其中：**

- sz: 表示压缩后的ziplist大小。
- compressed: 是个柔性数组（flexible array member），存放压缩后的ziplist字节数组。

**真正表示quicklist的数据结构是同名的quicklist这个struct：**

- head: 指向头节点（左侧第一个节点）的指针。
- tail: 指向尾节点（右侧第一个节点）的指针。
- count: 所有ziplist数据项的个数总和。
- len: quicklist节点的个数。
- fill: 16bit，ziplist大小设置，存放list-max-ziplist-size参数的值。
- compress: 16bit，节点压缩深度设置，存放list-compress-depth参数的值。

### 内存二次管理平衡问题

- 每个quicklist节点上的ziplist越短，则内存碎片越多。内存碎片多了，有可能在内存中产生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个quicklist节点上的ziplist只包含一个数据项，这就蜕化成一个普通的双向链表了。

- 每个quicklist节点上的ziplist越长，则为ziplist分配大块连续内存空间的难度就越大。有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间分配给ziplist的情况。这同样会降低存储效率。这种情况的极端是整个quicklist只有一个节点，所有的数据项都分配在这仅有的一个节点的ziplist里面。这其实蜕化成一个ziplist了。

配置参数`list-max-ziplist-size`:

当取正值的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项。

当取负值的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度。这时，它只能取-1到-5这五个值，每个值含义如下：

- 5: 每个quicklist节点上的ziplist大小不能超过64 Kb。（注：1kb => 1024 bytes）
- 4: 每个quicklist节点上的ziplist大小不能超过32 Kb。
- 3: 每个quicklist节点上的ziplist大小不能超过16 Kb。
- 2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（-2是Redis给出的默认值）
- 1: 每个quicklist节点上的ziplist大小不能超过4 Kb。