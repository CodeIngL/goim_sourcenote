### goIm源码阅读之整体流程

看完了所有goIm组件源码，我想还是有童鞋还不知道很多东西。
我们现在重新来看一下整个流程

---
#### 推送  
假设系统现在要求一个推送。外部系统走的是http接口
也就是对应http.go中的某个接口

阅读下源码

	func InitHTTP() (err error) {
			//。。。。。。省略
			httpServeMux.HandleFunc("/1/push", Push)
			httpServeMux.HandleFunc("/1/pushs", Pushs)
			httpServeMux.HandleFunc("/1/push/all", PushAll)
			httpServeMux.HandleFunc("/1/push/room", PushRoom)
			httpServeMux.HandleFunc("/1/server/del", DelServer)
			httpServeMux.HandleFunc("/1/count", Count)
			//。。。。。。省略
	}

对于logic组件，对外开放了这6个http接口。我们选取其中的一个来简单的走一遍。  

#### 以Push为例
	func Push(w http.ResponseWriter, r *http.Request) {

		//省略部分代码---功能限定请求方式

		//省略部分代码---功能定义一大串变量
		
		defer retPWrite(w, r, res, &body, time.Now()) //defer返回函数，记录了一次推送时间

		//body处理
		if bodyBytes, err = ioutil.ReadAll(r.Body); err != nil {
			log.Error("ioutil.ReadAll() failed (%s)", err)
			res["ret"] = InternalErr
			return
		}
		body = string(bodyBytes)

		//userId解析
		if userId, err = strconv.ParseInt(uidStr, 10, 64); err != nil {
			log.Error("strconv.Atoi(\"%s\") error(%v)", uidStr, err)
			res["ret"] = InternalErr
			return
		}
		subKeys = genSubKey(userId)
		
		//写入kafka
		for serverId, keys = range subKeys {
			if err = mpushKafka(serverId, keys, bodyBytes); err != nil {
				res["ret"] = InternalErr
				return
			}
		}
		res["ret"] = OK
		return
	}

关注下subKey的产生:

	func genSubKey(userId int64) (res map[int32][]string) {
		var (
			err    error
			i      int
			ok     bool
			key    string
			keys   []string
			args   = proto.GetArg{UserId: userId}
			reply  = proto.GetReply{}
			client *xrpc.Clients
		)
		res = make(map[int32][]string)
		
		client = getRouterByUID(userId)
		
		client.Call(routerServiceGet, &args, &reply)
		
		for i = 0; i < len(reply.Servers); i++ {
			key = encode(userId, reply.Seqs[i])
			if keys, ok = res[reply.Servers[i]]; !ok {
				keys = []string{}
			}
			keys = append(keys, key)
			res[reply.Servers[i]] = keys
		}
		return
	}

>创建一个res(subkeys)  
通过userId使用一致性hash得到相应router组件对应的rpcClient  
远程调用router组件的rpcGet的接口。获得了seqs和comets信息     
然后遍历comets数组，处理userId和seq 形成key(userID_SEQ)    
将key按cometId分类，形成一个数组keys。作为res的键值对(cometId,keys),
这个res就是返回的subKeys

最后遍历subkey(res),将相关信息以及body组装写入kafka中

	func mpushKafka(serverId int32, keys []string, msg []byte) (err error) {

		//省略部分代码---功能组件一个kafk消息，topic为define.KAFKA_MESSAGE_MULTI
	
		//省略部分代码---功能json化得到byte
		
		producer.Input() <- &sarama.ProducerMessage{Topic: KafkaPushsTopic, Value: sarama.ByteEncoder(vBytes)}//写入channel
	}

然后某个协程handleSucess回调成功记录日志

push接口返回调用方相关信息(主要是res)

**这样只完成了消息的写入，还没推送给客户端**

----
### 推送

推送工作是由相应的job组件负责。  
我们现在来看一下，job的流程   
job组件的kafka.go里面去  

	go func() {
		for msg := range cg.Messages() {
			log.Info("deal with topic:%s, partitionId:%d, Offset:%d, Key:%s msg:%s", msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value)
			push(msg.Value)
			cg.CommitUpto(msg)
		}
	}()

