---
title: 'Actor的DeathWatch'
date: 2019-11_28 9:40:33
---
  
应用场景：拿到一个actor的ActorRefk, 想在actor terminated时收到响应消息，调Context.Watch(actor)

#### 添加Watch
Context.Watch(actor)的实现逻辑，
```
public IActorRef Watch(IActorRef subject)
{
    var a = (IInternalActorRef)subject
    if (!a.Equals(Self) && !WatchingContains(a))
    {
        MaintainAddressTerminatedSubscription(() =>
        {
            a.SendSystemMessage(new Watch(a, _self)); // ➡➡➡ NEVER SENTHE SAME SYSTEM MESSAGE OBJECT TO TWO ACTORS
            _state = _state.AddWatching(a, null);
        }, a);
    }
    return a;
}
```
1. 不能watch自己
2. 已经watched不处理
3. MaintainAddressTerminatedSubscription的作用是，当前watch的所有actor中，是否有远程actor，如果有则订阅远程地址termiate消息，在remote中有远程地terminate时能收到响应消息，从而处理远程actor的监控。
4. 给远程actor发Watch消息，远程actor收到Watch消息时，记录到_state.watchedBy列表
5. 添加watching列表

```
private void MaintainAddressTerminatedSubscription(Action block, IActorRef change = null)
{
    if (IsNonLocal(change))
    {
        var had = HasNonLocalAddress();
        block();
        var has = HasNonLocalAddress()
        if (had && !has)
            UnsubscribeAddressTerminated();
        else if (!had && has)
            SubscribeAddressTerminated();
    }
    else
    {
        block();
    }
}
```

#### 触发Watch
actor的stop会最终触发FinishTerminated
```
protected void TellWatchersWeDied()
{
    var watchedBy = _state
        .GetWatchedBy()
        .ToList()
    if (!watchedBy.Any()) return;
    try
    {
        foreach (var w in watchedBy) SendTerminated(false(IInternalActorRef)w);
        foreach (var w in watchedBy) SendTerminated(true, (IInternalActorRef)w);
    }
    finally
    {
        _state = _state.ClearWatching();
    }
}
```
先通知远程watchBy的actors，再通知本地watchBy的actors，如，先用RemoteDaemon发了消息再RemoteDaemon发它监控的terminate