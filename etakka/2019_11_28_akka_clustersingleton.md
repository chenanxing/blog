---
title: 'Actor的ClusterSingleton'
date: 2019-11_28 19:40:33
---

akka集群中的单例怎么实现
#### 对比mb的stub实现
mobile server的stub是启动时通过game1远程创建entity实现。所有game进程创建好之后，由game1触发create_all_stub, 
伪代码
```
def on_all_game_ready(self):
    if GameServerRepo.game_server.sid == self.primary_game:
		self._init_stubs()

def _init_stubs(self):
    AllStubEntity = XXX
    for entityname in AllStubEntity:
        creat_entity_anywhere(entityname)
```


```
class ServerStubEntity()
    def __init__:
        on_ready()
        
    def on_ready():
        register_to_all()
```
stubentity创建完成时注册自己给所有进程，有集群成员加入时同时同步这些global entity

#### ClusterSignleton 使用
用法上主要是ClusterSingletonManager和ClusterSingletonProxy
```
system.ActorOf(ClusterSingletonManager.Props(
    singletonProps: Props.Create<MySingletonActor>(),
    terminationMessage: PoisonPill.Instance,
    settings: ClusterSingletonManagerSettings.Create(system).WithRole("worker")),
    name: "consumer");
```

```
system.ActorOf(ClusterSingletonProxy.Props(
    singletonManagerPath: "/user/consumer",
    settings: ClusterSingletonProxySettings.Create(system).WithRole("worker")),
    name: "consumerProxy");
```
1. ClusterSingletonManager创建一个实例actor,第一个参数是Actor属性，第二个参数是终止消息，默认是PoisonPill.Instance,第三个参数是role，用来启动这个singleton的节点过滤，即有的进程可以不用启动这个ClusterSingletonManager。
2. ClusterSingletonProxy是singleton的代理，传参路径就可以访问单例actor对象。ClusterSingletonProxy会向单例所在的节点发Identify消息来获得一个远程actor
```
Receive<TryToIdentifySingleton>(_ =>
{
    var oldest = _membersByAge.FirstOrDefault();
    if (oldest != null && _identityTimer != null)
    {
        var singletonAddress = neRootActorPath(oldest.Address)_singletonPath;
        Log.Debug("Trying to identify singleton [{0}]"singletonAddress);
        Context.ActorSelection(singletonAddress).Tell(neIdentifidentityId));
    }
});
```


#### ClusterSignleton实现
ClusterSignleton要处理两个逻辑，一是如何协调哪个节点来创建单例actor,二是当前singleton节点退出集群时，如何有另一个有效节点重建这个单例。

```
else if (e.FsmEvent is OldestChangedBuffer.InitialOldestState)
{
    var initialOldestStat(OldestChangedBuffer.InitialOldestState)e.FsmEvent;
    _oldestChangedReceived = true;
    var isSelfOldes_cluster.SelfAddress.Equals(initialOldestState.Oldes
    if (isSelfOldest && initialOldestState.SafeToBeOldest)
        return GoToOldest();
    else if (isSelfOldest)
        retGoTo(ClusterSingletonState.BecomingOldest).Using(BecomingOldestData(null));
    else
        return GoTo(ClusterSingletonState.Younger).Using(YoungerData(initialOldestState.Oldest));
}
```

```
private State<ClusterSingletonState, IClusterSingletonDataGoToOldest()
{
    Log.Info("Singleton manager [{0}] starting singleton actor"_cluster.SelfAddress);
    var singleton = Context.Watch(Context.ActorOf(_singletonProps_settings.SingletonName));
    return
        GoTo(ClusterSingletonState.Oldest).Using(neOldestData(singleton, false));
}
```

1. 协调Node:ClusterSignletonMangaer是一个状态机，用oldest节点创建singleton actor,GoToOldest就是直正创建singleton的地方Context.ActorOf，其它节点成员younger状态
2. 如果决定Oldest:OldestChangedBuffer. OldestChangedBuffer订阅cluster消息，根据成员变化改变_membersByAge状态。

```
private void TrackChanges(Action block)
{
    var before = _membersByAge.FirstOrDefault();
    block();
    var after = _membersByAge.FirstOrDefault();

    // todo: fix neq comparison
    if (!Equals(before, after))
        _changes = _changes.Enqueue(neOldestChanged(MemberAddressOrDefault(after)));
}
```

3. 转移singleton: 当Oldest状态变化，节点成员oldest后，会请求一条HandOverToMe来交接props,由props重建singleton
```
if (oldestChanged.Oldest.Equals(_cluster.SelfAddress))
{
    if (youngerData.Oldest == null) return GoToOldest();
    else if (_removed.ContainsKey(youngerData.Oldereturn GoToOldest();
    else
    {
Peer(youngerData.Oldest).Tell(HandOverToMe.Ine);
        reGoTo(ClusterSingletonState.BecomingOldest).Usew BecomingOldestData(youngerData.Oldest));
    }
}
```