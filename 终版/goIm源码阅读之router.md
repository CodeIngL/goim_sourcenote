## goIm源码阅读之router组件

### 小小说明 ### 
*goIm*是*毛剑大神*的作品，是关于*推送*的一个分布式项目，值得相关从事IM的同学好好学习下。  
想我这种弱鸡，还没毕业的菜鸟更是要仔细的学习下这东西。  

router组件结构如下:  
  
	`---router
		`---bucket.go
		`---cleaner.go
		`---cleaner_test.go
		`---config.go
		`---main.go
		`---monitor.go
		`---router-example.conf
		`---router-log.xml
		`---rpc.go
		`---session.go
		`---signal.go  

---
### 随便look  

先任意选取一个文件进行分析下，有go基础的童鞋可以直接跳过.  
以bucket.go为例       

		import (
			"goim/libs/define"
			"sync"
			"time"
		)

其中后面两个是系统包，可以直接略过。特别注意的是*goim/libs/define*  
这是goIm要用到的一些结构，常量，通用定义的包，一看到有define，应该就明白了  
它的目录是结构  

	`---define
		`---kafka.go
		`---operation.go
		`---room.go  
里面都是一些常量。

	kafka.go中
		KAFKA_MESSAGE_MULTI          = "multiple"       //multi-userid push 用户推送
		KAFKA_MESSAGE_BROADCAST      = "broadcast"      //broadcast push    广播推送
		KAFKA_MESSAGE_BROADCAST_ROOM = "broadcast_room" //broadcast room push 房间广播

至于有什么差异，后文会讲到。

	operation.go

		// handshake
		OP_HANDSHAKE       = int32(0)
		OP_HANDSHAKE_REPLY = int32(1)
		握手状态

		// heartbeat
		OP_HEARTBEAT       = int32(2)
		OP_HEARTBEAT_REPLY = int32(3)
		心跳状态

		// send text messgae
		OP_SEND_SMS       = int32(4)
		OP_SEND_SMS_REPLY = int32(5)
		消息状态

		// kick user
		OP_DISCONNECT_REPLY = int32(6) 
		取消连接

		// auth user
		OP_AUTH       = int32(7)
		OP_AUTH_REPLY = int32(8)
		认证状态

		// handshake with sid
		OP_HANDSHAKE_SID       = int32(9)
		OP_HANDSHAKE_SID_REPLY = int32(10)    
		带session的握手

		// raw message
		OP_RAW = int32(11)
		行消息

		// room
		OP_ROOM_READY = int32(12)
		房间响应

		// proto
		OP_PROTO_READY  = int32(13)
		OP_PROTO_FINISH = int32(14)
		协议状态

		// for test
		OP_TEST       = int32(254)
		OP_TEST_REPLY = int32(255)
		测试状态

这些都是一些操作的定义

	room.go中
    	NoRoom = -1 //没有房间

以上就是整个define包里的东西

---


### 正式开刀 ###

看完了上面的小菜，下面进入程序的正式分析。

首先从main.go开始，这是肯定的

	func main() {

		flag.Parse()	//解析命令行中的参数.....干嘛来着，还真的不知道，不过不影响  

		InitConfig() 	//初始化配置

		runtime.GOMAXPROCS(Conf.MaxProc)
		log.LoadConfiguration(Conf.Log)
		defer log.Close()
		log.Info("router[%s] start", VERSION)
		// start prof
		perf.Init(Conf.PprofAddrs)
		// start monitor
		if Conf.MonitorOpen {
			InitMonitor(Conf.MonitorAddrs)
		}
		// start rpc
		buckets := make([]*Bucket, Conf.Bucket)
		for i := 0; i < Conf.Bucket; i++ {
			buckets[i] = NewBucket(Conf.Session, Conf.Server, Conf.Cleaner)
		}
		if err := InitRPC(buckets); err != nil {
			panic(err)
		}
		// block until a signal is received.
		InitSignal()
	}
代码量很短。简洁明了，我们看一下逻辑。
 
我们关注下

	InitConfig()

