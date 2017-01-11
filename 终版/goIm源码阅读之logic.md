### goIm组件之logic
goIm中的每个组件整体逻辑都是很类似的，前面的流程几乎一样。不做详细的介绍了

#### 小小介绍

直接进main

流程介绍
>初始化配置对象
使用goConf去设置内存配置对象  
设置核心数  
开启日志  
开启监控  
开启http监控  
注册到router的RPC  

稍微提下这个注册  

	func InitRouter(addrs map[string]string) (err error) {
		var (
			network, addr string
		)
		routerRing = ketama.NewRing(ketama.Base)
		for serverId, bind := range addrs {
			var rpcOptions []xrpc.ClientOptions
			for _, bind = range strings.Split(bind, ",") {
				if network, addr, err = inet.ParseNetwork(bind); err != nil {
					log.Error("inet.ParseNetwork() error(%v)", err)
					return
				}
				options := xrpc.ClientOptions{
					Proto: network,
					Addr:  addr,
				}
				rpcOptions = append(rpcOptions, options)
			}
			// rpc clients
			rpcClient := xrpc.Dials(rpcOptions)
			// ping & reconnect
			rpcClient.Ping(routerServicePing)
			routerRing.AddNode(serverId, 1)
			routerServiceMap[serverId] = rpcClient
		}
		routerRing.Bake()
		return
	}
>**整个函数的功能就是把所有的router节点使用ketama一致性处理，  
然后在routerServiceMap保存相应节点和rcp客户端的对象，  
同时维持使用协程维持rpc(远程ping)的可用性**

---
视线回到logic的main.go中，走完router的RPC，执行一个mergeCount  
这个是干什么呢

	func MergeCount() {
		var (
			c                     *xrpc.Clients
			err                   error
			roomId, server, count int32
			counter               map[int32]int32
			roomCount             = make(map[int32]int32)
			serverCount           = make(map[int32]int32)
		)
		// all comet nodes
		for _, c = range routerServiceMap {
			if c != nil {
				if counter, err = allRoomCount(c); err != nil {
					continue
				}
				for roomId, count = range counter {
					roomCount[roomId] += count
				}
				if counter, err = allServerCount(c); err != nil {
					continue
				}
				for server, count = range counter {
					serverCount[server] += count
				}
			}
		}
		RoomCountMap = roomCount
		ServerCountMap = serverCount
	}

代码不长，就一个for循环  
> 功能就是找到router的节点和相应的rpcclient  
 执行allRoomCount(c)，得到一个map(roomId,count)这是在一个router上的  
 然后合并统计，roomId的实际数量
 执行allServerCount(c)同理
 server的实际数量

然后并发执行syncCount()，每秒统计下

接着初始化rpc接口

 	InitRPC(NewDefaultAuther())

做了什么呢 继续看呗 
首先newDefaultAuther()，这个是默认的认证demo，这个默认的认证实在太简单了。

	func InitRPC(auther Auther) (err error) {
		c = &RPC{auther: auther}
		rpc.Register(c)
		for i := 0; i < len(Conf.RPCAddrs); i++ {
			go rpcListen(network, addr)
		}
	}

逻辑十分简单，一眼看透。

接下来初始化对外的HTTP接口，提供推送

	InitHTTP();

最后初始化kafka

	InitKafka(Conf.KafkaAddrs);

至此logic的逻辑也全部完成了

#### 尾声  

上面毛大的注释好像是写错了吧，应该是router nodes







