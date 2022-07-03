# kafka消息流转过程

1、基础架构

见 **基础架构**

2、服务器启动阶段
首先，我们会启动 Zookeeper 服务器，作为集群管理服务器。接着，启动 Kafka Server。Kafka Server 会向 Zookeeper 服务器注册信息，接着启动线程池监听客户端的连接请求。最后，启动生产者和消费者，连接到 Zookeeper 服务器，从 Zookeeper 服务器获取到对应的 Kafka Server 信息。

2、生产者发送消息阶段

见 **数据发送流程**

3、Kafka存储消息阶段

见 **保存数据**


4、消费者拉取消息阶段

在消费者启动时，其会连接到 zk 注册节点，之后根据所连接 topic 的 partition 个数和消费者个数，进行 partition 分配。一个 partition 最多只能被一个线程消费，但一个线程可以消费多个 partition。

见 **消费数据**