干甚呢,贴**伪源码**。

	// InitConfig init the global config.
	func InitConfig() (err error) {
		Conf = NewConfig()
		gconf = goconf.New()
		gconf.Parse(confFile)
		gconf.Unmarshal(Conf)
	}

毛剑大大做了注释，相当的明显啊。初始化一个全局的配置对象。  
**不过我们需要详细的介绍下，因为这些是通用的操作**  
重点是:
>**每个组件的流程都是这样类似的**  

##### 先看 

	Conf = NewConfig()  

Conf是一个全局的配置对象。以下是它的结构：  

	type Config struct {
		// base section
		PidFile    string   `goconf:"base:pidfile"`		//linux下防止进程多个的冲突
		Dir        string   `goconf:"base:dir"`			//项目路径
		Log        string   `goconf:"base:log"`			//日志的配置文件
		MaxProc    int      `goconf:"base:maxproc"`		//最大核心数，记得从某个go版本开始。默认是当前电脑的核心数了
		PprofAddrs []string `goconf:"base:pprof.addrs:,"` //代码性能监控的url
		// rpc
		RPCAddrs []string `goconf:"rpc:addrs:,"`		//goRPC的url
		// bucket
		Bucket            int           `goconf:"bucket:bucket"`	//桶？？干嘛用，默认为核心数
		Server            int           `goconf:"bucket:server"`	//服务？？是一个数字看起来是描述一个东西的数量，未知
		Cleaner           int           `goconf:"bucket:cleaner"`	
		BucketCleanPeriod time.Duration `goconf:"bucket:clean.period:time"`	//桶清除时间？？ 看起来和上面的桶和Cleaner有极大的关系
		// session
		Session       int           `goconf:"session:session"`		//会话 但是不知道是来标识什么特征的
		SessionExpire time.Duration `goconf:"session:expire:time"`	//回话过期时间，一小时
		// monitor
		MonitorOpen  bool     `goconf:"monitor:open"`				//是否开启
		MonitorAddrs []string `goconf:"monitor:addrs:,"`			//监控地址。
	}

注释中有疑问的下面会说明 。  


重新看一下conf对象的初始化干了什么事情  

	return &Config{
			// base section
			PidFile:    "/tmp/goim-router.pid",
			Dir:        "./",
			Log:        "./router-log.xml",
			MaxProc:    runtime.NumCPU(),
			PprofAddrs: []string{"localhost:6971"},
			// rpc
			RPCAddrs: []string{"localhost:9090"},
			// bucket
			Bucket:            runtime.NumCPU(),
			Server:            5,
			Cleaner:           1000,
			BucketCleanPeriod: time.Hour * 1,
			// session
			Session:       1000,
			SessionExpire: time.Hour * 1,
		}
这代表一些默认的配置(如果配置文件中不存在的话)

---

##### 接着

	gconf = goconf.New()

这个是干嘛的呢，它是用来封装配置文件router-example里面的一组组配置的（简单的来说就是对应一个配置文件对象），
包括相应的注释，键值对之类的,我们可以看一下这个的结构  

	// Config is the key-value configuration object.
	type Config struct {
		data      map[string]*Section  //配置是按组的。每组多个键值对使用Section来包装的，key是该组的名字
		dataOrder []string			   //以每组配置的顺序，将每组的名字组成的一个数组
		file      string			   //配置文件路径
		Comment   string			   //配置的注释符号
		Spliter   string			   //分割符号
	}

这样好像不是很明了的样子,我来举个配置的例子，来自router-example.conf文件  

	# rpc listen and service
	[rpc]

	# The rpc server network@ip:port bind.
	#
	# bind localhost:8092
	addrs tcp@localhost:7270

[rpc]上面使用以rpc这个Section(这个组的注释),addrs上面的是该键值对的注释说明  

项目的默认是

	func New() *Config {
		//注释符是# ;分割符是' '
		return &Config{Comment: Comment, Spliter: Spliter, data: map[string]*Section{}} 
	} 

