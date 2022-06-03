# channel底层实现

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

为了并发的goroutines之间的通讯，golang使用了管道channel。

    可以通过一个goroutines向channel发送数据，然后从另一个goroutine接收它。

    通常我们会使用make来创建channel   -----   make(chan valType, [size])。

        写入 c <- data
    
        读取 data ：= <-c

**Golang的channel分为缓冲和非缓冲的两种。主要区别：缓冲chanel是同步的，非缓冲channel是非同步的。**

    举个例子：
         
        c1 := make(chan int)      // 无缓冲： c1 <- 1，当前协程阻塞。

        c2 := make(chan int, 1)   // 有缓冲： c2 <- 1，当前协程不会阻塞，c2 <- 2，此时1没有被取走，当前协程才阻塞。

备注：

    1、给一个nil channel发送数据，造成永远阻塞。
    
    2、从一个nil channel接收数据，造成永远阻塞。
    
    3、给一个已经关闭的channel发送数据，引起panic。
    
    4、从一个已经关闭的channel接收数据，立即返回一个零值（false : bool, 0 : int, 0.0 : float, "" : string, nil : pointer, function, interface, slice, channel, map）。
        
    5、channel关闭多次会引起panic，channel不能close大于1次。
    
    6、可以用两个返回值(valeu, b := <-c)来捕获channel是否关闭，b取值false或true。


## 缓冲channel的调度：

### 写数据

![](/uploads/upload_e0ce9c67e8e0ba7333e396bdfe7c713e.png)

1. 锁定通道
2. 确定写入，尝试从recvq中取G，将数据写入G，唤醒G。
3. 如果recvq为空，则确定buf是否可用，可以则把数据写入buf。
4. 若buf满，则数据保存在当前的写协程，并挂在sendq等待唤醒。
5. 写完成，释放锁。
    
### 读数据：
![](/uploads/upload_1185acd7dce12574ebf53ef4ed668e46.png)

1. 锁定通道
2. 尝试从sendq取取出一个G。
3. 如果没有缓冲区，直接取出G中的数据，然后唤醒G，结束，解锁。
4. 如果有缓冲区（满），从缓冲区首取出数据，唤醒G，G将数据写入缓冲区尾，结束，解锁。
5. 如果没有等待写的G，且缓冲区有数据，读取buf数据，结束，解锁。
6. 如果没有等待写的G，且缓冲区没有数据，将当前读取的G加到recvq队列，挂起，等待唤醒，结束，解锁。

## 非缓冲channel的调度：
    
### 写数据：
    
1. 协程1往channel写数据，读队列为空，加入写队列，阻塞当前协程1，等待有其他协程n从channel读数据。
        
2. 协程1往channel写数据，读队列非空（协程2阻塞在读队列），协程1将数据交给协程2并唤醒协程2等待调度，协程1返回，调度协程2取得数据并返回。
           
## 读数据：
        
1. 协程2从channel读数据，写队列非空（协程1阻塞在写队列），协程2从写队列取出数据并唤醒协程1等待调度，协程2携带数据返回，调度协程1返回。
        
2. 协程2从channel读数据，写队列为空，加入读队列，阻塞当前协程2，等待有其他协程n往channel写数据。