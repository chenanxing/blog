---
title: '2019_11_25_akka_cluster2'
date: 2019-11_25 9:40:33
---

### 从example入手

```
     +-------------+ +------------+ +------------+
     |   backend   | |  backend   | |  backend   |
     +-------------+ +------------+ +------------+
            XX             X              XX
             XX            X             XX
              XX           X           XX
            +------------+ X+--------------+
            |  frontend  |  |  frontend    |
            +------------+  +--------------+
```

exampel功能：搭建一组服务，两个前端服务三个后端服务，frontend收到消息后，负载均衡到backend处理，再返回消息。

对比mobileserver的处理，是这样的，gate server和game server通过gamemanager实现长连接，并通过gamemanager跟gate和game的连接状态，实时同步给其它gate和game，从而获得game的负载状态。服务发现是通过stub初始化后通知gamemanager,gamemanager再广播给所有连接的gate和game，对于单点stub处理不过来的业务，stub再管理多个entity,再作路由下发到entity处理。

akka的cluster实现思路是，开始提供种子节点seedNodes（由启动配置决定），随后的节点启动连接到（？）种子节点，再通过gossip广播到其它节点，其中gossip主要成员变量是vectorlock（向量时钟，简单理解是版本号，用于版本冲突时的处理）和members(集群中的成员)。节点通过订阅集群中的成员消息来感知其它成员的存在。例子中，两个backend为种子节点，其它成员通过接收成员消息得到其它成员地址。

    
### 几个概念
- vectorclock 向量时钟：分布式系统用来检测多副本冲突，简单理解是git版本管理中的新版本，旧版本与版本冲突处理。对比逻辑时钟，逻辑时钟只保存一个时钟来获得分布系统中事件发生的时序，逻辑时钟无法知道其它进程的状态。向量时钟则保存更多的数据（各个副本的版本号），用于版本数据检测。
- gossip 流言协议：理解是定时向其它节点同步自己的gossip副本状态，最终达到数据一致。
- memberstate 成员状态：节点成员状态包括joining,up,leaving,down,removed。节点开始向种子节点申请加入joing,加入成功收到welcome消息同步gossip副本变成up状态，主动退出leaving,移除removed,连不到节点变成down状态。
- SeedNode 种子节点：每个集群都必须先配置种子节点，作为其它节点启动后连接的节点，连接后同步gossip副本。seednodes的第一个节点作为leader节点。

### 实现细节
Cluster在akka的system可以看作单例，

取本地cluster的方法
```
Cluster.Get(system);
```
取远程cluster的方法

```
ActorSelection ClusterCore(Address address)
{
    return Context.ActorSelection(new RootActorPath(address) / "system" / "cluster" / "core" / "daemon");
}
```

cluster会创建一个actor(ClusterCoreSupervisor)
```
class ClusterCoreSupervisor:ReceiveActor
{
    private IActorRef _publisher;
    private IActorRef _coreDaemon;
    ...
}
```
ClusterCoreSupervisor的两个子Actor:_publisher,_coreDaemon

- _publisher的作用是处理订阅事件，很容易理解这是本地的消息订阅，记录订阅者和消息类型，跟常用的事件消息实现类型，而远程的gossip副本通过gossipto同步。每当gossip发生变化，通过Publish(gossip)同步给所有订阅者

- _coreDaemon则是cluster的核心实现

ClusterCoreDaemon结构的几个重要成员
- Gossip _latestGossip：每个cluster维护着一个Gossip副本，可以理解为版本内容，通过gossip协议（定时发送给其它节点，最终收敛），使得gossip副本的数据达到最终一致
- ImmutableList(Address) _seedNodes：种子节点，也就是节点启动时首先连接的节点，需要在启动配置中配置seednodes
- IActorRef _seedNodeProcess:启动actor,启动时负责向种子节点发出initJoin->Join完成建立加入集群的工作。

#### 启动逻辑
```
       +------------+       +----------+
       |            |       |          |
       | normalNode |       | seedNode |
       |            |       |          |
       +-----+------+       +-----+----+
+---------+  |                    |
| uninit  |  |      initJoin      |
+---------+  +-------------------->
             |      initJoinAck   |
             +<-------------------+
+----------+ |                    |
| tryJoining |      Jointo        |
+----------+ +-------------------->
             |                    |
             |      Welcome       |
   +------+  <--------------------+
   | init |  |      gossip        |
   +------+  +------------------->+

```
1. 一个普通节点会向种子节点s发送initJoin消息，此时普通节点的状态为Uninitialized;
 
