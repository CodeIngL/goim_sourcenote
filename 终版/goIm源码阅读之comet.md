### goIm组件之comet

#### 分析了又分析的流程
先初始化配置对象，  
然后使用goConf去设置相应的配置。  
开启核心数，  
开启日志  
然后开启监控  
开启http监控  
注册到router的RPC上上去  
还有。。。。。  


#### 正文
看comet组件中的main  


##### 首先

首先进入InitConfig看看
逻辑是一模一样的
先初始化话一个全局的配置
这个全局的配置的默认配置：

	func NewConfig() *Config {
		return &Config{
			// base section
			PidFile:   "/tmp/goim-comet.pid",
			Dir:       "./",
			Log:       "./comet-log.xml",
			MaxProc:   runtime.NumCPU(),
			PprofBind: []string{"localhost:6971"},
			StatBind:  []string{"localhost:6972"},
			Debug:     true,
			// tcp
			TCPBind:      []string{"0.0.0.0:8080"},
			TCPSndbuf:    1024,
			TCPRcvbuf:    1024,
			TCPKeepalive: false,
			// websocket
			WebsocketBind: []string{"0.0.0.0:8090"},
			// websocket tls
			WebsocketTLSOpen:     false,
			WebsocketTLSBind:     []string{"0.0.0.0:8095"},
			WebsocketCertFile:    "../source/cert.pem",
			WebsocketPrivateFile: "../source/private.pem",
			// flash safe policy
			FlashPolicyOpen: false,
			FlashPolicyBind: []string{"0.0.0.0:843"},
			// proto section
			HandshakeTimeout: 5 * time.Second,
			WriteTimeout:     5 * time.Second,
			TCPReadBuf:       1024,
			TCPWriteBuf:      1024,
			TCPReadBufSize:   1024,
			TCPWriteBufSize:  1024,
			// timer
			Timer:     runtime.NumCPU(),
			TimerSize: 1000,
			// bucket
			Bucket:        1024,
			CliProto:      5,
			SvrProto:      80,
			BucketChannel: 1024,
			// push
			RPCPushAddrs: []string{"localhost:8083"},
		}
	}

>下面属性介绍可以先忽略，主要是理解不恰当，你可以参考，但是不要去记住，因为你记不住  

    PidFile *nix下进程相关
    Dir 目录路径
    Log 日志配置
    MaxProc 核心数
    PprofBind 监控地址
    StatBind ？？未知
    Debug true 默认debug

    TCPBind tcp绑定地址
    TCPSndbuf 发送包大小
    TCPRcvbuf 接收包大小
    TCPKeepalive 是否长链 默认false

    WebsocketBind websocket绑定地址

    WebsocketTLSOpen 是否对websocket开启TLS认证 默认是false
    WebsocketTLSBind TLS认证绑定地址
    WebsocketCertFile 公钥文件
    WebsocketPrivateFile 私钥文件

    FlashPolicyOpen
    FlashPolicyBind
    直接忽略，我是不用的    

    HandshakeTimeout 握手超时时间 5s
    WriteTimeout 写的超时时间
    TCPReadBuf 1K
    TCPWriteBuf 1K
    TCPReadBufSize 1k
    TCPWriteBufSize 1K

    Timer 定时器
    TimerSize 1000

    Bucket 桶数量1K
    CliProto 5个
    SvrProto 80个
    BucketChannel 1024个

    RPCPushAddrs 推送地址

----

### 接着
**逻辑和router一模一样**  
然后根据配置文件设置相应的属性  
然后设置调试模式  
设置核心数  
设置日志  
然后设置 white list log  
设置监控  

---
### 重点来了
每个组件的的重点都在**initXXX**

	InitLogicRpc(Conf.LogicAddrs)
