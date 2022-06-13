# 内存回收

- 有过期时间
- 没有过期时间
- 热点数据
- 冷点数据

![](/uploads/upload_5d7f98fa788a4c142b912f7ffcef39d3.png)

## 过期键删除

```
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

redisDB存储结构图：

![](/uploads/upload_698ec7425e0b76f351957e0e78f55ffe.png)

Redis本质上就是一个大的key-value，key就是字符串，value有是几种对象：字符串、列表、有序列表、集合、哈希等，这些key-value都是存储在redisDb的dict中的。

过期字典expires和键值对空间dict存储的内容并不完全一样，过期字典expires的key是指向Redis对应对象的指针，其value是long long型的unix时间戳，前面的EXPIRE和PEXPIRE相对时长最终也会转换为时间戳，来看下过期字典expires的结构。

![](/uploads/upload_9b11ddace14a824419dc079fb1c0932c.png)

### 删除策略

- 定时删除：在设置键的过期时间的同时，创建定时器，让定时器在键过期时间到来时，即刻执行键值对的删除；

- 定期删除：每隔特定的时间对数据库进行一次扫描，检测并删除其中的过期键值对；

- 惰性删除：键值对过期暂时不进行删除，至于删除的时机与键值对的使用有关，当获取键时先查看其是否过期，过期就删除，否则就保留；

三种策略都有各自的优缺点：定时删除对内存使用率有优势，但是对CPU不友好，惰性删除对内存不友好，如果某些键值对一直不被使用，那么会造成一定量的内存浪费，定期删除是定时删除和惰性删除的折中。

![](/uploads/upload_f4cec4f3d338ddc3e27b1c36c2535b10.png)

定期删除的实现：

- 该算法是个自适应的过程，当过期的key比较少时那么就花费很少的cpu时间来处理，如果过期的key很多就采用激进的方式来处理，避免大量的内存消耗，可以理解为判断过期键多就多跑几次，少则少跑几次；

- 由于Redis中有很多数据库db，该算法会逐个扫描，本次结束时继续向后面的db扫描，是个闭环的过程；

- 定期删除有快速循环和慢速循环两种模式，主要采用慢速循环模式，其循环频率主要取决于server.hz，通常设置为10，也就是每秒执行10次慢循环定期删除，执行过程中如果耗时超过25%的CPU时间就停止；

- 慢速循环的执行时间相对较长，会出现超时问题，快速循环模式的执行时间不超过1ms，也就是执行时间更短，但是执行的次数更多，在执行过程中发现某个db中抽样的key中过期key占比 **低于25%** 则跳过；

主体意思：定期删除是个自适应的闭环并且概率化的抽样扫描过程，过程中都有执行时间和cpu时间的限制，如果触发阈值就停止，可以说是尽量在不影响对客户端的响应下润物细无声地进行的。

## 内存淘汰机制

为了保证Redis的安全稳定运行，设置了一个max-memory的阈值，那么当内存用量到达阈值，新写入的键值对无法写入，此时就需要内存淘汰机制。

- noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错；

- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中移除最近最少使用的 key；

- allkeys-random：当内存不足以容纳新写入数据时，在键空间中随机移除某个 key；

- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key；

- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key；

- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除；


## 过期删除与内存淘汰机制的关系

过期健删除策略强调的是对过期健的操作，如果有健过期而内存足够，Redis不会使用内存淘汰机制来腾退空间，这时会优先使用过期健删除策略删除过期健。

内存淘汰机制强调的是对内存数据的淘汰操作，当内存不足时，即使有的健没有到达过期时间或者根本没有设置过期也要根据一定的策略来删除一部分，腾退空间保证新数据的写入。