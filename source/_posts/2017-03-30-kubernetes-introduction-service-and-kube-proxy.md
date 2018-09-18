---
layout: post
title: "kubernetes 简介：service 和 kube-proxy 原理"
excerpt: "每个服务都一个固定的虚拟 ip，自动并且动态地绑定后面的 pod，所有的网络请求直接访问服务 ip，服务会自动向后端做转发。"
categories: blog
tags: [ kubernetes, container, network, load_balance, service]
comments: true
share: true
---


## 简介

在 kubernetes 集群中，网络是非常基础也非常重要的一部分。对于大规模的节点和容器来说，要保证网络的连通性、网络转发的高效，同时能做的 ip 和 port 自动化分配和管理，并让用户用直观简单的方式来访问需要的应用，这是需要复杂且细致设计的。

kubernetes 在这方面下了很大的功夫，它通过 `service`、`dns`、`ingress` 等概念，解决了服务发现、负载均衡的问题，也大大简化了用户的使用和配置。

这篇文章就讲解如何配置 kubernetes 的网络，最终从集群内部和集群外部都能访问应用。

## 跨主机网络配置：flannel

一直以来，kubernetes 并没有专门的网络模块负责网络配置，它需要用户在主机上已经配置好网络。kubernetes 对网络的要求是：容器之间（包括同一台主机上的容器，和不同主机的容器）可以互相通信，容器和集群中所有的节点也能直接通信。

