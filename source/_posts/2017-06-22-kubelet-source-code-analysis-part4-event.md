---
layout: post
title: "kubelet 源码分析： 事件处理"
excerpt: "kubelet 需要把关键步骤中的执行事件发送到 apiserver，这样客户端就能通过查询知道整个流程发生了哪些事情，不需要登录到 kubelet 所在的节点查看日志的内容或者容器的运行状态。"
categories: blog
tags: [kubernetes, kubelet]
comments: true
share: true
---

前几篇源码分析的文章介绍了 kubelet 提供的各种功能，这篇文章继续介绍 kubelet 的源码部分：事件机制。事件并不是 kubelet 对外提供的功能，但是对于 kubernetes 系统却非常重要。

## kubelet 事件机制

我们知道 kubernetes 是分布式的架构，apiserver 是整个集群的交互中心，客户端主要和它打交道，kubelet 是各个节点上的 worker，负责执行具体的任务。对于用户来说，每次创建资源的时候，除了看到它的最终状态（一般是运行态），希望看到资源执行的过程，中间经过了哪些步骤。这些反馈信息对于调试来说非常重要，有些任务会失败或者卡在某个步骤，有了这些信息，我们就能够准确地定位问题。

kubelet 需要把关键步骤中的执行事件发送到 apiserver，这样客户端就能通过查询知道整个流程发生了哪些事情，不需要登录到 kubelet 所在的节点查看日志的内容或者容器的运行状态。

## 事件机制源码分析

这部分我们讲直接分析 kubelet 的源码，了解事件机制实现的来龙去脉。

### 谁会发送事件？

kubernetes 是以 pod 为核心概念的，不管是 deployment、statefulSet、replicaSet，最终都会创建出来 pod。因此事件机制也是围绕 pod 进行的，在 pod 生命周期的关键步骤都会产生事件消息。比如 Controller Manager 会记录节点注册和销毁的事件、Deployment 扩容和升级的事件；kubelet 会记录镜像回收事件、volume 无法挂载事件等；Scheduler 会记录调度事件等。这篇文章只关心 kubelet 的情况，其他组件实现原理是一样的。

查看 `pkg/kubelet/kubelet.go` 文件的代码，你会看到类似下面的代码：

```go
kl.recorder.Eventf(kl.nodeRef, api.EventTypeWarning, events.ContainerGCFailed, err.Error())
```

上面这行代码是容器 GC 失败的时候出现的，它发送了一条事件消息，通知 apiserver 容器 GC 失败的原因。除了 kubelet 本身之外，kubelet 的各个组件（比如 imageManager、probeManager 等）也会有这个字段，记录重要的事件，读者可以搜索源码去看 kubelet 哪些地方会发送事件。

`recorder` 是 kubelet 结构的一个字段：

```go
type kubelet struct {
    ...
    // The EventBroader to use
    recorder    record.EventRecorder
    ...
}
```

它的类型是 `record.EventRecorder`，这是个定义了三个方法的  interface，代码在  `pkg/client/record/event.go` 文件中：

```go
type EventRecorder interface {
	Event(object runtime.Object, eventtype, reason, message string)

	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})

	PastEventf(object runtime.Object, timestamp unversioned.Time, eventtype, reason, messageFmt string, args ...interface{})
```

这里的三个方法都是记录事件用的，`Eventf` 就是封装了类似 `Printf` 的信息打印机制，内部也会调用 `Event`，而 `PastEventf` 允许用户传进来自定义的时间戳，因此可以设置事件产生的时间。我们后面会详解介绍它们参数的意思和内部实现。

### EventRecorder 和 EventBroadcaster

我们已经知道了 `recorder` 就是事件的负责人，那么接下来就要了解它是怎么实现事件发送机制的。不过在那之前，先让我们找到 `recorder` 是什么时候被创建的？

