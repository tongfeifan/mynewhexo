---
title: cat源码阅读(一)-架构设计解读
date: 2018-07-06 13:29:17
tags: [CAT, 监控, 源码阅读, 架构设计]
---

CAT(Central Application Tracking)是基于Java开发的实时应用监控平台，为大众点评网提供了全面的监控服务和决策支持。作为大众点评网基础监控组件，它已经在中间件框架（MVC框架，RPC框架，数据库框架，缓存框架等）中得到广泛应用，为点评各业务线提供系统的性能指标、健康状况、基础告警等。

目前拍拍贷已经有500多个应用接入了CAT监控。而我在接手CAT项目过程中，较为全面地阅读了CAT的源码，故在此分享。

## CAT数据结构设计

### Mesaage

CAT中的消息上报时的基本单位抽象类称为消息(Message)，`Message`可分为四大类，实现代码分别继承自`Message`抽象类。分别是`Event`、`Transaction`、`Heartbeat`、`Metric`。关于`Message`类的各子类含义，官方有个领域建模图可表示：
![domainModel](/images/domainModel.png)

### 消息树

所有消息都可被组织进消息树（MessageTree），`Transaction`类型的消息可作为消息树节点，而其他消息只可作为消息树的叶子节点。也就是`Transaction`是一个可嵌套的递归结构。结构可表示如下图：
![MessageTreeStruct](/images/MessageTreeStruct.png)

有时候以时序图的方式来表示也许会更清晰：
![MessageTreeTimeline](/images/MessageTreeTimeline.png)

我们可以简单理解为，若消息B为`Transaction`消息A的子节点，那么消息B就发生在A的执行期间。

`MessageTree`类中的属性`messageId`表示`MessageTree`本身，其构成为：{domain}-{ip}-{timestamp}-{自增index}；另外还有两个属性，分别是`parentMessageId`, `rootMessageId`。`parentMessageId`表示开启这一段代码执行的调用方；`rootMessageId`表示这一段代码的调用方的最终源头。上图中，MessageTree2的`parentMessageId`和`rootMessageId`皆为`MessageTree1`的messageId。这两个属性在之后CAT的调用链分析与分布式调用链分析中发挥了关键作用。

## CAT设计

CAT系统划分为三个模块，分别是客户端、服务端、以及portal。在实际部署时，会分为`cat-client.jar`和`cat-home.war`。其中，`cat-client.jar`为客户端jar包，服务端与portal的功能都集成在`cat-home.war`中。

### CAT客户端设计

CAT的客户端主要用作监控埋点，可简单分为埋点、上报两部分。
其设计图如下：
![CATclientDesign](/images/CATclientDesign.png)

由于`Conext`维护在ThreadLocal中，因此每一个thread都会拥有一份自己的`Context`，`Context`中会维护一个stack用来存储transaction，当新transaction开启时入栈，结束时出栈。当栈内压入第一个transaction时开始构造`MessageTree`；栈空时认为一个`MessageTree`结束，此时将该`MessageTree`发送给待发送队列。

埋点部分流程如下所示：
![CATclient](/images/CATclient.png)

上图可分为开始阶段和结束阶段：
- 一场CAT的埋点始于`newTransaction`方法：
    - 此时会判断是否已经有`context`，`context`由ThreadLocal维护，记录当前上下文，如IP、线程号 等信息，而每个`context`中会维护一个`stack`。换句话说，CAT的一场埋点是线程隔离的。
    - 在判断并创建`context`之后，会将新建transaction压入栈中，若此时栈为空，那么会将当前transaction记录在messageTree中，作为当前messageTree的根节点。
    - 之后在该线程中的所有新`Transaction`、`Event`、`Heartbeat`等都会作为stack顶端transaction的子节点被记录，另外新`Transaction`操作会被压入栈中。
- 而埋点的结束，是一个出栈的过程，当最终栈为空时，则认为这段埋点监控结束，将构造的整个messageTree进行flush，即分配给队列，准备上报。

