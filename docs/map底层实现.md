# map底层实现

采用哈希查找表，有一个key通过哈希函数得到哈希值，再由哈希值将key对应到不同的桶（bucket）中，当哈希映射到同一个桶中时，采用链表解决冲突。

```go
// A header for a Go map.
type hmap struct {
	count     int      //map中元素个数
	flags     uint8    //状态标识，比如是否被写，是否迁移等，因为map不是线程安全的，所以操作时需要判断flags 
	B         uint8    // log_2 of # of buckets (最多可放 loadFactor * 2^B 个元素。 即 6.5*2^B)
	noverflow uint16   //overflow buckets近似数
	hash0     uint32   //hash seed，随机哈希种子，可以减少哈希碰撞
    buckets  unsafe.Pointer //存储数据的buckets数组指针，大小2^B，如果count=0，可能是nil
	oldbuckets unsafe.Pointer //一半大小老的bucket数组，只有在扩容过程中非nil
    nevacuate  uintptr        //扩容进度标识，小于此地址的buckets已迁移完成
	extra *mapextra    // 可以减少GC扫描，当key和value都内联（inline）时，就会使用该字段
}
```

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
//如果key和value都不包含指针，并行内联（<=128字节），使用extra来存储overflow bucket，这样可以避免GC扫描整个map。
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```

```go
// A bucket for a Go map. map bucket 结构
type bmap struct {
	//tophash哈希值的高8位，如果tophash[0] < minTopHash，表示这个桶的搬迁状态
	tophash [bucketCnt]uint8
	//以下字段没有显示定义，但编译时编译器会自动加上。
	//keys 每个桶最多转8个元素
	//values
	//overflow pointer 发生碰撞时使用
}
```

根据哈希函数将key生成一个哈希值，其中低位用来判断桶的位置，高位用来确定在桶中那个cell，

- 低位哈希值就是哈希值的低B位。hmap中的B，比如B位5，2^5=32,即该map有32个桶，只需要取哈希值的低5位就可以知道这个key落在那个桶中。

- 高位哈希值是指哈希值的高8位，即（bmap中的tophash），根据tophash来确定key在桶中的哪个位置。每个桶可以存储8个key-value,为了优化对齐，go采用了key放在一起，value放在一起的存储方式.

- 当不同的key根据哈希得到tophash和地位hash都一样时，发生了哈希碰撞，这时就体现了overflow pointer的作用了，桶溢出时就存储在overflow。如果overflow也溢出了，那就给overflow bucket建一个overflow bucket，用指针连起来就形成了链式结构。

## 扩容

### 扩容条件

1. 装填因子是否大于6.5

    装填因子 = 元素个数 / 桶个数
    
    大于6.5时，说明快要填满，需要扩容。
    
2. overflow bucket是否太多

    当bucket数量小于2^15， 但overflow bucket大于桶数量时。
    当bucket数量大于等于2^15， 但overflow bucket大于2^15。
    
### 扩容方式

装填因子过大时直接翻倍：B+1；扩容后需要重新计算每一项数据在新hash中的位置。每一次访问旧的buckets时就迁移一部分到新的bucket里，知道完全迁移，旧bucket被GC回收。

## 查找

![](/uploads/upload_5b265b34f14904524659bd80396ab9c3.png)

1. 根据key计算哈希值。
2. 根据hash值的低B位确定在哪个bucket。
3. 根据hash值的高8位，确定在桶中的存储位置。
4. 当前bucket未找到则查找对应的overflow bucket。
5. 对应位置有数据则对比完整的hash值，确定是否是要查找的数据。
6. 如果有旧bucket（在扩容状态），则优先查旧bucket。

## 插入

![](/uploads/upload_1172b54ab789406c7d5a33d1e22497d3.png)

1. 1. 2. 3. 同上
2. key存在则更新，不存在则插入。


## 删除

![](/uploads/upload_3e48eb50d0d39b8abca139c7db943805.png)

1. 1. 2. 3. 同上
2. 若找到对应的key，把对应的tophash里打上空的标记。

## 搬迁

![](/uploads/upload_b5c3471e1e89234eaf1d7ab7f98af109.png)

## 哈希冲突

1. 开放地址方法

　　（1）线性探测

　　　按顺序决定值时，如果某数据的值已经存在，则在原来值的基础上往后加一个单位，直至不发生哈希冲突。　

　　（2）再平方探测

　　　按顺序决定值时，如果某数据的值已经存在，则在原来值的基础上先加1的平方个单位，若仍然存在则减1的平方个单位。随之是2的平方，3的平方等等。直至不发生哈希冲突。

　　（3）伪随机探测

　　　按顺序决定值时，如果某数据已经存在，通过随机函数随机生成一个数，在原来值的基础上加上随机数，直至不发生哈希冲突。
　　　
　　（4）双散列
　　
　　 F(i) = i * hash2(x)
　　 
2. 链式地址法（HashMap的哈希冲突解决方法）

　　对于相同的键值值，使用链表进行连接。使用数组存储每一个链表。

　　优点：

　　（1）拉链法处理冲突简单，且无堆积现象，即非同义词决不会发生冲突，因此平均查找长度较短；

　　（2）由于拉链法中各链表上的结点空间是动态申请的，故它更适合于造表前无法确定表长的情况；

　　（3）开放定址法为减少冲突，要求装填因子α较小，故当结点规模较大时会浪费很多空间。而拉链法中可取α≥1，且结点较大时，拉链法中增加的指针域可忽略不计，因此节省空间；

　　（4）在用拉链法构造的散列表中，删除结点的操作易于实现。只要简单地删去链表上相应的结点即可。
　　缺点：

　　指针占用较大空间时，会造成空间浪费，若空间用于增大散列表规模进而提高开放地址法的效率。

3. 建立公共溢出区

　　建立公共溢出区存储所有哈希冲突的数据。

4. 再哈希法

　　对于冲突的哈希值再次进行哈希处理，直至没有哈希冲突。