在 [kubelet 启动流程](http://cizixs.com/2017/06/06/kubelet-source-code-analysis-part-1) 这篇文章中，我们讲到 `RunKubelet` 中会初始化 `EventBroadcaster` 和 `Recorder`，对应的代码如下：
 
`cmd/kubelet/app/server.go#RunKubelet`：
```go
eventBroadcaster := record.NewBroadcaster()
kubeDeps.Recorder = eventBroadcaster.NewRecorder(api.EventSource{Component: "kubelet", Host: string(nodeName)})
eventBroadcaster.StartLogging(glog.V(3).Infof)

if kubeDeps.EventClient != nil {
	eventBroadcaster.StartRecordingToSink(&unversionedcore.EventSinkImpl{Interface: kubeDeps.EventClient.Events("")})
} else {
	glog.Warning("No api server defined - no events will be sent to API server.")
}
```

正如名字所示的那样， `eventBroadcaster` 是个事件广播器，`StartLogging` 和 `StartRecordingToSink` 创建了两个不同的事件处理函数，分别把事件记录到日志和发送给 apiserver。而 `NewRecorder` 新建了一个 `Recoder` 对象，通过它的 `Event`、`Eventf` 和 `PastEventf` 方法，用户可以往里面发送事件，`eventBroadcaster` 会把接收到的事件发送个多个处理函数，比如这里提到的写日志和发送到 apiserver。

知道了 `EventBroadcaster` 的功能，我们来看看它的实现：

`pkg/client/record/event.go`
```go
type EventBroadcaster interface {
	StartEventWatcher(eventHandler func(*api.Event)) watch.Interface
	StartRecordingToSink(sink EventSink) watch.Interface
	StartLogging(logf func(format string, args ...interface{})) watch.Interface

	NewRecorder(source api.EventSource) EventRecorder
}
```

`EventBroadcaster` 是个接口类型，`NewRecorder` 新建一个 `EventRecoder` 对象，它就像一个事件记录仪，用户可以通过它记录事件，它在内部会把事件发送给 `EventBroadcaster`。

此外，`EventBroadcaster` 定义了三个 `Start` 开头的方法，它们用来添加事件处理 handler 。其中核心方法是 `StartEventWatcher`，它会在后台启动一个 goroutine，不断从 EventBroadcaster 提供的管道中接收事件，然后调用 `eventHandler` 处理函数对事件进行处理。`StartRecordingToSink` 和 `StartLogging` 是对 `StartEventWatcher` 的封装，分别实现了不同的处理函数（发送给 apiserver 和写日志）。

至此，`EventBroadcaster` 的工作原理就比较清晰了：它通过 `EventRecorder` 提供接口供用户写事件，内部把接收到的事件发送给处理函数。处理函数是可以扩展的，用户可以通过 `StartEventWatcher` 来编写自己的事件处理逻辑，`kubelet` 默认会使用 `StartRecordingToSink` 和 `StartLogging`，也就是说任何一个事件会同时发送给 apiserver，并打印到日志中。

知道了 `EventBroadcaster` 做的事情，接下来我们就要分析它是怎么做的。这些内容可以分为三个部分：

1. `EventRecorder` 是怎么把事件发送给 `EventBroadcaster` 的？
2. `EventBroadcaster` 是怎么实现事件广播的？
3. `StartRecodingToSink` 内部是如何把事件发送到 apiserver 的？

分析完以上三点，我们就能知道事件的整个流程。

### 发送事件的过程

通过上面的分析，我们知道事件是通过 `EventRecorder` 对象发送出来的，它的具体实现在 `pkg/event/record/event.go`：

```go
type recorderImpl struct {
	source api.EventSource
	*watch.Broadcaster
	clock clock.Clock
}

func (eventBroadcaster *eventBroadcasterImpl) NewRecorder(source api.EventSource) EventRecorder {
	return &recorderImpl{source, eventBroadcaster.Broadcaster, clock.RealClock{}}
}

func (recorder *recorderImpl) Event(object runtime.Object, eventtype, reason, message string) {
	recorder.generateEvent(object, unversioned.Now(), eventtype, reason, message)
}

func (recorder *recorderImpl) Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{}) {
	recorder.Event(object, eventtype, reason, fmt.Sprintf(messageFmt, args...))
}

func (recorder *recorderImpl) PastEventf(object runtime.Object, timestamp unversioned.Time, eventtype, reason, messageFmt string, args ...interface{}) {
	recorder.generateEvent(object, timestamp, eventtype, reason, fmt.Sprintf(messageFmt, args...))
}
```

`recorderImpl` 是具体的实现，`eventBroadcaster.NewRecorder` 会创建一个指定 `EventSource` 的 `EventRecorder`，`EventSource` 指明了哪个节点的哪个组件。

recorder 对外暴露了三个方法：`Event`、`Eventf` 和 `PastEventf`，它们的内部最终都是调用 `generateEvent` 方法：

```go
func (recorder *recorderImpl) generateEvent(object runtime.Object, timestamp unversioned.Time, eventtype, reason, message string) {
	ref, err := api.GetReference(object)
    ......

	if !validateEventType(eventtype) {
		glog.Errorf("Unsupported event type: '%v'", eventtype)
		return
	}

	event := recorder.makeEvent(ref, eventtype, reason, message)

	if pod, ok := object.(*api.Pod); ok && pod.ObjectMeta.Labels != nil {
		// add the labels in pod to event
		event.ObjectMeta.Labels = map[string]string{}
		for k, v := range pod.ObjectMeta.Labels {
			event.ObjectMeta.Labels[k] = v
		}
	}

	event.Source = recorder.source

	go func() {
		defer utilruntime.HandleCrash()
		recorder.Action(watch.Added, event)
	}()
}
```

`generateEvent` 就是根据传入的参数，生成一个 `api.Event` 对象，并发送出去。它各个参数的意思是：

- object：哪个组件/对象发出的事件，比如 kubelet 产生的事件会使用 node 对象
- timestamp：事件产生的时间
- eventtype：事件类型，目前有两种：`Normal` 和 `Warning`，分别代表正常的事件和可能有问题的事件，定义在 `pkg/api/types.go` 文件中，未来可能有其他类型的事件扩展
- reason：事件产生的原因，可以在 `pkg/kubelet/events/event.go` 看到 kubelet 定义的所有事件类型
- message：事件的具体内容，用户可以理解的语句

`makeEvent` 就是根据参数构建 `api.Event` 对象，自动填充时间戳和 namespace：

```go
func (recorder *recorderImpl) makeEvent(ref *api.ObjectReference, eventtype, reason, message string) *api.Event {
	t := unversioned.Time{Time: recorder.clock.Now()}
	namespace := ref.Namespace
	if namespace == "" {
		namespace = api.NamespaceDefault
	}
	return &api.Event{
		ObjectMeta: api.ObjectMeta{
			Name:      fmt.Sprintf("%v.%x", ref.Name, t.UnixNano()),
			Namespace: namespace,
		},
		InvolvedObject: *ref,
		Reason:         reason,
		Message:        message,
		FirstTimestamp: t,
		LastTimestamp:  t,
		Count:          1,
		Type:           eventtype,
	}
}
```

**注意 Event 事件的名字的构成**，它有两部分：事件关联对象的名字和当前的时间，中间用点隔开。

`api.Event` 这个结构体定义在 `pkg/api/types.go`：

```go
type Event struct {
	unversioned.TypeMeta `json:",inline"`
	ObjectMeta `json:"metadata,omitempty"`

	InvolvedObject ObjectReference `json:"involvedObject,omitempty"`

	Reason string `json:"reason,omitempty"`
	Message string `json:"message,omitempty"`
	Source EventSource `json:"source,omitempty"`
	Type string `json:"type,omitempty"`

	FirstTimestamp unversioned.Time `json:"firstTimestamp,omitempty"`
	LastTimestamp unversioned.Time `json:"lastTimestamp,omitempty"`
	Count int32 `json:"count,omitempty"`
}
```

除了所有的 kubernetes 资源都有的 `unversioned.TypeMeta`（资源的类型和版本，对应了 yaml 文件的 `Kind` 和 `apiVersion` 字段） 和 `ObjectMera` 字段（资源的元数据，比如 name、nemspace、labels、uuid、创建时间等）之外，还有和事件本身息息相关的字段，比如事件消息、来源、类型，以及数量（kubernetes 会把多个相同的事件汇聚到一起）和第一个事件的发生的时间等。

中间有个 `InvolvedObject` 字段，它其实指向了和事件关联的对象，如果是启动容器的事件，这个对象就是 Pod。

至此，我们就疏通了事件是怎么创建出来的。下面看看事件是怎么发出去的，发送是通过 `recorder.Action()` 实现的。找到对应的代码部分，竟然简单得只有一句话，把对象封装一下，发送到 `m.incoming` 管道。

```go
// Action distributes the given event among all watchers.
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	m.incoming <- Event{action, obj}
}
```

`Broadcaster` 是 `Recoder` 内部的对象，调用 `NewRecoder` 的时候 `EventBroadcaster` 传给它的。接下来，我们要分析 `EventBroadcaster` 的实现。

### EventBroadcaster 实现事件的分发

`EventBroadcaster` 也在 `pkg/event/record/event.go` 文件中：

```
type eventBroadcasterImpl struct {
	*watch.Broadcaster
	sleepDuration time.Duration
}

func NewBroadcaster() EventBroadcaster {
	return &eventBroadcasterImpl{watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull), defaultSleepDuration}
}
```

它的核心组件是 `watch.Broadcaster`，`Broadcaster` 就是广播的意思，主要功能就是把发给它的消息，广播给所有的监听者（watcher）。它的实现代码在 `pkg/watch/mux.go`，我们不再深入剖析，不过这部分代码如何使用 golang channel 是值得所有读者学习的。

简单来说，`watch.Broadcaster` 是一个分发器，内部保存了一个消息队列，可以通过 `Watch` 创建监听它内部的 worker。当有消息发送到队列中，`watch.Broadcaster` 后台运行的 goroutine 会接收消息并发送给所有的 watcher。而每个 `watcher` 都有一个接收消息的 channel，用户可以通过它的 `ResultChan()` 获取这个 channel 从中读取数据进行处理。

前面说过 `StartLogging` 和 `StartRecordingToSink` 都是启动一个事件处理的函数，我们就以后者为例，看看事件的处理过程：

```go
func (eventBroadcaster *eventBroadcasterImpl) StartRecordingToSink(sink EventSink) watch.Interface {
	randGen := rand.New(rand.NewSource(time.Now().UnixNano()))
	eventCorrelator := NewEventCorrelator(clock.RealClock{})
	return eventBroadcaster.StartEventWatcher(
		func(event *api.Event) {
			recordToSink(sink, event, eventCorrelator, randGen, eventBroadcaster.sleepDuration)
		})
}
```

`StartRecordingToSink` 就是对 `StartEventWatcher` 的封装，将处理函数设置为 `recordToSink`。我们先看看 `StartEventWatcher` 的代码：

```
func (eventBroadcaster *eventBroadcasterImpl) StartEventWatcher(eventHandler func(*api.Event)) watch.Interface {
	watcher := eventBroadcaster.Watch()
	go func() {
		defer utilruntime.HandleCrash()
		for {
			watchEvent, open := <-watcher.ResultChan()
			if !open {
				return
			}
			event, ok := watchEvent.Object.(*api.Event)
			if !ok {
				continue
			}
			eventHandler(event)
		}
	}()
	return watcher
}
```

它启动一个 goroutine，不断从 `watcher.ResultChan()` 中读取消息，然后调用 `eventHandler(event)` 对事件进行处理。

而我们的处理函数就是 `recordToSink`，它的代码是下一节的重点。

### 事件的处理过程

`recordToSink` 负责把事件发送到 apiserver，这里的 sink 其实就是和 apiserver 交互的 restclient， event 是要发送的事件，eventCorrelator 在发送事件之前先对事件进行预处理。

```go
func recordToSink(sink EventSink, event *api.Event, eventCorrelator *EventCorrelator, randGen *rand.Rand, sleepDuration time.Duration) {
	eventCopy := *event
	event = &eventCopy
	result, err := eventCorrelator.EventCorrelate(event)
	if result.Skip {
		return
	}
	
	tries := 0
	for {
		if recordEvent(sink, result.Event, result.Patch, result.Event.Count > 1, eventCorrelator) {
			break
		}
		tries++
		if tries >= maxTriesPerEvent {
			glog.Errorf("Unable to write event '%#v' (retry limit exceeded!)", event)
			break
		}
		// 第一次重试增加随机性，防止 apiserver 重启的时候所有的事件都在同一时间发送事件
		if tries == 1 {
			time.Sleep(time.Duration(float64(sleepDuration) * randGen.Float64()))
		} else {
			time.Sleep(sleepDuration)
		}
	}
}
```

`recordToSink` 对事件的处理分为两个步骤：`eventCorrelator.EventCorrelate` 会对事件做预处理，主要是聚合相同的事件（避免产生的事件过多，增加 etcd 和 apiserver 的压力，也会导致查看 pod 事件很不清晰）；`recordEvent` 负责最终把事件发送到 apiserver，它会重试很多次（默认是 12 次），并且每次重试都有一定时间间隔（默认是 10 秒钟）。


```go
func recordEvent(sink EventSink, event *api.Event, patch []byte, updateExistingEvent bool, eventCorrelator *EventCorrelator) bool {
	var newEvent *api.Event
	var err error
	
	// 更新已经存在的事件
	if updateExistingEvent {
		newEvent, err = sink.Patch(event, patch)
	}
	// 创建一个新的事件
	if !updateExistingEvent || (updateExistingEvent && isKeyNotFoundError(err)) {
		event.ResourceVersion = ""
		newEvent, err = sink.Create(event)
	}
	if err == nil {
		// we need to update our event correlator with the server returned state to handle name/resourceversion
		eventCorrelator.UpdateState(newEvent)
		return true
	}

    // 如果是已知错误，就不要再重试了；否则，返回 false，让上层进行重试
	switch err.(type) {
	case *restclient.RequestConstructionError:
		glog.Errorf("Unable to construct event '%#v': '%v' (will not retry!)", event, err)
		return true
	case *errors.StatusError:
		if errors.IsAlreadyExists(err) {
			glog.V(5).Infof("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		} else {
			glog.Errorf("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		}
		return true
	case *errors.UnexpectedObjectError:
	default:
	}
	glog.Errorf("Unable to write event: '%v' (may retry after sleeping)", err)
	return false
}
```
它根据 `eventCorrelator` 的结果来决定是新建一个事件还是更新已经存在的事件，并根据请求的结果决定是否需要重试（返回值为 false 说明需要重试，返回值为 true 表明已经操作成功或者忽略请求错误）。`sink.Create` 和 `sink.Patch` 是自动生成的 apiserver 的 client，对应的代码在： `pkg/client/clientset_generated/internalclientset/typed/core/internalversion/event_expansion.go` 。

到这里，事件总算完成了它的使命，到了目的地。但是我们略过了 `EventCorrelator` 的部分，它在发送之前对事件做过滤和聚合处理，以免产生大量的事件给 apiserver 和 etcd 带来太大的压力。

### EventCorrelator：事件的预处理

`EventCorrelator` 的代码在 `pkg/client/record/event_cache.go` 文件中，从文件名可以猜测出它对事件做了缓存。

```go
func NewEventCorrelator(clock clock.Clock) *EventCorrelator {
	cacheSize := maxLruCacheEntries
	return &EventCorrelator{
		filterFunc: DefaultEventFilterFunc,
		aggregator: NewEventAggregator(
			cacheSize,
			EventAggregatorByReasonFunc,
			EventAggregatorByReasonMessageFunc,
			defaultAggregateMaxEvents,
			defaultAggregateIntervalInSeconds,
			clock),
		logger: newEventLogger(cacheSize, clock),
	}
}

// EventCorrelate filters, aggregates, counts, and de-duplicates all incoming events
func (c *EventCorrelator) EventCorrelate(newEvent *api.Event) (*EventCorrelateResult, error) {
	if c.filterFunc(newEvent) {
		return &EventCorrelateResult{Skip: true}, nil
	}
	aggregateEvent, err := c.aggregator.EventAggregate(newEvent)
	if err != nil {
		return &EventCorrelateResult{}, err
	}
	observedEvent, patch, err := c.logger.eventObserve(aggregateEvent)
	return &EventCorrelateResult{Event: observedEvent, Patch: patch}, err
}
```

`EventCorrelator` 内部有三个对象：`filterFunc`、`aggregator` 和 `logger`，它们分别对事件进行过滤、把相似的事件汇聚在一起、把相同的事件记录到一起。使用 `NewEventCorrelator` 初始化的时候内部会自动创建各个对象的默认值，`EventCorrelate` 会以此调用三个对象的方法，并返回最终的结果。现在它们的逻辑是这样的：

- `filterFunc`：目前不做过滤，也就是说所有的事件都要经过后续处理，后面可能会做扩展
- `aggregator`：如果在最近 10 分钟出现过 10 个相似的事件（除了 message 和时间戳之外其他关键字段都相同的事件），aggregator 会把它们的 message 设置为 `events with common reason combined`，这样它们就完全一样了
- `logger`：这个变量的名字有点奇怪，其实它会把相同的事件（除了时间戳之外其他字段都相同）变成同一个事件，通过增加事件的 `Count` 字段来记录该事件发生了多少次。经过 `aggregator` 的事件会在这里变成同一个事件

`aggregator` 和 `logger` 都会在内部维护一个缓存（默认长度是 4096），事件的相似性和相同性比较是和缓存中的事件进行的，也就是说它并在乎 kubelet 启动之前的事件，而且如果事件超过 4096 的长度，最近没有被访问的事件也会被从缓存中移除。这也是这个文件中带有 `cache` 的原因。它们的内部实现并不复杂，有兴趣的读者请自行阅读相关源码。

## Event 总结

通过这篇文章，我们了解到了整个事件机制的来龙去脉。最后，我们再做一个总结，看看事件流动的整个过程：

1. kubelet 通过 `recorder` 对象提供的 `Event`、`Eventf` 和 `PastEventf` 方法产生特性的事件
2. `recorder` 根据传递过来的参数新建一个 `Event` 对象，并把它发送给 `EventBroadcaster` 的管道
3. `EventBroadcaster` 后台运行的 goroutine 从管道中读取事件消息，把它广播给之前注册的 handler 进行处理
4. kubelet 有两个 handler，它们分别把事件记录到日志和发送给 apiserver。记录到日志很简单，直接打印就行
5. 发送给 apiserver 的 handler 叫做 `EventSink`，它在发送事件给 apiserver 之前会先做预处理
6. 预处理操作是 `EventCorrelator` 完成的，它会对事件做过滤、汇聚和去重操作，返回处理后的事件（可能是原来的事件，也可能是新创建的事件）
7. 最后通过 restclient （eventClient） 调用对应的方法，给 apiserver 发送请求，这个过程如果出错会进行重试
8. apiserver 接收到事件的请求把数据更新到 etcd

事件的产生过程是这样的，那么这些事件都有什么用呢？它一般用于调试，用户可以通过 `kubectl` 命令获取整个集群或者某个 pod 的事件信息。`kubectl get events` 可以看到所有的事件，`kubectl describe pod PODNAME` 能看到关于某个 pod 的事件。对于前者很好理解，kubectl 会直接访问 apiserver 的 event 资源，而对于后者 kubectl 还根据 pod 的名字进行搜索，匹配 InvolvedObject 名称和 pod 名称匹配的事件。

我们来思考一下事件机制的框架，有哪些我们可以借鉴的设计思想呢？我想最重要的一点是：**需求决定实现**。

Event 和 kubernetes 中其他的资源不同，它有一个很重要的特性就是它可以丢失。如果某个事件丢了，并不会影响集群的正常工作。事件的重要性远低于集群的稳定性，所以我们看到事件整个流程中如果有错误，会直接忽略这个事件。

事件的另外一个特性是它的数量很多，相比于 pod 或者 deployment 等资源，事件要比它们多很多，而且每次有事件都要对 etcd 进行写操作。整个集群如果不加管理地往 etcd 中写事件，会对 etcd 造成很大的压力，而 etcd 的可用性是整个集群的基础，所以每个组件在写事件之前，会对事件进行汇聚和去重工作，减少最终的写操作。

## 参考资料

- [Kubernetes Events介绍（下）](https://www.kubernetes.org.cn/1195.html)
- [Application Introspection and Debugging](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application-introspection/)