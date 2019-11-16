---
layout: post
title: RPC 青铜时代—— Flex BlazeDS 和 Spring Remoting
date: 2019-05-10
categories: rpc
tags: [java,rpc]
description: 系统间通信技术 blazeds 和 spring remoting
---

#### 系统间交互

2012年第一次到公司上班，问的第一个问题就是 Flex 怎么和后台的 Java 代码交互呢？那时候 low 哥的头发还没全白，不知道他还记不记得。

远程调用是多系统之间交互绕不过去的一关，一个系统一旦引入网络就会带来更多的不确定性和复杂性。现在的系统间通信基础基于 TCP/IP 协议之上，但是在应用层，依然有很多不同的选择和架构方式会对整个交互过程和效果产生影响。在几年的工作中，也陆续接触了不少和通信层架构相关的东西，个人感觉一个比较好的通信框架需要具备以下的特点：

1. 高性能，网络通信是整个系统开销中不容忽视的一个环节，能否高效完成网络传输，是评价通信框架的主要标准，主要的衡量指标比如 QPS 等，如果自己写一个通信框架，那么你最起码性能要优于最常见的 HTTP 通信方式。
2. 兼容性，主要是多语言的支持，不同的系统可能会采用不同的编程语言，通信协议的主要目标是要完成系统交互，如果只能支持部分的变成语言，那会限制它的应用场景，兼容性差的例子：Java RMI，兼容性好的例子：WebService、HTTP。
3. 功能性，系统的交互通过通信框架完成，那么很多通用功能也可以抽取到通信框架来实现，常见的负载均衡、流量控制、服务编排调度、监控和追踪、安全等。可以把这些内容概括为服务治理，潜台词就是，我们希望通过引入通信框架，来管理数量众多的服务，知晓目前系统的状态。
4. 易用性，框架的主要功能就是封装和提供编程集成标准。易用性包括了对开发使用友好，不需要太多网络层相关的知识就可以上手使用；对调试和测试友好，出现问题要能够快速定位解决；低侵入性，要做到100%的无侵入性难度很大，但是应该尽量减少对特定编程环境和编程模式的依赖，特别是在稳定运行的旧系统引入新的通信架构，应该尽量避免对系统架构大动干戈，这些都是容易使用和推广的重要指标。

#### Flex BlazeDS

