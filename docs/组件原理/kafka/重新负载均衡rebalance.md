# 重新负载均衡rebalance

## 什么情况下会发生消费者的重新负载均衡呢？

- 当consumer group 新增消费者；
 -consumer group有消费者退出时，比如主机停机；
- topic新增分区时，分区的数量发生变化时； 

kafka consuemr 的 rebalance 机制规定了一个 consumer group 下的所有 consumer 如何达成一致来分配订阅 topic 的每个分区。实现重新负载均衡机制：范围分区和轮询分区；


## 谁来执行rebalance以及管理consumer group呢？

Kafka 提供了一个角色:coordinator 来执行对于consumer group 的管理；

## consumer group如何确定自己的coordinate?

消费者向kafka集群中的任意一个broker发送一个GroupCoordinatorRequest请求，服务端会返回一个负载最小的broker节点的id，并将改broker设置为Coordinate；之后该group中的所有consumer成员会和Coordinate进行通信；

## Rebalance的执行分为两个过程

在执行rebalance之前，需要保证Coordinate已经确定好了；

1. JoinGroup 的过程

join:consumer加入到consumer group时，在这一步中，所有的consumer会向Coordinate发送joinGroup请求，Coordinate收到所有的consumer发来的joinGroup请求，会从中选择一个consumer为leader角色（班长），并给组成员返回一个response信息；

response信息中有leaderid,

members：组成员信息,consumer group 中全部的消费者的订阅信息，只有发给给leader的members才有消息，其他消费者的members为null；

membermetadate:对应消费者的订阅信息；

generationid:年代信息，对于每一轮 rebalance， generationid 都会递增。

总体来说：joingroup的过程就是每个消费者向Coordinate发送一个请求，来确定消息者中谁是他们的班长（leader）,班长会拿到班级名单和班级订阅的资料书（订阅的消息）；

## Synchronizing Group State 阶段

每个消费者都会向 coordinator 发送 syncgroup 请求，不过只有 leader 节点会发送消费分区分配方案，其他消费者的请求没起什么作用。当 leader 把方案发给 coordinator 以后， coordinator 会把结果设置到 SyncGroupResponse 中返回给每一个消费者。这 样所有成员都知道自己应该消费哪个分区。

consumer group 的分区分配方案是在客户端执行的!Kafka 将这个权利下放给客户端主要是因为这样做可以有更好的灵活性