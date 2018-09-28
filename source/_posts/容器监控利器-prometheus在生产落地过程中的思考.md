---
title: 容器监控利器-prometheus在生产落地过程中的思考
date: 2018-09-28 12:20:15 
tags: [容器, 监控, prometheus, 落地]
---

## 背景

随着容器技术的不断推进，容器云也在同时不断发展，随之而来需要解决的问题便是容器及容器云的监控。目前容器监控的主流方案为prometheus。本文是在prometheus落地过程中的一些思考。


## 社区活跃

目前容器监控的主流方案为prometheus。作为CFCN社区的第二款产品（第一款为kubernets），prometheus在社区拥有极高的支持度，容器界主流成品几乎都做到了对prometheus的原生支持。并且由于prometheus采用了pull模式，对于其他产品来说，接入成本极低，只需为prometheus提供一个pull metrics的接口即可；由此我认为开源界接入prometheus的比例会不断升高。

## 功能丰富

### 查询
prometheus server端实际上是一个时间序列数据库，并且提供了查询引擎promQL，promQL本身提供的查询能力较为丰富。

PromQL定义了四种数据类型，分别是instance vector(可理解为时间序列上的某一个点)，range vector(可理解为时间序列上的一段)，scalar(浮点数)，string(字符串，目前尚未使用)。PromQL便是通过操作符以及聚合函数操作这四种数据类型构成了查询表达式。

PromQL所提供的操作符，一般用于操作instance vector。除了常规的加减乘除之外，promQL还提供了Aggregation operators，如
```
sum (calculate sum over dimensions)
min (select minimum over dimensions)
max (select maximum over dimensions)
avg (calculate the average over dimensions)
stddev (calculate population standard deviation over dimensions)
stdvar (calculate population standard variance over dimensions)
count (count number of elements in the vector)
count_values (count number of elements with the same value)
bottomk (smallest k elements by sample value)
topk (largest k elements by sample value)
quantile (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions)
```

另外PromQL到目前为止(2.4版本)还提供了30余种function，可对数据进行花样查询和聚合。

### 报警

prometheus体系中包含了报警管理- Alertmanager，Alertmanager被作为独立组件发布，独立部署。在部署Alertmanager后，只需在prometheus配置文件中配置Alertmanager的ip、port即可完成对接。Alertmanager的报警配置，依赖于promQL，同时配置报警模板以及notify通道即可使用。Alertermanager实现了报警的分组、抑制与安静功能，可以在配置之后，配合以上功能灵活地零时操作报警。目前已经集成的notify通道有：
```
DingTalk
IRC Bot
JIRAlert
Phabricator / Maniphest
prom2teams: forwards notifications to Microsoft Teams
SMS: supports multiple providers
Telegram bot
```

### 采集

prometheus在采集方面，充分发挥了其社区活跃的优势，在容器层面，google的容器信息采集器`cadvisor`对prometheus提供了非常好的支持，同时kubernets项目也原生提供了prometheus的采集接口。在机器层面的信息采集，prometheus提供了机器层面的采集器`node_exporter`。对于主流语言java, python等prometheus也提供了相应的sdk client可以方便地采集应用层与业务层的监控数据。

## 第三方集成

Prometheus系统自身极度专注于监控，非常纯粹。然而在落地过程中实际存在着各种场景与需求，此时便需要结合一些第三方集成或定制化开发来达到我们的目的。

### 配置
对于prometheus的配置，是基于yml配置文件进行的，所以每当需要操作prometheus的时候需要操作大量yml文件，关于此可考虑自行开发一个配置文件管理系统即可，也有对于的开源方案可参考：[promgen](https://github.com/line/promgen)

### 服务发现

每当有新的监控target上线时，若是使用静态抓取配置，则需要每次修改配置项，如此操作比较麻烦，比较幸运的是prometheus集成了对主流服务发现中心/注册中心的集成，配置上对应的服务发现中心即可。

然而如果使用的服务发现中心未被prometheus集成，也无需担心，prometheus支持基于文件的服务发现，即将target配置在json文件中。prometheus会定期读取json文件，更新target列表。那么基于此，我们只需针对所使用的服务发现中心，定期拉取服务列表，更新至json文件就可实现自定义的服务发现。

### 高可用/长期存储

Prometheus作为一款监控产品，其server的初始定位只是用于短期存储，默认存储时间为15天，可在启动时通过`--storage.tsdb.retention`进行设定。目前prometheus自身并没有过多地考虑如何进行长期存储，但是prometheus提供了remote read 和remote write的，通过该功能可实现长期存储。

关于prometheus的长期存储方案，我认为[Thanos](https://github.com/improbable-eng/thanos)项目是一个较为优秀的解决方案。

在长期存储备份方面，他采用了[slidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar#solution)模式进行设计，通过与prometheus部署在一起的sidecar，将prometheus的存储文件备份到object storage（目前只支持gae, s3，可自行扩展）。另外Thanos可部署一个store组件， 可用来高效查询object storage。sidecar和store组件都会提供相同的store API，另外Thanos可部署一个query组件，通过store API查询sidecar或者store组件。另外query组件对外提供了与prometheus完全相同的查询dashboad与api，通过这一套方式Thanos实现了prometheus的长期存储，以及对长期存储数据和近期数据的相同查询。

另外结合Thanos，可达到prometheus的高可用部署，只需部署两台完全相同的prometheus server，便可基本达到高可用。两者唯一区别即在external_labels配置replica，那么在部署thanos query的时候添加 flag ` --query.replica-label replica`即可实现在查询时候的dereplica。