```
// send InitJoin to remaining seed nodes (except myself)
foreach (var seed in _remainingSeeds.Select(
    x => Context.ActorSelection(Context.Parent.Path.ToStringWithAddress(x))))
    seed.Tell(new InternalClusterAction.InitJoin());
```
2. 种子节点收到initJoin消息返回initJoinAck(这里要检查状态是否可达);

```
public void InitJoin()
{
    var selfStatus = _latestGossip.GetMember(SelfUniqueAddress).Status;
    if (Gossip.RemoveUnreachableWithMemberStatus.Contains(selfStatus))
    {
        _cluster.LogInfo("Sending InitJoinNack message from node [{0}] to [{1}]", SelfUniqueAddress.Address,
            Sender);
        // prevents a Down and Exiting node from being used for joining
        Sender.Tell(new InternalClusterAction.InitJoinNack(_cluster.SelfAddress));
    }
    else
    {
        // TODO: add config checking
        _cluster.LogInfo("Sending InitJoinNack message from node [{0}] to [{1}]", SelfUniqueAddress.Address,
            Sender);
        Sender.Tell(new InternalClusterAction.InitJoinAck(_cluster.SelfAddress));
    }
}
```
3. 收到initJoinAck后，_seedNodeProcess已经完成了自己的任务，会stop掉，给父节点(也就是ClusterCoreDaemon这个actor)发JoinTo消息，然后stop掉自己。ClusterCoreDaemon给seedNode发Jointo消息
```
else if (message is InternalClusterAction.InitJoinAck)
{
    // first InitJoinAck reply, join existing cluster
    var initJoinAck = (InternalClusterAction.InitJoinAck)message;
    Context.Parent.Tell(new ClusterUserAction.JoinTo(initJoinAck.Address));
    Context.Stop(Self);
}
```
4. 种子节点响应返回Welcome消息（这里主要判断地址成员是否有效，重复ip，port等）

```
var localMembers = _latestGossip.Membe
// check by address without uid to make sure that node with same host:port is not allowed
// to join until previous node with that host:port has been removed from the cluster
var localMember = localMembers.FirstOrDefault(m => m.Address.Equals(node.Address));
if (localMember != null && localMember.UniqueAddress.Equals(node))
{
    // node retried join attempt, probably due to lost Welcome message
    _cluster.LogInfo("Existing member [{0}] is joining again.", node);
    if (!node.Equals(SelfUniqueAddress))
    {
        Sender.Tell(new InternalClusterAction.Welcome(SelfUniqueAddress, _latestGossip));
    }
}
```
5. 收到Welcome消息，握手基本完成, 将自己的状态变成Initialized,发布消息,然后同步gossip

```
_cluster.LogInfo("Welcome from [{0}]", from.Address);
_latestGossip = gossip.Seen(SelfUniqueAddress);
AssertLatestGossip();
Publish(_latestGossip);
if (!from.Equals(SelfUniqueAddress))
    GossipTo(from, Sender);
BecomeInitialized();
```

### 订阅消息
publisher
```
ClusterDomainEventPublisher
{
     _latestGossip = Gossip.Empty;
    _eventStream = Context.System.EventStream;
    ...
}
```
订阅事件结构很简单，维护一个gossip副本和一个事件管理器（同进程内的eventlistener），给本进程内订阅的actor一个注册监听的功能，实现Subscribe和Unsubscribe。每当本进程的gossip副本发生变化，即同步给所有订阅的cluster事件的actor。cluster事件包括MemberJoined，MemberUp，MemberExited等。