这里我们需要特别注意**Section**这个结构，这是键值对(map)的封装,具体来看一下  

    // Section is the key-value data object.
    type Section struct {
        data         map[string]string // key:value  
        dataOrder    []string
        dataComments map[string][]string // key:comments
        Name         string
        comments     []string
        Comment      string
    }
	data			就是我们所说的map键值对
	dataOrder 		根据配置的顺序加进来的key
	dataComments 	对于每个键值对的说明
	Name			名字
	comments		整个的注释
	Comment			注释符号

同样是配置文件来对应下  

		[monitor]
		# monitor listen
		open true
		addrs 0.0.0.0:7374
这个配置对应的**Section**是这样的  
**data:(open,true),(addrs,0.0.0.0:7374)**   
**dataOrder:open,addrs**  
**dataComments:(open,[]String{"monitor listen"}),(addrs,[]string{})**    
**name:monitor**    
**comments:[]string{}**    
**Comment:#**   


---
##### 继续

    gconf.Parse(confFile)

解析confFile配置文件，这东西哪里来的，就在上面的一个变量。  
这个confFile是在init函数中，这个init函数，只要学go都应该知道的。
加载的时候回执行 init()函数。这个的值是  

	flag.StringVar(&confFile, "c", "./router-example.conf", " set router config file path")

这个是go自带包的东西没什么好说的。继续看正文

	func (c *Config) Parse(file string) error {
		os.Open(file)
		c.file = file
		c.ParseReader(f)
		//呈现的是伪代码
	}

一眼望去就是很简单的委托给了**ParseReader**来工作
这个函数做了什么呢

---

我们继续看这个函数有点长，功能就是上面提到了对**gconf**对象的填充
简单的流程是这样：

>	使用行读取函数,出错活着文件结束，结束读取。  
	否则读取该行，并去掉两边空格。  
	空行和#开头的追加的comments中  

>	碰到[表明是一个section,没有]结尾会抛出一个错误来,指明行号  
	然后得的这个Section的名字，就在[]中间的符合  
	TIP：
		sectionStr := row[1 : len(row)-1]
		然后 sectionStr = strings.TrimSpace(sectionStr)
	这样我觉得会更好，不过毛大应该有自己的考虑，因为下面都是去了两边的空格
 
 这里再次说明的是为什么毛大采用的**map[String]*Section**呢
 因为一个[xxx]下面会有很多键值对，的所以封装成一个**Section**会更好。

			s, ok := c.data[sectionStr]
			if !ok {
				s = &Section{data: map[string]string{}, dataComments: map[string][]string{}, comments: comments, Comment: c.Comment, Name: sectionStr}
				c.data[sectionStr] = s
				c.dataOrder = append(c.dataOrder, sectionStr)
			} else {
				return errors.New(fmt.Sprintf("section: %s already exists at %d", sectionStr, line))
			}
			section = s
			comments = []string{}
			continue

这里很重要，能更好理解细节，

> 不允许使用同样的[xxx]的节点，如果不存在节点，就新生成一个  
初始化一个空的data，一个描述data的注释，然后是针对这个[xxx]的注释，然后是分割符，名字是节点的名字  
接着按顺序添加dataOrder进节点的名字  
然后赋值section   
清空注释  
继续 


>接下来一行行读取有用的键值对，没用分割符（空格）报错并提示行号  
然后对key和value删除两边空格赋值，处理注释  
赋值进section  

最后成功解析返回(Parse(confFile)过程完成)

---

##### 最后

	gconf.Unmarshal(Conf)

这个的作用是,由配置文件组装的**gconf**对象，去对**Conf**对象进行相关的设置，以及覆盖。


源码有点长，这里就不贴了，请需要的童鞋对着源码来看本文。  
这里说的比较浅显，比喻不当但是应该不会影响一个懂go语法的人理解
重点就是利用了放射，根据tag来设置

>利用反射包，得到了vv 也就是Config的包装对象(value)  
然后判断下vv.kind进行简单检查

>使用反射包得到字段rv,类型rt,字段数量n

