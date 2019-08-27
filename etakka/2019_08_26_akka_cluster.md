---
title: '2019_08_26_akka_cluster'
date: 2019-08_26 9:40:33
---
# Akka学习cluster篇

开始学习akka的cluster模块，还是先从example入手，看cluster提供了什么功能。再分析cluster实现中的一些概念，最后从源码层面研究实现的细节。

### cluster提供了什么功能
1.特点。
- 容错性：通过定时检测消息重启进程
- 去中心化：p2p协议gossip实现，没有gamemanager这样的单点问题
- 可伸缩：可动态加入和退出节点，加入和退出事件会同步给集群上的所有节点，广播带了向量时钟的gossip协议，实现成员副本数据的最终一致性）
- 无单点故障：多节点服务，leader节点挂了，其余节点中的一个会成为leader

2.集群订阅消息
集群中每个节点会有这5个状态joining,up,leaving,down,removed。
通过subscribe方法订阅节点状态的消息,比如actor启动时监听
```
protected override void PreStart()
{
    Cluster.Subscribe(Self, ClusterEvent.InitialStateAsEvents, new []{ typeof(ClusterEvent.IMemberEvent), typeof(ClusterEvent.UnreachableMember) });
}
```
当成员变化会收到对应消息,添加成员，删除成员等

```
protected override void OnReceive(object message)
{
    var up = message as ClusterEvent.MemberUp;
    if (up != null)
    {
        var mem = up;
        Log.Info("Member is Up: {0}", mem.Member);
    } else if(message is ClusterEvent.UnreachableMember)
    {
        var unreachable = (ClusterEvent.UnreachableMember) message;
        Log.Info("Member detected as unreachable: {0}"unreachable.Member);
    }
    else if (message is ClusterEvent.MemberRemoved)
    {
        var removed = (ClusterEvent.MemberRemoved) message;
        Log.Info("Member is Removed: {0}", removed.Member);
    }
    else if (message is ClusterEvent.IMemberEvent)
    {
        //IGNORE                
    }
    else if (message is ClusterEvent.CurrentClusterState)
    {
        
    }
    else
    {
        Unhandled(message);
    }
}
```



3.路由与负载均衡。路由和负载均衡就是结合routing模块和成员状态的订阅，在一个actor中管理节点成员，通过路由算法分发消息。对比bigworld,类似stub下创建entity,创建成功后注册到stub,由stub负责路由工作。
伪代码
```
protected override void OnReceive(object message)
{
    case ClusterEvent.MemberUp:
        router.add(member)
    case ClusterEvent.MemberRemoved:
        router.remove(member)
    case Message:
        router.route(message)
}
```

4.Cluster Singleton集群中的单例。也就是bigworld中的stub

### 从example入手

### 几个概念
- vectorclock 向量时钟
- gossip 流言协议
- memberstate 成员状态
- SeedNode 种子节点

### 实现细节