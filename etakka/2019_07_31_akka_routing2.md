---
title: '2019_07_31_akka_routing2'
date: 2019-07-31 9:40:33
---
# Akka学习routing篇

分别从router的创建和发消息深入了解其实现

其实要实现一个路由功能，就是路由，路由策略，路由节点三个部分组成。akka也不例外，整理了下类关系图，看看它的router结构

#### Router类关系
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_31_akka_routing/akka_routing201.png?raw=true)

- RouterActorCell:路由actor,包括配置属性（通过判断配置决定创建方式是pool还是group）,维护Routee(路由节点)和RoutingLogic（路由策略），响应GetRoutees,AddRoutee,RemoveRoutee,Terminated四个系统消息，其中Terminated消息由加入时给actor发watch消息，actor停止时会通知回Terminated（如果不是系统消息，则根据路由策略向路由节点转发消息）
- RouterActor:是RouterActorCell的actorBase,用来响应消息
- RouterPoolActor:继承RouterActor,响应AdjustPoolSize，调整actor池大小
- ResizeablePoolActor:继承RouterActor，响应Resize消息，调整actor池大小
- Router:RouterActorCell的成员变量，包含路由actor和路由策略
- RoutingLogic:路由策略，具体的路由策略通过实现虚方法select

结合类关系和创建例子，一个router创建过程就是RouterActorCell初始过程
1. 根据RouterConfig配置的pool/group方法创建不同的actorbase(RouterActor or RouterPoolActor)，负责响应不同的消息方法和转发路由消息
2. 根据RouterConfig配置的routingLogic创建Router，如果是pool方式，由router创建子actor,并管理，把actorref记录在routee上;如果是group方法，传入actor path,记在routees上，由ActorSelection的方式向actor发消息

#### Pool Vs Group
RouterActorCell启动方法中会根据routerconfig的配置方式加入routees
```
if (RouterConfig is Pool pool)
{
    var nrOfRoutees = pool.GetNrOfInstances(System);
    if (nrOfRoutees > 0)
        AddRoutees(Vector.Fill<Routee>(nrOfRoutees)(() pool.NewRoutee(RouteeProps, this)));
}
else if (RouterConfig is Group group)
{
    // must not use group.paths(system) for old (nre-compiled) custom routers
    // for binary backwards compatibility reasons
    var deprecatedPaths = group.Path
    var paths = deprecatedPaths == null
            ? group.GetPaths(System)?.ToArray()
            : deprecatedPaths.ToArray(
    if (paths.NonEmpty())
        AddRoutees(paths.Select(p => group.RouteeFor(this)).ToList());
}
```
- pool方式：创建子actor

```
internal virtual Routee NewRoutee(Props routeeProps, IActorContext context)
{
    return neActorRefRoutee(context.ActorOf(EnrichWithPoolDispatcher(routProps, context)));
}
```
- group方式：通过actorpath引用actor

```
internal Routee RouteeFor(string path, IActorContext context)
{
    return new ActorSelectionRoutee(context.ActorSelection(path));
}
```

#### 发消息
给一个router发消息
```
public override void SendMessage(Envelope envelope)
{
    if (RouterConfig.IsManagementMessage(envelope.Message))
        base.SendMessage(envelope);
    else
        Router.Route(envelope.Message, envelope.Sender);
}
```
如果是系统消息，直接自已处理。系统消息指的是GetRoutees,AddRoutee,RemoveRoutee,Terminated这几个消息，另外如果是pool，还要处理resize，AdjustPoolSize消息

AdjustPoolSize消息由用户主动发起调用，比如：

```
actor.Tell(new AdjustPoolSize(4));
actor.Tell(new AdjustPoolSize(-2));
```

Resize消息由ResizePoolCell自己发起，根据消息数量动态调整
        
```
public override void SendMessage(Envelope envelope)
{
    if(!(RouterConfig.IsManagementMessage(envelope.Message)) &&
        resizer.IsTimeForResize(_resizeCounter.GetAndIncrement()&&
        _resizeInProgress.CompareAndSet(false, true))
    {
        base.SendMessage(new Envelope(new Resize(), SelfSystem));
    
    base.SendMessage(envelope);
}
```
其中由于SendMessage可能被多个线程执行，_resizeCounter，_resizeInProgress需要通过c#的Interlock实现线程同步

