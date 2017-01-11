### goIm组件之job

#### 瞎扯
这是最后一个组件了，但这个组件是很有用的。
是不是呢？是的啊。自问自答有点神经啊

#### 分析 

还是老规矩
看job组件的main函数
前面的流程基本是一样的
一个全局配置

	type Config struct {
		Log        string   `goconf:"base:log"`
		ZKAddrs    []string `goconf:"kafka:zookeeper.list:,"`
		ZKRoot     string   `goconf:"kafka:zkroot"`
		KafkaTopic string   `goconf:"kafka:topic"`
		// comet
		Comets      map[int32]string `goconf:"-"`
		RoutineSize int64            `goconf:"comet:routine.size"`
		RoutineChan int              `goconf:"comet:routine.chan"`
		// push
		PushChan     int `goconf:"push:chan"`
		PushChanSize int `goconf:"push:chan.size"`
		// timer
		Timer     int `goconf:"timer:num"`
		TimerSize int `goconf:"timer:size"`
		// room
		RoomBatch  int           `goconf:"room:batch"`
		RoomSignal time.Duration `goconf:"room:signal:time"`
		// monitor
		MonitorOpen  bool     `goconf:"monitor:open"`
		MonitorAddrs []string `goconf:"monitor:addrs:,"`
	}

上面流程基本一样，只要心存个概念就好

然后main的流程就到了InitComet()

	func InitComet(addrs map[int32]string, options CometOptions) (err error) {
		var (
			serverId      int32
			bind          string
			network, addr string
		)
		for serverId, bind = range addrs {
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
			rpcClient.Ping(CometServicePing)
			// comet
			c := new(Comet)
			c.serverId = serverId
			c.rpcClient = rpcClient
			c.pushRoutines = make([]chan *proto.MPushMsgArg, options.RoutineSize)
			c.roomRoutines = make([]chan *proto.BoardcastRoomArg, options.RoutineSize)
			c.broadcastRoutines = make([]chan *proto.BoardcastArg, options.RoutineSize)
			c.options = options
			cometServiceMap[serverId] = c
			// process
			for i := int64(0); i < options.RoutineSize; i++ {
				pushChan := make(chan *proto.MPushMsgArg, options.RoutineChan)
				roomChan := make(chan *proto.BoardcastRoomArg, options.RoutineChan)
				broadcastChan := make(chan *proto.BoardcastArg, options.RoutineChan)
				c.pushRoutines[i] = pushChan
				c.roomRoutines[i] = roomChan
				c.broadcastRoutines[i] = broadcastChan
				go c.process(pushChan, roomChan, broadcastChan)
			}
			log.Info("init comet rpc: %v", rpcOptions)
		}
		return
	}

commet组件的工作可不小,值得我们深究下  
前面是老套路，循环遍历，组装对应rpcClient,开协程维持rpc的可用性(ping)。  
然后就是new Comet  
设置相应的配置，包括Id,rpcClient,以及三个类型的chan数组，分别对应push,room,broadcast  
然后就是这个comet以键值对的方式存入内存（cometServiceMap）  

接着就是创建三种类型的chan，并发来负责推送(go c.process(pushChan,roomChan,broadcastChan))　　

#### 推送关键

	// process
	func (c *Comet) process(pushChan chan *proto.MPushMsgArg, roomChan chan *proto.BoardcastRoomArg, broadcastChan chan *proto.BoardcastArg) {
		var (
			pushArg      *proto.MPushMsgArg
			roomArg      *proto.BoardcastRoomArg
			broadcastArg *proto.BoardcastArg
			reply        = &proto.NoReply{}
			err          error
		)
		for {
			select {
			case pushArg = <-pushChan:
				// push
				err = c.rpcClient.Call(CometServiceMPushMsg, pushArg, reply)
				if err != nil {
					log.Error("rpcClient.Call(%s, %v, reply) serverId:%d error(%v)", CometServiceMPushMsg, pushArg, c.serverId, err)
				}
				pushArg = nil
			case roomArg = <-roomChan:
				// room
				err = c.rpcClient.Call(CometServiceBroadcastRoom, roomArg, reply)
				if err != nil {
					log.Error("rpcClient.Call(%s, %v, reply) serverId:%d error(%v)", CometServiceBroadcastRoom, roomArg, c.serverId, err)
				}
				roomArg = nil
			case broadcastArg = <-broadcastChan:
				// broadcast
				err = c.rpcClient.Call(CometServiceBroadcast, broadcastArg, reply)
				if err != nil {
					log.Error("rpcClient.Call(%s, %v, reply) serverId:%d error(%v)", CometServiceBroadcast, broadcastArg, c.serverId, err)
				}
				broadcastArg = nil
			}
		}
	}

