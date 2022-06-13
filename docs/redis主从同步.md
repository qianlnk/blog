# 主从同步

## 完全同步

![](/uploads/upload_441707f7ff5222ab92bd2823b7f06c06.png)

1. slave 服务启动，slave 会建立和master 的连接，发送sync 命令。

2. master启动一个后台进程将数据库快照保存到RDB文件中

    注意：此时如果生成RDB文件过程中存在写数据操作会导致RDB文件和当前主redis数据不一致，所以此时master 主进程会开始收集写命令并缓存起来。

3. master 就发送RDB文件给slave

4. slave 将文件保存到磁盘上，然后加载到内存恢复

5. master把缓存的命令转发给slave

注意：后续master收到的写命令都会通过开始建立的连接发送给slave。

当master 和slave 的连接断开时slave 可以自动重新建立连接。如果master 同时收到多个slave 发来的同步连接命令，只会启动一个进程来写数据库镜像，然后发送给所有slave。


**完全同步的问题：**

在redis2.8之前从redis每次同步都会从主redis中复制全部的数据，如果从redis是新创建的从主redis中复制全部的数据这是没有问题的，但是，如果当从redis停止运行，再启动时可能只有少部分数据和主redis不同步，此时启动redis仍然会从主redis复制全部数据，这样的性能肯定没有只复制那一小部分不同步的数据高。

## 部分同步

![](/uploads/upload_ddbcc9ec35bb9d9950d796eb1b6eb7da.png)

从机连接主机后，会主动发起 PSYNC 命令，从机会提供 master的runid(机器标识，随机生成的一个串) 和 offset（数据偏移量，如果offset主从不一致则说明数据不同步），主机验证runid 和 offset 是否有效， runid 相当于主机身份验证码，用来验证从机上一次连接的主机，如果runid验证未通过则，则进行全同步，如果验证通过则说明曾经同步过，根据offset同步部分数据。