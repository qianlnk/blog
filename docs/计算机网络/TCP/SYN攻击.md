
# SYN攻击

泛洪伪造IP发送SYN包，使得服务器发送SYN+ACK无人应答，占满半连接队列，使服务器不能为正常用户服务。

避免方式：

1. 通过修改 Linux 内核参数，控制队列大小和当队列满时应做什么处理。
    1. 通过修改 Linux 内核参数，控制队列大小和当队列满时应做什么处理。
       
        ```
        net.core.netdev_max_backlog
        ```
    2. SYN_RCVD 状态连接的最大个数
        
        ```
        net.ipv4.tcp_max_syn_backlog
        ```
    3. 超出处理能时，对新的 SYN 直接回报 RST，丢弃连接
    
        ```
        net.ipv4.tcp_abort_on_overflow
        ```
        
2. 启动syn_cookie
    
    ```
    net.ipv4.tcp_syncookies = 1
    ```
    
    - 当 「 SYN 队列」满之后，后续服务器收到 SYN 包，不进入「 SYN 队列」; 计算出一个 cookie 值，再以 SYN + ACK 中的「序列号」返回客户端，
    - 服务端接收到客户端的应答报文时，服务器会检查这个 ACK 包的合法性。如果合法，直接放入到 「 Accept 队列」。
    - 最后应用通过调用 accpet() socket 接口，从「 Accept 队列」取出的连接。