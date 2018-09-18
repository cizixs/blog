---
layout: post
title: "kubernetes 简介：controller manager 和自动扩容"
excerpt: "kubernetes 提供了一个非常重要的特性，它能保证某个 pod 一定会有特定数量的副本在运行，这个功能是 controller manager 实现的。"
categories: blog
tags: [ kubernetes, container, ]
comments: true
share: true
---

## controller manager 简介

在配置了[调度器](http://cizixs.com/2017/03/10/kubernetes-intro-scheduler)之后，我们的 kubernetes 集群已经能够实现计算资源（容器或者 pod）自动选择合适的节点去运行，看起来很不错，但是 kubernetes 还提供另外一个很重要的特性——保证特定副本的 pod 在运行。为什么呢？

因为在大规模的集群中，总会出现各种各样想不到的错误，某个运行的 pod 会因为各种问题无法运行（代码有 bug、内存泄漏、磁盘满了、权限问题甚至是宿主机故障），而我们希望 kubernetes 能提供一定的高可用和自动化功能，它能够检测出有 pod 异常退出并对它进行重启，或者启动新的 pod 替代它，保证我们的服务不受异常 pod 的影响。

其实除了 pod 之外，往外扩展还有其他需要自动化处理的东西。比如自动检测管理的节点是否正常，如果发现节点异常就不要继续往它上面调度资源了，而且需要把节点上运行的 pod 转移到其他节点。kubernetes master 上运行的 controller manager 会运行多个不同的 manager 分别负责不同的功能，这篇文章我们只关注 **Replication Manager**：它保证在任何时间某个 pod 都有特定数量的副本在运行。

replication manager 的出现有很多用处，上面提到了错误修复，另外一个重要的作用是提供服务的扩容，我们可以在访问量增长的时候增加副本的数量，实现服务的横向扩展。

## 参数

controller manager 主要的工作就是和 apiserver 通信，获取集群的特定信息，然后做出响应的反馈动作。它的启动配置参数很多，比较常用的参数包括：

参数    |   含义    |   默认值
---     |   ----    |   ---
--address=0.0.0.0   |   监听的 HTTP 地址    |  
--alsologtostderr   |   是否把日志打印到标准输出    | false
--cluster-name |   集群名，会添加到实体名称最前面    |  kubernetes
--controller-start-interval   |   启动 controller manager 之间的时间间隔  | 0
--deleting-pods-burst   |   node 不可用的时候，删除 pod 的 burst 值  |  1
--deleting-pods-qps     |   node 不可用的时候，删除 pod 的速率  |   0.1
--deployment-controller-sync-period |   同步 deployments 的频率周期 |   30s
--kube-api-burst    |   和 API Server 通信的 burst 值   |   30
--kube-api-qps   |   和 API Server 通信的 QPS 值     |   20
--kubeconfig    |   包含了认证和 master 地址信息的配置文件地址  |   
--log-dir   |   日志目录地址    |   
--master    |   master API 的地址，会覆盖 kubeconfig 里面 master API 的值   |
--node-monitor-grace-period |   node 被认为不可使用的失联时间，必须是 node 向 API Server 注册/更新状态频率的整数倍 | 40s
--port  |   controller manger 监听的 HTTP 端口     |    10252
--profiling |   是否开启 profiling。开启后，可以在 `host:port/debug/pprof` 看到  | true
--v     |   日志级别    |   0

## 配置和使用

下载或者编译 kube-controller-manager 的二进制文件，然后如下运行它（可以使用 systemd 或者其他软件进行管理）：

```bash
# /opt/kubernetes/bin/kube-controller-manager \
    --logtostderr=false \
    --v=4 \
    --master=172.17.8.100:8080 \
    --log_dir=/var/log/kubernetes \
    --root_ca_file=/srv/kubernetes/ca.crt \
    --service-account-private-key-file=/srv/kubernetes/server.key
```

启动 `kube-controller-manager` 之后，就可以创建 `replication controller` 自动监控和管理 pod 了。和 Pod 的配置文件一样，我们也需要编写对应的 `ReplicationController` yaml 文件，沿用我们之前的例子，它的配置如下：

```
# cat nginx-log-rc.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-server
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
        env: dev
    spec:
      containers:
      - name: nginx-server
        image: 172.16.1.41:5000/nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /var/log/nginx
          name: nginx-logs
      - name: log-output
        image: 172.16.1.41:5000/busybox
        command:
        - bin/sh
        args: [-c, 'tail -f /logdir/access.log']
        volumeMounts:
        - mountPath: /logdir
          name: nginx-logs
      volumes:
      - name: nginx-logs
        emptyDir: {}
```

注意 `replicas: 2` 这一行，它告诉 kubernetes：在集群中运行两个 pod。这就像领导（应用的创建者）布置的任务，kubernetes 会想办法做到，而使用者不用关心它到底是怎么实现的。

使用 `kubectl create -f nginx_rc.yml` 可以创建 `ReplicationController`，命令成功执行后，可以使用 `kubectl get rc` 查看刚刚创建的 `rc`（`ReplicationController` 的简写） 资源：

```
# kubectl get rc
NAME           DESIRED   CURRENT   READY     AGE
nginx-server   2         2         1         9s
whoami         3         3         3         3h
[root@localhost ~]# kubectl describe rc nginx-server
Name:           nginx-server
Namespace:      default
Image(s):       172.16.1.41:5000/library/nginx,172.16.1.41:5000/library/busybox
Selector:       app=nginx
Labels:         app=nginx
                env=dev
Replicas:       2 current / 2 desired
Pods Status:    2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Volumes:
  nginx-logs:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
Events:
  FirstSeen     LastSeen        Count   From                            SubObjectPath   Type            Reason                  Message
  ---------     --------        -----   ----                            -------------   --------        ------                  -------
  14s           14s             1       {replication-controller }                       Normal          SuccessfulCreate        Created pod: nginx-server-4s7vw
  14s           14s             1       {replication-controller }                       Normal          SuccessfulCreate        Created pod: nginx-server-zl85w

[root@localhost ~]# kubectl get pods
NAME                 READY     STATUS    RESTARTS   AGE
nginx-server-4s7vw   2/2       Running   1          20s
nginx-server-zl85w   2/2       Running   0          20s

```

怎么验证 `rc` 可以保证特定 pod 副本运行的功能呢？ 如果你手动删除某个 pod `kubectl delete pod <podname>`，过一段时间可以看到又有一个新的 pod 创建并运行。当然如果运行的 pod 数量大于指定的数目，controller manager 也会自动删除多余的 pod，保证正确的副本数量。

## 扩容

在互联网时代，用户和流量在快速增长，后台的服务也需要应对随时变化的网络流量。云计算的一个重要任务就是能够自动伸缩——根据用户的流量来自动地增加或者减少服务的数量，实现高可用和高性能以及最优的资源利用率。有了 Replication Manager，kubernetes 也能够很简单地修改某个 pod 的副本数量。

```bash
[root@localhost ~]# kubectl scale --replicas=3 rc/nginx-server

[root@localhost ~]# kubectl get pods
NAME                 READY     STATUS    RESTARTS   AGE
nginx-server-4s7vw   2/2       Running   1          8m
nginx-server-xzqmt   2/2       Running   0          5m
nginx-server-zl85w   2/2       Running   0          8m
```

通过增加副本数来提供服务能力的前提是，多个副本能够提供无差别的服务，这就需要服务做到无状态。对于有状态的服务，还需要其他额外的配置，这里就不介绍了。

## 问题

如果 pod 被分配到不同的机器上，可以看到它们网络都是使用 docker 的默认 bridge 模式，也就是说它们的网络地址都是私有的，无法通信。比如，在我的环境中，两个 pod 的 ip 都是在 `172.17.0.0/16` 这个网段。这就导致一个问题：不同主机上的 pod 无法通信。kubernetes 本身并不提供网络解决方案，但它对网络有一个要求：所有的 pod 都能够互相通信，网络的配置可以选择 flannel，之前也[有一篇文章](http://cizixs.com/2016/06/15/flannel-overlay-network)介绍过。

另外一个问题是，我们怎么访问不同 pod 的服务？最简单的办法是我们从 kubernetes 中获取某个 rc 中所有 pod 的 ip 地址，然后在应用程序中和某个建立连接去使用它提供的服务。但是这个办法把所有的复杂度都转移给应用程序的编写，显然是不符合 kubernetes 这类集群管理的设计原则，它们本来就是为了让应用更容易部署和使用的。

其实这两个问题都和网络有关，我们会在下一篇文章讲讲 kubernetes 网络和服务（service）的概念。
