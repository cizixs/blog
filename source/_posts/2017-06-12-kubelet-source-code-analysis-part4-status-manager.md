---
layout: post
title: "kubelet 源码分析：statusManager 和 probeManager"
excerpt: "statusManager 负责维护状态信息，并把 pod 状态更新到 apiserver，但是它并不负责监控 pod 状态的变化，而是提供对应的接口供其他组件调用，比如 probeManager。probeManager 会定时去监控 pod 中容器的健康状况，一旦发现状态发生变化，就调用 statusManager 提供的方法更新 pod 的状态。"
categories: blog
tags: [kubernetes, kubelet, container, golang]
comments: true
share: true
---

## 简介

在 kubelet 初始化的时候，会创建 statusManager 和 probeManager，两者都和 pod 的状态有关系，因此我们放到一起来讲解。

statusManager 负责维护状态信息，并把 pod 状态更新到 apiserver，但是它并不负责监控 pod 状态的变化，而是提供对应的接口供其他组件调用，比如 probeManager。probeManager 会定时去监控 pod 中容器的健康状况，一旦发现状态发生变化，就调用 statusManager 提供的方法更新 pod 的状态。

```go
klet.statusManager = status.NewManager(kubeClient, klet.podManager)
klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.runner,
		containerRefManager,
		kubeDeps.Recorder)
```

## StatusManager

statusManager 对应的代码在 `pkg/kubelet/status/status_manager.go` 文件中，

```go
type PodStatusProvider interface {
	GetPodStatus(uid types.UID) (api.PodStatus, bool)
}

type Manager interface {
	PodStatusProvider

	Start()

	SetPodStatus(pod *api.Pod, status api.PodStatus)
	SetContainerReadiness(podUID types.UID, containerID kubecontainer.ContainerID, ready bool)
	TerminatePod(pod *api.Pod)
	RemoveOrphanedStatuses(podUIDs map[types.UID]bool)
}
```

这个接口的方法可以分成三组：获取某个 pod 的状态、后台运行 goroutine 执行同步工作、修改 pod 的状态。修改状态的方法有多个，每个都有不同的用途：

- SetPodStatus：如果 pod 的状态发生了变化，会调用这个方法，把新状态更新到 apiserver，一般在 kubelet 维护 pod 生命周期的时候会调用
- SetContainerReadiness：如果健康检查发现 pod 中容器的健康状态发生变化，会调用这个方法，修改 pod 的健康状态
- TerminatePod：kubelet 在删除 pod 的时候，会调用这个方法，把 pod 中所有的容器设置为 terminated 状态
- RemoveOrphanedStatuses：删除孤儿 pod，直接把对应的状态数据从缓存中删除即可

`Start()` 方法是在 kubelet 运行的时候调用的，它会启动一个 goroutine 执行更新操作：

```go
const syncPeriod = 10 * time.Second

func (m *manager) Start() {
	......
	glog.Info("Starting to sync pod status with apiserver")
	syncTicker := time.Tick(syncPeriod)
	// syncPod and syncBatch share the same go routine to avoid sync races.
	go wait.Forever(func() {
		select {
		case syncRequest := <-m.podStatusChannel:
			m.syncPod(syncRequest.podUID, syncRequest.status)
		case <-syncTicker:
			m.syncBatch()
		}
	}, 0)
}
```

这个 goroutine 就能不断地从两个 channel 监听数据进行处理：`syncTicker` 是个定时器，也就是说它会定时保证 apiserver 和自己缓存的最新 pod 状态保持一致；`podStatusChannel` 是所有 pod 状态更新发送到的地方，调用方不会直接操作这个 channel，而是通过调用上面提到的修改状态的各种方法，这些方法内部会往这个 channel 写数据。

`m.syncPod` 根据参数中的 pod 和它的状态信息对 apiserver 中的数据进行更新，如果发现 pod 已经被删除也会把它从内部数据结构中删除。

## ProbeManager