看到这边在收取kafka的信息，然后调用了push。看一下源码 

	func push(msg []byte) (err error) {
		m := &proto.KafkaMsg{} 

		//省略部分代码----功能msg转m
		
		switch m.OP {
		case define.KAFKA_MESSAGE_MULTI:
			pushChs[rand.Int()%Conf.PushChan] <- &pushArg{ServerId: m.ServerId, SubKeys: m.SubKeys, Msg: m.Msg, RoomId: define.NoRoom}
		}
		//关注我们的部分。写入pushChan
	}

这些chan在哪里被读出呢。找到

	func processPush(ch chan *pushArg) {
		var arg *pushArg
		for {
			arg = <-ch
			mPushComet(arg.ServerId, arg.SubKeys, arg.Msg)
		}
	}

这个函数在一些协程里工作,arg进行了读取，调用了mPushComet(....)

	// mPushComet push a message to a batch of subkeys
	func mPushComet(serverId int32, subKeys []string, body json.RawMessage) {
		var args = proto.MPushMsgArg{
			Keys: subKeys, P: proto.Proto{Ver: 0, Operation: define.OP_SEND_SMS_REPLY, Body: body},
		}
		if c, ok := cometServiceMap[serverId]; ok {
			c.Push(&args)
		}
	}
这个毛剑大神在上面写了注释，批量的推送信息
看到关键Push操作，进入源码之后
	// user push
	func (c *Comet) Push(arg *proto.MPushMsgArg) (err error) {
		c.pushRoutines[num] <- arg
	}
又是写入了channel，然后探索哪里写出

>看到这个InitComet方法中有这个细节我们观察下	

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

开了很多协程，我们找到我们需要的东西**c.process(...)**

	// process
	func (c *Comet) process(pushChan chan *proto.MPushMsgArg, roomChan chan *proto.BoardcastRoomArg, broadcastChan chan *proto.BoardcastArg) {
		//省略部分代码
		for {
			select {
			case pushArg = <-pushChan:
				// push
				err = c.rpcClient.Call(CometServiceMPushMsg, pushArg, reply)
				if err != nil {
					log.Error("rpcClient.Call(%s, %v, reply) serverId:%d error(%v)", CometServiceMPushMsg, pushArg, c.serverId, err)
				}
				pushArg = nil
		}
		//省略部分代码
	}
这里从push中读出啊,直接rpc远程的comet组件。在comet中找到对应的接口

	// Push push a message to a specified sub key
	func (this *PushRPC) MPushMsg(arg *proto.MPushMsgArg, reply *proto.MPushMsgReply) (err error) {
		//省略部分代码
		for n, key = range arg.Keys {
			bucket = DefaultServer.Bucket(key)
			if channel = bucket.Channel(key); channel != nil {
				if err = channel.Push(&arg.P); err != nil {
					return
				}
				reply.Index = int32(n)
			}
		}
		return
	}
功能只有一个，找到对应的channel写入  

	// Push server push message.
	func (c *Channel) Push(p *proto.Proto) (err error) {
		select {
		case c.signal <- p:
		default:
		}
		return
	}
写入到c.signal这个chan，哪里有写入，哪里就有写出。寻找源码

	// Ready check the channel ready or close?
	func (c *Channel) Ready() *proto.Proto {
		return <-c.signal
	}

在这里方法中被取出，哪里调用的呢，继续看

	func (server *Server) dispatchWebsocket(key string, conn *websocket.Conn, ch *Channel) {
		for {
			p = ch.Ready()
			switch p {
			case proto.ProtoFinish:
				//省略
			case proto.ProtoReady:
				for {
					p.WriteWebsocket(conn)
				}
			default:
				p.WriteWebsocket(conn)
			}
		}
	}

这个方法在websocket.go中开了协程进行处理，关注上面的
	**proto.ProtoReady:**
这样就写回了websocket的链接，写会给客户端，也就是完成了整个的推送。

### 尾声

我们只是简单介绍了推送的流程，长连接的维持这些细节并没有，好好推敲。有兴趣的童鞋可以继续看一下。比较简单。tcp的方式也是类似的