点击进去查看

	func InitLogicRpc(addrs []string) (err error) {
		var (
			bind          string
			network, addr string
			rpcOptions    []xrpc.ClientOptions
		)
		for _, bind = range addrs {
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
		logicRpcClient = xrpc.Dials(rpcOptions)
		// ping & reconnect
		logicRpcClient.Ping(logicServicePing)
		log.Info("init logic rpc: %v", rpcOptions)
		return
	}

循环遍历addrs，得到network,addr对应配置文件看一下

	[logic]
	# logic service rpc address
	# set(logic1, logic2)
	#
	# Examples:
	#
	# rpc.addrs tcp@localhost:7170,tcp@localhost:7170
	rpc.addrs tcp@localhost:7170
举个简单例子，network是tcp addr是localhost:7170  
这将被封装成一个ClientOptions。它的定义如下:

	// Rpc client options.
	type ClientOptions struct {
		Proto string
		Addr  string
	}
循环遍历将这个ClientOptions封装成一个数组。并传递经Dial(....)里面  
我们可以看一下Dial(...)的源码


	// Dial connects to an RPC server at the specified network address.
	func Dial(options ClientOptions) (c *Client) {
		c = new(Client)
		c.options = options
		c.dial()
		return
	}

	// Dial connects to an RPC server at the specified network address.
	func (c *Client) dial() (err error) {
		var conn net.Conn
		conn, err = net.DialTimeout(c.options.Proto, c.options.Addr, dialTimeout)
		if err != nil {
			log.Error("net.Dial(%s, %s), error(%v)", c.options.Proto, c.options.Addr, err)
		} else {
			c.Client = rpc.NewClient(conn)
		}
		return
	}

简单的说明，这里有封装了一个有效的Client，这是关于rpc的client，来源于毛大自己的封装。结构如下:

	// Client is rpc client.
	type Client struct {
		*rpc.Client
		options ClientOptions
		quit    chan struct{}
		err     error
	}

一个rpc的一个客户端(有效的)  
一个客户端的的属性组成的数组:包含net类型和地址  
一个通道  
一个错误字段  

---  

>rpc的客户端里面的字段是这样的

	// Client represents an RPC Client.
	// There may be multiple outstanding Calls associated
	// with a single Client, and a Client may be used by
	// multiple goroutines simultaneously.
	type Client struct {
		codec ClientCodec

		reqMutex sync.Mutex // protects following
		request  Request

		mutex    sync.Mutex // protects following
		seq      uint64
		pending  map[uint64]*Call
		closing  bool // user has called Close
		shutdown bool // server has told us to stop
	}

里面要加锁。因为这个对象在并发的时候可能会被共享


接下来使用ping来rpc调用远程维持远程logic组件的的链接     
如果短线会尝试重连
时间为1S单位  
当字段quit被写入，rpcClient会失效并关闭

------

### 继续
返回main.go
然后进行monitor的监控，直接略过。  之后定义了很多变量，我们需要仔细看一下

	// new server
	buckets := make([]*Bucket, Conf.Bucket)
	for i := 0; i < Conf.Bucket; i++ {
		buckets[i] = NewBucket(BucketOptions{
			ChannelSize:   Conf.BucketChannel,
			RoomSize:      Conf.BucketRoom,
			RoutineAmount: Conf.RoutineAmount,
			RoutineSize:   Conf.RoutineSize,
		})
	}
	round := NewRound(RoundOptions{
		Reader:       Conf.TCPReader,
		ReadBuf:      Conf.TCPReadBuf,
		ReadBufSize:  Conf.TCPReadBufSize,
		Writer:       Conf.TCPWriter,
		WriteBuf:     Conf.TCPWriteBuf,
		WriteBufSize: Conf.TCPWriteBufSize,
		Timer:        Conf.Timer,
		TimerSize:    Conf.TimerSize,
	})
	operator := new(DefaultOperator)
	DefaultServer = NewServer(buckets, round, operator, ServerOptions{
		CliProto:         Conf.CliProto,
		SvrProto:         Conf.SvrProto,
		HandshakeTimeout: Conf.HandshakeTimeout,
		TCPKeepalive:     Conf.TCPKeepalive,
		TCPRcvbuf:        Conf.TCPRcvbuf,
		TCPSndbuf:        Conf.TCPSndbuf,
	})

上面来源于main.go中

这里涉及到的结构挺多的，我们先先穿插几个结构看看吧  

>room.go

	type Room struct {
		id     int32  //标示 
		rLock  sync.RWMutex //锁
		next   *Channel  //下一个Channel 这是一个包装的channel 
		drop   bool  //是否无效  
		Online int // dirty read is ok  允许脏读
	}

>童鞋们可以看一下里面的各个操作，简单一眼明了  
对于put操作  
将一个channel插入头部，并且在线房间数++  

>对于del操作  
删除一个channel 当onlien数为0的时候，标示这个房间抛弃了

>对于push操作  
是向房间放入消息，如果chan满了，就丢弃

>对于Close操作  
关闭所有的chan，就是关闭了


### comet组件中的logic.go
规定logic组件rpc调用的接口  
其中一个包内函数

	func connect(p *proto.Proto) (key string, rid int32, heartbeat time.Duration, err error) {
		var (
			arg   = proto.ConnArg{Token: string(p.Body), Server: Conf.ServerId}
			reply = proto.ConnReply{}
		)
		if err = logicRpcClient.Call(logicServiceConnect, &arg, &reply); err != nil {
			log.Error("c.Call(\"%s\", \"%v\", &ret) error(%v)", logicServiceConnect, arg, err)
			return
		}
		key = reply.Key
		rid = reply.RoomId
		heartbeat = 5 * 60 * time.Second
		return
	}

这是Proto的一个方法实现  
连接参数数 一个Key 一个rid 一个心跳时间 错误信息    
远程RPC的调用   
调用参数为token 由于是连接直接就是body中的内容，
还有一个标示自己组件的ServerId 响应参数是key和一个roomId
调用对应的远程RPC方法 

	// Connect auth and registe login
	func (r *RPC) Connect(arg *proto.ConnArg, reply *proto.ConnReply) (err error) {
		if arg == nil {
			err = ErrConnectArgs
			log.Error("Connect() error(%v)", err)
			return
		}
		var (
			uid int64
			seq int32
		)
		uid, reply.RoomId = r.auther.Auth(arg.Token)
		if seq, err = connect(uid, arg.Server, reply.RoomId); err == nil {
			reply.Key = encode(uid, seq)
		}
		return
	}

连接的验证和登录，对于每个commet返回一个key 和一个RoomId   
首先是验证规则 

	uid, reply.RoomId = r.auther.Auth(arg.Token)
	
---

	func (a *DefaultAuther) Auth(token string) (userId int64, roomId int32) {
		var err error
		if userId, err = strconv.ParseInt(token, 10, 64); err != nil {
			userId = 0
			roomId = define.NoRoom
		} else {
			roomId = 1 // only for debug
		}
		return
	}

这个干了什么呢，直接解析token为一个整数，如果解析成功就赋值userId = 0 roomId为 -1  
否则roomId=1 

//观看证调用无论 ParseInt如何解析 返回的只有两种 （0，-1），（token的十进制，1）说明了一个问题
就是这里可能是给我们一个例子而已，看出来了，因为这是一DefaultAuther的类  

回头看roomId是返回过来了 
然后
	if seq, err = connect(uid, arg.Server, reply.RoomId); err == nil {
			reply.Key = encode(uid, seq)
		}

	函数定义
	func connect(userID int64, server, roomId int32) (seq int32, err error) {
		var (
			args   = proto.PutArg{UserId: userID, Server: server, RoomId: roomId}
			reply  = proto.PutReply{}
			client *xrpc.Clients
		)
		if client, err = getRouterByUID(userID); err != nil {
			return
		}
		if err = client.Call(routerServicePut, &args, &reply); err != nil {
			log.Error("c.Call(\"%s\",\"%v\") error(%v)", routerServicePut, args, err)
		} else {
			seq = reply.Seq
		}
		return
	}

这里干嘛来着呢
 这里又进行了一次RPC的调用，
 组成的参数是 userId肯定是0，或者token对应十进制值，server:commet组件的标示，roomId也只有两个属性-1 1
 然后根据userId得带了client相应的

 if client, err = getRouterByUID(userID); err != nil {
		return
	}

接着看一下这个函数


func getRouterByUID(userID int64) (*xrpc.Clients, error) {
	return getRouterByServer(routerRing.Hash(strconv.FormatInt(userID, 10)))
}

func getRouterByServer(server string) (*xrpc.Clients, error) {
	if client, ok := routerServiceMap[server]; ok {
		return client, nil
	} else {
		return nil, ErrRouter
	}
}

使用了一致性的hash
使用产生的hash值找到对应的值xrpcClient 这个routerServiceMap结构实在logic里面main执行的，这也看出来了commet组件必须在logic之后使用

然后就是正式的远程rpc了 对应的是
	routerServicePut            = "RouterRPC.Put"

func (r *RouterRPC) Put(arg *proto.PutArg, reply *proto.PutReply) error {
	reply.Seq = r.bucket(arg.UserId).Put(arg.UserId, arg.Server, arg.RoomId)
	return nil
}

直接使用相应的桶放进相应的东东，为了利用桶，还是根据通数量，使用userID去余做简单的映射 
然后返回桶，并放置相应的信息


// Put put a channel according with user id.
func (b *Bucket) Put(userId int64, server int32, roomId int32) (seq int32) {
	var (
		s  *Session
		ok bool
	)
	b.bLock.Lock()
	if s, ok = b.sessions[userId]; !ok {
		s = NewSession(b.server)
		b.sessions[userId] = s
	}
	if roomId != define.NoRoom {
		seq = s.PutRoom(server, roomId)
	} else {
		seq = s.Put(server)
	}
	b.counter(userId, server, roomId, true)
	b.bLock.Unlock()
	return
}

这是干什么呢
首先加锁，因为这些桶必然会被重复利用的
然后观察是否有session，没有新建，并使用userId作为键值
然后判断roomI的是不是 -1 不是说明则放置sever和RoomId形成的room
否则直接放置seq

然后这个session是什么东西呢

type Session struct {
	seq     int32
	servers map[int32]int32           // seq:server
	rooms   map[int32]map[int32]int32 // roomid:seq:server with specified room id
}

一个seq
一个server
一个rooms


// PutRoom put a session in a room according with subkey.
func (s *Session) PutRoom(server int32, roomId int32) (seq int32) {
	var (
		ok   bool
		room map[int32]int32
	)
	seq = s.Put(server)
	if room, ok = s.rooms[roomId]; !ok {
		room = make(map[int32]int32)
		s.rooms[roomId] = room
	}
	room[seq] = server
	return
}

// Put put a session according with sub key.
func (s *Session) Put(server int32) (seq int32) {
	seq = s.nextSeq()
	s.servers[seq] = server
	return
}

func (s *Session) nextSeq() int32 {
	s.seq++
	return s.seq
}

1、现将session中的seq加1 然后映射到server上去，
2、然后根据roomId找到room 实际上上面的正是因为是一个demo所以只写了-1和1其实应该还有很多种的
3、没有room则在该session中新建下
4、然后把server和seq设置到room中，这样room就能找到seq和server了然后返回，否泽只执行1


接着调用
b.counter(userId, server, roomId, true)

// counter incr or decr counter.
func (b *Bucket) counter(userId int64, server int32, roomId int32, incr bool) {
	var (
		sm map[int64]int32
		v  int32
		ok bool
	)
	if sm, ok = b.userServerCounter[server]; !ok {
		sm = make(map[int64]int32, b.session)
		b.userServerCounter[server] = sm
	}
	if incr {
		sm[userId]++
		b.roomCounter[roomId]++
		b.serverCounter[server]++
	} else {
		// WARN:
		// if decr a userid but key not exists just ignore
		// this may not happen
		if v, _ = sm[userId]; v-1 == 0 {
			delete(sm, userId)
		} else {
			sm[userId] = v - 1
		}
		b.roomCounter[roomId]--
		b.serverCounter[server]--
	}
}// counter incr or decr counter.
func (b *Bucket) counter(userId int64, server int32, roomId int32, incr bool) {
	var (
		sm map[int64]int32
		v  int32
		ok bool
	)
	if sm, ok = b.userServerCounter[server]; !ok {
		sm = make(map[int64]int32, b.session)
		b.userServerCounter[server] = sm
	}
	if incr {
		sm[userId]++
		b.roomCounter[roomId]++
		b.serverCounter[server]++
	} else {
		// WARN:
		// if decr a userid but key not exists just ignore
		// this may not happen
		if v, _ = sm[userId]; v-1 == 0 {
			delete(sm, userId)
		} else {
			sm[userId] = v - 1
		}
		b.roomCounter[roomId]--
		b.serverCounter[server]--
	}
}
这函数是干什么用的呢
注释上说是原子的增加或者减少数量，也就是统计用来着

最后东西完成，返回seq

得到seq后，使用
		reply.Key = encode(uid, seq)
得到了key值
这里只是简单的拼接了uid和seq使用‘_’来拼接

最后两次rpc调用放回
connect 返回 key,roomId,心跳时间5分钟，

这个函数是被
operation.go
func (operator *DefaultOperator) Connect(p *proto.Proto) (key string, rid int32, heartbeat time.Duration, err error) {
	key, rid, heartbeat, err = connect(p)
	return
}
中调用的
这个会在某些地方被调用


首先我们来看先我们熟悉的一个东西

在commet组件中的main.go里面，这是一个初始化对websocket的支持
首先就是解析，然后遍历绑定到相应的端口，这是非常简单的操作
然后就是这样的了
映射到ServeWebSocket来处理每个客户端请求

func ServeWebSocket(w http.ResponseWriter, req *http.Request) {
	if req.Method != "GET" {
		http.Error(w, "Method Not Allowed", 405)
		return
	}
	ws, err := upgrader.Upgrade(w, req, nil)
	if err != nil {
		log.Error("Websocket Upgrade error(%v), userAgent(%s)", err, req.UserAgent())
		return
	}
	defer ws.Close()
	var (
		lAddr = ws.LocalAddr()
		rAddr = ws.RemoteAddr()
		tr    = DefaultServer.round.Timer(rand.Int())
	)
	log.Debug("start websocket serve \"%s\" with \"%s\"", lAddr, rAddr)
	DefaultServer.serveWebsocket(ws, tr)
}

方式是这样的，请求方式一定是get，这是硬性要求，然后
将http升级到websocket
接下来使用一个DefaultServer来处理ws和tr


这个函数又做了什么呢


func (server *Server) serveWebsocket(conn *websocket.Conn, tr *itime.Timer) {
	var (
		err error
		key string
		hb  time.Duration // heartbeat
		p   *proto.Proto
		b   *Bucket
		trd *itime.TimerData
		ch  = NewChannel(server.Options.CliProto, server.Options.SvrProto, define.NoRoom)
	)
	// handshake
	trd = tr.Add(server.Options.HandshakeTimeout, func() {
		conn.Close()
	})
	// must not setadv, only used in auth
	if p, err = ch.CliProto.Set(); err == nil {
		if key, ch.RoomId, hb, err = server.authWebsocket(conn, p); err == nil {
			b = server.Bucket(key)
			err = b.Put(key, ch)
		}
	}
	if err != nil {
		conn.Close()
		tr.Del(trd)
		log.Error("handshake failed error(%v)", err)
		return
	}
	trd.Key = key
	tr.Set(trd, hb)
	// hanshake ok start dispatch goroutine
	go server.dispatchWebsocket(key, conn, ch)
	for {
		if p, err = ch.CliProto.Set(); err != nil {
			break
		}
		if err = p.ReadWebsocket(conn); err != nil {
			break
		}
		//p.Time = *globalNowTime
		if p.Operation == define.OP_HEARTBEAT {
			// heartbeat
			tr.Set(trd, hb)
			p.Body = nil
			p.Operation = define.OP_HEARTBEAT_REPLY
		} else {
			// process message
			if err = server.operator.Operate(p); err != nil {
				break
			}
		}
		ch.CliProto.SetAdv()
		ch.Signal()
	}
	log.Error("key: %s server websocket failed error(%v)", key, err)
	tr.Del(trd)
	conn.Close()
	ch.Close()
	b.Del(key)
	if err = server.operator.Disconnect(key, ch.RoomId); err != nil {
		log.Error("key: %s operator do disconnect error(%v)", key, err)
	}
	if Debug {
		log.Debug("key: %s server websocket goroutine exit", key)
	}
	return
}

定义一群变量
然后使用tr.add去建立一个握手 
然后去获得一个proto
然后去认证下，
接着把rpc过来的key也就时seq_usrId创建一个新的Bucket
并把key和chanel一起放置在里面，这样就有信息了
然后设置连接的时长
这一个是一个握手的过程
然后起一个携程来处理逻辑
下面一个死循环
获得一个pro
然后检查连是否正常
然后pro的属性是心跳，就进行时间的重新设定
如果不是的话就处理消息
然后写的树龄++
阻塞

现在看go 携程中做了什么事情



// dispatch accepts connections on the listener and serves requests
// for each incoming connection.  dispatch blocks; the caller typically
// invokes it in a go statement.
func (server *Server) dispatchWebsocket(key string, conn *websocket.Conn, ch *Channel) {
	var (
		p   *proto.Proto
		err error
	)
	if Debug {
		log.Debug("key: %s start dispatch websocket goroutine", key)
	}
	for {
		p = ch.Ready()
		switch p {
		case proto.ProtoFinish:
			if Debug {
				log.Debug("key: %s wakeup exit dispatch goroutine", key)
			}
			goto failed
		case proto.ProtoReady:
			for {
				if p, err = ch.CliProto.Get(); err != nil {
					err = nil // must be empty error
					break
				}
				if err = p.WriteWebsocket(conn); err != nil {
					goto failed
				}
				p.Body = nil // avoid memory leak
				ch.CliProto.GetAdv()
			}
		default:
			// TODO room-push support
			// just forward the message
			if err = p.WriteWebsocket(conn); err != nil {
				goto failed
			}
		}
	}
failed:
	if err != nil {
		log.Error("key: %s dispatch websocket error(%v)", key, err)
	}
	conn.Close()
	// must ensure all channel message discard, for reader won't blocking Signal
	for {
		if p == proto.ProtoFinish {
			break
		}
		p = ch.Ready()
	}
	if Debug {
		log.Debug("key: %s dispatch goroutine exit", key)
	}
	return
}
上面的注释说的很明白，然后整个websocket就完成了
最后连接失败后，做一些收尾的工作

tcp也差不多这个逻辑。至此整个commet的流程已经完成








现在来看下这个包装的Channel

type Channel struct {
	RoomId   int32
	CliProto Ring
	signal   chan *proto.Proto
	Writer   bufio.Writer
	Reader   bufio.Reader
	Next     *Channel
	Prev     *Channel
}

结构 双向链表
 RoomId 房间Id 标示所属房间
 Next 后指针
 Prev 前指针
 Writer 写阅读器 受包装过
 Reader 度阅读器 受包装过
 signal 信号，也就是实际的消息组合 类型是proto包内的Proto
 CliProto Ring类型

 这个ring是什么类型的呢
 type Ring struct {
	// read
	rp   uint64
	num  uint64
	mask uint64
	// TODO split cacheline, many cpu cache line size is 64
	// pad [40]byte
	// write
	wp   uint64
	data []proto.Proto
}

rp 读
num 数量
mask 掩码
wp 写
data 实际数据



------
whiteList 这结构是代表白名单的东西 拿来调试用的

























    