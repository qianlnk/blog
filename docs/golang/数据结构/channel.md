# Channel

## Channel实现原理
channel主要用于进程内各goroutine间通信
### 数据结构：
它主要由环形队列、等待队列组成等
1. 环形队列：
环形队列作为其缓冲区，队列的长度是创建chan时指定的
2. 等待队列：
被阻塞的goroutine将会挂在channel的等待队列中：；
		从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞等待。
		向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞等待。

## 读写
向channel写数据：
		如果缓冲区中有空余位置，则将数据写入缓冲区，结束发送
		如果缓冲区内没有空余位置，则将当前协程加入sendq（等待写消息队列）队列，进入睡眠并等待被读协程唤醒
向channel读数据
		如果缓冲区中有数据，则从缓冲区中取出数据，结束读取过程
		如果缓冲区中有数据，则从将当前协程加入recvq（等待读消息队列）队列，进入睡眠并等待被写协程唤醒
关闭channel：
		关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。
		关闭channel会把sendq中的G全部唤醒，但这些G会panic。

## Channel缓冲区特点
### 同步与非同步：
无缓冲的 channel 是同步的，
有缓冲的 channel 是非同步的，缓冲满时发送阻塞
		 
channel无缓冲时，发送阻塞直到数据被接收，接收阻塞直到读到数据；
channel有缓冲时，当缓冲满时发送阻塞，当缓冲空时接收阻塞。

### nil：
如果给一个nil的 channel 发送数据，会造成永远阻塞。 
如果从一个nil的 channel 中接收数据，会造成永久阻塞。

### panic： 
关闭值为nil的channel
关闭已经被关闭的channel
向已经关闭的channel写数据
给一个已经关闭的channel发送数据，引起panic

### 特点：
关闭的管道读数据仍然可以读数据
从一个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回一个零值 

## Channel线程安全
### 原因：
使用场景就是多线程，为了保证数据的一致性，必须实现线程安全

### 原理：
channel的底层实现中，hchan结构体中采用Mutex锁来保证数据读写安全。在对数据进行入队和出队操作时，必须先获取互斥锁，才能操作channel数据

## channel死锁
### 概念：
协程永久阻塞
两个以上的协程的执行过程中，由于竞争资源或由于彼此通信而造成的一种阻塞的现象。

### 场景：
无缓冲channel只写不读
无缓冲channel读在写后面
多个协程互相等待

## Channel应用场景
channel适用于数据在多个协程中流动的场景

① 任务定时
比如超时处理：
select {
    case <-time.After(time.Second):
定时任务
select {
    case <- time.Tick(time.Second)

② 解耦生产者和消费者
可以将生产者和消费者解耦出来，生产者只需要往channel发送数据，而消费者只管从channel中获取数据。

③ 控制并发数
一般要配合sync使用

### 退出程序怎么防止无缓冲channel没有消费完（channel还留有数据没有）
退出时将生产者关闭，不会产生多余的数据给消费者

### 怎么控制上游生产速度过快
channel长度可以与上下游的速度比例成线性关系