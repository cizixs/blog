---
layout: post
title: "kubernetes 简介：调度器和调度算法"
excerpt: "scheduler 是 kubernetes 的调度器，主要的任务是把定义的 pod 分配到集群的节点上。听起来非常简单，但有很多要考虑的问题。"
categories: blog
tags: [kubernetes, container, scheduler]
comments: true
share: true
---

## 简介

scheduler 是 kubernetes 的调度器，主要的任务是把定义的 pod 分配到集群的节点上。听起来非常简单，但有很多要考虑的问题：

- 公平：如何保证每个节点都能被分配资源
- 资源高效利用：集群所有资源最大化被使用
- 效率：调度的性能要好，能够尽快地对大批量的 pod 完成调度工作
- 灵活：允许用户根据自己的需求控制调度的逻辑

sheduler 是作为单独的程序运行的，启动之后会一直坚挺 API Server，获取 `PodSpec.NodeName` 为空的 pod，对每个 pod 都会创建一个 binding，表明该 pod 应该放到哪个节点上。

## 参数说明

Scheduler 的参数是相对因为比较少的，因为它做的事情比较清晰，而且需要配置的地方比较少。下面是常见的参数列表和解释（不同版本 kubernetes 提供的参数可能会有出入，请以实际为准）：

参数    |   意思    |   默认值
--- |   --- |   ---
--address   |   监听地址    |   "0.0.0.0"   
--port  |   调度器监听的端口    |   10251
 --algorithm-provider   |   提供调度算法的对象    |   "DefaultProvider"
 --master   |   kubernetes API Server 的 HTTP API 地址  |   
 --profiling    |   是否开启 profiling，开启后可以在 `host:port/debug/pprof` 访问 profile 信息 |   true
 --scheduler-name   |   调度器名称，用来唯一确定该调度器  |   "default-scheduler"   
 --kube-api-burst   |   和 API Server 通信的时候最大 burst 值 | 100
 --kube-api-qps     |   和 API Server 通信的时候 QPS 值    |   50  
 --log_dir      |   日志保存的目录  |   
 --policy-config-file   |   json 配置文件，用来指定调度器的 filters 和 priorities，可以参考 `examples/scheduler-policy-config.json` 文件   |   

## 安装和使用

scheduler 的安装比较简单，它重要的参数就是 master 的 ip 地址，也是唯一必须指定的地址：

```bash
/opt/kubernetes/bin/kube-scheduler \
    --logtostderr=false \
    --v=4 \
    --log_dir=/var/log/kubernetes \
    --master=172.17.8.100:8080
```

运行之后，我们可以创建 pod 来测试。前一篇文章我们讲到需要手动指定 pod 要放到哪台 node 上，这次就不需要我们来做了，因为调度系统帮我们自动完成了这个步骤。来看看配置文件：

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx-server
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

这里我们没有指定 pod 的 `nodeName` 属性，创建看看：

``` bash
# kubectl create -f nginx-log.yml
```

等一段时间，可以使用 `kubectl describe pod` 来查看 pod 的信息，在输出的最后部分的事件列表中，我们可以看到第一条就是调度信息。下面这条记录说明了调度的时间，使用的调度器（`default-scheduler`），以及调度的最终结果（`172.17.8.101`）：

```bash
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath                   Type            Reason          Message
  ---------     --------        -----   ----                    -------------                   --------        ------          -------
  11m           11m             1       {default-scheduler }                                    Normal          Scheduled       Successfully assigned nginx-server to 172.17.8.101
  11m           11m             1       {kubelet 172.17.8.101}  spec.containers{nginx-server}   Normal          Pulling         pulling image "172.16.1.41:5000/library/nginx"
......

```

**NOTE**：这篇文章运行的实体都是 pod，但是实际上 ReplicationController（1.5 之后的版本更名为 ReplicaSet） 在实际更常用，我们会在后面的文章介绍。

## 调度过程

调度分为几个部分：首先是过滤掉不满足条件的节点，这个过程称为 `predicate`；然后对通过的节点按照优先级排序，这个是 `priority`；最后从中选择优先级最高的节点。如果中间任何一步骤有错误，就直接返回错误。

predicate 有一系列的算法可以使用：

- `PodFitsResources`：节点上剩余的资源是否大于 pod 请求的资源
- `PodFitsHost`：如果 pod 指定了 NodeName，检查节点名称是否和 NodeName 匹配
- `PodFitsHostPorts`：节点上已经使用的 port 是否和 pod 申请的 port 冲突
- `PodSelectorMatches`：过滤掉和 pod 指定的 label 不匹配的节点
- `NoDiskConflict`：已经 mount 的 volume 和 pod 指定的 volume 不冲突，除非它们都是只读

如果在 predicate 过程中没有合适的节点，pod 会一直在 `pending` 状态，不断重试调度，直到有节点满足条件。经过这个步骤，如果有多个节点满足条件，就继续 priorities 过程：
按照优先级大小对节点排序。

优先级由一系列键值对组成，键是该优先级项的名称，值是它的权重（该项的重要性）。这些优先级选项包括：

- `LeastRequestedPriority`：通过计算 CPU 和 Memory 的使用率来决定权重，使用率越低权重越高。换句话说，这个优先级指标倾向于资源使用比例更低的节点
- `BalancedResourceAllocation`：节点上 CPU 和 Memory 使用率越接近，权重越高。这个应该和上面的一起使用，不应该单独使用
- `ImageLocalityPriority`：倾向于已经有要使用镜像的节点，镜像总大小值越大，权重越高

通过算法对所有的优先级项目和权重进行计算，得出最终的结果。

## 自定义调度器

除了 kubernetes 自带的调度器，你也可以编写自己的调度器。通过 `spec:schedulername` 参数指定调度器的名字，可以为 pod 选择某个调度器进行调度。

比如下面的 pod 选择 `my-scheduler` 进行调度，而不是默认的 `default-scheduler`：

```bash
apiVersion: v1
kind: Pod
metadata:
  name: annotation-second-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulername: my-scheduler
  containers:
  - name: pod-with-second-annotation-container
    image: gcr.io/google_containers/pause:2.0
```

调度器的编写请参考 kubernetes 默认调度器的实现，最核心的内容就是读取 apiserver 中 pod 的值，根据特定的算法找到合适的 node，然后把调度结果会写到 apiserver。

## 参考资料

- [Improving Kubernetes Scheduler Performance](https://coreos.com/blog/improving-kubernetes-scheduler-performance.html)
- [kubernetes Scheduler 文档](https://github.com/kubernetes/kubernetes/blob/master/docs/devel/scheduler.md)
- [Comparison of Container Schedulers](https://medium.com/@ArmandGrillet/comparison-of-container-schedulers-c427f4f7421#.sw4kxxdp9)
- [How does Kubernetes scheduler work](http://carmark.github.io/2015/12/21/How-does-Kubernetes-scheduler-work/)
- [多个 schedulers 的配置和使用方法](http://kubernetes.io/docs/admin/multiple-schedulers/)
- [Scheduling Your Kubernetes Pods With Elixir](https://deis.com/blog/2016/scheduling-your-kubernetes-pods-with-elixir/)
- [Kubernetes from the ground up: the scheduler](http://kamalmarhubi.com/blog/2015/11/17/kubernetes-from-the-ground-up-the-scheduler/)
