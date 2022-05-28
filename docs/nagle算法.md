
# nagle算法

nagle 算法讲的是减少发送端频繁的发送小包给对方。

Nagle 算法要求，当一个 TCP 连接中有在传数据（已经发出但还未确认的数据）时，小于 MSS（Maxitum Segment Size 最大报文大小） 的报文段就不能被发送，直到所有的在传数据都收到了 ACK。同时收到 ACK 后，TCP 还不会马上就发送数据，会收集小包合并一起发送。

默认情况下 Nagle 算法都是启用的，Java 可以通过 setTcpNoDelay(true);来禁用 Nagle 算法。

![](/uploads/upload_4c241d067c84826266c7624c4086afb8.png)

## Nagle 算法的意义在哪里

Nagle 算法的作用是减少小包在客户端和服务端直接传输，一个包的 TCP 头和 IP 头加起来至少都有 40 个字节，如果携带的数据比较小的话，那就非常浪费了。就好比开着一辆大货车运一箱苹果一样。