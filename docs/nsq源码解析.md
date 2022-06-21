---
type: slide
tags: Templates, Talk, slide
slideOptions:
  transition: convex
  theme: white
  allottedMinutes: 30
---

<style>
.reveal section img{border:0}

pre{text-align: left !important;}
.hljs {background: #fefefe;color: #777;}
pre code .gutter.linenumber {
    color: #bfbfbf !important;
    border-right: 3px solid #6DBFFF !important;
}
.reveal pre code {
    max-height: 500px;
}
</style>

# <span style="color: #00BFFF;">NSQ</span>源码解析

---


```go
1. 为什么使用MQ
2. 消息队列对比
3. NSQ VS kafka
4. 认识NSQ
    1. NSQ特性
    2. NSQ组件
5. NSQ入口
6. NSQ主要源码
7. NSQLookupd源码
8. 源码亮点
```

---

## 为什么使用MQ

----

流程异步化、代码解耦合、流量削峰、高可用、高吞吐量、广播分发，达到数据的最终一致性，满足具体的业务场景需求。

---

## 消息队列对比

----

![](/uploads/upload_39f0d60bfd1e2ccca56f254c34aaefe6.png)

---

## NSQ VS kafka

----

### 资源消耗

![](/uploads/upload_5e47187231a33a17ba1208e078d50ab0.png)


----

### 运维维护

----

![](/uploads/upload_3ee1a2cbc46bde361a40966cfcc066e1.png)

----

### 业务能力


![](/uploads/upload_ca37f4cb2b4413e7702ab1bbb8a8cff1.png)

----

### 附加能力

![](/uploads/upload_85c9951ff945e5a4d12f8b50b4bb84e9.png)


----

kafka有topic和partition的概念，而nsq有topic和channel的概念，topic这一层上大家概念还比较类似，都是一种订阅的消息类型。到partition和channel这一层就不同了。

----

### 工作原理

kafka的producer产生特定种类的topic，分发到各个partition中，是没有重复消息的，comsumer本身订阅的是某一个topic，而每个partition只能连接1个comsumer，1个comsumer是可以处理多个partition的，而且其中会根据comsumer的数量做负载均衡。

nsq的channel就不同了，所有的channel都会拿到一份topic传来全部信息传到1个nsqd里，comsumer订阅的不是不仅是topic，还要包含指定channel，一个nsqd接的comsumer数量也可以是多个，nsqlookup会根据comsumer的订阅的channel将其指到特定的nsqd上。

----

### 使用场景

kafka更像是一个消息的分发器件，对于消息的分发实现topic级别的负载均衡，是针对不同消息的处理模式。对同一消息的不同处理，因为kafka本身会将所有消息固化，接不同的comsumer组去处理的时候只要将offset置到指定位置即可。

nsq比kafka多了一层，对同一消息可能不同业务场景需求也不同，因此用不同的channel去接topic，这样consumer连接channel对应的nsqd去做处理，这是因为nsq本身不做消息的固化，处理完了除了内存里的其他就丢了（除非将 –mem-queue-size 设置为 0），因此用这种处理模式。

----

### 区别

kafka消息会固化，存文件，nsq默认是不保存的
kafka消息因为固化下来，所以是保序的，nsq传递时候通常是无序的，当然你也可以保留下信息去check时间戳，因此nsq更适合处理数据量大但是彼此间没有顺序关系的消息。

---

## 认识NSQ

NSQ 最初是由 bitly 公司开源出来的一款简单易用的分布式消息中间件，它可用于大规模系统中的实时消息服务，并且每天能够处理数亿级别的消息。

----

![](/uploads/upload_0309490addcdc4d1605ff78d0c37461d.png)


----

### NSQ特性

- 分布式：它提供了分布式的、去中心化且没有单点故障的拓扑结构，稳定的消息传输发布保障，能够具有高容错和高可用特性。

- 易于扩展：它支持水平扩展，没有中心化的消息代理（ Broker ），内置的发现服务让集群中增加节点非常容易。

- 运维方便：它非常容易配置和部署，灵活性高。

- 高度集成：现在已经有官方的 Golang、Python 和 JavaScript 客户端，社区也有了其他各个语言的客户端库方便接入，自定义客户端也非常容易。

----

### NSQ组件

----

![](/uploads/upload_3f354368475530c397929ec13e2ce447.gif)

----

Topic：一个 topic 就是程序发布消息的一个逻辑键，当程序第一次发布消息时就会创建 topic。

Channels：channel 与消费者相关，是消费者之间的负载均衡， channel 在某种意义上来说是一个“队列”。每当一个发布者发送一条消息到一个 topic，消息会被复制到所有消费者连接的 channel 上，消费者通过这个特殊的 channel 读取消息，实际上，在消费者第一次订阅时就会创建 channel。 Channel 会将消息进行排列，如果没有消费者读取消息，消息首先会在内存中排队，当量太大时就会被保存到磁盘中。

Messages：消息构成了我们数据流的中坚力量，消费者可以选择结束消息，表明它们正在被正常处理，或者重新将他们排队待到后面再进行处理。每个消息包含传递尝试的次数，当消息传递超过一定的阀值次数时，我们应该放弃这些消息，或者作为额外消息进行处理。

----

#### 基础架构

----

![](/uploads/upload_d5f0340beb221e9f742d651494b68c43.png)


----

nsqd ： 一个负责接收、排队、转发消息到客户端的守护进程

nsqlookupd ： 管理拓扑信息并提供最终一致性的发现服务的守护进程

nsqadmin ： 一套 Web 用户界面，可实时查看集群的统计数据和执行各种各样的管理任务

utilities ： 常见基础功能、数据流处理工具，如 nsq_stat、nsq_tail、nsq_to_file、nsq_to_http、nsq_to_nsq、to_nsq

---

### NSQD 入口

![](/uploads/upload_be3f971efc96adbd19fd90947cb77456.png)

----

#### 1、svc优雅的启用与退出程序

```go=
type Service interface {
    // Init is called before the program/service is started and after it's
    // determined if the program is running as a Windows Service.
    Init(Environment) error

    // Start is called after Init. This method must be non-blocking.
    Start() error

    // Stop is called in response to syscall.SIGINT, syscall.SIGTERM, or when a
    // Windows Service is stopped.
    Stop() error
}
```

----

#### 2、加载启动项和配置

```go=
	opts := nsqd.NewOptions()

	flagSet := nsqdFlagSet(opts)
	flagSet.Parse(os.Args[1:])

	rand.Seed(time.Now().UTC().UnixNano())

	if flagSet.Lookup("version").Value.(flag.Getter).Get().(bool) {
		fmt.Println(version.String("nsqd"))
		os.Exit(0)
	}
```

----

```go=
	var cfg config
	configFile := flagSet.Lookup("config").Value.String()
	if configFile != "" {
		_, err := toml.DecodeFile(configFile, &cfg)
		if err != nil {
			logFatal("failed to load config file %s - %s", configFile, err)
		}
	}
	cfg.Validate()

	options.Resolve(opts, flagSet, cfg)
```

----

#### 3、加载历史数据（ nsqd.LoadMetadata ）、持久化最新数据（ nsqd.PersistMetadata ），然后开启协程，进入 nsqd.Main() 主函数


```go=
    err = p.nsqd.LoadMetadata()
    ...
    err = p.nsqd.PersistMetadata()
    ...
    go func() {
	    err := p.nsqd.Main()
	    if err != nil {
	    	p.Stop()
	    	os.Exit(1)
	    }
    }()
```

----

### 4、跟踪nsqd.Main

----

```go=
func (n *NSQD) Main() error {
	exitCh := make(chan error)
	var once sync.Once
	exitFunc := func(err error) {    //出错时只执行一次
		once.Do(func() {
			if err != nil {
				n.logf(LOG_FATAL, "%s", err)
			}
			exitCh <- err
		})
	}

    //启动tcp服务
	n.waitGroup.Wrap(func() {
		exitFunc(protocol.TCPServer(n.tcpListener, n.tcpServer, n.logf))
	})

    //启动http服务
	httpServer := newHTTPServer(n, false, n.getOpts().TLSRequired == TLSRequired)
	n.waitGroup.Wrap(func() {
		exitFunc(http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf))
	})
    
    //启动https服务
	if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
		httpsServer := newHTTPServer(n, true, true)
		n.waitGroup.Wrap(func() {
			exitFunc(http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf))
		})
	}

    //启动三个协程
    
    //启动循环监控队列信息: 这个函数的作用是用来维护 channel 中的延时队列（deferedPQ）和等待消费确认队列(inflightPQ)。将超时的消息重新加入消费队列中。
	n.waitGroup.Wrap(n.queueScanLoop)
    //启动节点信息管理: 同步信息给lookupd
    n.waitGroup.Wrap(n.lookupLoop)
    //统计信息：同步信息给状态监护进程
	if n.getOpts().StatsdAddress != "" {
		n.waitGroup.Wrap(n.statsdLoop)
	}

	err := <-exitCh
	return err
}
```

----

5、http接口

----

```go=
	router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))
	router.Handle("GET", "/info", http_api.Decorate(s.doInfo, log, http_api.V1))

	// v1 negotiate
	router.Handle("POST", "/pub", http_api.Decorate(s.doPUB, http_api.V1))
	router.Handle("POST", "/mpub", http_api.Decorate(s.doMPUB, http_api.V1))
	router.Handle("GET", "/stats", http_api.Decorate(s.doStats, log, http_api.V1))

	// only v1
	router.Handle("POST", "/topic/create", http_api.Decorate(s.doCreateTopic, log, http_api.V1))
	router.Handle("POST", "/topic/delete", http_api.Decorate(s.doDeleteTopic, log, http_api.V1))
	router.Handle("POST", "/topic/empty", http_api.Decorate(s.doEmptyTopic, log, http_api.V1))
	router.Handle("POST", "/topic/pause", http_api.Decorate(s.doPauseTopic, log, http_api.V1))
	router.Handle("POST", "/topic/unpause", http_api.Decorate(s.doPauseTopic, log, http_api.V1))
	router.Handle("POST", "/channel/create", http_api.Decorate(s.doCreateChannel, log, http_api.V1))
	router.Handle("POST", "/channel/delete", http_api.Decorate(s.doDeleteChannel, log, http_api.V1))
	router.Handle("POST", "/channel/empty", http_api.Decorate(s.doEmptyChannel, log, http_api.V1))
	router.Handle("POST", "/channel/pause", http_api.Decorate(s.doPauseChannel, log, http_api.V1))
	router.Handle("POST", "/channel/unpause", http_api.Decorate(s.doPauseChannel, log, http_api.V1))
	router.Handle("GET", "/config/:opt", http_api.Decorate(s.doConfig, log, http_api.V1))
	router.Handle("PUT", "/config/:opt", http_api.Decorate(s.doConfig, log, http_api.V1))

	// debug
	router.HandlerFunc("GET", "/debug/pprof/", pprof.Index)
	router.HandlerFunc("GET", "/debug/pprof/cmdline", pprof.Cmdline)
	router.HandlerFunc("GET", "/debug/pprof/symbol", pprof.Symbol)
	router.HandlerFunc("POST", "/debug/pprof/symbol", pprof.Symbol)
	router.HandlerFunc("GET", "/debug/pprof/profile", pprof.Profile)
	router.Handler("GET", "/debug/pprof/heap", pprof.Handler("heap"))
	router.Handler("GET", "/debug/pprof/goroutine", pprof.Handler("goroutine"))
	router.Handler("GET", "/debug/pprof/block", pprof.Handler("block"))
	router.Handle("PUT", "/debug/setblockrate", http_api.Decorate(setBlockRateHandler, log, http_api.PlainText))
	router.Handler("GET", "/debug/pprof/threadcreate", pprof.Handler("threadcreate"))
```

----

6、tcp接口

----

```go=
func (p *tcpServer) Handle(conn net.Conn) {
	p.nsqd.logf(LOG_INFO, "TCP: new client(%s)", conn.RemoteAddr())

	// The client should initialize itself by sending a 4 byte sequence indicating
	// the version of the protocol that it intends to communicate, this will allow us
	// to gracefully upgrade the protocol away from text/line oriented to whatever...
	buf := make([]byte, 4)
	_, err := io.ReadFull(conn, buf)
	if err != nil {
		p.nsqd.logf(LOG_ERROR, "failed to read protocol version - %s", err)
		conn.Close()
		return
	}
	protocolMagic := string(buf)

	p.nsqd.logf(LOG_INFO, "CLIENT(%s): desired protocol magic '%s'",
		conn.RemoteAddr(), protocolMagic)

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
		prot = &protocolV2{nsqd: p.nsqd}
	default:
		protocol.SendFramedResponse(conn, frameTypeError, []byte("E_BAD_PROTOCOL"))
		conn.Close()
		p.nsqd.logf(LOG_ERROR, "client(%s) bad protocol magic '%s'",
			conn.RemoteAddr(), protocolMagic)
		return
	}

	client := prot.NewClient(conn)
	p.conns.Store(conn.RemoteAddr(), client)

	err = prot.IOLoop(client)
	if err != nil {
		p.nsqd.logf(LOG_ERROR, "client(%s) - %s", conn.RemoteAddr(), err)
	}

	p.conns.Delete(conn.RemoteAddr())
	client.Close()
}
```

----

#### 7、整体流程图

![](/uploads/upload_73f8cc8e26eabc6be84c0004040a2416.png)

----

#### 8、优先队列

[inFlightPqueue](https://github.com/nsqio/nsq/blob/master/nsqd/in_flight_pqueue.go)

[deferredPqueue](https://github.com/nsqio/nsq/tree/master/internal/pqueue/pqueue.go)

----

#### 9、消息分发

![](/uploads/upload_e0b9be064cfe4cb61715e911b11aad60.png)

---

### NSQLookupd

----

#### 整体流程

![](/uploads/upload_0163e879cd10c16a21d13f36673fbf0b.png)


----

#### lookupLoop

1、nsqd每隔15秒发送ping包给lookupd
2、topic发生变化通知到lookupd
3、channel发生变化通知到lookupd

---

### 源码亮点

----

#### 1、svc优雅的启用与退出程序

github.com/judwhite/go-svc


----

#### 2、waitGroup的使用

```go=
type NSQD struct {
    ...
    exitChan             chan int
    waitGroup            util.WaitGroupWrapper
    ...
}

func (n *NSQD) Exit() {
    ...
    close(n.exitChan)
    n.waitGroup.Wait()
    ...
}
```

----

#### 3、Decorate装饰器

从路由 `router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))`，可以看出 `httpServer` 通过 `http_api.Decorate` 装饰器实现对各 http 路由进行 handler 装饰，如加 log 日志、V1 协议版本号的统一格式输出等；

----

#### 4、锁与原子操作 RWMutex/atomic.Value

从下面的代码中可以看到，当需要获取一个 topic 的时候，先用读锁去读(此时如果有写锁将被阻塞)，若存在则直接返回，若不存在则使用写锁新建一个；另外，使用 atomic.Value 进行结构体某些字段的并发存取值，保证原子性。

----

```go=
func (n *NSQD) GetTopic(topicName string) *Topic {
	// most likely, we already have this topic, so try read lock first.
	n.RLock()
	t, ok := n.topicMap[topicName]
	n.RUnlock()
	if ok {
		return t
	}

	n.Lock()

	t, ok = n.topicMap[topicName]
	if ok {
		n.Unlock()
		return t
	}
	deleteCallback := func(t *Topic) {
		n.DeleteExistingTopic(t.name)
	}
	t = NewTopic(topicName, n, deleteCallback)
	n.topicMap[topicName] = t

	n.Unlock()

	n.logf(LOG_INFO, "TOPIC(%s): created", t.name)
	// topic is created but messagePump not yet started

	// if loading metadata at startup, no lookupd connections yet, topic started after load
	if atomic.LoadInt32(&n.isLoading) == 1 {
		return t
	}

	// if using lookupd, make a blocking call to get the topics, and immediately create them.
	// this makes sure that any message received is buffered to the right channels
	lookupdHTTPAddrs := n.lookupdHTTPAddrs()
	if len(lookupdHTTPAddrs) > 0 {
		channelNames, err := n.ci.GetLookupdTopicChannels(t.name, lookupdHTTPAddrs)
		if err != nil {
			n.logf(LOG_WARN, "failed to query nsqlookupd for channels to pre-create for topic %s - %s", t.name, err)
		}
		for _, channelName := range channelNames {
			if strings.HasSuffix(channelName, "#ephemeral") {
				continue // do not create ephemeral channel with no consumer client
			}
			t.GetChannel(channelName)
		}
	} else if len(n.getOpts().NSQLookupdTCPAddresses) > 0 {
		n.logf(LOG_ERROR, "no available nsqlookupd to query for channels to pre-create for topic %s", t.name)
	}

	// now that all channels are added, start topic messagePump
	t.Start()
	return t
}
```

----

#### 5、优先队列-小顶堆


优先级队列（ Priority Queue ） - 通过数据结构最小堆（ min heap ）实现，pub 一条消息时立即就排好序（优先级通过 Priority-timeout 时间戳排序），最近到期的放到最小堆根节点；取出一条消息直接从最小堆的根节点取出，时间复杂度很低。

----

#### 6、队列设计 - 延时/运行队列

延时队列（ deferredPQ ） - 通过 DPUB 发布消息时带有 timeout 属性值实现，表示从当前时间戳多久后可以取出来消费；

运行队列（ inFlightPQ ） - 正在被消费者 consumer 消费的消息放入运行队列中，若处理失败或超时则自动重新放入（ Requeue ）队列，待下一次取出再次消费；消费成功（ Finish ）则删除对应的消息。

----

#### 7、分布式 - 去中心化/无 SPOF

nsq 被设计以分布的方式被使用，客户端连接到指定 topic 的所有生产者 producer 实例。没有中间人，没有消息代理 broker ，也没有单点故障（ SPOF - single point of failure ）。
这种拓扑结构消除单链，聚合，消费者直接连接所有生产者。从技术上讲，哪个客户端连接到哪个 nsq 不重要，只要有足够的消费者 consumer 连接到所有生产者 producer，以满足大量的消息，保证所有东西最终将被处理。
对于 nsqlookupd，高可用性是通过运行多个实例来实现。他们不直接相互通信和数据被认为是最终一致。如果某个 nsqd 出现问题，down 机了，会和 nsqlookupd 断开，这样客户端从 nsqlookupd 得到的 nsqd 的列表永远是可用的。客户端连接的是所有的 nsqd，一个出问题了就用其他的连接，所以也不会受影响。

----

#### 8、 高可用、大吞吐量

高可用性（ HA ） - 通过集群化部署多个 nsqd, nsqlookupd 节点，可实现同时多生产者、多消费者运行，单一节点出现故障不影响系统运行；每个节点启动时都会先从磁盘读取未处理的消息，极端情况下，会丢失少量还未来得及存盘的内存中消息。

10 亿/天 - 通过 goroutine, channel 充分利用 golang 语言的协程并发特性，可高并发处理大量消息的生产与消费。例如 message 为 10 byte 大小，则 50( nsq 节点数) * 10(字节) 86400(一天秒数) 25(每秒处理消息数) = 10 亿，可见达到十亿级别的吞吐量，通过快速部署节点即可实现。

----

#### 9、快速扩容

nsq 集群很容易配置（多种参数设定方式：命令行 > 配置文件 > 默认值）和部署（编译的二进制可执行文件没有运行时依赖），通过简单设置初始化参数，运维 Ops 就可以快速增加 nsqd 或 nsqlookupd 节点，为 Topic 引入一个新的消费者，只需启动一个配置了 nsqlookup 实例地址的 nsq 客户端。无需为添加任何新的消费者或生产者更改配置，大大降低了开销和复杂性。

通过容器化管理多个实例将非常快速进行生产者、消费者的扩缩容，加上容器的流量监控、熔断、最低节点数等功能，保证了集群中 nsqd 的高效运行。

----

#### 10、消息多路分发 & 负载均衡

Topic 和 Channel 都没有预先配置。Topic 由第一次发布消息到命名的 Topic 或第一次通过订阅一个命名 Topic 来创建。Channel 被第一次订阅到指定的 Channel 创建。Topic 和 Channel 的所有缓冲的数据相互独立，防止缓慢消费者造成对其他 Channel 的积压（同样适用于 Topic 级别）。

----

多路分发 - producer 会同时连上 nsq 集群中所有 nsqd 节点，当然这些节点的地址是在初始化时，通过外界传递进去；当发布消息时，producer 会随机选择一个 nsqd 节点发布某个 Topic 的消息；consumer 在订阅 subscribe 某个Topic/Channel时，会首先连上 nsqlookupd 获取最新可用的 nsqd 节点，然后通过 TCP 长连接方式连上所有发布了指定 Topic 的 producer 节点，并在本地用 tornado 轮询每个连接，当某个连接有可读事件时，即有消息达到，处理即可。

----

负载均衡 - 当向某个 Topic 发布一个消息时，该消息会被复制到所有的 Channel，如果 Channel 只有一个客户端，那么 Channel 就将消息投递给这个客户端；如果 Channel 的客户端不止一个，那么 Channel 将把消息随机投递给任何一个客户端，这也可以看做是客户端的负载均衡；

----

#### 不足之处

nsq 保证消息至少被投递消费一次（幂等消费），当某个 nsqd 节点出现故障时，极端情况下内存里面的消息还未来得及存入磁盘，这部分消息将丢失；通过分布式多个 consumer 消费，会因为消息处理时长、网络延迟等导致消息重排，再次消费顺序与写入顺序不一致，因此在高可靠性、顺序性方面略存在不足，应根据具体的业务场景进行取舍。