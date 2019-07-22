---
title: '2019_07_14_akka_remote（未完）'
date: 2019-07-14 21:10:33
---
# Akka学习remote篇

akka的ActorSystem是单进程的，而remote模块给ActorSystem提供了跨进程通信的机制，cluster是Akka在remote模块基础上实现集群。下面先介绍remote模块的功能和使用例子，然后第二部分是其实现细节。

### remote模块提供了什么功能
1. 位置透明(Location transparency with RemoteActorRef)
    通过少量配置实现，像本地actor一样调用远程actor。基本原理是，一个actor引用是IActorRef,而LocalActorRef和RemoteActorRef都继承IActorRef,在调用Tell时会调用对应实例的虚函数Tell（LocalActorRef or RemoteActorRef）
    一个简单的例子：
```
using (var system = ActorSystem.Create("Deployer", ConfigurationFactory.ParseString(@"
            akka {  
                actor{
                    provider = remote
                    deployment {
                        /remoteecho {
                            remote = ""akka.tcp://DeployTarget@localhost:8090""
                        }
                    }
                }
                remote {
                    dot-netty.tcp {
                        port = 0
                        hostname = localhost
                    }
                }
            }")))
var localEcho = system.ActorOf(Props.Create(() => new EchoActor()), "localecho");
localEcho.Tell(new Hello("hi from localEcho!"))
var remoteEcho = system.ActorOf(Props.Create(() => new EchoActor()), "remoteecho");
remoteEcho.Tell(new Hello("hi from remoteEcho!"))
```
创建system时添加配置 /remoteecho,system.ActorOf通过actor的名字参数可以分别创建本地actor和远程actor,拿到IActorRef的情况下，与本地或远程actor来说是通信透明的。

2 远程定位(remote addressing)
    在知道actor位置的情况下,与这个actor通信要用到ActorSelection，还可以给actor发送Identify消息拿到这个Actor的IActorRef。
    ActorSelection的使用例子
```
system.ActorSelection("/user/remoteecho").Tell(new Hello("hi from selection!"));
```
其中/remoteecho是上面配置的"akka.tcp://DeployTarget@localhost:8090",
这个地址叫actorPath,由Protocol,ActorSystem,Address,Path构成,无非就是定位到远程对应的actor地址,ActorSelection内部会解析这个地址给远程actorsystem的一个特定actor(RemoteSystemDaemon)发消息，再找到对应actor。下图是actorPath名字的约定图示
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_14_akka_remote/akka_remote01.png?raw=true)

3 远程发消息(remote message)
    通过ActorSelection定位actor,或者拿到IActorRef调用Tell都可以给对应actor发消息。底层逻辑是底层管理transport和EndPointManager, tranports管理底层连接，EndPointManager负责上层发送接收和管理连接。后面会具体讲实现细节。
    
4 远程发布actor(remote depoly)
    在Akka中如何在远程创建一个actor（同BigWorld.createBaseRemotely）
    还是上面的例子
```
var remoteEcho1 = system.ActorOf(Props.Create(() => new EchoActor()), "remoteecho");
```
配置指定remoteecho为"akka.tcp://DeployTarget@localhost:8090",本地actor和远程actor的创建都是system.ActorOf，不同的只是actor名字参数。在远程创建中，最终调用的是RemoteActorRefProvider的actorof,创建一个RemoteActorRef.RemoteActorRef是远程actor的一个本地引用，创建过程是给远程一个特殊actor(remote)发消息DaemonMsgCreate，远程actor处理消息HandleDaemonMsgCreate并在本地创建对应actor(_system.Provider.ActorOf)
    
5 默认的分布式(Distributed by Default)
    transport:Akka.Remote默认支持tcp，底层用dot-netty库作为网络库。
    Address：ip+port或者hostname
    endpoint: 一个endpoint管理一个远程连接，endPointManager管理所有endPoint,而EndPointManager是顶层Actor的一个子Actor（）
    assoication: 两个endpoint之间的连接，初始化时打开端口监听连接。
    ![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_14_akka_remote/akka_remote02.png?raw=true)
	
### 实现细节
transports和endpointManager就两个比较重要的类，transports负责管理底层连接和endpointManager负责上层发送接收