>然后进行循环得到 确切的值vf，类型tf,以及tag   
如果tag中等于- 或者 "" 或者是omitempty，直接略过
	
>对tag切成三个使用分隔符':'   
如果切割后后长度小于2 则抛出错误.因为tag不符合我们的规范

>然后tagArr0,付给section，tagarr1付给key  
查询下section是否存在，不存在直接略过

>查询下section中的key是否存储，没有直接略过。  
接下来就是放射的处理了  
根据实际的类型，进行相关的设置  
对于特殊的"memory"和"time"就特殊的处理下  
总体还是比较简单的  
最后返回

这时候Conf的最终设置完成(router组件main.go中InitConfig()执行完成)

----


重新跳回main.go接下来根据配置对象，设置相应的配置，
包括：  
	1. **核心数**,     
	2. **日志配置**,   
	3. **prof监控**,  
	4.**monitor监控(监控http是否是正常)**  

最后就是开启rpc  **goIm各个组件之间的通信采用的方式是标准库自带的rpc方式** 性能应该还可以。  

>rpc初始化相关代码，需要重点关注下。  

	buckets := make([]*Bucket, Conf.Bucket)
	for i := 0; i < Conf.Bucket; i++ {
		buckets[i] = NewBucket(Conf.Session, Conf.Server, Conf.Cleaner)
	}

**这里出现了上文存在疑问的桶，只是上文的是数量(int)，这个东西比较难翻译，我只能很差的取个名字了：桶  
桶的结构是这样。**

	type Bucket struct {
		bLock             sync.RWMutex
		server            int                       // session server map init num
		session           int                       // bucket session init num
		sessions          map[int64]*Session        // userid->sessions
		roomCounter       map[int32]int32           // roomid->count
		serverCounter     map[int32]int32           // server->count
		userServerCounter map[int32]map[int64]int32 // serverid->userid count
		cleaner           *Cleaner                  // bucket map cleaner
	}
>**桶的作用，就是防止超大的map出现，拆分成一个小的map，什么之类的都好。
由于golang的go并发性，锁一个小map的好处比锁这个超大map肯定是好多了的。**  

默认的配置数量是16,16,16


接下来正式初始化rpc  

	func InitRPC(bs []*Bucket) (err error)

实际的工作就是:初始化一个routerRpc的结构然后向rpc注册  
然后RouterRPC的所有符合规范的接口都被导出，注册到RPC的服务上。

**至此，router模块完成所有的构造**

---------------------------------------

#### 更多补充  

#### 疑问探究 ####

>值得说明的是从头到尾没有说明**router组件**的作用，  
实际上首先从名字上就能模糊的推出这是一个路由的功能。  
但模糊推出是不符合我们程序员严谨的精神，所以我们需要具体的了解这是干什么来的。

>我们可以观察其rpc接口了，如上面所说goIm组件是通过rpc的方式来通信的。  
因此我们可以就此判断router的具体作用。  
同时，值得注意的是该组件里面，没有调用其他的rpc组件。因而更加容易分析。

#### 答案呈现 ####

现在我们进入router组件，rpc.go来看一下  

rpc.go

	ping接口，什么都没搞，目的就是验证远程调用是否有效
		func (r *RouterRPC) Ping(arg *proto.NoArg, reply *proto.NoReply) error {
			return nil
		}
	put接口，根据请求的参数（userId，server，RoomId）。返回一个seq给调用方，
		func (r *RouterRPC) Put(arg *proto.PutArg, reply *proto.PutReply) error {
			reply.Seq = r.bucket(arg.UserId).Put(arg.UserId, arg.Server, arg.RoomId)
			return nil
		}
		这里会根据userId映射到桶里面，找到相应的桶，然后将信息放置进去
	del接口，根据请求的参数（userId，seq，RoomId）。返回一个has给调用方，
		func (r *RouterRPC) Del(arg *proto.DelArg, reply *proto.DelReply) error {
			reply.Has = r.bucket(arg.UserId).Del(arg.UserId, arg.Seq, arg.RoomId)
			return nil
		}
	xxxx接口，若干

