---
title: k8s中的https证书与权限
date: 2018-10-31 18:50:52
tags: [k8s, https, 容器云]
---

## 背景

k8s通过api-server对外提供服务，而k8s系统作为集群的调度控制中心如果没有权限控制，其安全性会有很大隐患，所以在k8s中使用了https进行认证，同时引入了RBAC作为官方推荐的权限授权和控制方式(非https端口没有权限认证)。

## https认证

### 证书

为了方便后面的讨论，我们先回顾一下https的一些基础知识。https认证分为单向认证和双向认证。双向认证不仅需要服务端提供证书，客户端也需要提供证书，在两者交换证书互相确认身份之后交换密钥，然后开始加密通讯。而单向认证则不需要客户端提供证书，只需要服务端自证身份即可。k8s中使用的以双向认证为主。

- **CA根证书：** CA根证书有CA中心签发，一般作为根证书，可用于签发其他证书，在k8s中可用于签发api-server的服务端证书和各个组件的客户端证书。
- **csr：** 证书签名请求（Certificate Signing Request）在申请证书时候提交的文件。包含自身信息（国家、地区、机构、名称），k8s会读取其中的机构、名称分别对应k8s中的group和user；包含主机信息（作为服务器时，接收的https请求地址与此不一致，则会拒绝）
- **客户端/服务端证书：** 该证书来自CA中心或者CA根证书根据csr内容签发，包含了公钥信息，在https握手阶段自证身份以及交换公钥。
- **私钥文件：** 私钥文件同样来自于CA中心或者CA根证书，客户端/服务端证书在被签发时，会同时颁发一个私钥文件。私钥文件用于https握手阶段交换公钥之后的通讯加密方式协商阶段。

更详细的https基础知识，请自行google。

### 证书生成工具

目前在k8s部署过程中，主流使用的证书生成工具是cfssl和cfssljson。

### 证书配置

关于k8s的证书配置，网上相关的教程很多，在此不做赘述。只提部分较为关键的点。

- 前文已经提到csr中的名称、组织（CN, O）在k8s中作为user和group使用，k8s中预设了部分角色绑定，如：clusterrole中cluster-admin已经绑定group system:master；clusterrole中的controller, schedule也已经绑定相应的user。所以在为k8s原生组件申请证书时，直接填写对应的CN或者O即可。[官网罗列了预设的角色绑定](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles)。k8s的角色绑定关系查询语句`kubectl get clusterrolebindings -o wide`。
- k8s中预设的角色`system:kubelet-api-admin`觉有kubelet-api的权限，但是k8s没有为其绑定任何user, group，若kubelet关闭了匿名访问，开启认证后，使用类似`kubectl logs`等交互命令时可能会得到类似报错：`Error from server (Forbidden): Forbidden (user=kubernetes, verb=get, resource=nodes, subresource=proxy) ( pods/log busybox0-n9nbw)`此时解决问题的关键是将user与该角色进行绑定：`kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes`。
- 前文已经提到在csr中会包含主机信息，若地址与接收的http请求地址不一致时，会拒绝该请求。在为api-server编写主机信息时，经常会有遗漏，需要注意不单单要写本机host，cluster-ip也写在里面。

## kubelet部署时的认证

部分教程在部署kubelet时，没有为kubelet组件制作证书与密钥，如果学习者看到的教程没有对这一段进行解释，也许会感到疑惑。实际上kubelet的证书最终是通过kube-controller-manager签发给kubelet（kube-controller-manager 需要配置 --cluster-signing-cert-file 和 --cluster-signing-key-file 参数）。过程如下：

1. kubeadm 通过kubeadm token create 命令创建token，并通过kubectl config命令将该token写入bootstrap文件。
2. kubelet在启动的时候会检查config文件，如果没有config文件，则会携带bootstrap中的token想api-server发送csr请求。
3. 在api-server中approve该csr请求后，kube-controller-manager会签发证书与密钥给kubelete，为kubelet生成config文件，并集成证书和密钥。此时，kubelete获得了自己证书和密钥，进入了https常规交互过程。

采用了该方式之后，免去了制作csr文件的繁琐步骤。另外，随着kubeadm工具的发展，以上认证步骤会变得越来越简单，甚至覆盖所有组件的部署。