probeManager 检测 [pod 中容器的健康状态](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)，目前有两种 probe：readiness 和 liveness。**readinessProbe** 检测容器是否可以接受请求，如果检测结果失败，则将其从 service 的 endpoints 中移除，后续的请求也就不会发送给这个容器；livenessProbe 检测容器是否存活，如果检测结果失败，kubelet 会杀死这个容器，并重启一个新的（除非 RestartPolicy 设置成了 Never）。

并不是所有的 pod 中的容器都有健康检查的探针，如果没有，则不对容器进行检测，默认认为容器是正常的。在每次创建新 pod 的时候，kubelet 都会调用 `probeManager.AddPod(pod)` 方法，它对应的实现在 `pkg/kubelet/prober/prober_manager.go` 文件中：

```go
func (m *manager) AddPod(pod *api.Pod) {
	m.workerLock.Lock()
	defer m.workerLock.Unlock()

	key := probeKey{podUID: pod.UID}
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name

		if c.ReadinessProbe != nil {
			key.probeType = readiness
			if _, ok := m.workers[key]; ok {
				glog.Errorf("Readiness probe already exists! %v - %v",
					format.Pod(pod), c.Name)
				return
			}
			w := newWorker(m, readiness, pod, c)
			m.workers[key] = w
			go w.run()
		}

		if c.LivenessProbe != nil {
			key.probeType = liveness
			if _, ok := m.workers[key]; ok {
				glog.Errorf("Liveness probe already exists! %v - %v",
					format.Pod(pod), c.Name)
				return
			}
			w := newWorker(m, liveness, pod, c)
			m.workers[key] = w
			go w.run()
		}
	}
}
```

遍历 pod 中的容器，如果其定义了 readiness 或者 liveness，就创建一个 worker，并启动一个 goroutine 在后台运行这个 worker。

`pkg/kubelet/prober/worker.go`：

```go
func (w *worker) run() {
	probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second
	probeTicker := time.NewTicker(probeTickerPeriod)

	defer func() {
		probeTicker.Stop()
		if !w.containerID.IsEmpty() {
			w.resultsManager.Remove(w.containerID)
		}

		w.probeManager.removeWorker(w.pod.UID, w.container.Name, w.probeType)
	}()

	time.Sleep(time.Duration(rand.Float64() * float64(probeTickerPeriod)))

probeLoop:
	for w.doProbe() {
		// Wait for next probe tick.
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
			// continue
		}
	}
}

func (w *worker) doProbe() (keepGoing bool) {
	defer func() { recover() }() 
	defer runtime.HandleCrash(func(_ interface{}) { keepGoing = true })

    // pod 没有被创建，或者已经被删除了，直接跳过检测，但是会继续检测
	status, ok := w.probeManager.statusManager.GetPodStatus(w.pod.UID)
	if !ok {
		glog.V(3).Infof("No status for pod: %v", format.Pod(w.pod))
		return true
	}

	// pod 已经退出（不管是成功还是失败），直接返回，并终止 worker
	if status.Phase == api.PodFailed || status.Phase == api.PodSucceeded {
		glog.V(3).Infof("Pod %v %v, exiting probe worker",
			format.Pod(w.pod), status.Phase)
		return false
	}

    // 容器没有创建，或者已经删除了，直接返回，并继续检测，等待更多的信息
	c, ok := api.GetContainerStatus(status.ContainerStatuses, w.container.Name)
	if !ok || len(c.ContainerID) == 0 {
		glog.V(3).Infof("Probe target container not found: %v - %v",
			format.Pod(w.pod), w.container.Name)
		return true 
	}

    // pod 更新了容器，使用最新的容器信息
	if w.containerID.String() != c.ContainerID {
		if !w.containerID.IsEmpty() {
			w.resultsManager.Remove(w.containerID)
		}
		w.containerID = kubecontainer.ParseContainerID(c.ContainerID)
		w.resultsManager.Set(w.containerID, w.initialValue, w.pod)
		w.onHold = false
	}

	if w.onHold {
		return true
	}

	if c.State.Running == nil {
		glog.V(3).Infof("Non-running container probed: %v - %v",
			format.Pod(w.pod), w.container.Name)
		if !w.containerID.IsEmpty() {
			w.resultsManager.Set(w.containerID, results.Failure, w.pod)
		}
		// 容器失败退出，并且不会再重启，终止 worker
		return c.State.Terminated == nil ||
			w.pod.Spec.RestartPolicy != api.RestartPolicyNever
	}

    // 容器启动时间太短，没有超过配置的初始化等待时间 InitialDelaySeconds
	if int32(time.Since(c.State.Running.StartedAt.Time).Seconds()) < w.spec.InitialDelaySeconds {
		return true
	}

    // 调用 prober 进行检测容器的状态
	result, err := w.probeManager.prober.probe(w.probeType, w.pod, status, w.container, w.containerID)
	if err != nil {
		return true
	}

	if w.lastResult == result {
		w.resultRun++
	} else {
		w.lastResult = result
		w.resultRun = 1
	}

    // 如果容器退出，并且没有超过最大的失败次数，则继续检测
	if (result == results.Failure && w.resultRun < int(w.spec.FailureThreshold)) ||
		(result == results.Success && w.resultRun < int(w.spec.SuccessThreshold)) {
		return true
	}

    // 保存最新的检测结果
	w.resultsManager.Set(w.containerID, result, w.pod)

	if w.probeType == liveness && result == results.Failure {
		// 容器 liveness 检测失败，需要删除容器并重新创建，在新容器成功创建出来之前，暂停检测
		w.onHold = true
	}

	return true
}
```

