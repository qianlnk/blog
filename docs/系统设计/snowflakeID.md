# snowflake ID

snowflake是Twitter开源的分布式ID生成算法，结果是一个Long型的ID。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的序列号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0。

特点：

- 作为ID，肯定是唯一的；
- 自增，依赖时间戳生成，序列号有序递增；
- 支持非常大的业务ID生成，最大支持2^10=1024个业务节点，同一个节点一毫秒最多生成2^12=4096个ID，41位毫秒级时间可以使用（2^41 - 1）/(1000*60*60*24*365)=69.73，大约70年；
- 实现简单，不依赖于其他第三方组件，甚至都不需要任何import。

结果是一个Long型的ID，64位，结构图如下

![](/uploads/upload_fb42b9890e71522e8a728c3f0831faf9.png)


1. 第1位固定是0，表示正数；
2. 第2-42共41位表示时间戳，当前时间的时间戳减去开始时间的时间戳；
3. 业务节点ID，每个节点固定的值；
4. 毫秒内的序列号。