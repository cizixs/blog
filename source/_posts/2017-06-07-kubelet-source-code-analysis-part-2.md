---
layout: post
title: "kubelet 源码分析：pod 新建流程"
excerpt: "在上一篇文章中，我们分析了 kubelet 是怎么从命令行进行解析参数、怎么根据配置初始化各种对象、以及最终怎么创建出来 `Kubelet`并运行的。这篇文章我们就接着分析，当有新的 pod 分配到该节点的时候，kubelet 是怎么处理的。"
categories: blog
tags: [kubernetes, kubelet, container, golang]
comments: true
share: true
---

在[上一篇文章](http://cizixs.com/2017/06/06/kubelet-source-code-analysis-part-1)中，我们分析了 kubelet 是怎么从命令行进行解析参数、怎么根据配置初始化各种对象、以及最终怎么创建出来 `Kubelet`并运行的。这篇文章我们就接着分析，当有新的 pod 分配到该节点的时候，kubelet 是怎么处理的。

## syncLoop

`syncLoop` 是 kubelet 的主循环方法，它从不同的管道（文件、URL 和 apiserver）监听变化，并把它们汇聚起来。当有新的变化发生时，它会调用对应的处理函数，保证 pod 处于期望的状态。如果 pod 没有变化，它也会定期保证所有的容器和最新的期望状态保持一致。这个方法是 for 循环，不会退出。

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	glog.Info("Starting kubelet main sync loop.")
	
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			glog.Infof("skipping pod synchronization - %v", rs)
			time.Sleep(5 * time.Second)
			continue
		}
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
	}
}
```

这里的代码主逻辑是 `for` 循环，不断调用 `syncLoopIteration` 方法。在此之前创建了两个定时器： `syncTicker` 和 `housekeepingTicker`，即使没有需要更新的 pod 配置，kubelet 也会定时去做同步和清理工作。如果在每次循环过程中出现比较严重的错误，kubelet 会记录到 `runtimeState` 中，遇到错误就等待 5 秒中继续循环。注意第二个参数变成了 `SyncHandler` 类型，这是一个 interface，定义了处理不同情况的接口，我们在在后面会看到它的具体方法。

我们继续看 `syncLoopIteration`，这个方法就是对多个管道做遍历，发现任何一个管道有消息就交给 handler 去处理。

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	kl.syncLoopMonitor.Store(kl.clock.Now())
	select {
	case u, open := <-configCh:
		switch u.Op {
		case kubetypes.ADD:
			glog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			glog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletiontimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			glog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			glog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			glog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			glog.Errorf("Kubelet does not support snapshot update")
		}

		// 收到消息之后就把对应的来源标记为 ready 状态
		kl.sourcesReady.AddSource(u.Source)

	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				glog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*api.Pod{pod})
			} else {
				glog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		glog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		kl.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				glog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			glog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*api.Pod{pod})
		}
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			glog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			glog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				glog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
	kl.syncLoopMonitor.Store(kl.clock.Now())
	return true
}
```

可以看到，它会从以下管道中获取消息：

- configCh：读取配置事件的管道，就是之前讲过的通过文件、URL 和 apiserver 汇聚起来的事件
- syncCh：定时器管道，每次隔一段事件去同步最新保存的 pod 状态
- houseKeepingCh：housekeeping 事件的管道，做 pod 清理工作
- plegCh：PLEG 状态，如果 pod 的状态发生改变（因为某些情况被杀死，被暂停等），kubelet 也要做处理
- livenessManager.Updates()：健康检查发现某个 pod 不可用，一般也要对它进行重启

需要注意的是， `switch-case` 语句从管道中读取数据的时候，不像一般情况下那样会从上到下按照顺序，只要任何管道中有数据，`switch` 就会选择执行对应的 `case` 语句。

每个管道的处理思路大同小异，我们只分析用户通过 apiserver 添加新 pod 的情况，也就是 `handler.HandlePodAdditions(u.Pods)` 这句话的处理逻辑。

## HandlePodAddtions

```go
func (kl *Kubelet) HandlePodAdditions(pods []*api.Pod) {
	start := kl.clock.Now()

	sort.Sort(sliceutils.PodsByCreationTime(pods))

	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		kl.podManager.AddPod(pod)

		if kubepod.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}
		......
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
		kl.probeManager.AddPod(pod)
	}
}
```
对于事件中的每个 pod，执行以下操作：

- 把所有的 pod 按照创建日期进行排序，保证最先创建的 pod 会最先被处理
- 把它加入到 `podManager` 中，因为 `podManager` 是 kubelet 的 source of truth，所有被管理的 pod 都要出现在里面。如果 `podManager` 中找不到某个 pod，就认为这个 pod 被删除了
- 如果是 mirror pod调用其单独的方法
- 验证 pod 是否能在该节点运行，如果不可以直接拒绝
- 把 pod 分配给给 worker 做异步处理
- 在 `probeManager` 中添加 pod，如果 pod 中定义了 readiness 和 liveness 健康检查，启动 goroutine 定期进行检测

