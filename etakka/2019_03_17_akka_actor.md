## AKKA学习actor篇

### 什么是actor
```
The actor model provides an abstraction that allows you to think about your code in terms of communication, not unlike people in a large organization. The basic characteristic of actors is that they model the world as stateful entities communicating with each other by explicit message passing.
```
akka.net对actor模型的定义，actor模型中，由有状态的实体消息传递相互通信构成。

理解：
- Actor是并发模型的一种抽象。每个actor之间不共享状态，通过发消息的机制通信，避免了多线程的锁（实际由global队列的原子操作控制）。作用就是可以高效的利用多核cpu，而不需要担心锁。
- Actor适合分布式。每个执行单元都应该是一个actor,无须关心每个actor的具体位置，单机或者分布式中有同样的操作方式（remote,cluster）
- 比如skynet是actor模型的一种实现，每个actor是一个lua vm,每个actor的mailbox挂在全局的globalqueue,由工作线程worker从globalqueue取出消息执行，并控制每个actor只能被一个worker同时执行。
- akka中的actor,跟面向对象相似，在akka中一切都是actor。actorcell本身带有状态，行为，邮箱，子Actor和监管策略。

### AKKA中的actor
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_17_akka_actor/akka_actor01.png?raw=true)

上图是akka.net文档对[actor](https://getakka.net/articles/concepts/actors.html)的描述

理解：
1. Mailbox:邮箱，actor间通信通过给邮箱发消息，调用过程是往接收actor邮箱发消息->记为邮箱有消息->线程池worker检查邮箱->接收actor触发收消息
2. Behavior:行为，处理消息的行为
3. State:状态,actor维护着一个私有的状态，不共享
4. SupervisorStrategy:监管策略,一个actor对子actor的错误处理，有两种:AllForOneStrategy和OneForOneStrategy
5. Event-driven thread:每个actor只由单线程同时执行，akka中提供了多种dispatch的可选方案，默认是使用dotnet的threadpool。
6. ActorRef:指向一个actor,actorRef可以在本机也可以是远程
7. Actor Systems: 一个app只有一个actorsystems,管理所有actor，actor之间组成树结构，至少有三个actor（/user: 守护Actor，/system: 系统守护者，/: 根守护者）

### AKKA中的线程模型
上面介绍了akka的actor类的基本结构。本篇主要写actor在akka中是如何实现的。
```
protected ExecutorServiceConfigurator ConfigureExecutor()
{
    var executor = Config.GetString("executor");
    switch (executor)
    {
        case null:
        case "":
        case "default-executor":
        case "thread-pool-executor":
            return new ThreadPoolExecutorServiceFactory(Config, Prerequisites);
        case "fork-join-executor":
            return new ForkJoinExecutorServiceFactory(Config, Prerequisites);
        case "current-context-executor":
            return new CurrentSynchronizationContextExecutorServiceFactory(Config, Prerequisites);
        case "task-executor":
            return new DefaultTaskSchedulerExecutorConfigurator(Config, Prerequisites);
        default:
            ...
        }
```
从任务执行的配置可以看到，akka提供了多种任务执行方式。
- ThreadPoolExecutorServiceFactory：根据App.Domain的isFullTrust调用TreadPool.UnsafeQueueUserWorkItem或者ThreadPool.QueueUserWorkItem
- ForkJoinExecutorServiceFactory:使用[DedicatedThreadPool](https://github.com/helios-io/DedicatedThreadPool).QueueUserWorkItem,（没具体看，大概是对线程池耗时任务的优化）
- CurrentSynchronizationContextExecutorServiceFactory：使用TaskScheduler.FromCurrentSynchronizationContext
- DefaultTaskSchedulerExecutorConfigurator：TaskScheduler.Default  (dotnet的Task,Task是线程池的上层封闭，对耗时任务不放到线程池中而另起线程)

这里从默认配置方式ThreadPool深入了解。

#### 知识准备
[dotnet threadpool](https://www.codeproject.com/Articles/89284/Parallel-programming-in-NET-Internals)
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_17_akka_actor/akka_actor02.png?raw=true)

ThreadPoolGlobals:全局队列

```
public static readonly ThreadPoolWorkQueue workQueue = new ThreadPoolWorkQueue();
```

ThreadPoolWorkQueueThreadLocals:本线程队列，ThreadStatic属性，初始化加入WorkStealingQueueList,本线程队列和主队列都没有任务时从别的线程队列偷一任务
```
[ThreadStatic]
public static ThreadPoolWorkQueueThreadLocals threadLocals;
```

QueueUserWorkItem:线程池的入口，给TreadPool加入任务
```
public static bool QueueUserWorkItem(WaitCallback callBack, object state)
{
    ...
    ThreadPoolGlobals.workQueue.Enqueue(tpcallBack, forceGlobal: true);
    return true;
}
```

加入队列可选加入全局队列或本线程队列
```
public void Enqueue(object callback, bool forceGlobal)
{
    ...
    ThreadPoolWorkQueueThreadLocals tl = null;
    if (!forceGlobal)
        tl = ThreadPoolWorkQueueThreadLocals.threadLocals;
    if (null != tl)
    {
        tl.workStealingQueue.LocalPush(callback);
    }
    else
    {
        workItems.Enqueue(callback);
    }
    EnsureThreadRequested();
}
```
dispatch中取任务执行

```
internal static bool Dispatch()
{
    while (ThreadPool.KeepDispatching(startTickCount))
    {
        bool missedSteal = false;
        // Use operate on workItem local to try block so it can be enregistered 
        object workItem = outerWorkItem = workQueue.Dequeue(tl, ref missedSteal);
        ...
        task.ExecuteFromThreadPool(currentThread);
    }
}
```


ThreadPool的2个workqueue,一个gobal和一些local.先从local找任务，再去找global,都找不出随机一个local偷一个任务
```
public object Dequeue(ThreadPoolWorkQueueThreadLocals tl, ref bool missedSteal)
{
    WorkStealingQueue localWsq = tl.workStealingQueue;
    object callback;
    if ((callback = localWsq.LocalPop()) == null && // first try the local queue
    !workItems.TryDequeue(out callback)) // then try the global queue
    {
        // finally try to steal from another thread's local queue
        WorkStealingQueue[] queues = WorkStealingQueueList.Queues;
        int c = queues.Length;
        Debug.Assert(c > 0, "There must at least be a queue for this thread.");
        int maxIndex = c - 1;
        int i = tl.random.Next(c);
        while (c > 0)
        {
            i = (i < maxIndex) ? i + 1 : 0;
            WorkStealingQueue otherQueue = queues[i];
            if (otherQueue != localWsq && otherQueue.CanSteal)
            {
                callback = otherQueue.TrySteal(ref missedSteal);
                if (callback != null)
                {
                    break;
                }
            }
            c--;
        }
    }
    return callback;
}
```
上面就是threadpool的基本逻辑，akka中的默认actor消息执行是threadpool方式

### akka中的actor实现（如何结合线程模型）
ICanTell抽象类的Tell方法是actor的消息发起，actor消息有两类systemMessage和mailboxMessage.
Tell->TellInternal->SendMessage->Dispatcher.Dispatch
发消息实际是给目标actor的mailbox插入Envelope(Envelope包装了message和sender)
```
public virtual void Dispatch(ActorCell cell, Envelope envelope)
{
    var mbox = cell.Mailbox;
    mbox.Enqueue(cell.Self, envelope);
    RegisterForExecution(mbox, true, false);
}
```

然后RegisterForExecution，标记mbox有任务执行，同时保证了一个mailbox只加入threadpool一次
```
internal bool RegisterForExecution(Mailbox mbox, bool hasMessageHint, bool hasSystemMessageHint)
{
    if (mbox.CanBeScheduledForExecution(hasMessageHint, hasSystemMessageHint)) //This needs to be here to ensure thread safety and no races
    {
        if (mbox.SetAsScheduled())
        {
            ExecuteTask(mbox);
            return true;
        }
        return false;
    }
    return false;
}
```
ExecuteTask调用Executor.Execute(run)，这里就是具体的任务执行器的调度，比如threadpool方案，Execute就是给线程池抛任务，到这里就完成了actor模型与线程模型的结合
```
public override void Execute(IRunnable run)
{
    ThreadPool.UnsafeQueueUserWorkItem(Executor, run);
}
```

Mailbox本身是一个IRunnable，当它被线程执行时，分别处理了系统消息(resume,stop等)和邮箱消息
```
public void Run()
{
    try
    {
        if (!IsClosed()) // Volatile read, needed here
        {
            Actor.UseThreadContext(() =>
            {
                ProcessAllSystemMessages(); // First, deal with any system messages
                ProcessMailbox(); // Then deal with messages
            });
        }
    }
    finally
    {
        SetAsIdle(); // Volatile write, needed here
        Dispatcher.RegisterForExecution(this, false, false); // schedule to run again if there are more messages, possibly
    }
}
```

ProcessMailbox,TryDequeue对MessageQueue出队列，消息执行
```
private void ProcessMailbox(int left, long deadlineTicks)
{
    while (ShouldProcessMessage())
    {
        if (!TryDequeue(out var next)) return;
            DebugPrint("{0} processing message {1}", Actor.Self, next);
        Actor.Invoke(next);
        ProcessAllSystemMessages();
        ...
        break;
    }
}
```

最后Actor的Invoke调用的ReceiveMessage,最终根据actor的Receive处理消息
```
protected virtual void ReceiveMessage(object message)
{
    var wasHandled = _actor.AroundReceive(_state.GetCurrentBehavior(), message);
    if (System.Settings.AddLoggingReceive && _actor is ILogReceive)
    {
        //TODO: akka alters the receive handler for logging, but the effect is the same. keep it this way?
        var msg = "received " + (wasHandled ? "handled" : "unhandled") + " message " + message + " from " + Sender.Path;
        Publish(new Debug(Self.Path.ToString(), _actor.GetType(), msg));
    }
}
```
以上就是一个actor从SendMessage到ReceiveMessage的流程

总结：akka中的actor消息机制的分配默认使用dotnet的ThreadPool.ThreadPool的2个workqueue,一个global和一些local.并发队列global的workItem使用的是ConcurrentQueue。