刚从事工作的时候，公司还大规模使用基于 BlazeDS 的 Adobe Flex + Java 技术体系。关于Flex，目前已经很少使用了，在十年前，Flash最流行的时候也算是炙手可热的技术了，很多现在前端框架正在践行的理念，比如大前端、模块化、双向数据绑定等，借助强大的 avm 虚拟机和 actionscript 在 Flex 中都有所应用。虽然最终因为种种原因被抛弃，但是个人觉得Flex是一个非常有前瞻性的技术，有兴趣的同学，可以自行了解一下 [Apache Flex](http://flex.apache.org/)。

Flex+Java开发的 B/S 应用系统中，B系统（客户端系统）和S系统（服务端系统）完全分离，各自独立运行在不同的CPU和虚拟机中，B系统主要负责“展现层”逻辑，而S系统主要负责“领域层”和“数据源层”逻辑，因此 Flex+Java 所开发的企业应用系统是异构的分布式系统。BlazeDS就是解决异构的客户端系统和服务器端系统如何通信的问题。

BlazeDS 的核心功能包括远程调用和消息服务，这边主要讲远程调用。

由于 BlazeDS 是一种 RPC 框架，在开始讲 BlazeDS 架构之前，先引入 RPC 的定义会有助于理解。Remote Procedure Call ，是指计算机 A 上的进程，调用另外一台计算机 B 上的进程，其中 A 上的调用进程被挂起，而 B 上的被调用进程开始执行，当值返回给 A 时，A 进程继续执行。调用方可以通过使用参数将信息传送给被调用方，而后可以通过传回的结果得到信息。而这一过程，对于开发人员来说是透明的。

Bruce Jay Nelson 的论文 Implement Remote Procedure Calls，定义了 RPC 的调用标准，后面所有 RPC 框架，都是按照这个标准模式来的。

![RCP](/postsimg/rpc/rcp-1540803585-31147.png)

当客户端的应用想发起一个远程调用时，它实际上是通过本地调用本地调用方的 Stub。它负责将调用的接口、方法和参数，通过约定的洗衣规范进行编码，并通过本地的 RPCRuntime 进行传输，将调用网络包发送到服务器。

服务器的 RPCRumtime 收到请求后，交给提供方 Stub 进行解码，然后调用服务端的方法，服务端执行方法，返回结果，提供方 Stub 将返回结果编码后，发送给客户端，客户端的 RPCRuntime 收到结果，发给调用方 Stub 解码得到结果，返回给客户端。

这里面分了三个层次，对于用户称和服务端，都像本地调用一样，专注于业务逻辑的处理就可以了。对于 Stub层，处理双方约定好的语法、语义、封装、解封装。对于 RPCRuntime，主要处理高性能的传输，以及网络的错误和异常。

下面开始 BlazeDS 架构的介绍

一个 BlazeDS 应用通常包括了客户端的 Flex 应用程序和服务端的 Java EE 应用。

##### Flex 应用程序

![flexclient](/postsimg/rpc/flexclient-1540805448-14924.png)

在客户端，由Flex RPC 或者 `Message` 组件发起会话请求，由`Channel`将参数或命令使用指定的网络协议（HTTP/s ，AMF（Action Message Format 是基于HTTP的二进制传输协议））与服务器端进行会话。

`Operation` 代表远程服务中的单个方法，`RemoteObject` 委托 `mx.rpc.remoting.Operation`，WebService 使用 `mx.rpc.soap.Operation` 来调用远程服务器中的方法，注意，这些都是 actionscript 类，可以看成是**调用方 Stub**，这两个 `Operation` 都是从 `mx.rpc.AbstractInvoker`继承而来的。`AbstractInvoker` 是所有RPC 调用的抽象基类，HttpService访问的是远程的 URL，不涉及远程方法调用，因此直接从 `AbstractInvoker` 继承。

`AsyncRequest` 则提供了这样的一种机制：它允许对远程`Destination`进行多次请求，并在服务器处理完请求后会调用请求者。由于Flex的远程调用都是异步的，不能保证请求和响应以同一顺序完成。假设A和B一次发出对同一个 `Destination` 的远程调用请求，如果 B 请求的响应先于 A 返回，那么 `AsyncRequest` 也可以保证请求和回调的意义对应。`AbstractInvoker` 借助 `AsyncRequest` 进行远程调用，因此也获得了这个能力。

`Channel` 这是代表一个到远端 `Endpoint` 的连接。并且可以在不同的 `Destination` 调用中共享。

![asyncrequest](/postsimg/rpc/asyncreque-1540807442-23728.png)

`AsyncRequest` 是一个 `Producer`，也就是说，它是一个消息的生产者，可以通过`Channel`将消息传输到服务端。

`Message` 特指 BlazeDS 能识别的格式封装的数据，包括消息头和消息体。消息头一般用于存储消息的处理指令，消息体包含传输的业务数据。Flex 应用和 BlazeDS 之间透明地序列化和反反序列化。

![Message](/postsimg/rpc/message-1540825368-1671463798.png)

`Message` 接口定义了消息的基本数据类型，最终要的是 `destination` 和 `body`，前者表示消息的目的地，后者是消息的内容。另一个重要的属性 messageId 则是唯一表示一条消息。`AsyncMessage` 则用于消息服务，同时也是服务端返回 `Acknowledge` 的基类。调用失败的时候，BlazeDS 使用 `ErrorMessage` 封装错误消息。`CommandMessage` 用于传输内部命令，如连接和断开，登录和注销，订阅和取消订阅。

`Producer` 和 `Consumer` 用于消息服务，前者是消息的生产者，用于发布，后者是消息的消费者用于订阅。二者都有属性 `destionation` 和 `subtopic`，`destination` 指定了消息的远程目的，`subtopic` 则限定了消息的类型。使用`Consumer.subscribe` 来订阅消息，一旦订阅，就可以接受 `Producer` 发布的消息，接收消息有两种方法：主动接收和被动接收，如果使用实时或者轮询`Channel`，那么一旦有新消息，`Consumer`会收到 `acknowledge`事件通知，否则需要调用 `Consumer.receive` 来接收消息，不需要接收消息，使用`Consumer.unsubscribe`取消对消息的订阅。

下面看一下一个典型的 Flex 客户端是如何发起一次调用的，伪代码大致如下

```actionscript
var ro:RemoteObject = new RemoteObject(destination);
ro.endpoint = "http://localhost:8080/JavaServer/messagebroker/amf";
var op:AbstractOperation= getOperation(ro,method);
var faultHandler:function = defaultFaultHandler;
var errorFn:Function = function(event:FaultEvent):void {faultHandler.call(...);}
var resultFn:Function = functin(event:ResultEvent):void {callback.call(callback,event);}
op.addEventListener(FaultEvent.FAULT, errorFn);
op.addEventListener(ResultEvent.RESULT, resultFn);
var token:AsyncToken = op.send();
token.addResponder(new mx.rpc.AsyncResponder(resultFn,faultHandler));
```

当使用`RemoteObject` 作为远程调用入口的时候，首先需要指明 `destination`，在内部，我们一般定义为远程的`ServiceName`对应到Spring的BeanName，Java端会自行解析；设置远程的 `Endpoint` 也就是服务提供者的监听端点；调用的方法，也就是后台对应Java类的public方法，Flex端会对应到一个`AbstractOperation`实例；整个通信框架全部都是异步的，因此数据的处理和异常处理全部采用回调方法；最终方法的调用使用 `op.send()` 来触发，实现被定义在`mx.rpc.remoting.mxml.Operation` 的 send 方法，调用后会返回一个 `AsyncToken`，在其中添加`AsyncResponder` 也可以进行回调事件的处理。

```actionscript
override public function send(... args:Array):AsyncToken
    {
        if (Concurrency.SINGLE == concurrency && activeCalls.hasActiveCalls())
        {
            var token:AsyncToken = new AsyncToken(null);
			var message:String = resourceManager.getString(
				"rpc", "pendingCallExists");
            var fault:Fault = new Fault("ConcurrencyError", message);
            var faultEvent:FaultEvent = FaultEvent.createEvent(fault, token);
            new AsyncDispatcher(dispatchRpcEvent, [faultEvent], 10);
            return token;
        }
        // We delay endpoint initialization until now because MXML codegen may set
        // the destination attribute after the endpoint and will clear out the
        // channelSet.
        if (asyncRequest.channelSet == null && remoteObject.endpoint != null)
        {
            remoteObject.mx_internal::initEndpoint();
        }
        return super.send.apply(null, args);
}
```

`mx.rpc.remoting.Operation:send` 方法

```actionscript
override public function send(... args:Array):AsyncToken
{
    if (!args || (args.length == 0 && this.arguments))
    {
        if (this.arguments is Array)
        {
            args = this.arguments as Array;
        }
        else
        {
            args = [];
            for (var i:int = 0; i < argumentNames.length; ++i)
            {
                args[i] = this.arguments[argumentNames[i]];
            }
        }
    }
    var message:RemotingMessage = new RemotingMessage();
    message.operation = name;
    message.body = args;
    message.source = RemoteObject(service).source;
    return invoke(message);
}
```

消息组装好了，对应 `RemotingMessage`，消息体就是调用的参数，`operation`变量对应方法的名称。之后调用 `AbstractInvoker` 的 `invoke` 方法：

```actionscript
mx_internal function invoke(message:IMessage, token:AsyncToken = null) : AsyncToken
{
    if (token == null)
        token = new AsyncToken(message);
    else
        token.setMessage(message);
    activeCalls.addCall(message.messageId, token);
    var fault:Fault;
    try
    {
        //asyncRequest.invoke(message, new AsyncResponder(resultHandler, faultHandler, token));
        asyncRequest.invoke(message, new Responder(resultHandler, faultHandler));
        dispatchRpcEvent(InvokeEvent.createEvent(token, message));
    }
    catch(e:MessagingError)
    {
        _log.warn(e.toString());
        var errorText:String = resourceManager.getString(
            "rpc", "cannotConnectToDestination" ,
            [ asyncRequest.destination ]);
        fault = new Fault("InvokeFailed", e.toString(), errorText);
        new AsyncDispatcher(dispatchRpcEvent, [FaultEvent.createEvent(fault, token, message)], 10);
    }
    catch(e2:Error)
    {
        _log.warn(e2.toString());
        fault = new Fault("InvokeFailed", e2.message);
        new AsyncDispatcher(dispatchRpcEvent, [FaultEvent.createEvent(fault, token, message)], 10);
    }
    return token;
}
```

1. `activeCalls.addCall(message.messageId, token)`; 将`AysncToken` 放入一个活动调用队列中，这样就可以取消尚未返回的调用
2. `asyncRequest.invoke(message, new Responder(resultHandler, faultHandler))`; 将调用委托给 `AsyncRequest` 对象，并注册回调函数
3. `new AsyncDispatcher(dispatchRpcEvent, [FaultEvent.createEvent(fault, token, message)], 10)`; 延迟10毫秒发出`FaultEvent` 延迟是为了可以调用
`Operation` 的 `send` 并得到 `AsyncToken` 再注册监听。这样可以保证在发生异常的时候，能够注册到监听，进行相应的处理。

`operation` 把实际的工作交给了 `AsyncRequest` [作为 `AbstractInvoke`r 的一个属性],它创建并初始化了一个 `ChanelSet` 的实例，然后把委托工作交给 `ChannelSet`。
`AsyncRequest` 使用其基类 `MessageAgent` 创建和实例化 `ChannelSet`，并将自身实例和消息作为参数调用 `ChannelSet` 的`send`方法，`ChannelSet` 查找可用的 `Channel`并使用 `Channel` 的 `send` 方法。

```actionscript
public function send(agent:MessageAgent, message:IMessage):void {
    if (connected) {
        // Filter out any commands to trigger connection establishment, and ack them locally.
        if ((message is CommandMessage) && (CommandMessage(message).operation == CommandMessage.TRIGGER_CONNECT_OPERATION)) {
            var ack:AcknowledgeMessage = new AcknowledgeMessage();
            ack.clientId = agent.clientId;
            ack.correlationId = message.messageId;
            agent.acknowledge(ack, message);
            return;
        }
        // If this ChannelSet targets a clustered destination, request the
        // endpoint URIs for the cluster.
        if (!_hasRequestedClusterEndpoints && clustered) {
            var msg:CommandMessage = new CommandMessage();
            // Fetch failover URIs for the correct destination.
            if (agent is AuthenticationAgent) {
                msg.destination = initialDestinationId;
            }
            else {
                msg.destination = agent.destination;
            }
            msg.operation = CommandMessage.CLUSTER_REQUEST_OPERATION;
            _currentChannel.sendClusterRequest( new ClusterMessageResponder(msg, this ));
            _hasRequestedClusterEndpoints = true;
        }
        _currentChannel.send(agent, message);
    }
    else {
        // Filter out duplicate messages here while waiting for the underlying Channel to connect.
        if (_pendingMessages[message] == null) {
            _pendingMessages[message] = true;
            _pendingSends.push( new PendingSend(agent, message));
        }
        if (!_connecting) {
            if ((_currentChannel == null) || (_currentChannelIndex == -1))
                hunt();
            if (_currentChannel is NetConnectionChannel) {
                // Insert a slight delay in case we've hunted to a
                // NetConnection channel that doesn't allow a reconnect
                // within the same frame as a disconnect.
                if (_reconnectTimer == null) {
                    _reconnectTimer = new Timer(1, 1);
                    _reconnectTimer.addEventListener(TimerEvent.TIMER, reconnectChannel);
                    _reconnectTimer.start();
                }
            } else // No need to wait with other channel types.
            {
                connectChannel();
            }
        }
    }
}
```

1. `ChannelSet` 首先判断其状态是否为已经连接
2. 如果还没有连接，从 `ChannelSet` 保护的 `Channel` 列表中按顺序取出一个`Channel`，并设置为当前 `Channel`，否则直接返回当前 `Channel`，调用的设置代码为 `hunt`

```actionscript
private function hunt():Boolean {
	if (_channels.length == 0) {
		var message:String = resourceManager.getString(
			"messaging", "noAvailableChannels" );
		throw new NoChannelAvailableError(message);
	}
	// Advance to next channel, and reset to beginning if all Channels in the set
	// have been attempted.
	if (++_currentChannelIndex >= _channels.length) {
		_currentChannelIndex = -1;
		return false ;
	}
	// If we've advanced past the first channel, indicate that we're hunting.
	if (_currentChannelIndex > 0)
		_hunting = true;
	// Set current channel.
	if (configured) {
		if (_channels[_currentChannelIndex] != null) {
			_currentChannel = _channels[_currentChannelIndex];
		}
		else
		{
			_currentChannel = ServerConfig.getChannel(_channelIds[
									_currentChannelIndex], _clustered);
			_currentChannel.setCredentials(_credentials);
			_channels[_currentChannelIndex] = _currentChannel;
		}
	} else {
		_currentChannel = _channels[_currentChannelIndex];
	}
	// Ensure that the current channel is assigned failover URIs it if was lazily instantiated.
	if ((_channelFailoverURIs != null) && (_channelFailoverURIs[_currentChannel.id] != null))
		_currentChannel.failoverURIs = _channelFailoverURIs[_currentChannel.id];
	return true ;
}
```

3. 使用当前 `Channel` 发送 ping 命令，如果失败重复步骤2
4. 最后设置 `ChannelSet` 为已经连接。

我们使用的 RPC 组件是`RemoteObject`，因此 `AMFChannel` 的`send`方法将会被`AsyncRequest`间接调用，`AMFChannel` 使用 `NetConnection` 传输消息，`NetConnection` 使用post的方式向URL地址发送请求，进入Web工程，返回结果则会调用 agent 里面注册的各种回调函数进行处理。

整个 Flex-BlazeDS 通信的前端，全部是基于 Actionscript 构建，运行在 AVM 上。并且除了 `AMFChannel` 和后台 BlazeDS 交互之外，Flex端也有支持 SOAP 协议的 WebService ，和完全构建在 HTTP 上的 HTTPService，分别可以用来构建 webservice 客户端和 REST 风格的客户端。

##### Web 应用程序

接着讲一下Web应用程序的处理，主要讲 BlazeDS 的应用

![balzeDs-Java](/postsimg/rpc/balzedsjav-1540893022-22598.png)

Web 服务器接收到客户端 POST 的请求后，将此请求转发给注册在 Web 服务器的`MessageBrokerServlet`，`MessageBrokerServlet` 是 BlazeDS 用于接收 HTTP 请求的常驻 Web 服务器代理，它接收到 HTTP 请求后，根据请求的 URL 特性查找对应的`Endpoint`。并将请求转发给匹配的 Endpoint。

客户端发出消息的是 `AMFChannel`，`MessageBrokerServlet` 会将消息发送给 `AMFEndpoint`处理，`AMFEndpoint` 的方法 `createFilterChain`

```java
protected AMFFilter createFilterChain() {
    AMFFilter serializationFilter = new SerializationFilter(getLogCategory());
    AMFFilter batchFilter = new BatchProcessFilter();
    AMFFilter sessionFilter = new SessionFilter();
    AMFFilter envelopeFilter = new LegacyFilter(this);
    AMFFilter messageBrokerFilter = new MessageBrokerFilter(this);
    serializationFilter.setNext(batchFilter);
    batchFilter.setNext(sessionFilter);
    sessionFilter.setNext(envelopeFilter);
    envelopeFilter.setNext(messageBrokerFilter);
    return serializationFilter;
}
```

可以看出，这个方法没有特殊行为，只是组装了一个责任链

1. serializationFilter 负责从前端传过来的消息流中反序列化出消息对象，它位于整个责任链的链首，因为从前端传过来的消息流中的消息是二进制的，必须将其反序列化。
才能得到Java对象。后续的操作也就可以使用这个对象。
2. batchFilter 用于将批量消息装换为单个调用；
3. sessionFilter追加SessionID到URL中，以便Flex应用中可以使用ServletSession
4.  enelopeFilter 检查消息格式，用于兼容Flex1.5的消息
5. messageBrokerFilter是最后的处理环节，负责调用业务逻辑。

`AMFEndpoint` 类在启动时组装此责任链，然后在 service 方法中调用它，这些行为在 `BaseHTTPEndpoint` 中规定。

`MessageBrokerFilter` 是负责整个责任链和关系代码最重要的一环。看一下在Flex 中调用 sayHello() 的时序图。

![blazeds](/postsimg/rpc/blazeds-1540952088-28535.png)

1. 当 Flex 前端将消息传递到服务器时，第一个接收到消息的是注册在 Servlet 容器中的 `MessageBrokerServlet`，它默认将截获所有匹配 `*/messagebroker/*` 的 URL 请求，被激活后`MessageBrokerServlet` 使用请求的 URL 向`MessageBroker` 查询匹配的`Endpoint`。

2. MessageBroker 根据 URL 的特征，循环所有已注册的Endpoint，并返回匹配到的AMFEndpoint。

3. MessageBrokerServlet 在查询到合适的AMFEndPoint后，将消息转发给AMFEndPoint的service方法，

4. AMFEndPoint 激活其在启动时创建的责任链，最后，位于责任链末端的MessageBrokerFilter的invoke方法被调用。

5. MessageBrokerFilter在对消息进行简单处理之后，又调用AMFEndPoint的serviceMessage。

6. AMFEndPoint 根据不同的消息类型做出不同的相应，在本例中消息是普通的AsyncMessage ，因此AMFEndPoint会调用MessageBroker的routeMessageToService。

7. MessageBroker 根据消息中包含的Destination 信息找到对应的Service，根据配置，找到的将会是一个RemoteService的实例，这里的Service是BlazeDS提供的位于MessageBroker和业务逻辑质检的桥，并非业务服务层。

8. RemoteService在调用业务逻辑后，返回业务逻辑的执行结果，在本例中RemoteService会使用 JavaAdapter 调用HelloWorldFacade的sayHello(String name)，并返回字符串。

9. MessageBroker得到RemoteService返回的结果后，将结果包装成为消息

10. AMFEndPoint 继续向调用者返回响应消息

11. MessageBrokerFilter 将响应消息写入 HTTP 的 response对象。

12. MessageBrokerServlet完成本次请求的响应，将控制权交给servlet，容器负责将响应内容返回到前端浏览器。完成请求和响应在后端的全部过程。

####  Spring Remoting

公司的产品覆盖整个供应链套件，后台的系统也有大量的交互，基本采用的是 WebService 和 Spring + Hessian 混合的方式，WebService 应用在异构系统，主要是 PDA 等终端和外部例如 SAP ERP、Oracle 财务系统的对接。内部系统因为绝大多数采用 Spring 框架，因此采用基于 Spring + Hessian 的 RPC 架构。WebService的技术并没有太多可以分析，主要采用 CXF+Spring。这两组方案，都可以看做是 Spring Remoting 的实现。

Spring Remoting 使用“Service Accessor”和“Service Exporter”组合对应前面典型 RPC 架构的调用方 Stub 和 提供方 Stub。

![springremoting](/postsimg/rpc/springremo-1540980026-4516.png)

具象一点来说，Service Exporter 主要将远程服务对象发布，使用远程接口协议（RMI、HTTP、SOAP 等等）接收服务请求，然后对请求内容进行解析（un-marshaling 或者叫做 decode）,利用解析后的请求来调用本地的远程服务对象，调用完成之后，将结果重新编码（marshaling 或者 encode）再返回给请求的客户端。Service Accessor 则刚好相反，将请求编码后发送，并解析返回的响应。

Spring Remoting 主要的工作，在于巧妙地结合 ProxyFactoryBean 技术，自动生成远程调用代理，调用方只需要了解远程服务的接口。调用的细节，包括通信、异常处理等全部交给调用代理来实现，整个过程就和使用一个本地的服务一样，实现“透明的远程调用”，当然这里和本地调用还是存在一定的区别，本地调用时按照引用传参，而“透明的远程调用”显然是按值传参的，这里只需要稍微注意即可。

这里使用 Spring Boot 的例子做一下简单的使用说明一下 Spring Remoting 对于 RPC 调用的封装模式

服务端的 Exporter 代码为：

```java

    @Bean
    CabBookingService bookingService() {
        return new CabBookingServiceImpl();
    }

    @Bean(name = "/booking")
    RemoteExporter hessianService(CabBookingService service) {
        HessianServiceExporter exporter = new HessianServiceExporter();
        exporter.setService(bookingService());
        exporter.setServiceInterface(CabBookingService.class);
        return exporter;
    }

    @Bean
    TourWS tourWS(){
        return new TourWSImpl();
    }

    @Bean
    SimpleJaxWsServiceExporter tourWs(){
        SimpleJaxWsServiceExporter exporter = new SimpleJaxWsServiceExporter();
        exporter.setBaseAddress("http://localhost:8081/");
        return exporter;
    }
```

客户端的 Accessor 代码为

```java

    @Bean
    public HessianProxyFactoryBean hessianInvoker() {
        HessianProxyFactoryBean invoker = new HessianProxyFactoryBean();
        invoker.setServiceUrl("http://localhost:9091/booking");
        invoker.setServiceInterface(CabBookingService.class);
        return invoker;
    }

    @Bean
    public JaxWsPortProxyFactoryBean jaxWsInvoker() throws MalformedURLException {
        JaxWsPortProxyFactoryBean invoker = new JaxWsPortProxyFactoryBean();
        try {
            invoker.setWsdlDocumentUrl(new URL("http://localhost:8081/TourWS?wsdl"));
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
        invoker.setServiceName("TourWS");
        invoker.setPortName("TourWSImplPort");
        invoker.setNamespaceUri("http://rpc.demo.example.com/");
        invoker.setServiceInterface(TourWS.class);
        return invoker;
    }
```

首先看一下 Service Exporter 的实现策略

以 `HessianServiceExporter` 为例，它本身实现了 Spring MVC 模块的 `HttpRequestHandler`，声明Bean的时候，使用 "/booking" 作为BeanName。使用这样格式的BeanName ，Spring 容器会在将其注册在 `BeanNameUrlHandlerMapping` 中的 `handlerMap` 中。这样可以通过URL访问到 `HessianServiceExporter`。

![hessian](/postsimg/rpc/hessian-1541065821-6330.png)

1. 首先通过 `HandlerMapping`（在这里就是 `BeanNameUrlHandlerMapping`） 找到对应的 handler（实际上就是注册到Spring 容器中的 `HessianServiceExporter`），按照Spring MVC 的规范，会封装为 `HandlerExecutionChain` 以便执行拦截器等。
2. 处理完拦截器后，根据 handler ，获取 `HandlerAdapter`，因为 `HessianServiceExporter` 实现了 `HttpRequestHandler` 所以对应的适配器就是 `HttpRequestHandlerAdapter`
3. `HessianServiceExporter` 继承 `HessianExporte`r，并且在Bean初始化完成的时候，会实例化一个 `HessianSkeleton` 实例，`this.skeleton = new HessianSkeleton(getProxyForService(), getServiceInterface())`，`getProxyForService()`方法，是Spring 的 `RemoteExporter` 提供的方法，主要作用是生成对应服务实例的代理类，对应到本例就是 `CabBookingServiceImpl`，这种做法便于添加统一的拦截器，比如`RemoteInvocationTraceInterceptor`等可以用于远程调用时候的追踪，或者增加一些安全认证功能。
4. 触发 `HessianServiceExporter` 的 `handleRequest` 方法，由于Hessian是基于二进制的压缩协议，入参和出参使用的是request和response对应的流。首先校验请求流的Hessian的版本和数据基本格式。接着使用输入输出流作为参数调用 `skeleton.invoke(in, out);`
5. `HessianSkeleton` 首先从输入流中按照Hessian协议解码出需要调用的方法和调用的参数，然后通过反射机制来调用的目标方法（实际上调用的是代理对象），最终将返回结果使用 Hessian 编码，写入到 response对应的输出流中，完成调用。

接着看一下 Service Accessor 的实现

以 `HessianProxyFactoryBean` 为例，熟悉Spring的同学很容易就知道这是一个 FactoryBean，生成的 Bean 是一个代理对象。在我们的场景中，不难想象，需要生成的是一个访问Hessian服务端的客户端。在 Hessian 官方文档中给出的使用样例是

```java
    String url = "http://www.caucho.com/hessian/test/basic";
    HessianProxyFactory factory = new HessianProxyFactory();
    Basic basic = (Basic) factory.create(Basic.class, url);
    System.out.println("Hello: " + basic.hello());
```

这样的使用方式，如果我们采用正常的 Spring 声明 Bean 的方式，其实很难实现，因为你只有 Basic 接口、而 Basic 的实现需要通过 `HessianProxyFactory` 来生成，这两者很难通过 XML 描述或者类似 JavaBean 的声明方式联系在一起。而FactoryBean 就是为了解决这类问题和设计的。

> `FactoryBean` 是一个生成 Bean 的 Bean，主要的作用是允许我们自定义一些比较复杂的 Bean 的实例化，或者增加一些额外的逻辑。`FactoryBean` 在进行第三方框架和 Spring 的集成的时候使用较多，因为第三方框架的对象初始化往往比较复杂，使用 Spring 的 XML 配置配置来实现比较繁琐；此外如果要进行一些额外的拓展，例如增加 AOP 逻辑等，那么使用 `FactoryBean` 生成一个代理对象，就是一个很好的选择。
>
>如果从Spring 全局的角度来看，关于 Factory 模式，在DDD中，其主要的目的是为了解决复杂的实体的生成。Spring 的 BeanFactory 正是如此，复杂的 IOC 注入、AOP 增强，都是通过Spring 容器这个大型的Bean加工厂来生产的。FactoryBean的引入，则完善这个架构，提供了用户可以自定义的“Factory”来初始化、增强对应的 Object，当然在Spring 容器里面，你首先要是一个Bean，所以可以叫做 Object-Factory-Bean。

![hessianProxyFactoryBean](/postsimg/rpc/hessianpro-1541121467-10188.png)

1. 首先在 `HessianProxyFactoryBean` 中组合一个 `HessianProxyFactory` 实例，可以通过声明 setter 方法的方式，设置参数，例如超时、版本、认证信息等等。
2. 在属性设置完成之后，使用 `HessianProxyFactory#create` 生成 `HessianProxy` 访问代理，`this.hessianProxy = createHessianProxy(this.proxyFactory);`
3. 生成的 `hessianProxy` 其实已经可以直接提供给用户使用但是这个对象实际上是 Hessian 的代理对象，这样用户需要处理 Hessian 的 Exception，为了提供一致性的 `Remoting Accessor` 体验，于是在最外层又封装了一层代理，`this.serviceProxy = new ProxyFactory(getServiceInterface(), this).getProxy(getBeanClassLoader());`，因为HessianProxyFactoryBean 也实现了 `MethodInterceptor`接口，因此其也可以作为一个AOP的增强，具体的逻辑在 `HessianClientInterceptor` 的 invoke 方法中，主要就是调用 `hessianProxy` 完成请求，并且处理异常。
4. 最终层层代理到 `HessianProxy`，其将要访问的参数编码成为 Hessian 格式，并使用 `HessianConnection` 通过 HTTP 协议发送，请求最终会到达我们之前的讲到的 `Remoting Exporter`。
5. 当远端执行完成，从 `HessianConnection` 的 `InputStream` 中，可以得到响应的字节流。根据所调用的方法的返回类型，将响应字节流解码成对应的类型，返回给调用的客户端，完成调用。

##### 总结

Flex+BlazeDS+Java 的整个交互，实际上是构建在 HTTP 协议之上的 RPC 通信架构，通信的目标是运行在浏览器上的客户端和运行在数据中心的服务器，传输的内容采用 AMF 进行进行二进制压缩，严格意义上来说并不算系统之间的通信RPC，而是前后台通信架构。但是整个结构上非常符合 RPC 的标准套路。在对于 Channel 的抽象和拓展、后台服务定义和查找、异步调用上做得都很出色，整个框架也非常易用。但是很明显的，整套架构缺乏服务治理的功能，也缺少从架构层面对分布式应用的支持，语言上也只是支持 ActionScript3 和 Java 之间的通信，如果要拓展更多的语言，则不能使用 AMF 协议则需要进行一定的改造。但是在其出现的年代，SOA技术的实现还主要是 WebService 和 ESB 为主流，服务治理的工作还是 ESB 的最大卖点。作为通信框架而言，Flex + BlazeDS 设计精良，代码架构也很优雅，现在依然有借鉴和参考的价值。

>这套框架，是我接触和维护的第一套开发框架，现在除了少数的旧项目维护，基本已经没有再用到了。回想起来，对于重业务逻辑的企业内部系统来说，整套框架提供了很好的开发效率和用户体验的平衡：Flex MXML 的组件化特点和 Flex Builder 开发工具，对于我们以表单和列表操作为主的UI来说，开发速度非常快；同时，我们基于 RemoteObject 封装了一套全异步的远程调用工具，相比那时候刚刚起步的Ajax技术，更加易用和稳定；ActionScript3.0 的可调试、强类型，对于我们水平层次不齐的开发团队也大有裨益。同时，浏览器兼容性、富客户端体验、RIA桌面支持等在做技术选型的时候都是加分项。
>
>不过随着更多互联网技术的兴起，Flex 显得愈发笨重。公司也在逐步向互联网化的转型，在实现一个物流协同平台的时候，我们意识到使用 Flex 的页面访问相比 Ajax 技术实现的轻量级前端过于缓慢，大量的前端缓存加载会让用户消耗大量的事件等待首屏加载，虽然之后可以实现良好的富用户体验，但是并非每一个用户都会像企业内部用户一样耐心等待。
>
>后来随着乔布斯对 Flash 的批评，在移动端 Flash 逐渐失去地位，Adobe 基于 Flash构建的跨平台应用蓝图也宣告终结。随后，Flex 和 BlazeDS 被捐赠给 Apache 基金会。（写这篇文章的时候 2018年10月份，我又上去看了一下，版本停留在 4.7.3 已经一年多没有更新了）
>
>同时Flex人员在招聘难度上较大，基本上我们都需要自行培养，尽管整体技术很易用，但是很多人对学习一门可能出了公司很难再找到新工作的技术并非很感兴趣。基于上面的种种原因，公司也基本放弃了对这套架构的研发投入，组织团队开发了基于 jQuery 等更加常用的 JavaScript 技术栈进行产品的更迭，整个应用架构也从原来的伪分布式逐步向分布式 SOA 架构迁移，对于通信架构也需要重新作出选型。

应用之间的 RPC 框架从 Nelson 的论文提出，到 Spring Remoting 中的 Exporter 和 Accessor 抽象，本质上并没有不同。Remoting 更多的是集成已有的成熟框架，利用 Spring 强大的拓展能力，将其融合，提供一致性的使用。同时我们也看到了使用动态代理在应对复杂性方便的优势，在面向接口编程中，如果实现类是一个动态代理而非一个具体的类，往往会带来更强的拓展性。

在实际使用上，我们并非为每一个实现接口都什么对应的 Bean，而是通过自定义标签的方式，进一步简化了接口暴露和接口使用。同时也拓展接口暴露方式，通过对请求进行包装和自定义请求解析器，也可以支持 REST 风格的 HTTP 调用。

最大的问题还是在于缺乏原生的分布式支持、服务治理功能的缺失。在生产环境中，我们通过部署高可用的负载均衡器（Nginx/F5）来进行负载均衡、流量控制。但是在动态服务发现、注册、动态负载均衡、服务状态、链路跟踪等等功能的需求，还是让我们有点捉襟见肘，同时高并发的请求都通过 SLB 来进行中转，在性能上也不是最优。内部开发了类似 ELK 的日志追踪系统弥补了部分不足，但是整体的框架对公司服务化架构的要求而言还是显得过于羸弱。

![entire](/postsimg/rpc/entire-1541139003-2266.png)