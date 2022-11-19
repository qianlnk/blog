# Select原理
go的 select 为 golang 提供了多路 IO 复用机制，和其他 IO 复用一样，用于检测是否有读写事件是否 ready。linux 的系统 IO 模型有 select，poll，epoll，go 的 select 和 linux 系统 select 非常相似。

select 结构组成主要是由 case 语句和执行的函数组成 select 实现的多路复用是：每个线程或者进程都先到注册和接受的 channel（装置）注册，然后阻塞，然后只有一个线程在运输，当注册的线程和进程准备好数据后，装置会根据注册的信息得到相应的数据。

## 数据结构：
c：表示case语句操作的管道，由于select数据结构中仅能存放一个管道，所以case语句只能处理一个管道
kind：类型kind表示该case的类型，分为读channel、写channel和default
elem：数据数据存放的地址

## 实现逻辑：

通过源码包里面的selectgo()函数 
selectgo（）函数会从一组case语句中挑选一个case，然后返回命中case的下标。

## 总结：
		Select语句中当所有的case被阻塞的时候，就执行的是default
		select语句中除default外，每个case操作一个channel，要么读要么写 select语句中除default外，各case执行顺序是随机的 
		select语句中如果没有default语句，则会阻塞等待任一case 
		select语句中读操作要判断是否成功读取，关闭的channel也可以读取

## select应用场景
1. 永久阻塞
   不希望主函数退出（协程处理任务的时候），使用select{}
2. 限时等待