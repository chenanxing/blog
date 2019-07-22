分为两部分，一是建立连接，二是发消息
### 建立连接
发消息的入口是actorRef的tell方法
```
public void Tell(object message, IActorRef sender = null)
{
    if (sender == null && ActorCell.Current != null && ActorCell.Current.Self != null)
        sender = ActorCell.Current.Self;
        DeliverSelection(Anchor as IInternalActorRef, sender, new ActorSelectionMessage(message, Path, wildCardFanOut: false));
}
```
其中DeliverSelection通过多态实现了localActor和remoteActor的tellInternal,这里看remoteActor的tellInternal

```
remoteactor.TellInternal(){
    Remote.Send(message, sender, this)
}
```
remote的send实际调用的是_endpointManager的tell,构造一个endpointmanager.send消息
```
public override void Send(object message, IActorRef sender, RemoteActorRef recipient)
{
    if (_endpointManager == null)
    {
        throw new RemoteTransportException("Attempted to send remote message but Remoting is not running.", null);
    }
    _endpointManager.Tell(new EndpointManager.Send(message, recipient, sender), sender ?? ActorRefs.NoSender);
}
```

再看看endpointmanager对send消息的处理（endpointmanager是一个actor,每个actor都有处理消息的状态机,这里就是Send消息的响应方法）
```
Receive<Send>(send =>
{
    var recipientAddress = send.Recipient.Path.Address;
    IActorRef CreateAndRegisterWritingEndpoint(int? refuseUid) _endpoints.RegisterWritableEndpoint(recipientAddress, CreateEndpoint(recipientAddressend.Recipient.LocalAddressToUse, _transportMapping[send.Recipient.LocalAddressToUse], _settingwriting: true, handleOption: null, refuseUid: refuseUid), uid: null, refuseUid: refuseUid
    // pattern match won't throw a NullReferenceException if one is returned WritableEndpointWithPolicyFor
    _endpoints.WritableEndpointWithPolicyFor(recipientAddress).Match()
        .With<Pass>(
            pass =>
            {
                pass.Endpoint.Tell(send);
            })
        .With<Gated>(gated =>
        {
            if (gated.TimeOfRelease.IsOverduCreateAndRegisterWritingEndpoint(gated.RefuseUid).Tell(send);
            else Context.System.DeadLetters.Tell(send);
        })
        .With<WasGated>(wasGated =>
        {
            CreateAndRegisterWritingEndpoint(wasGated.RefuseUid).Tell(send);
        })
        .With<Quarantined>(quarantined =>
        {
            // timeOfRelease is only used for garbage collection reasons, therefore it is ignored here. still have
            // the Quarantined tombstone and we know what UID we don't want to accept, so use it.
            CreateAndRegisterWritingEndpoint(quarantined.Uid).Tell(send);
        })
        .Default(msg => CreateAndRegisterWritingEndpoint(null).Tell(send));
});
```
其中_addressToWritable记录endpoint的连接状态
如果该endpoint是pass状态（已经建立好tcp连接）调用endpoint的tell方法

其它状态，先接立连接。_endpoints.createEndPoint()
endpoint的createEndPoint,是创建一个endpointWriter,endpointWriter初始化prestart时通过网络库dotnetty的client方法AssociateAsync建立连接.
```
private async Task<object> AssociateAsync()
{
    try
    {
        return new Handle(await Transport.Associate(RemoteAddress, _refuseUid).ConfigureAwait(false));
    }
    catch (Exception e)
    {
        return new Status.Failure(e.InnerException ?? e);
    }
}
```

```
public async Task<AkkaProtocolHandle> Associate(Address remoteAddress, int? refuseUid)
// Prepare a Task and pass its completion source to the manager
var statusPromise = new TaskCompletionSource<AssociationHandle>()   manager.Tell(new 

AssociateUnderlyingRefuseUid(SchemeAugmenter.RemoveScheme(remoteAddress), statusPromiseefuseUid))   return (AkkaProtocolHandle)await statusPromise.Task.ConfigureAwait(false);

```

akkaprotocolmanager处理AssociateUnderlyingRefuseUid消息AssociateInternal
```
protected override async Task<AssociationHandle> AssociateInternal(Address remoteAddress)
{
    var clientBootstrap = ClientFactory(remoteAddress);
    var socketAddress = AddressToSocketAddress(remoteAddress);
    socketAddress = await MapEndpointAsync(socketAddress).ConfigureAwait(false);
    var associate = await clientBootstrap.ConnectAsync(socketAddress).ConfigureAwait(false);
    var handler = (TcpClientHandler)associate.Pipeline.Last();
    return await handler.StatusFuture.ConfigureAwait(false);
}
```
clientBootstrap是一个dotnettytransport的client,网络库用的dotnetty完成tcp连接,返回一个TcpClientHandler，注册到_addressToWritable记录endpoint的连接状态,endpointwriter创建完会创建endpointreader, 两端建立双向通道（两边都建立好endpointwriter和endpointreader）,同时writer会创建DefaultMessageDispatcher用于远程消息分发。
上述建立连接的actor消息流向图
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_21_akka_remote2/akka_remote201.png?raw=true)

### 发消息
tell方法，给endpointwriter发Send消息，endpointWrite处理Send消息
private bool WriteSend(EndpointManager.Send send)

```
{
    ...
     var pdu = _codec.ConstructMessage(send.Recipient.LocalAddressToUse, send.Recipient,
                    this.SerializeMessage(send.Message), send.SenderOption, send.Seq, _lastAck);
    var ok = _handle.Write(pdu);
    ...
}
```

远程的TcpServerHandler处理消息ChannelRead

```
InboundPayload
.With<InboundPayload>(ip =>
{
    var pdu = DecodePdu(ip.Payload);
    
    var ackAndMessage = TryDecodeMessageAndAck(payload);
    _msgDispatch.Dispatch(ackAndMessage.MessageOption.Recipient,
                                ackAndMessage.MessageOption.RecipientAddress,
                                ackAndMessage.MessageOption.SerializedMessage,
                                ackAndMessage.MessageOption.SenderOptional);
}
```
其中_msgDispatch是DefaultMessageDispatcher,收到消息再分发到对应的actor处理消息

上述发消息的actor消息流向图
![image](https://github.com/chenanxing/blog/blob/master/etakka/2019_07_21_akka_remote2/akka_remote202.png?raw=true)

后续总结下路由router部分，再研究下cluster,persistent,singleactor(stub)部分