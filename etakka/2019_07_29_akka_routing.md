先从akka的routing的概念，再到用法入手，再深入了解其实现细节
#### akka中routing的概念
![image](![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_29_akka_routing/akka_routing01.png?raw=true))
其中：

- router:路由器，也就是消息分发器，成员变量包括路由算法routeLogic和routee处理消息的节点
- routeLogic：路由算法，如轮询（RoundRobin）,随机（RandomRouting）,一致性hash(ConsistentHashingRouting)等
- routees:最终处理消息的节点worker

配置发布router的两种方式：
- pool:只告诉pool Worker的actor类型，由pool来创建和管理具体的actor，有resize方法动态改变pool的大小
- group:只传actor的path,通过ActorSelection的方法给actor发消息

#### 用法举例
### 通过代码的方式发布router
```
var props = Props.Create<Worker>().WithRouter(new RoundRobinPool(5));
var actor = system.ActorOf(props, "worker");
```

### 通过配置的方式发布router
#### Pool方式
告诉router使用Worker来创建5个routee,由router管理，是router的子actor
```
akka.actor.deployment {
  /workers {
    router = round-robin-pool
    nr-of-instances = 5
  }
}
var props = Props.Create<Worker>().WithRouter(FromConfig.Instance);
var actor = system.ActorOf(props, "workers");
```

#### Group方式
告诉router routee的路径/user/workers/w1等，router通过actorselection的方式与router通信，不监控routee
```
akka.actor.deployment {
  /workers {
    router = round-robin-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
var props = Props.Create<Worker>().WithRouter(FromConfig.Instance);
var actor = system.ActorOf(props, "workers");
```


#### 提供的几个路由算法
- RoundRobinRoutingLogic:轮询，按顺序轮流发给routee
- RandomRoutingLogic:随机顺序轮流发给routee
- SmallestMailboxRoutingLogic:策略是选择最少处理消息的routee
![image](![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_29_akka_routing/akka_routing02.png?raw=true))
- BroadcastRoutingLogic
- ScatterGatherFirstCompletedRoutingLogic
- TailChoppingRoutingLogic
- ConsistentHashingRoutingLogic

#### 实现细节（未完）
Router
    logic
    routees=[]
    Pool
    Group
Routee

ResizablePoolActor(RouterPoolActor)
ResizablePoolCell(RoutedActorCell)

RoutingLogic
	RoundRobinRoutingLogic
	RandomRoutingLogic
	SmallestMailboxRoutingLogic
	BroadcastRoutingLogic
	ScatterGatherFirstCompletedRoutingLogic
	TailChoppingRoutingLogic
	ConsistentHashingRoutingLogic