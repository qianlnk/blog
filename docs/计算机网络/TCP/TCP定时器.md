
# TCP定时器

1. **连接建立定时器**（connection establishment timer）

    当发送端发送SYN想建立一个新的连接时，会开启连接建立定时器，如果没有收到对端的ACK包将进行重传。重传次数由`/proc/sys/net/ipv4/tcp_syn_retries`设置。
    
    重试全部失败后，connect 调用返回 -1，调用超时。
    
    ![](/uploads/upload_3e6df909831e8898c18ab05a4b0bf620.png)



2. **重传定时器**（retransmission timer）

    在发送数据包的时候没有收到ACK时用到。
    
    重传定时器的时间是动态计算的，取决于 RTT 和重传的次数。

    • 重传时间间隔是指数级退避，直到达到 120s 为止
    • 重传次数是15次（这个值由操作系统的 `/proc/sys/net/ipv4/tcp_retries2`  决定)，总时间将近 15 分钟。


    ![](/uploads/upload_4f36f3313833fb81c570ee625e999f8b.png)


3. **延迟ACK定时器**
    
    在 TCP 收到数据包以后在没有数据包要回复时，不马上回复 ACK。这时开启一个定时器，等待一段时间看是否有数据需要回复。如果期间有数据要回复，则在回复的数据中捎带 ACK，如果时间到了也没有数据要发送，则也发送 ACK。在 Centos7 上这个值为 40ms。
   
    ![](/uploads/upload_781c754b4f21cd10890141cc38054d50.png)


4. **坚持定时器**（persist timer）

    persist timer是专门为零窗口探测而准备的。我们都知道TCP利用滑动窗口来实现流量控制，当接收端B接收到窗口为零时，发送端A此时不能再发送数据，发送端此时会开启一个persist定时器，超时后发送一个特殊的报文给接收端看对方的窗口是否已经恢复。这个特殊报文只有一个字节，就是零窗口探测包。
    
    ![](/uploads/upload_786487d0c0ad742a461124321af87adf.png)


5. **保活定时器**（keepalive timer）

    如果通信以后一段时间有再也没有传输过数据，怎么知道对方是不是已经挂掉或者重启了呢？于是 TCP 提出了一个做法就是在连接的空闲时间超过 2 小时，会发送一个探测报文，如果对方有回复则表示连接还活着，对方还在，如果经过几次探测对方都没有回复则表示连接已失效，客户端会丢弃这个连接。
    
    参数：
    
    - tcp_keepalive_time: 最后一次数据交换到TCP发送第一个保活探测报文的时间，即允许连接空闲的时间，默认7200s
    - tcp_keepalive_intvl: 保活探测报文的重传时间，默认75s
    - tcp_keepalive_probes: 保活探测报文的发送次数，默认9次

    ![](/uploads/upload_9c903799ccd300b60a2d365214050423.png)


6. **FIN_WAIT_2定时器**

    四次挥手过程中，主动关闭的一方收到 ACK 以后从 FIN_WAIT_1 进入 FIN_WAIT_2 状态等待对端的 FIN 包的到来，FIN_WAIT_2 定时器的作用是防止对方一直不发送 FIN 包，防止自己一直傻等。
    
    参数：
    
    - tcp_fin_timeout: 等待接收FIN报文时间

    ![](/uploads/upload_39c587c02f834c5e18e8f32662b817af.png)

7. **TIME_WAIT 定时器**
 
    TIME_WAIT 定时器也称为 2MSL 定时器，可能是这七个里面名气最大的，主动关闭连接的一方在 TIME_WAIT 持续 2 个 MSL 的时间，超时后端口号可被安全的重用。
    
    ![](/uploads/upload_788085b965b3faf3293cfb17096b900a.png)
    
    TIME_WAIT存在的意义有两个：
    
    • 可靠的实现 TCP 全双工的连接终止（处理最后 ACK 丢失的情况）
    • 避免当前关闭连接与后续连接混淆（让旧连接的包在网络中消逝）