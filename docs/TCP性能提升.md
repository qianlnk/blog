
# TCP性能提升


![](/uploads/upload_28dccb2f86635c74fb7da42556b052a0.png)


## 优化三次握手策略

|端|策略|TCP内核参数|参数说明|
|---|---|---|---|
|客户端|减少重发SYN次数|tcp_syn_reties|默认5次|
|服务端|调整tcp半连接队列长度|tcp_max_syn_backlog、somaxconn、backlog|backlog有web服务决定|
|服务端|开启syncookies功能|tcp_syncookies|0-关闭<br><br>1-仅当 SYN 半连接队列放不下时，再启用它<br><br>2-无条件开启<br><br>针对SYN攻击设置成1|
|服务端|调整SYN+ACK报文的重传次数|tcp_synack_reties|默认5次|
|服务端|调整tcp全连接队列长度|min(backlog, somaxconn)|
|服务端|关闭RST复位报文|tcp_abort_on_overflow|默认0<br><br> 0-如果 accept 队列满了，那么 server 扔掉 client 发过来的 ack<br><br>如果 accept 队列满了，server 发送一个 RST 包给 client，表示废掉这个握手过程和这个连接|
|两端|绕过三次握手|tcp_fastopn|0 关闭<br><br>1 作为客户端使用 Fast Open 功能<br><br>2 作为服务端使用 Fast Open 功能<br><br>3 无论作为客户端还是服务器，都可以使用 Fast Open 功能|


## 优化四次挥手策略

|端|策略|TCP内核参数|参数说明|
|---|---|---|---|
|主动方/被动方|调整FIN重传次数|tcp_orphan_retries||
|主动方|调整FIN_WAIT2的时间|tcp_fin_timeout||
|主动方|调整孤儿连接的上限|tcp_max_orphans||
|主动方|调整TIME_WAIT状态的上限|tcp_max_tw_buckets||
|主动方|复用TIME_WAIT状态的连接|tcp_tw_reuse、tcp_timestamps||
|被动方|出现大量CLOSE_WAIT|查看自身程序是不是有问题||


## 传输性能

|策略|TCP内核参数|参数说明|
|---|---|---|
|开启窗口扩大因子|tcp_window_scalng|默认开启|
|调整发送缓冲区范围|tcp_wmen||
|调整接收缓冲区范围|tcp_rmen||
|打开接收缓冲区动态调节|tcp_moderate_rcvbuf||
|调整内存范围|tcp_men||

![](/uploads/upload_25504d6bdad8bccdc81894037634ad3e.png)


![](/uploads/upload_abdcd1d6ddadc86aa868227f4ff59e66.png)