每次检测的时候都会用 `w.resultsManager.Set(w.containerID, result, w.pod)` 来保存检测结果，`resultsManager` 的代码在 `pkg/kubelet/prober/results/results_manager.go`：

```go
func (m *manager) Set(id kubecontainer.ContainerID, result Result, pod *api.Pod) {
	if m.setInternal(id, result) {
		m.updates <- Update{id, result, pod.UID}
	}
}

func (m *manager) setInternal(id kubecontainer.ContainerID, result Result) bool {
	m.Lock()
	defer m.Unlock()
	prev, exists := m.cache[id]
	if !exists || prev != result {
		m.cache[id] = result
		return true
	}
	return false
}

func (m *manager) Updates() <-chan Update {
	return m.updates
}
```

它把结果保存在缓存中，并发送到 `m.updates` 管道。对于 liveness 来说，它的管道消费者是 kubelet，还记得 `syncLoopIteration` 中的这段代码逻辑吗？

```go
case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				glog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			glog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*api.Pod{pod})
		}
```

因为 liveness 关系者 pod 的生死，因此需要 kubelet 的处理逻辑。而 readiness 即使失败也不会重新创建 pod，它的处理逻辑是不同的，它的处理代码同样在 `pkg/kubelet/prober/prober_manager.go`：

```go
func (m *manager) Start() {
	go wait.Forever(m.updateReadiness, 0)
}

func (m *manager) updateReadiness() {
	update := <-m.readinessManager.Updates()

	ready := update.Result == results.Success
	m.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)
}
```

`proberManager` 启动的时候，会运行一个 goroutine 定时读取 readinessManager 管道中的数据，并根据数据调用 `statusManager` 去更新 apiserver 中 pod 的状态信息。负责 Service 逻辑的组件获取到了这个状态，就能根据不同的值来决定是否需要更新 endpoints 的内容，也就是 service 的请求是否发送到这个 pod。

具体执行检测的代码在 `pkg/kubelet/prober/prober.go` 文件中，它会根据不同的 prober 方法（exec、HTTP、TCP）调用对应的处理逻辑，而这些具体的逻辑代码是在 `pkg/probe/` 文件夹中，三种方法的实现都不复杂，就不再详细解释了。