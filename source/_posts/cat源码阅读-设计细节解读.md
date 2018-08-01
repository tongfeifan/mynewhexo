---
title: cat源码阅读(二)-设计细节解读
date: 2018-07-08 13:29:17
tags: [CAT, 监控, 源码阅读，架构设计]
---

## 背景

CAT作为一款开源监控平台已经被许多大小公司所采用，如携程、陆金所、拍拍贷等大中型公司，以及其他各种创业公司，CAT在各大小公司一直稳定运行。这源于其优秀的细节设计。故本文将会分析CAT的一些设计细节，这些设计实现了较高的可用性，且较为复杂，值得一说。CAT主要的组成部分有： 客户端、服务端、存储几部分，下面分别对这几部分的设计做一些分析。

客户端的主要设计点有调用链和路由，服务端的主要设计点有数据分析和报表（上文已讨论），存储的主要设计点在于文件存储模块。

本文主要分析以下几点设计：

1. [客户端路由](#客户端路由)
2. [分布式调用链设计](#分布式调用链设计)
3. [文件存储设计](#文件存储设计)

## 客户端路由

### 概述

CAT的设计中，客户端路由是由客户端和服务器端共同实现的。通过服务端的参与，CAT实现了可在portal中动态配置服务地址；通过服务端分配`domain`与路由（此处`domain`可理解为应用名称），实现了相同`domain`的客户端消息都上报到同一台server端实例中，避免了server端互相通信等复杂设计。

### 设计图

![routerDesign](/images/routerDesign.png)

### 详述

在路由过程，我将其分为四个环节，分别是启动->获取路由->domain路由计算->构造/更新Channel。

其中启动环节只在客户端启动的时候会触发。之后三个环节除客户端启动外，也会持续地轮询执行。

- 启动(上图1，2)：在客户端启动时，`ClientConfigManager`会从配置文件中读取serverAddress，将serverAddress交给`ChannelManager`。
- 获取路由（上图3）：`ChannelManager`在获取服务器地址后，会像该服务器的router接口发送请求，请求体中携带客户端名`domain`
- domain路由计算（上图4）：服务端在接受路由计算请求后，便会从XML或者DB中读取ip-port列表，然后会依据每个domain的hashcode分配两个ip-port给该domain（上图中代码所示）
- 构造/更新Channel（上图5）:服务端将分配的两个ip-port返回给客户端，由此客户端就得到了两个ip-port，客户端会将第一个ip-port作为主上报服务器，将第二个ip-port作为备份，用作failover，由此构造一个channal。

经过以上四个步骤，客户端就完成了路由计算过程，`TcpSocketSender`只需获取channel，持续上报消息即可。

## 分布式调用链设计

CAT可以通过定制化客户端而实现对分布式调用链的跟踪。CAT的分布式调用链在服务端主要是在展示的时候得到体现，所以可以通过阅读CAT的logview部分，探索CAT的分布式调用链设计机制。另外CAT服务端会对消息的type属性的某些默认值进行特殊处理，而分布式调用链的处理，正式通过对type为`RemoteCall`的Event消息进行处理，来组合分布式调用链。

### logview展示消息树

![CatMessageTreeShow](/images/CatMessageTreeShow.png)

通过阅读展示代码，并梳理过程得到上图，我们发现CAT的分布式调用链关键是一个type为`RemoteCall`的Event，并且可以模拟构造远程调用所形成的messageTree，如下图。

![messageTreeRemoteCall](/images/messageTreeRemoteCall.png)

### 如何使用分布式调用链监控

在CAT的官方文档中，几乎没有提及如何使用CAT的分布式调用链监控。然后CAT是提供了这一功能的，并且有较好的支持。我们通过阅读源码，理解了在服务端CAT是如何展示分布式调用链的，那么在客户端自然也就可以配合实现，从而开始使用CAT的分布式调用链追踪。

### 调用方客户端
在拍拍贷内部，对okhttp进行了Aspectj编程，达成了使用okhttp包进行远程调用时的监控。核心代码段如下：
```java
PropertyContext context = new PropertyContext();
Cat.logRemoteCallClient(context);

request = request.newBuilder()
        .addHeader(CatConstants.HTTP_HEADER_ROOT_MESSAGE_ID, context.getProperty(Cat.Context.ROOT))
        .addHeader(CatConstants.HTTP_HEADER_PARENT_MESSAGE_ID, context.getProperty(Cat.Context.PARENT))
        .addHeader(CatConstants.HTTP_HEADER_CHILD_MESSAGE_ID, context.getProperty(Cat.Context.CHILD))
        .addHeader("X-PPD-CAT-APP", Cat.getManager().getDomain())
        .build();

ret = proceed(request);
```

通过代码可以看到，在请求中，将生成的childMessageId(可通过createMessageId方法生成)，放入header中，传递给了被调用方

### 被调用方客户端
而对http被调用方，则在filter获取header，并设置当前messageTree的messageId。

```java
PropertyContext propertyContext = new PropertyContext();
propertyContext.addProperty(Cat.Context.ROOT, request.getHeader(CatConstants.HTTP_HEADER_ROOT_MESSAGE_ID));
propertyContext.addProperty(Cat.Context.PARENT, request.getHeader(CatConstants.HTTP_HEADER_PARENT_MESSAGE_ID));
propertyContext.addProperty(Cat.Context.CHILD, request.getHeader(CatConstants.HTTP_HEADER_CHILD_MESSAGE_ID));
Cat.logRemoteCallServer(propertyContext);
```

```java
public static void logRemoteCallServer(Context ctx) {
	MessageTree tree = Cat.getManager().getThreadLocalMessageTree();
	String messageId = ctx.getProperty(Context.CHILD);
	String rootId = ctx.getProperty(Context.ROOT);
	String parentId = ctx.getProperty(Context.PARENT);

	if (messageId != null) {
		tree.setMessageId(messageId);
	}
	if (parentId != null) {
		tree.setParentMessageId(parentId);
	}
	if (rootId != null) {
		tree.setRootMessageId(rootId);
	}
}
```

如此一来，CAT的远程调用的两个MessageTree就串起来了，这也达成了分布式调用追踪的目的。


## 文件存储设计

### 概述

CAT为了高可用、高效地进行消息存储，设计了一套文件存储结构。此文件将同一小时内上报的消息归集在一个目录进行存储,存储文件各目录内分为索引文件、数据文件两部分。系统以IP、消息上报时间小时窗口、消息树序列号（1小时内序列号）便可快速定位某一个消息。

### 设计图

![CatfileDesign](/images/CatfileDesign.png)

### 详述

消息存储文件分为索引文件、数据文件。索引文件路径结构为`/{yyyymmdd}/{hh24}/{domain}-{serverIP}.idex2`，数据文件路径结构与索引文件相比仅后缀不同，其结构为`/{yyyymmdd}/{hh24}/{domain}-{serverIP}.dat2`。

索引文件分为heander段和segment段，其中segment段便是有多个索引块顺序存储构成；而header段内每一条数据由64位存储，前32位为IP，后32位为segmentIdex。header段通过此方式维护了消息IP、segmentIndex与索引块的关系。segmentIndex可由消息树序列号(messageSeq)计算得知。因此一个消息可通过其IP、消息树序列号快速定位索引。

索引块中每一条数据，由64位存储，前40位为blockAddress, 后24位为block内offset的，通过blockAddress和blockOffset可快速定位一条messageTree在数据文件中的位置。

数据文件由多个block构成，每个block内的messageTree都上报自相同domain的客户端。

