
# RST攻击

RST 攻击也称为伪造 TCP 重置报文攻击，通过伪造 RST 报文来关闭掉一个正常的连接。

源 IP 地址伪造非常容易，不容易被伪造的是序列号，RST 攻击最重要的一点就是构造的包的序列号要落在对方的滑动窗口内，否则这个 RST 包会被忽略掉，达不到攻击的效果。

**tcpkill 工具使用及原理介绍**

![](/uploads/upload_362cd21c304431f59dd02f68a60e343d.png)


tcpkill 假冒了 A 和 B 的 IP发送了 RST 包给通信的双方，那问题来了，伪造 ip 很简单，它是怎么知道当前会话的序列号的呢？

tcpkill 的原理跟 tcpdump 差不多，会通过 libpcap 库抓取符合条件的包。 因此只有有数据传输的 tcp 连接它才可以拿到当前会话的序列号，通过这个序列号伪造 IP 发送符合条件的 RST 包。

![](/uploads/upload_6154000476b48b2acafe0fff975b72b0.png)


可以看到 tcpkill 对每个端发送了 3 个RST 包，这是因为在高速数据传输的连接上，根据当前抓的包计算的序列号可能已经不再 TCP 连接的窗口内了，这种情况下 RST 包会被忽略，因此默认情况下 tcpkill 未雨绸缪往后计算了几个序列号。还可以指定参数-n指定更多的 RST 包，比如tcpkill -9

局限:

• 无法杀掉一条僵死连接 (无法抓包获取 seq)

**killcx**

killcx 是一个用 perl 写的在 linux 下可以关闭 TCP 连接的脚本，无论 TCP 连接处于什么状态。


![](/uploads/upload_02a53660769e14a437d82d33e3d059e2.png)


前 5 个包都很正常，三次握手加上一次数据传输，有趣的事情从第 6 个包开始

1. 第 6 个包是 killcx 伪造 IP 向服务端 B 发送的一个 SYN 包
2. 第 7 个包是服务端 B 回复的 ACK 包，里面包含的 SEQ 和 ACK 号
3. 第 8 个包是 killcx 伪造 IP 向服务端 B 发送的 RST 包
4. 第 9 个包是 killcx 伪造 IP 向客户端 A 发送的 RST 包

![](/uploads/upload_95def3f2eee6e131a367c37863aaea3a.png)


小结

• tcpkill 采用了比较保守的方式，抓取流量等有新包到来的时候，获取 SEQ/ACK 号，这种方式只能杀掉有数据传输的连接

• killcx 采用了更加主动的方式，主动发送 SYN 包获取 SEQ/ACK 号，这种方式活跃和非活跃的连接都可以杀掉