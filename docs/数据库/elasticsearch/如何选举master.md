
# 如何选举master

ElasticSearch 的选主是 ZenDiscovery 模块负责，源码分析将首发在: https://gitee.com/rodert/JavaPub

对所有可以成为 Master 的节点（node.master: true）根据 nodeId 排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是 Master 节点。

如果对某个节点的投票数达到一定的值 **（可以成为master节点数n/2+1）** 并且该节点自己也选举自己，那这个节点就是master。否则重新选举。

(当然也可以自己设定一个值，最小值设定为超过能成为 **Master节点的n/2+1**，否则会出现脑裂问题。discovery.zen.minimum_master_nodes)