到这里还是不知道router组件干了什么的话，请看其他组件的源码解析。这样会形成一个整体的架构
下面的说明是基于浏览过其他组件的源码。  

---

>我们选取put接口分析。  
首先userId是指用户Id，server是指commet组件的标示ID，roomId是指房间Id，上面的接口通常会先定位一个桶。
里面的函数是这样写的  

		func (r *RouterRPC) bucket(userId int64) *Bucket {
			idx := int(userId % r.BucketIdx)
			// fix panic
			if idx < 0 {
				idx = 0
			}
			return r.Buckets[idx]
		}
>直接使用取余定位桶，毛大在上面写了一个    
>//fix panic  
感觉调用会出现userId可能是负数。无法想明白可能是网络传输的问题？？userId是负数？？  
如果频率高的话，可能r.Bucket[0]与其他可能不平衡  

接着调用桶的put操作

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
>首先是查找userId的session是否已经存在，不存在就创建一个session，这个session是这样的

	type Session struct {
		seq     int32
		servers map[int32]int32           // seq:server
		rooms   map[int32]map[int32]int32 // roomid:seq:server with specified room id
	}

	// NewSession new a session struct. store the seq and serverid.
	func NewSession(server int) *Session {
		s := new(Session)
		s.servers = make(map[int32]int32, server)
		s.rooms = make(map[int32]map[int32]int32)
		s.seq = 0
		return s
	}

>session是这样的，  
一个增序的seq   
一个servers seq和server(commet组件ID)的映射   
一个rooms  roomId和(seq:server)的映射   


>返回Put函数来看，如果根据userId对应的session。设置相应的信息。  
如果存在房间编码。

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
首先还是放入server

	func (s *Session) nextSeq() int32 {
		s.seq++
		return s.seq
	}

	// Put put a session according with sub key.
	func (s *Session) Put(server int32) (seq int32) {
		seq = s.nextSeq()
		s.servers[seq] = server
		return
	}
seq先自增,然后放置seq和server(comet标示Id)  

然后放入房间，根据roomId放入，然后roomId的seq在放入server(cometId)返回  

### TIP：
>**整体上对于一个用户来说，在整个分布式上，就有这样的逻辑，  
根据一个userId，进行一致性hash，就能找到相应的router组件（router组件是多个的），  
定位到某个router组件后，根据userId，进行取余hash找到对应的桶   
定位到确定的桶后，还是依据userId,就能找到相应session。  
然后session里面保存了这个用户的自身很多信息。**   

这个session的结构如下  

	type Session struct {
		seq     int32					  // 最新的序列
		servers map[int32]int32           // 最新seq:server(comet组件的Id)
		rooms   map[int32]map[int32]int32 // 最新roomid:seq:server with specified room id
	}

**对于servers和rooms对应的最大key。可能会不一致，当且仅单put操作时，roomId!=NoRoom时一致**

>然后桶还要统计下，因为是put操作，需要更新统计数量

	// counter incr or decr counter.
	func (b *Bucket) counter(userId int64, server int32, roomId int32, incr bool) {
		//。。。省略部分
		if sm, ok = b.userServerCounter[server]; !ok {
			sm = make(map[int64]int32, b.session)
			b.userServerCounter[server] = sm
		}
		if incr {
			sm[userId]++
			b.roomCounter[roomId]++
			b.serverCounter[server]++
		} else {
			if v, _ = sm[userId]; v-1 == 0 {
				delete(sm, userId)
			} else {
				sm[userId] = v - 1
			}
			b.roomCounter[roomId]--
			b.serverCounter[server]--
		}
	}

>简单的介绍下改函数。根据server(commentId)找到或创建一个map，大小session值  
map的userId对应数量加1，  
roomId加1，  
server加1，   
若是decr则是减1  

最后rpc结束返回一个Seq给调用方。  

### 结尾 ####

从上面分析来，router只保存相关信息，可以认为就是一个内存数据库。  
由于这个hash是ketama算法，得失还是有的。
























    


