## akka的actor并发benchmark

akka.net的解决方案中有spawnbenchmark工程

![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_25_akka_spawn_benchmark/akka_actor_benchmark01.png?raw=true)

[SpawnBenchmark](https://github.com/akkadotnet/akka.net/blob/dev/src/benchmark/SpawnBenchmark/Program.cs)

```
private void StartRun(int n)
{
    Console.WriteLine($"Start run {n}");
    var start = Stopwatch.ElapsedMilliseconds;
    Context.ActorOf(SpawnActor.Props).Tell(new SpawnActor.Start(7, 0));
    Context.Become(Waiting(n - 1, start));
}
```
- 测试逻辑是，创建一个actor(rootactor),每个actor创建10个子actor,递归深度7次，也就是10,000,000(10的7次方)个actor.全部actor创建完，给parent actor发消息直到root actor。计算从rootactor创建到rootactor收到消息计算耗时。

####  本地测试环境：【i5-7500, Win10】

![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_25_akka_spawn_benchmark/akka_actor_spawn_benchmark.png?raw=true)

#### 对比go的goroutine
网上有个同样测试逻辑的工程，分别对libmill,go（goroutine），python(gevent)做了百万并发benchmark

[Skynet 1M concurrency](https://github.com/atemerev/skynet)

下面是网上的一些测试数据

![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_25_akka_spawn_benchmark/akka_actor_spawn_benchmark02.png?raw=true)

- 这里主要是对比go的goroutine并发性能

#### 本地测试环境：【i5-7500, Win10】

![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_03_25_akka_spawn_benchmark/akka_actor_spawn_benchmark03.png?raw=true)

可以看到，akka.net跟go的actor并发性能还是差很多