显而易见的是，只要有chan有数据读出，就rpc调用远程comet组件的相应接口,十分简洁  


回到main中。接下来开启监控

然后初始化RoomBucket

	//room
	InitRoomBucket(round,
		RoomOptions{
		BatchNum:   Conf.RoomBatch,
		SignalTime: Conf.RoomSignal,
		})

		
---
##### 接着

	func MergeRoomServers() {
		var (
			c           *Comet
			ok          bool
			roomId      int32
			serverId    int32
			roomIds     map[int32]struct{}
			servers     map[int32]struct{}
			roomServers = make(map[int32]map[int32]struct{})
		)
		// all comet nodes
		for serverId, c = range cometServiceMap {
			if c.rpcClient != nil {
				if roomIds = roomsComet(c.rpcClient); roomIds != nil {
					// merge room's servers
					for roomId, _ = range roomIds {
						if servers, ok = roomServers[roomId]; !ok {
							servers = make(map[int32]struct{})
							roomServers[roomId] = servers
						}
						servers[serverId] = struct{}{}
					}
				}
			}
		}
		RoomServersMap = roomServers
	}
这个干了什么事呢,和logic源码分析中**MergeCount**的功能类似,不多说  
相应的信息保存在内存(RoomServersMap)中  
之后也是开协程，定时统计  

---

#### 继续
	InitPush()

前面InitComet中就有了相关的Chanel现在是搞鸡巴。不得不看一下源码

	func InitPush() {
		pushChs = make([]chan *pushArg, Conf.PushChan)
		for i := 0; i < Conf.PushChan; i++ {
			pushChs[i] = make(chan *pushArg, Conf.PushChanSize)
			go processPush(pushChs[i])
		}
	}

一堆的chan，搞基呢。继续深入

	// push routine
	func processPush(ch chan *pushArg) {
		var arg *pushArg
		for {
			arg = <-ch
			mPushComet(arg.ServerId, arg.SubKeys, arg.Msg)
		}
	}

阻塞了啊哇，ch中读了什么，什么时候有数据，感觉我们要仔细看一下。源码搜索中   
然后我们马上定位到job组件中的**push(...)**操作中，看到。。。好吧不能再多说了  
说多了就会扯到其他组件了，反正这里做的的就是等ch有数据，有数据就执行**mPushComet(arg.ServerId, arg.SubKeys, arg.Msg)**

	// mPushComet push a message to a batch of subkeys
	func mPushComet(serverId int32, subKeys []string, body json.RawMessage) {
		var args = proto.MPushMsgArg{
			Keys: subKeys, P: proto.Proto{Ver: 0, Operation: define.OP_SEND_SMS_REPLY, Body: body},
		}
		if c, ok := cometServiceMap[serverId]; ok {
			if err := c.Push(&args); err != nil {
				log.Error("c.Push(%v) serverId:%d error(%v)", args, serverId, err)
			}
		}
	}

这一步马上就找到了上面的所说的cometServiceMap并找到了某个特定Comet，然后Push
我们再看一下

	// user push
	func (c *Comet) Push(arg *proto.MPushMsgArg) (err error) {
		num := atomic.AddInt64(&c.pushRoutinesNum, 1) % c.options.RoutineSize
		c.pushRoutines[num] <- arg
		return
	}

好，写入channel了，打赌读出的地方肯定是调用rpc给comet组件了.  
为什么这样说呢，请童鞋注意下**c.pushRoutines**这在前面就已经提到了啊。如果你不知道请看上面  
的**process(...)**

最后初始化kafka，需要提出的是kafka这里是第三方的，所以不是我们关注的重点，可以当做一个黑盒。
不影响你的阅读，当然，里面的逻辑也是十分的简单

至此job组件完成整个操作

-----