上报过程则是从队列中取得messageTree，进行编码，通过tcpSocket上报至服务端。

### CAT服务端设计

CAT的服务端大致设计，如下
![CATprocess](/images/CATprocess.png)

从图中可以看到，cat客户端的埋点数据是通过tcpsocket上报到服务端的。服务端的一切服务是通过监听上报消息开始启动。当服务端的tcpsocketReceiver收到消息以及解码后，便会创建或者寻找一个当前时间的period（在每一小时的开始会创建一个period。），如下代码所示。

```java
public void consume(MessageTree tree) {
    String domain = tree.getDomain();
    String ip = tree.getIpAddress();

    if (!m_blackListManager.isBlack(domain, ip)) {
        long timestamp = tree.getMessage().getTimestamp();
        Period period = m_periodManager.findPeriod(timestamp);

        if (period != null) {
            period.distribute(tree);
        } else {
            m_serverStateManager.addNetworkTimeError(1);
        }
    } else {
        m_black++;

        if (m_black % CatConstants.SUCCESS_COUNT == 0) {
            Cat.logEvent("Discard", domain);
        }
    }
}
```

在这之后，period会为每一个analyzer维护一个period task，period将消息分配至每一个period task的队列中。

```java
public void distribute(MessageTree tree) {
    m_serverStateManager.addMessageTotal(tree.getDomain(), 1);
    boolean success = true;
    String domain = tree.getDomain();

    for (Entry<String, List<PeriodTask>> entry : m_tasks.entrySet()) {
        List<PeriodTask> tasks = entry.getValue();
        int length = tasks.size();
        int index = 0;
        boolean manyTasks = length > 1;

        if (manyTasks) {
            index = Math.abs(domain.hashCode()) % length;
        }
        PeriodTask task = tasks.get(index);
        boolean enqueue = task.enqueue(tree);

        if (enqueue == false) {
            if (manyTasks) {
                task = tasks.get((index + 1) % length);
                enqueue = task.enqueue(tree);

                if (enqueue == false) {
                    success = false;
                }
            } else {
                success = false;
            }
        }
    }

    if (!success) {
        m_serverStateManager.addMessageTotalLoss(tree.getDomain(), 1);
    }
}
```

而后analyzer从队列中消费消息并处理。analyzer可以理解一个数据分析器，用来处理收到的Message，并聚合生成报表。

```java
public void analyze(MessageQueue queue) {
    while (!isTimeout() && isActive()) {
        MessageTree tree = queue.poll();
        if (tree != null) {
            try {
                process(tree);
            } catch (Throwable e) {
                m_errors++;
                if (m_errors == 1 || m_errors % 10000 == 0) {
                    Cat.logError(e);
                }
            }
        }
    }
    while (true) {
        MessageTree tree = queue.poll();
        if (tree != null) {
            try {
                process(tree);
            } catch (Throwable e) {
                m_errors++;
                if (m_errors == 1 || m_errors % 10000 == 0) {
                    Cat.logError(e);
                }
            }
        } else {
            break;
        }
    }
}

```

在CAT的设计中，所有`analyzer`可以都拥有自己的queue。因此，所有`analyzer`可以相互独立的各自分析数据。另外CAT的设计中，对于`analyzer`的扩展做得非常方便，只需要继承其`AbstractMessageAnalyzer`抽象类，并实现`process`方法，便可添加自定义的analyzer。
拍拍贷也通过该方式扩展统计了许多内部指标。

其中DumpAnalyzer较为特殊，它不进行任何数据处理，只用作原始数据存储。（在原始数据的文件存储这部分，CAT设计了自己的索引与存储结构，其设计较为精妙，会在后续文章中进行分析。）

CAT中一般都是以1小时作为一个时间窗口，各个analyzer会对当前小时窗口内的数据进行聚合处理，生成report表，并且在当前小时窗口内维护在内存中，在当前小时窗口结束后，将其落地写入文件或DB中。
