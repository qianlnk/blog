
# 有序集合zset 及 跳跃表

ZSet结构同时包含一个字典和一个跳跃表，跳跃表按score从小到大保存所有集合元素。字典保存着从member到score的映射。这两种结构通过指针共享相同元素的member和score，不会浪费额外内存。

![](/uploads/upload_22a98b79bac16b43b31dbfc4c4f40db7.png)

```c
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```


**ZSet中跳表的实现细节**

- 随机层数的实现原理

跳表是一个概率型的数据结构，元素的插入层数是随机指定的。Willam Pugh在论文中描述了它的计算过程如下：

- 指定节点最大层数 MaxLevel，指定概率 p， 默认层数 lvl 为1 

- 生成一个0~1的随机数r，若r<p，且lvl<MaxLevel ，则lvl ++

- 重复第 2 步，直至生成的r >p 为止，此时的 lvl 就是要插入的层数。

redis中生成随机层数源码：

```c
//redis.h
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

//src/z_set.c
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**节点删除**

![](/uploads/upload_4401a179dbe5b9c57d9dd6692a4a3fd5.png)


跳表元素的删除与普通链表相比增加了索引层的判断，如果结点是非索引结点则正常处理，如果结点是索引结点那边需要进行索引层结点的处理。

## 跳跃表

如何确定最高索引层数m呢？

如果一个链表有 n 个结点，如果每两个结点取出一个结点建立索引，那么第一级索引的结点数是 n/2，第二级索引的结点数是n/4，以此类推第 m 级索引的结点数为 n/(2^m)，前面说过最高层结点数为2，因此存在关系：
![](/uploads/upload_8133f85d89c15ee6da1d265fdbbcdfec.png)


算上最底层的原始链表，整个跳表的高度为h=logn(底数为2)，每一层需要遍历的结点数是d，那么整个过程的复杂度为:O(d*logn)。

d表明了层间结点的稀疏程度，也就是每隔2个结点选取索引结点、或者每隔3个结点选取索引结点，每个4个结点选取索引结点......