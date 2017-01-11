### 细节补充

goIm整体上介绍完了，单这水准远远只够使用的，要进一步改造还要了解更多的知识

----

#### 关注点之kafka

首先我们还是要关注的地方是  
define包下的

	package define
	// Kafka message type Commands
	const (
		KAFKA_MESSAGE_MULTI          = "multiple"       //multi-userid push
		KAFKA_MESSAGE_BROADCAST      = "broadcast"      //broadcast push
		KAFKA_MESSAGE_BROADCAST_ROOM = "broadcast_room" //broadcast room push
	)
了解细节的方式之一，就是仔细的将系统对应起来  
这几个是kafka推送的方式的标示，之前我说道，Im系统对外有6个接口，这和这些是怎么对应的呢

>http 接口:

		httpServeMux.HandleFunc("/1/push", Push)
		httpServeMux.HandleFunc("/1/pushs", Pushs)
		httpServeMux.HandleFunc("/1/push/all", PushAll)
		httpServeMux.HandleFunc("/1/push/room", PushRoom)
		httpServeMux.HandleFunc("/1/server/del", DelServer)
		httpServeMux.HandleFunc("/1/count", Count)

我们简单的对应下:   
> Push -----> KAFKA_MESSAGE_MULTI  
> Pushs -----> KAFKA_MESSAGE_MULTI  
> PushAll  -----> KAFKA_MESSAGE_BROADCAST  
> PushRoom -----> KAFKA_MESSAGE_BROADCAST_ROOM  
> DelServer  ------> router组件RPC接口"RouterRPC.DelServer"    
> Count  ------> logic本身存储的信息
---
上面6个接口我们主要关注的重点是kafka**相关的,即推送相关的四个。   
另外其中只有Count接口是get的，其他接口都是post的方式

---

---


#### 关注点之ring.go 
让我们把关注点放到Commet组件的ring.go上，这个我一开始毫无头绪，后来  
仔细看看就知道了。  
>一个环形，消费和生产。  
>默认是num的计算值(num是2的整数倍为num，否则是num的最高位为1的数左移一位)  
>mask，是掩码  
>目的是控制速率呢



-----
### 修改源码自己使用喽。怎么搞

上面的毛大的Im还是比较通用的，那我们如何去改造成自己的源码系统的呢

#### 如何改造

1. 认证方式改造  
>goIm中有一个简陋的认证，这个认证是在Comet组件请求logic组件时的认证。  
>因为涉及到去拿用户的信息，所以这个认证是应该需要的，而这个token，往往是
>保存在客户端的。

关注一下文件auth.go。坐标logic组件内

	// developer could implement "Auth" interface for decide how get userId, or roomId
	type Auther interface {
		Auth(token string) (userId int64, roomId int32)
	}

这个接口 非常明显，毛大在这上面都做了点评，就是我们开发者要自己实现的一个认证方式

**请求和返回json**

```json
{
    "ver": 102,
    "op": 10,
    "seq": 10,
    "body": {"data": "xxx"}
}
```

**请求和返回参数说明**

| 参数名     | 必选  | 类型 | 说明       |
| :-----     | :---  | :--- | :---       |
| ver        | true  | int | 协议版本号 |
| op         | true  | int    | 指令 |
| seq        | true  | int    | 序列号（服务端返回和客户端发送一一对应） |
| body          | true | string | 授权令牌，用于检验获取用户真实用户Id |