#### RoutingLogic
akka的router配置一共支持以下策略
```
router.type-mapping {
    from-code = "Akka.Routing.NoRouter"
    round-robin-pool = "Akka.Routing.RoundRobinPool"
    round-robin-group = "Akka.Routing.RoundRobinGroup"
    random-pool = "Akka.Routing.RandomPool"
    random-group = "Akka.Routing.RandomGroup"
    smallest-mailbox-pool = "Akka.Routing.SmallestMailboxPool"
    broadcast-pool = "Akka.Routing.BroadcastPool"
    broadcast-group = "Akka.Routing.BroadcastGroup"
    scatter-gather-pool="Akka.Routing.ScatterGatherFirstCompletedPool"
    scatter-gather-group="Akka.Routing.ScatterGatherFirstCompletedGroup"
    consistent-hashing-pool = "Akka.Routing.ConsistentHashingPool"
    consistent-hashing-group ="Akka.Routing.ConsistentHashingGroup"
    tail-chopping-pool = "Akka.Routing.TailChoppingPool"
    tail-chopping-group = "Akka.Routing.TailChoppingGroup"
}
```
其中注意到，smallest-mailbox没有group方式。看看它的策略选择方法select就知道原因

![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_31_akka_routing/akka_routing202.png?raw=true)

```
private Routee SelectNext(Routee[] routees)
{
    var winningScore = long.MaxValue
    // round robin fallback
    var winner = routees[(Interlocked.Increment(ref _next) int.MaxValue) %  routees.Length]
    for (int i = 0; i < routees.Length; i++)
    {
        var routee = routees[i];
        var cell = TryGetActorCell(routee);
        if (cell != null)
        {
            // routee can be reasoned about it's mailbox size
            var score = cell.NumberOfMessages;
            if (score == 0)
            {
                // no messages => instant win    
                return routee;
            
            if (winningScore > score)
            {
                winningScore = score;
                winner = routee;
            }
        }
    
    return winner;
}
```
通过cell的NumberOfMessages选出消息队列最小的那个routee来处理消息，而NumberOfMessages只会local actor返回真实的当前消息数，remote actor只会返回0

其它策略都分别有pool和group的方式：
- round-robin：轮询，每次加1取模
```
int index = (Interlocked.Increment(ref _next) & int.MaxValue)%size;
```
- random：随机选一个routee
```
routees[ThreadLocalRandom.Current.Next(routees.Length)]
```
- broadcast：每个routee都转发消息

```
new SeveralRoutees(routees);
```
- scatter-gather: 全部routee转发消息，只处理第一个返回的消息，其它返回忽略。使用场景是不关心哪个routee返回消息。
```
var tasks = _routees.Select(routee => routee.Ask(message, _within)).ToList();
var firstFinishedTask = await Task.WhenAny(tasks);
return await firstFinishedTask;
```
- tail-chopping:随机找一个routee转发消息，如果一定时间没响应，重新找一个发

```
_scheduler.Advanced.ScheduleRepeatedly(TimeSpan.Zero, _interval, async () =>
{
    var currentIndex = routeeIndex.GetAndIncrement();
    if (currentIndex >= _routees.Length) 
        retu
    try

        completion.TrySetResult(aw(_routees[currentIndex].Ask(messa_within)).ConfigureAwait(false));
    }
    catch (TaskCanceledException)
    {
        completion.TrySetResult(
            new Status.Failure(
                new AskTimeoutException($"Ask timed on {sender} after {_within}")));
    }
}, cancelable);
```
- consistent-hashing:一致性hash，根据message的hashkey选择routee,让同样hashkey的消息发到同一个routee,比如id可以固定分配到同一个routee。另外，每次选择target时，UpdateConsistentHash根据当前routees更新，当前routees由watch消息监控了节点状态。
```
var currentConsistentHash = UpdateConsistentHash();
if (currentConsistentHash.IsEmpty) reRoutee.NoRoutee;
else
{
    switch (hashData)
    {
        case byte[] bytes:
            recurrentConsistentHash.NodeFor(bytestee;
        case string data:
            recurrentConsistentHash.NodeFor(data)ee;
        default:
            recurrentConsistentHash.NodeFor(_systrialization.FindSerializerFor(hashDToBinary(hashData)).Routee;
    }
}
```