这里可以看到 `podManger` 和 `probeManager` 发挥用处了，它们两个的具体实现都不复杂，感兴趣的读者可以自行阅读相关的代码。

pod 具体会被怎么处理呢？我们再来看 `dispatchWorker` 方法，它的作用就是根据 pod 把任务发送给特定的执行者 `podWorkers`：

```go
func (kl *Kubelet) dispatchWork(pod *api.Pod, syncType kubetypes.SyncPodType, mirrorPod *api.Pod, start time.Time) {
	if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			kl.statusManager.TerminatePod(pod)
		}
		return
	}
	// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
				metrics.PodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	// Note the number of containers for new pods.
	if syncType == kubetypes.SyncPodCreate {
		metrics.ContainersPerPodCount.Observe(float64(len(pod.Spec.Containers)))
	}
}
```

`dispatchWork` 主要工作就是把接收到的参数封装成 `UpdatePodOptions`，调用 `kl.podWorkers.UpdatePod` 方法。`podWorkers` 的代码在 `pkg/kubelet/pod_workers.go` 文件中，它通过 `podUpdates` 字典保存了一个字典，每个 pod 的 id 作为 key，而类型为 UpdatePodOptions 的管道作为 value 传递 pod 信息。

```go
func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
	pod := options.Pod
	uid := pod.UID
	var podUpdates chan UpdatePodOptions
	var exists bool

	p.podLock.Lock()
	defer p.podLock.Unlock()
	if podUpdates, exists = p.podUpdates[uid]; !exists {
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates

		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
	if !p.isWorking[pod.UID] {
		p.isWorking[pod.UID] = true
		podUpdates <- *options
	} else {
		update, found := p.lastUndeliveredWorkUpdate[pod.UID]
		if !found || update.UpdateType != kubetypes.SyncPodKill {
			p.lastUndeliveredWorkUpdate[pod.UID] = *options
		}
	}
}
```

`UpdatePod` 会先去检查 `podUpdates` 字典是否已经存在对应的 pod，因为这里的新建的 pod，所以会调用 `p.managePodLoop()` 方法作为 goroutine 运行更新工作。也就是说对于管理的每个 pod，`podWorkers` 都会启动一个 goroutine 在后台执行，除此之外，它还会更新 `podUpdate` 和 `isWorking`，填入新 pod 的信息，并往 `podUpdates` 管道中发送接收到的 pod 选项信息。

`managePodLoop` 的代码如下：

```go
func (p *podWorkers) managePodLoop(podUpdates <-chan UpdatePodOptions) {
	var lastSyncTime time.Time
	for update := range podUpdates {
		err := func() error {
			podUID := update.Pod.UID
			
			status, err := p.podCache.GetNewerThan(podUID, lastSyncTime)
			if err != nil {
				return err
			}
			err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			if err != nil {
				return err
			}
			return nil
		}()
		// notify the call-back function if the operation succeeded or not
		if update.OnCompleteFunc != nil {
			update.OnCompleteFunc(err)
		}
		if err != nil {
			glog.Errorf("Error syncing pod %s, skipping: %v", update.Pod.UID, err)
			p.recorder.Eventf(update.Pod, api.EventTypeWarning, events.FailedSync, "Error syncing pod, skipping: %v", err)
		}
		p.wrapUp(update.Pod.UID, err)
	}
}
```
`managePodLoop` 调用 `syncPodFn` 方法去同步 pod，`syncPodFn` 实际上就是 `kubelet.SyncPod`。

## SyncPod

`SyncPod` 的内容比较长，我们这里就不贴出它的代码了，它做的事情包括：

- 如果是删除 pod，立即执行并返回
- 检查 pod 是否能运行在本节点，主要是权限检查（是否能使用主机网络模式，是否可以以 privileged 权限运行等）。如果没有权限，就删除本地旧的 pod 并返回错误信息
- 如果是 static Pod，就创建或者更新对应的 mirrorPod
- 创建 pod 的数据目录，存放 volume 和 plugin 信息
- 如果定义了 PV，等待所有的 volume  mount 完成（volumeManager 会在后台做这些事情）
- 如果有 image secrets，去 apiserver 获取对应的 secrets 数据
- 调用 container runtime 的 SyncPod 方法，去实现真正的容器创建逻辑

这里所有的事情都和具体的容器没有关系，可以看做是提前做的准备工作。最重要的事情发生在 `kl.containerRuntime.SyncPod()` 里，也就是上面过程的最后一个步骤，它调 runtime 执行具体容器的创建，对于 docker 来说，具体的代码位于 `pkg/kubelet/dockertools/docker_manager.go`：

