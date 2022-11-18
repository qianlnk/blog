
# elasticsearch 索引文档的过程

## 分发过程

![](/uploads/upload_1857d210f3339bc179b8a4695ec75e18.png)

1. 客户端向 Node 1 发送新建、索引或者删除请求。

2. 节点使用文档的 _id 确定文档属于分片 0 。请求会被转发到 Node 3，因为分片 0 的主分片目前被分配在 Node 3 上。

3. Node 3 在主分片上面执行请求。如果成功了，它将请求并行转发到 Node 1 和 Node 2 的副本分片上。一旦所有的副本分片都报告成功, Node 3 将向协调节点报告成功，协调节点向客户端报告成功。

**一图胜千文，记住这幅图，上面是文档在节点间分发的过程，接着说一下文档从接收到写入磁盘过程。**

## 写入磁盘过程

协调节点默认使用文档 ID 参与计算（也支持通过 routing），以便为路由提供合适的分片。

    shard = hash(document_id) % (num_of_primary_shards)
    
1、当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 MemoryBuffer，然后定时（默认是每隔 1 秒）写入到 Filesystem Cache，这个从 MomeryBuffer 到 Filesystem Cache 的过程就叫做 refresh；
2、当然在某些情况下，存在 Momery Buffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过 translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush；
3、在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。
4、flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512M）时；

```
1. translog 可以理解为就是一个文件，一直追加。
2. MemoryBuffer 应用缓存。
3. Filesystem Cache 系统缓冲区。
```

延伸阅读：Lucene 的 Segement:
```
1. Lucene 索引是由多个段组成，段本身是一个功能齐全的倒排索引。
2. 段是不可变的，允许 Lucene 将新的文档增量地添加到索引中，而不用从头重建索引。
3. 对于每一个搜索请求而言，索引中的所有段都会被搜索，并且每个段会消耗CPU 的时钟周期、文件句柄和内存。这意味着段的数量越多，搜索性能会越低。
4. 为了解决这个问题，Elasticsearch 会合并小段到一个较大的段，提交新的合并段到磁盘，并删除那些旧的小段。

```