### gossip副本同步
gossip副本的同步是实现cluster的状态一致的核心算法。
#### Gossip的结构
```
class Gossip
{
    readonly ImmutableSortedSet<Member> _members;
    readonly GossipOverview _overview;
    readonly VectorClock _version;
    ...
}
```
members维护各节点的信息（Address,status等）
version则是向量时钟([为什么需要向量时钟](https://zhuanlan.zhihu.com/p/56886156) 向量时钟这篇文章讲得比较清楚。)，通过
将版本信息多存储各各节点的版本信息来检测分布式系统中多副本更新的数据冲突，类似git版本管理。

#### GossipTo
加入种子点后发首先发一次GossipTo来同步副本状态，收到Gossip的处理大致：

```
public ReceiveGossipType ReceiveGossip(GossipEnvelope envelope)
{
    ...
    var comparison = remoteGossip.Version.CompareTo(localGossip.Version);
    ...
    switch (comparison)
    {
        case VectorClock.Ordering.Same:
            //same version
            talkback = !_exitingTasksInProgress && !remoteGossip.SeenByNode(SelfUniqueAddress);
            winningGossip = remoteGossip.MergeSeen(localGossip);
            gossipType = ReceiveGossipType.Same;
            break;
        case VectorClock.Ordering.Before:
            //local is newer
            winningGossip = localGossip;
            talkback = true;
            gossipType = ReceiveGossipType.Older;
            break;
        case VectorClock.Ordering.After:
            //remote is newer
            winningGossip = remoteGossip;
            talkback = !_exitingTasksInProgress && !remoteGossip.SeenByNode(SelfUniqueAddress);
            gossipType = ReceiveGossipType.Newer;
            break;
        default:
            ...
    }
}
```

- 版本比较：vectorclock的版本比较算法，参见VectorClock.cs的CompareOnlyTo，即是向量时钟的比较算法，可以参考之前的链接[为什么需要向量时钟](https://zhuanlan.zhihu.com/p/56886156)
- 如果版本一致：copy一份作为winningGossip
- 本地版本更新: winningGossip = localGossip
- 远程版本更新: winningGossip = remoteGossip
- 有版本冲突： Merge

###### merge方法
```
public Gossip Merge(Gossip that)
{
    //TODO: Member ordering import?
    // 1. merge vector clocks
    var mergedVClock = _version.Merge(that._version);

    // 2. merge members by selecting the single Member with highest MemberStatus out of the Member groups
    var mergedMembers = EmptyMembers.Union(Member.PickHighestPriority(this._members, that._members));

    // 3. merge reachability table by picking records with highest version
    var mergedReachability = this._overview.Reachability.Merge(mergedMembers.Select(m => m.UniqueAddress),
        that._overview.Reachability);

    // 4. Nobody can have seen this new gossip yet
    var mergedSeen = ImmutableHashSet.Create<UniqueAddress>();

    return new Gossip(mergedMembers, new GossipOverview(mergedSeen, mergedReachability), mergedVClock);
}
```

##### GossipTick和GossipSpeedupTick
除了启动时GossipTo同步副本状态，ClusterCoreDaemon还启动定时器去同步Gossip,这个消息是InternalClusterAction.GossipTick，通过settings.GossipInterval配置同步的间隔（默认配置2s）。
为了达到更快速的最终一致性，GossipTick会在需要时加速定时器触发GossipSpeedupTick
```
public void GossipTick()
{
    SendGossip();
    if (IsGossipSpeedupNeeded())
    {
        _cluster.Scheduler.ScheduleTellOnce(new TimeSpan(_cluster.Settings.GossipInterval.Ticks / 3), Self,
            InternalClusterAction.GossipSpeedupTick.Instance, ActorRefs.NoSender);
        _cluster.Scheduler.ScheduleTellOnce(new TimeSpan(_cluster.Settings.GossipInterval.Ticks * 2 / 3)Self,
            InternalClusterAction.GossipSpeedupTick.Instance, ActorRefs.NoSender);
    }
}
```
```
public bool IsGossipSpeedupNeeded()
{
    return _latestGossip.Overview.Seen.Count < _latestGossip.Members.Count / 2;
}
```

### 错误检测
为了监控节点的存活状态，ClusterDaemon会启定时器_failureDetectorReaperTaskCancellable
```
// start periodic cluster failure detector reaping (moving nodes condemned by the failure detector to unreachable list)
_failureDetectorReaperTaskCancellable =
    scheduler.ScheduleTellRepeatedlyCancelable(
        settings.PeriodicTasksInitialDelay.Max(settings.UnreachableNodesReaperInterval),
        settings.UnreachableNodesReaperInterval,
        Self,
        InternalClusterAction.ReapUnreachableTick.Instance,
        Self);
```

该定时器默认配置unreachable-nodes-reaper-interval = 1s，触发ReapUnreachableMembers
```
var newlyDetectedUnreachableMembers =
    localMembers.Where(member => !(
        member.UniqueAddress.Equals(SelfUniqueAddress) ||
        localOverview.Reachability.Status(SelfUniqueAddress, member.UniqueAddress)Reachability.ReachabilityStatus.Unreachable ||
        localOverview.Reachability.Status(SelfUniqueAddress, member.UniqueAddress)Reachability.ReachabilityStatus.Terminated ||
        _cluster.FailureDetector.IsAvailable(member.Address))).ToImmutableSortedSet
var newlyDetectedReachableMembers = localOverview.Reachability.AllUnreachableFrom(SelfUniqueAddress)
        .Where(node => !node.Equals(SelfUniqueAddress)_cluster.FailureDetector.IsAvailable(node.Address))
        .Select(localGossip.GetMember).ToImmutableHashSet();
```
通过FailureDetector检查gossip的各节点状态，地址是否可达等。