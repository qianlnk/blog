
# kafka 与 NSQ 对比

资源消耗

NSQ：

|进程|	启动时占用|
|---|---|
|nsqd|	9.2MB|
|nsqlookup|	8.5MB|

Kafka

|进程|	启动时占用|
|---|---|
|kafka|	299MB|
|zookeeper|	58MB|

运行与维护

||NSQ|	Kafka|
|---|---|---|
|依赖	|无	|Linux基础包、bash、jdk、java|
|耦合	|无！能以nsqd单进程提供完整服务，只在多节点分布式模式下需要nsqlookup	|依赖 zookeeper|
|日志|	标准输出，自行重定向|	zookeeper 1份日志，kafka 7份日志，其中两份日志按小时自动切割|
|配置|	10项左右，默认即是最优|	10多个独立配置文件，数百个配置项|
|性能优化|	默认开启 pprof。支持web可视化实时观测内存、协程等动态|	无
|异常排查|	错误日志中的栈，源码量小。不依赖网络问答也能在短时间内找出问题|	错误日志中的栈，深度的栈，巨量源码，排查需要深入了解其原理，大量阅读源码。否则只能通过互联网、查阅前人经验或大师级人脉。|

业务能力

||NSQ	|Kafka|
|---|---|---|
|数据安全|	单个nsqd实例内的数据，不支异地热备，实例在正常退出时，会做刷入磁盘操作，也有手动备份实例数据的工具。|	数据全在磁盘。多个节点间自动互为备份。|
|消息顺序|	不保证有序	|支持有条件的有序|
|消息投递|	至少一次，消费者需自行保持消息处理的幂等	|支持准确的一次|

附加能力

||	NSQ|	Kafka|
|---|---|---|
|界面化管理|	自带nsqadmin|	无，需额外安装第三方包
|基于http协议的pub|	nsqd自带|	无，需额外安装第三方包


# kafka nsq

kafka有topic和partition的概念，而nsq有topic和channel的概念，topic这一层上大家概念还比较类似，都是说一种订阅的消息类型。到partition和channel这一层就不同了。

**工作原理**

kafka的producer产生特定种类的topic，分发到各个partition中，是没有重复消息的，comsumer本身订阅的是某一个topic，而每个partition只能连接1个comsumer，1个comsumer是可以处理多个partition的，而且其中会根据comsumer的数量做负载均衡。

nsq的channel就不同了，所有的channel都会拿到一份topic传来全部信息传到1个nsqd里，comsumer订阅的不是不仅是topic，还要包含指定channel，一个nsqd接的comsumer数量也可以是多个，nsqlookup会根据comsumer的订阅的channel将其指到特定的nsqd上。

**使用场景**
kafka更像是一个消息的分发器件，对于消息的分发实现topic级别的负载均衡，是针对不同消息的处理模式。对同一消息的不同处理，因为kafka本身会将所有消息固化，接不同的comsumer组去处理的时候只要将offset置到指定位置即可。

nsq比kafka多了一层，对同一消息可能不同业务场景需求也不同，因此用不同的channel去接topic，这样consumer连接channel对应的nsqd去做处理，这是因为nsq本身不做消息的固化，处理完了除了内存里的其他就丢了（除非将 –mem-queue-size 设置为 0），因此用这种处理模式。

**区别**
kafka消息会固化，存文件，nsq默认是不保存的
kafka消息因为固化下来，所以是保序的，nsq传递时候通常是无序的，当然你也可以保留下信息去check时间戳，因此nsq更适合处理数据量大但是彼此间没有顺序关系的消息。