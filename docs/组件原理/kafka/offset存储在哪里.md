# offset存储在哪里

在kafka中，默认提供一个consumer_offsets的topic主题，offset信息写入到了这个topic分区中。分为50个分区，分区中保存了每一个consumer group 提交的offset值。


## 一个消费者组中offset应该保存在consumer_offsets中那个分区上呢？


例如：一个叫groupID为“hahaha”的消费组，里面的消费者提交的offset保存的分区=“hahaha”.hashCOde()%分区数（这里默认是50）=22，也就是保存在consumeroffsets22这个分区里；这50分区的命名consumeroffsets[0-49];

查看offset信息：h kafka-simple-consumer-shell.sh --topic _consumeroffsets --partition 35 --broker-list 这里是kafka集群的主机ip地址：9092用逗号隔开（如192...1:9092,192....2:9092等等） --formatter