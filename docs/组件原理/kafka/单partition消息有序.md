# 单partition消息有序

## 为什么只保证单partition有序

如果Kafka要保证多个partition有序，不仅broker保存的数据要保持顺序，消费时也要按序消费。假设partition1堵了，为了有序，那partition2以及后续的分区也不能被消费，这种情况下，Kafka 就退化成了单一队列，毫无并发性可言，极大降低系统性能。因此Kafka使用多partition的概念，并且只保证单partition有序。这样不同partiiton之间不会干扰对方。


## Kafka如何保证单partition有序？

1. producer发消息到队列时，通过加锁保证有序

    现在假设两个问题
    
    broker leader在给producer发送ack时，因网络原因超时，那么Producer 将重试，造成消息重复。
    
    先后两条消息发送。t1时刻msg1发送失败，msg2发送成功，t2时刻msg1重试后发送成功。造成乱序。

1. 解决重试机制引起的消息乱序

    为实现Producer的幂等性，Kafka引入了Producer ID（即PID）和Sequence Number。对于每个PID，该Producer发送消息的每个<Topic, Partition>都对应一个单调递增的Sequence Number。同样，Broker端也会为每个<PID, Topic, Partition>维护一个序号，并且每Commit一条消息时将其对应序号递增。对于接收的每条消息，如果其序号比Broker维护的序号）大一，则Broker会接受它，否则将其丢弃：
    
    - 如果消息序号比Broker维护的序号差值比一大，说明中间有数据尚未写入，即乱序，此时Broker拒绝该消息，Producer抛出InvalidSequenceNumber
    
    - 如果消息序号小于等于Broker维护的序号，说明该消息已被保存，即为重复消息，Broker直接丢弃该消息，Producer抛出DuplicateSequenceNumber
    
    - Sender发送失败后会重试，这样可以保证每个消息都被发送到broker