```go
func (dm *DockerManager) SyncPod(pod *api.Pod, _ api.PodStatus, podStatus *kubecontainer.PodStatus, pullSecrets []api.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {

    // 计算容器的变化
	containerChanges, err := dm.computePodContainerChanges(pod, podStatus)
    ......

    // 如果需要，先删除运行的容器
	if containerChanges.StartInfraContainer || (len(containerChanges.ContainersToKeep) == 0 && len(containerChanges.ContainersToStart) == 0) {
	    ......
		killResult := dm.killPodWithSyncResult(pod, kubecontainer.ConvertPodStatusToRunningPod(dm.Type(), podStatus), nil)
		......
	}

	podIP := ""
	if podStatus != nil {
		podIP = podStatus.IP
	}

	// 先创建 infrastructure 容器
	podInfraContainerID := containerChanges.InfraContainerId
	if containerChanges.StartInfraContainer && (len(containerChanges.ContainersToStart) > 0) {
		......
		// 通过 docker 创建出来一个运行的 pause 容器。
		// 如果镜像不存在，kubelet 会先下载 pause 镜像；
		// 如果 pod 是主机模式，容器也是；其他情况下，容器会使用 None 网络模式，让 kubelet 的网络插件自己进行网络配置
		podInfraContainerID, err, msg = dm.createPodInfraContainer(pod)
		......

        // 配置 infrastructure 容器的网络
		if !kubecontainer.IsHostNetworkPod(pod) {
			err = dm.networkPlugin.SetUpPod(pod.Namespace, pod.Name, podInfraContainerID.ContainerID())
		    ......
		}
	}
	......

	// 启动正常的容器
	for idx := range containerChanges.ContainersToStart {
		container := &pod.Spec.Containers[idx]
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, container.Name)
		result.AddSyncResult(startContainerResult)

		// containerChanges.StartInfraContainer causes the containers to be restarted for config reasons
		if !containerChanges.StartInfraContainer {
			isInBackOff, err, msg := dm.doBackOff(pod, container, podStatus, backOff)
			if isInBackOff {
				startContainerResult.Fail(err, msg)
				continue
			}
		}

		if err, msg := dm.tryContainerStart(container, pod, podStatus, pullSecrets, namespaceMode, pidMode, podIP); err != nil {
			startContainerResult.Fail(err, msg)
			utilruntime.HandleError(fmt.Errorf("container start failed: %v: %s", err, msg))
			continue
		}
	}
	return
}
```

这个方法的内容也非常多，它的主要逻辑是先比较传递过来的 pod 信息和实际运行的 pod（对于新建 pod 来说后者为空），计算出两者的差别，也就是需要更新的地方。然后先创建 infrastructure 容器，配置好网络，然后再逐个创建应用容器。

`dm.computePodContainerChanges` 根据最新拿到的 pod 配置，和目前实际运行的容器对比，计算出其中的变化，得到需要重新启动的容器信息。不管是创建、更新还是删除 pod，最终都会调用  `syncPod` 方法，所以这个结果涵盖了所有的可能性。

```go
type podContainerChangesSpec struct {
	StartInfraContainer  bool
	InfraChanged         bool
	InfraContainerId     kubecontainer.DockerID
	InitFailed           bool
	InitContainersToKeep map[kubecontainer.DockerID]int
	ContainersToStart    map[int]string
	ContainersToKeep     map[kubecontainer.DockerID]int
}
```

这个结构体中的内容可以分成三部分：infrastructure 变化信息，init containers 变化信息，以及应用 containers 变化信息。检测 infrastructure pod 有没有变化，只需要检查下面这些内容：

- pasue 镜像
- 网络模型有没有变化
- 暴露的端口号有没有变化
- 镜像拉取策略
- 环境变量

根据 infrastructure 容器的状态，其需要执行的操作可以分为三种情况：

- 容器还不存在，或者没有在运行状态：启动新的 pause 容器（这就是我们一直分析的 pod 新建的情况）
- 容器正在运行，但是新的 pod 配置发生了变化：杀掉 pause 容器，重新启动
- pause 容器已经运行，而且没有变化，不做任何事情

应用容器要重建的原因包括：

- 容器异常退出
- infrastructure 容器要重启（pod 新建也属于这种情况）
- init 容器运行失败
- container 配置的哈希值发生了变化（对 pod 的内容做了更新操作）
- liveness 检测失败

容器创建就是根据配置得到 docker client 新建容器需要的所有参数，最终发送给 docker API，这里不再赘述。创建应用容器的时候，会把 infrastructure 容器的网络模式和 pidMode 传过去，这也是 pod 中所有容器共享网络和 pid 资源的地方。

新建 pod 的逻辑就是这样的，更新和这个流程类似，删除的逻辑比这个简单，我们就不再一一解释。

## 总结

通过上一篇文章和这一篇文章，我们分析了 kubelet 的主要流程代码，包括它是怎么启动的，它是怎么处理新建 pod 的。这两篇代码分析已经把 kubelet 代码的骨架描绘出来了，但也有很多细节性的东西没有提及。我们在后面的文章中会继续分析 kubelet 一些组件的功能，让大家更全面地理解 kubelet 。那些我们没有提及的知识点，读者可以自己阅读源码。