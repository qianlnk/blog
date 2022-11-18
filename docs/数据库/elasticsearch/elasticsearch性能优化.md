# 性能优化

## 系统拓补

现状：

3 master 25 hot 23 warn

hot 2～10核 45G～60G
warn 1～6核 45G～60G
master 1～4核 20G

1900亿日志
125TB

索引分片 6
索引副本 1
refresh_interval: 30s

每个索引200G 或 按天自动滚动
使用别名进行写入，滚动时别名自动指向新索引

### Master节点设置

一般会设置３个专用的maste节点，以提供最好的弹性扩展能力。因为master节点不参与查询、索引操作，仅负责对于集群管理，所以在CPU、内存、磁盘配置上，都可以比数据节点低很多。

### Hot节点设置

索引节点（写节点），同时保持近期频繁使用的索引。 属于IO和CPU密集型操作，使用SSD的磁盘类型，保持良好的写性能；将节点设置为hot类型:

```
node.attr.box_type: hot 
```

针对index, 通过设置`index.routing.allocation.require.box_type： hot`可以设置将索引写入hot节点。


### Warm节点设置

用于不经常访问的read-only索引。由于不经常访问，一般使用普通的磁盘即可。内存、CPU的配置跟Hot节点保持一致即可。

当索引不再被频繁查询时，可通过index.routing.allocation.require.box_type： warm， 将索引标记为warm, 从而保证索引不写入hot节点，以便将SSD磁盘资源用在刀刃上。一旦设置这个属性，ES会自动将索引合并到warm节点。同时，也可以在elasticsearch.yml中设置 index.codec: best_compression 保证warm 节点的压缩配置

## 内存设置

50G内存 50%给ES，50%留给lucene

禁止swap，一旦允许内存与磁盘的交换，会引起致命的性能问题。 通过： 在elasticsearch.yml 中 bootstrap.memory_lock: true， 以保持JVM锁定内存，保证ES的性能。

GC设置：

- 使用默认设置

- 线程池设置

    当某个线程池active==threads时，表示所有线程都在忙，那么后续新的请求就会进入queue中，即queue>0，一旦queue大小超出限制，比如bulk的queue默认100，那么elasticsearch进程将拒绝请求（碰到bulk HTTP状态码429），相应的拒绝次数就会累加到rejected中。 对于被拒绝的请求：我们一般用如下的方法规避。 （ 1、记录失败的请求并重发 2、减少并发写的进程个数，同时加大每次bulk请求的size ）

批量请求 大小12M
或 10w条日志

## 分片设置

ES一旦创建好索引后，就无法调整分片的设置，而在ES中，一个分片实际上对应一个lucene 索引，而lucene索引的读写会占用很多的系统资源，因此，分片数不能设置过大；所以，在创建索引时，合理配置分片数是非常重要的

## maping 建模

1. 尽量避免使用nested（嵌套）或 parent/child（父子关系），能不用就不用；nested query慢， parent/child query 更慢，比nested query慢上百倍；因此能在mapping设计阶段搞定的（大宽表设计或采用比较smart的数据结构），就不要用父子关系的mapping。

2. 如果一定要使用nested fields，保证nested fields字段不能过多，目前ES默认限制是50。参考：
 
    ```
    index.mapping.nested_fields.limit ：50
    ```

3. 避免使用动态值作字段(key), 动态递增的mapping，会导致集群崩溃；同样，也需要控制字段的数量，业务中不使用的字段，就不要索引。控制索引的字段数量、mapping深度、索引字段的类型，对于ES的性能优化是重中之重。以下是ES关于字段数、mapping深度的一些默认设置：

    ```
    index.mapping.nested_objects.limit :10000
    index.mapping.total_fields.limit:1000
    index.mapping.depth.limit: 20
    ```
     
## 索引优化设置
    
1. 设置refresh_interval 为-1，同时设置number_of_replicas 为0，通过关闭refresh间隔周期（或增大刷新周期），同时不设置副本来提高写性能。

2. 修改index_buffer_size 的设置，可以设置成百分数，也可设置成具体的大小，大小可根据集群的规模做不同的设置测试。
    ```
    indices.memory.index_buffer_size：10%（默认）
    indices.memory.min_index_buffer_size： 48mb（默认）
    indices.memory.max_index_buffer_size
    ```
3. 修改translog相关的设置：
        
    a. 控制数据从内存到硬盘的操作频率，以减少硬盘IO。可将sync_interval的时间设置大一些。
    ```
    index.translog.sync_interval：5s(默认)。
    ```
        
    b. 控制tranlog数据块的大小，达到threshold大小时，才会flush到lucene索引文件。
    ```
    index.translog.flush_threshold_size：512mb(默认)
    ```
    
1. _id字段的使用，应尽可能避免自定义_id, 以避免针对ID的版本管理；建议使用ES的默认ID生成策略或使用数字类型ID做为主键。

2. _all字段及_source字段的使用，应该注意场景和需要，_all字段包含了所有的索引字段，方便做全文检索，如果无此需求，可以禁用；_source存储了原始的document内容，如果没有获取原始文档数据的需求，可通过设置includes、excludes 属性来定义放入_source的字段。

3. 合理的配置使用index属性，analyzed 和not_analyzed，根据业务需求来控制字段是否分词或不分词。只有 groupby需求的字段，配置时就设置成not_analyzed, 以提高查询或聚类的效率。

## 查询优化

1. query_string 或 multi_match的查询字段越多， 查询越慢。可以在mapping阶段，利用copy_to属性将多字段的值索引到一个新字段，multi_match时，用新的字段查询。

2. 查询结果集的大小不能随意设置成大得离谱的值， 如query.setSize不能设置成 Integer.MAX_VALUE， 因为ES内部需要建立一个数据结构来放指定大小的结果集数据。

3. QueryCache: ES查询的时候，使用filter查询会使用query cache, 如果业务场景中的过滤查询比较多，建议将querycache设置大一些，以提高查询速度。
   ```
   indices.queries.cache.size： 10%（默认），可设置成百分比，也可设置成具体值，如256mb。
   ```
   
## 设计调优

a. 根据业务增量需求，采取基于日期模板创建索引，通过 rollover API 滚动索引；(rollover API我会单独写一个代码案例做讲解，公众号：JavaPub)
b. 使用别名进行索引管理；（es的索引名不能改变，提供的别名机制使用非常广泛。）
c. 每天凌晨定时对索引做force_merge操作，以释放空间；
d. 采取冷热分离机制，热数据存储到SSD，提高检索效率；冷数据定期进行shrink操作，以缩减存储；
e. 采取curator进行索引的生命周期管理；
f. 仅针对需要分词的字段，合理的设置分词器；
g. Mapping阶段充分结合各个字段的属性，是否需要检索、是否需要存储等。

## 写入调优

写入前副本数设置为0；
写入前关闭refresh_interval设置为-1，禁用刷新机制；
写入过程中：采取bulk批量写入；
写入后恢复副本数和刷新间隔；
尽量使用自动生成的id。

## 查询调优

禁用wildcard；（通配符模式，类似于%like%）
禁用批量terms（成百上千的场景）；
充分利用倒排索引机制，能keyword类型尽量keyword；
数据量大时候，可以先基于时间敲定索引再检索；
设置合理的路由机制。