至于具体的网络方案，用户可以自己选择，目前使用比较多的是 flannel，因为它比较简单，而且刚好满足 kubernetes 对网络的要求。我们会使用 flannel vxlan 模式，具体的配置我在博客之前[有文章介绍过](http://cizixs.com/2016/06/15/flannel-overlay-network)，这里不再赘述。

以后 kubernetes 网络的发展方向是希望通过插件的方式来集成不同的网络方案， [CNI](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/#cni) 就是这一努力的结果，flannel 也能够通过 CNI 插件的形式使用。

## kube-proxy 和 service

配置好网络之后，集群是什么情况呢？我们可以创建 pod，也能通过 ReplicationController 来创建特定副本的 pod（这是更推荐也是生产上要使用的方法，即使某个 rc 中只有一个 pod 实例）。可以从集群中获取每个 pod ip 地址，然后也能在集群内部直接通过 `podIP:Port` 来获取对应的服务。

但是还有一个问题：**pod 是经常变化的，每次更新 ip 地址都可能会发生变化**，如果直接访问容器 ip 的话，会有很大的问题。而且进行扩展的时候，rc 中会有新的 pod 创建出来，出现新的 ip 地址，我们需要一种更灵活的方式来访问 pod 的服务。

### Service 和 cluster IP

针对这个问题，kubernetes 的解决方案是“服务”（service），每个服务都一个固定的虚拟 ip（这个 ip 也被称为 cluster IP），自动并且动态地绑定后面的 pod，所有的网络请求直接访问服务 ip，服务会自动向后端做转发。Service 除了提供稳定的对外访问方式之外，还能起到负载均衡（Load Balance）的功能，自动把请求流量分布到后端所有的服务上，服务可以做到对客户透明地进行水平扩展（scale）。

![](https://coreos.com/kubernetes/docs/latest/img/service.svg)

而实现 service 这一功能的关键，就是 kube-proxy。kube-proxy 运行在每个节点上，监听 API Server 中服务对象的变化，通过管理 iptables 来实现网络的转发。

> **NOTE**: kube-proxy 要求 NODE 节点操作系统中要具备 /sys/module/br_netfilter 文件，而且还要设置 bridge-nf-call-iptables=1，如果不满足要求，那么 kube-proxy 只是将检查信息记录到日志中，kube-proxy 仍然会正常运行，但是这样通过 Kube-proxy 设置的某些 iptables 规则就不会工作。

kube-proxy 有两种实现 service 的方案：userspace 和 iptables

- userspace 是在用户空间监听一个端口，所有的 service 都转发到这个端口，然后 kube-proxy 在内部应用层对其进行转发。因为是在用户空间进行转发，所以效率也不高
- iptables 完全实现 iptables 来实现 service，是目前默认的方式，也是推荐的方式，效率很高（只有内核中 netfilter 一些损耗）。

这篇文章通过 iptables 模式运行 kube-proxy，后面的分析也是针对这个模式的，userspace 只是旧版本支持的模式，以后可能会放弃维护和支持。

### kube-proxy 参数介绍

kube-proxy 的功能相对简单一些，也比较独立，需要的配置并不是很多，比较常用的启动参数包括：

参数    |   含义    |   默认值
--- |   --- |   ---
--alsologtostderr   |   打印日志到标准输出  |   false
--bind-address  |   HTTP 监听地址   |   0.0.0.0
--cleanup-iptables  |   如果设置为 true，会清理 proxy 设置的 iptables 选项并退出    |   false
--healthz-bind-address  |   健康检查 HTTP API 监听端口  |   127.0.0.1
--healthz-port  |   健康检查端口    |   10249
--iptables-masquerade-bit   |   使用 iptables 进行 SNAT 的掩码长度  |   14
--iptables-sync-period  |   iptables 更新频率   |   30s
--kubeconfig    |   kubeconfig 配置文件地址 |   
--log-dir   |   日志文件目录/路径    |   
--masquerade-all    |   如果使用 iptables 模式，对所有流量进行 SNAT 处理    |   false
--master    |   kubernetes master API Server 地址   |   
--proxy-mode    |   代理模式，`userspace` 或者 `iptables`， 目前默认是 `iptables`，如果系统或者 iptables 版本不够新，会 fallback 到 userspace 模式   | iptables
--proxy-port-range  |   代理使用的端口范围， 格式为 `beginPort-endPort`，如果没有指定，会随机选择 | `0-0`
--udp-timeout   |   UDP 空连接 timeout 时间，只对 `userspace` 模式有用 |   250ms
--v   |   日志级别    |   0

`kube-proxy` 的工作模式可以通过 `--proxy-mode` 进行配置，可以选择 `userspace` 或者 `iptables`。

### 实例启动和测试

我们可以在终端上启动 `kube-proxy`，也可以使用诸如 `systemd` 这样的工具来管理它，比如下面就是一个简单的 `kube-proxy.service` 配置文件

```
[root@localhost]# cat /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy Service
Documentation=http://kubernetes.com
After=network.target
Wants=network.target

[Service]
Type=simple
EnvironmentFile=-/etc/sysconfig/kube-proxy
ExecStart=/usr/bin/kube-proxy \
    --master=http://172.17.8.100:8080 \
    --v=4 \
    --proxy-mode=iptables
TimeoutStartSec=0
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

为了方便测试，我们创建一个 rc，里面有三个 pod。这个 pod 运行的是 [`cizixs/whoami` 容器](https://github.com/cizixs/whoami)，它是一个简单的 HTTP 服务器，监听在 3000 端口，访问它会返回容器的 hostname。

```bash
[root@localhost ~]# cat whoami-rc.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: whoami
spec:
  replicas: 3
  selector:
    app: whoami
  template:
    metadata:
      name: whoami
      labels:
        app: whoami
        env: dev
    spec:
      containers:
      - name: whoami
        image: cizixs/whoami:v0.5
        ports:
        - containerPort: 3000
        env:
          - name: MESSAGE
            value: viola
```

我们为每个 pod 设置了两个 label：`app=whoami` 和 `env=dev`，这两个标签很重要，也是后面服务进行绑定 pod 的关键。

为了使用 service，我们还要定义另外一个文件，并通过 `kubectl create -f ./whoami-svc.yml` 来创建出来对象：

```bash
apiVersion: v1
kind: Service
metadata:
  labels:
    name: whoami
  name: whoami
spec:
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
  selector:
    app: whoami
    env: dev
```

其中 `selector` 告诉 kubernetes 这个 service 和后端哪些 pod 绑定在一起，这里包含的键值对会对所有 pod 的 `labels` 进行匹配，只要完全匹配，service 就会把 pod 作为后端。也就是说，service 和 rc 并不是对应的关系，一个 service 可能会使用多个 rc 管理的 pod 作为后端应用。

`ports` 字段指定服务的端口信息：

- `port`：虚拟 ip 要绑定的 port，每个 service 会创建出来一个虚拟 ip，通过访问 `vip:port` 就能获取服务的内容。这个 port 可以用户随机选取，因为每个服务都有自己的 vip，也不用担心冲突的情况
- `targetPort`：pod 中暴露出来的 port，这是运行的容器中具体暴露出来的端口，一定不能写错
- `protocol`：提供服务的协议类型，可以是 `TCP` 或者 `UDP`

创建之后可以列出 service ，发现我们创建的 service 已经分配了一个虚拟 ip (10.10.10.28)，这个虚拟 ip 地址是不会变化的（除非 service 被删除）。查看 service 的详情可以看到它的 endpoints 列出，对应了具体提供服务的 pod 地址和端口。

```bash
[root@localhost ~]# kubectl get svc
NAME         CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes   10.10.10.1    <none>        443/TCP    19d
whoami       10.10.10.28   <none>        3000/TCP   1d

[root@localhost ~]# kubectl describe svc whoami
Name:                   whoami
Namespace:              default
Labels:                 name=whoami
Selector:               app=whoami
Type:                   ClusterIP
IP:                     10.10.10.28
Port:                   <unset> 3000/TCP
Endpoints:              10.11.32.6:3000,10.13.192.4:3000,10.16.192.3:3000
Session Affinity:       None
No events.
```

默认的 service 类型是 `ClusterIP`，这个也可以从上面输出看出来。在这种情况下，只能从集群内部访问这个 IP，不能直接从集群外部访问服务。如果想对外提供服务，我们后面会讲解决方案。

测试一下，访问 service 服务的时候可以看到它会随机地访问后端的 pod，给出不同的返回：

```bash
[root@localhost ~]# curl http://10.10.10.28:3000
viola from whoami-8fpqp
[root@localhost ~]# curl http://10.10.10.28:3000
viola from whoami-c0x6h
[root@localhost ~]# curl http://10.10.10.28:3000
viola from whoami-8fpqp
[root@localhost ~]# curl http://10.10.10.28:3000
viola from whoami-dc9ds
```

默认情况下，服务会随机转发到可用的后端。如果希望保持会话（同一个 client 永远都转发到相同的 pod），可以把 `service.spec.sessionAffinity` 设置为 `ClientIP`。

**NOTE**: 需要注意的是，服务分配的 cluster IP 是一个虚拟 ip，如果你尝试 `ping` 这个 IP 会发现它没有任何响应，这也是刚接触 kubernetes service 的人经常会犯的错误。实际上，这个虚拟 IP 只有和它的 port 一起的时候才有作用，直接访问它，或者想访问该 IP 的其他端口都是徒劳。

### 外部能够访问的服务

上面创建的服务只能在集群内部访问，这在生产环境中还不能直接使用。如果希望有一个能直接对外使用的服务，可以使用 `NodePort` 或者 `LoadBalancer` 类型的 Service。我们先说说 `NodePort` ，它的意思是在所有 worker 节点上暴露一个端口，这样外部可以直接通过访问 `nodeIP:Port` 来访问应用。

我们先把刚才创建的服务删除：

```bash
[root@localhost ~]# kubectl delete rc whoami
replicationcontroller "whoami" deleted

[root@localhost ~]# kubectl delete svc whoami
service "whoami" deleted

[root@localhost ~]# kubectl get pods,svc,rc
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.10.10.1   <none>        443/TCP   14h
```

对我们原来的 `Service` 配置文件进行修改，把 `spec.type` 写成 `NodePort` 类型：

```bash
[root@localhost ~]# cat whoami-svc.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: whoami
  name: whoami
spec:
  ports:
    - port: 3000
      protocol: TCP
      # nodePort: 31000
  selector:
    app: whoami
  type: NodePort
```

因为我们的应用比较简单，只有一个端口。如果 pod 有多个端口，也可以在 `spec.ports`中继续添加，只有保证多个 port 之间不冲突就行。

重新创建 rc 和 svc：

```bash
[root@localhost ~]# kubectl create -f ./whoami-svc.yml
service "whoami" created
[root@localhost ~]# kubectl get rc,pods,svc
NAME        DESIRED   CURRENT   READY     AGE
rc/whoami   3         3         3         10s

NAME              READY     STATUS    RESTARTS   AGE
po/whoami-8zc3d   1/1       Running   0          10s
po/whoami-mc2fg   1/1       Running   0          10s
po/whoami-z6skj   1/1       Running   0          10s

NAME             CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
svc/kubernetes   10.10.10.1     <none>        443/TCP          14h
svc/whoami       10.10.10.163   <nodes>       3000:31647/TCP   7s
```

需要注意的是，因为我们没有指定 `nodePort` 的值，kubernetes 会自动给我们分配一个，比如这里的 `31647`（默认的取值范围是 30000-32767）。当然我们也可以删除配置中 `# nodePort: 31000` 的注释，这样会使用 `31000` 端口。

`nodePort` 类型的服务会在所有的 worker 节点（运行了 kube-proxy）上统一暴露出端口对外提供服务，也就是说外部可以任意选择一个节点进行访问。比如我本地集群有三个节点：`172.17.8.100`、`172.17.8.101` 和 `172.17.8.102`：

```bash
[root@localhost ~]# curl http://172.17.8.100:31647
viola from whoami-mc2fg
[root@localhost ~]# curl http://172.17.8.101:31647
viola from whoami-8zc3d
[root@localhost ~]# curl http://172.17.8.102:31647
viola from whoami-z6skj
```

有了 `nodePort`，用户可以通过外部的 Load Balance 或者路由器把流量转发到任意的节点，对外提供服务的同时，也可以做到负载均衡的效果。

`nodePort` 类型的服务并不影响原来虚拟 IP 的访问方式，内部节点依然可以通过 `vip:port` 的方式进行访问。

`LoadBalancer` 类型的服务需要公有云支持，如果你的集群部署在公有云（GCE、AWS等）可以考虑这种方式。

## service 原理解析

目前 kube-proxy 默认使用 iptables 模式，上述展现的 service 功能都是通过修改 iptables 实现的。

我们来看一下从主机上访问 `service:port` 的时候发生了什么（通过 `iptables-save` 命令打印出来当前机器上的 iptables 规则）。

所有发送出去的报文会进入 KUBE-SERVICES 进行处理

```bash
*nat
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

KUBE-SERVICES 每条规则对应了一个 service，它告诉继续进入到某个具体的 service chain 进行处理，比如这里的 `KUBE-SVC-OQCLJJ5GLLNFY3XB`

```bash
-A KUBE-SERVICES -d 10.10.10.28/32 -p tcp -m comment --comment "default/whoami: cluster IP" -m tcp --dport 3000 -j KUBE-SVC-OQCLJJ5GLLNFY3XB
```

更具体的 chain 中定义了怎么转发到对应 endpoint 的规则，比如我们的 rc 有三个 pods，这里也就会生成三个规则。这里利用了 iptables 随机和概率转发的功能

```bash
-A KUBE-SVC-OQCLJJ5GLLNFY3XB -m comment --comment "default/whoami:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-VN72UHNM6XOXLRPW
-A KUBE-SVC-OQCLJJ5GLLNFY3XB -m comment --comment "default/whoami:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-YXCSPWPTUFI5WI5Y
-A KUBE-SVC-OQCLJJ5GLLNFY3XB -m comment --comment "default/whoami:" -j KUBE-SEP-FN74S3YUBFMWHBLF
```

我们来看第一个 chain，这个 chain 有两个规则，第一个表示给报文打上 mark；第二个是进行 DNAT（修改报文的目的地址），转发到某个 pod 地址和端口。

```bash
-A KUBE-SEP-VN72UHNM6XOXLRPW -s 10.11.32.6/32 -m comment --comment "default/whoami:" -j KUBE-MARK-MASQ
-A KUBE-SEP-VN72UHNM6XOXLRPW -p tcp -m comment --comment "default/whoami:" -m tcp -j DNAT --to-destination 10.11.32.6:3000
```

因为地址是发送出去的，报文会根据路由规则进行处理，后续的报文就是通过 flannel 的网络路径发送出去的。

`nodePort` 类型的 service 原理也是类似的，在 `KUBE-SERVICES` chain 的最后，如果目标地址不是 VIP 则会通过 `KUBE-NODEPORTS` ：

```bash
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

而 `KUBE-NODEPORTS` chain 和 `KUBE-SERVICES` chain 其他规则一样，都是转发到更具体的 `service` chain，然后转发到某个 pod 上面。

```bash
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/whoami:" -m tcp --dport 31647 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/whoami:" -m tcp --dport 31647 -j KUBE-SVC-OQCLJJ5GLLNFY3XB
```

## 不足之处

看起来 service 是个完美的方案，可以解决服务访问的所有问题，但是 service 这个方案（iptables 模式）也有自己的缺点。

首先，如果转发的 pod 不能正常提供服务，它不会自动尝试另一个 pod，当然这个可以通过 `readiness probes` 来解决。每个 pod 都有一个健康检查的机制，当有 pod 健康状况有问题时，kube-proxy 会删除对应的转发规则。

另外，`nodePort` 类型的服务也无法添加 TLS 或者更复杂的报文路由机制。

## 参考资料

- [Kubernetes 1.2 如何使用 iptables](http://blog.csdn.net/horsefoot/article/details/51249161)
- [Kubernetes User Guide: Service](https://kubernetes.io/docs/user-guide/services/)
- [Kubernetes User Guide: Debugging Services](https://kubernetes.io/docs/user-guide/debugging-services/)
- [Kubernetes Services and Ingress Under X-ray](http://containerops.org/2017/01/30/kubernetes-services-and-ingress-under-x-ray/)
- [CoreOS documentation: Overview of a Service](https://coreos.com/kubernetes/docs/latest/services.html)
