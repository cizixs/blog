---
layout: post
title: "kubelet 源码分析：Garbage Collect"
excerpt: "退出的容器也会继续占用系统资源，比如还会在文件系统存储很多数据、docker 应用也要占用 CPU 和内存去维护这些容器。docker 本身并不会自动删除已经退出的容器，因此 kubelet 就负起了这个责任。"
categories: blog
tags: [kubernetes, kubelet, container, golang]
comments: true
share: true
---

## kubernetes GC 简介

作为 kubernetes 中重要的组件，kubelet 接管了节点上容器相关的所有工作。除了最核心的任务：根据 apiserver 分配的 pod 创建容器之外，它还有其他很多事情要做，其中之一就是 GC（Garbage Collect）。

在运行一段时候之后，节点上会下载很多镜像，也会有很多因为各种原因退出的容器。为了保证节点能够正常运行，kubelet 要防止镜像太多占满磁盘空间，也要防止退出的容器太多导致系统运行缓慢或者出现错误。

GC 的工作不需要手动干预，kubelet 会周期性去执行，不过在启动 kubelet 进程的时候可以通过参数控制 GC 的策略。这篇文章会介绍 GC 的功能，然后跟着代码看一下它的实现。

### 容器 Garbage Collect

退出的容器也会继续占用系统资源，比如还会在文件系统存储很多数据、docker 应用也要占用 CPU 和内存去维护这些容器。docker 本身并不会自动删除已经退出的容器，因此 kubelet 就负起了这个责任。kubelet 容器的回收是为了删除已经退出的容器以节省节点的空间，提升性能。

容器 GC 虽然有利于空间和性能，但是删除容器也会导致错误现场被清理，不利于 debug 和错误定位，因此不建议把所有退出的容器都删除。因此容器的清理需要一定的策略，主要是告诉 kubelet 你要保存多少已经退出的容器。和容器 GC 有关的可以配置的 kubelet 启动参数包括：

- `minimum-container-ttl-duration`：container 结束多长时间之后才能够被回收，默认是一分钟
- `maximum-dead-containers-per-container`：每个 container 最终可以保存多少个已经结束的容器，默认是 1，设置为负数表示不做限制
- `maximum-dead-containers`：节点上最多能保留多少个结束的容器，默认是 -1，表示不做限制

也就是说默认情况下，kubelet 会自动每分钟去做容器 GC，容器退出一分钟之后就可以被删除，而且每个容器做多只会保留一个已经退出的历史容器。

### 镜像 Garbage Collect

镜像主要占用磁盘空间，虽然 docker 使用镜像分层可以让多个镜像共享存储，但是长时间运行的节点如果下载了很多镜像也会导致占用的存储空间过多。如果镜像导致磁盘被占满，会造成应用无法正常工作。docker 默认也不会做镜像清理，镜像一旦下载就会永远留在本地，除非被手动删除。

其实很多镜像并没有被实际使用，这些不用的镜像继续占用空间是非常大的浪费，也是巨大的隐患，因此 kubelet 也会周期性地去清理镜像。

镜像的清理和容器不同，是以占用的空间作为标准的，用户可以配置当镜像占据多大比例的存储空间时才进行清理。清理的时候会优先清理最久没有被使用的镜像，镜像被 pull 下来或者被容器使用都会更新它的最近使用时间。

启动 kubelet 的时候，可以配置这些参数控制镜像清理的策略：

- `image-gc-high-threshold`：磁盘使用率的上限，当达到这一使用率的时候会触发镜像清理。默认值为 90%
- `image-gc-low-threshold`：磁盘使用率的下限，每次清理直到使用率低于这个值或者没有可以清理的镜像了才会停止.默认值为 80%
- `minimum-image-ttl-duration`：镜像最少这么久没有被使用才会被清理，可以使用 `h`（小时）、`m`（分钟）、`s`（秒）和 `ms`（毫秒）时间单位进行配置，默认是 `2m`(两分钟)

也就是说，默认情况下，当镜像占满所在盘 90% 容量的时候，kubelet 就会进行清理，一直到镜像占用率低于 80% 为止。

## 容器 GC 的代码分析

我们在之前[分析 kubelet 启动](http://cizixs.com/2017/06/06/kubelet-source-code-analysis-part-1)的时候讲过，kubelet 会调用 `StartGarbageCollection` 启动 GC：

```go
func (kl *Kubelet) StartGarbageCollection() {
	loggedContainerGCFailure := false
	go wait.Until(func() {
		if err := kl.containerGC.GarbageCollect(kl.sourcesReady.AllReady()); err != nil {
			kl.recorder.Eventf(kl.nodeRef, api.EventTypeWarning, events.ContainerGCFailed, err.Error())
			loggedContainerGCFailure = true
		} else {
			var vLevel glog.Level = 4
			if loggedContainerGCFailure {
				vLevel = 1
				loggedContainerGCFailure = false
			}

			glog.V(vLevel).Infof("Container garbage collection succeeded")
		}
	}, ContainerGCPeriod, wait.NeverStop)

	loggedImageGCFailure := false
	go wait.Until(func() {
		if err := kl.imageManager.GarbageCollect(); err != nil {
			kl.recorder.Eventf(kl.nodeRef, api.EventTypeWarning, events.ImageGCFailed, err.Error())
			loggedImageGCFailure = true
		} else {
			var vLevel glog.Level = 4
			if loggedImageGCFailure {
				vLevel = 1
				loggedImageGCFailure = false
			}

			glog.V(vLevel).Infof("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)
}
```

容器 GC 和镜像 GC 分别是在独立的 goroutine 中执行的，我们先来分析容器 GC 的过程。containerGC 的创建是在 `pkg/kubelet/kubelet.go#NewMainKublet` 中完成的，对应的代码有：

```go
containerGCPolicy := kubecontainer.ContainerGCPolicy{
	MinAge:             kubeCfg.MinimumGCAge.Duration,
	MaxPerPodContainer: int(kubeCfg.MaxPerPodContainerCount),
	MaxContainers:      int(kubeCfg.MaxContainerCount),
}

containerGC, err := kubecontainer.NewContainerGC(klet.containerRuntime, containerGCPolicy)
if err != nil {
	return nil, err
}
klet.containerGC = containerGC
klet.containerDeletor = newPodContainerDeletor(klet.containerRuntime, integer.IntMax(containerGCPolicy.MaxPerPodContainer, minDeadContainerInPod))
```

`containerGCPolicy` 对应了我们上面提到的容器 GC 的策略，具体的初始化和实现的代码在 `pkg/kubelet/container/container_gc.go` 文件中：

```go
type ContainerGCPolicy struct {
	MinAge time.Duration
	MaxPerPodContainer int
	MaxContainers int
}

type ContainerGC interface {
	GarbageCollect(allSourcesReady bool) error
}

type realContainerGC struct {
	runtime Runtime
	policy ContainerGCPolicy
}

func NewContainerGC(runtime Runtime, policy ContainerGCPolicy) (ContainerGC, error) {
	if policy.MinAge < 0 {
		return nil, fmt.Errorf("invalid minimum garbage collection age: %v", policy.MinAge)
	}

	return &realContainerGC{
		runtime: runtime,
		policy:  policy,
	}, nil
}

func (cgc *realContainerGC) GarbageCollect(allSourcesReady bool) error {
	return cgc.runtime.GarbageCollect(cgc.policy, allSourcesReady)
}
```

这里的代码只是一层封装，最终会调用 `runtime.GarbageCollect`，对于 docker 来说，对应的代码是 `pkg/kubelet/dockertools/docker_manager.go`：

```go
func (dm *DockerManager) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool) error {
	return dm.containerGC.GarbageCollect(gcPolicy, allSourcesReady)
}
```

内容又被包装到 `dm.containerGC`，我们找到创建 dockerManager 的代码，看一下 `containerGC` 初始化部分：

```go
func NewDockerManager(...){
    ...
    dm.containerGC = NewContainerGC(client, podGetter, containerLogsDir)
    ...
}
```

这才是具体的 docker container GC 的实现， 对应的代码在 `pkg/kubelet/dockertools/container_gc.go#NewContainerGC`：

```go
type containerGC struct {
    // client 用来和 docker API 交互，比如获取容器列表、查看某个容器的详细信息等
	client           DockerInterface
	podGetter        podGetter
	containerLogsDir string
}

func NewContainerGC(client DockerInterface, podGetter podGetter, containerLogsDir string) *containerGC {
	return &containerGC{
		client:           client,
		podGetter:        podGetter,
		containerLogsDir: containerLogsDir,
	}
}

func (cgc *containerGC) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool) error {
	// 找到可以清理的容器列表，条件是不在运行并且创建时间超过 MinAge。
	// 这个步骤会过滤掉不是 kubelet 管理的容器，并且把容器按照创建时间进行排序（也就是说最早创建的容器会先被删除）
	// evictUnits 返回的是需要被正确回收的，第二个参数是 kubelet 无法识别的容器
	evictUnits, unidentifiedContainers, err := cgc.evictableContainers(gcPolicy.MinAge)
    ......

	// 删除无法识别的容器
	for _, container := range unidentifiedContainers {
		glog.Infof("Removing unidentified dead container %q with ID %q", container.name, container.id)
		err = cgc.client.RemoveContainer(container.id, dockertypes.ContainerRemoveOptions{RemoveVolumes: true})
		if err != nil {
			glog.Warningf("Failed to remove unidentified dead container %q: %v", container.name, err)
		}
	}

	// 如果 pod 已经不存在了，就删除其中所有的容器
	if allSourcesReady {
		for key, unit := range evictUnits {
			if cgc.isPodDeleted(key.uid) {
				cgc.removeOldestN(unit, len(unit)) // Remove all.
				delete(evictUnits, key)
			}
		}
	}

	// 执行 GC 策略，保证每个 POD 最多只能保存 MaxPerPodContainer 个已经退出的容器
	if gcPolicy.MaxPerPodContainer >= 0 {
		cgc.enforceMaxContainersPerEvictUnit(evictUnits, gcPolicy.MaxPerPodContainer)
	}

	// 执行 GC 策略，保证节点上最多有 MaxContainers 个已经退出的容器
	// 先把最大容器数量平分到 pod，保证每个 pod 在平均数量以下；如果还不满足要求的数量，就按照时间顺序先删除最旧的容器
	if gcPolicy.MaxContainers >= 0 && evictUnits.NumContainers() > gcPolicy.MaxContainers {
		// 先按照 pod 进行删除，每个 pod 能保留的容器数是总数的平均值
		numContainersPerEvictUnit := gcPolicy.MaxContainers / evictUnits.NumEvictUnits()
		if numContainersPerEvictUnit < 1 {
			numContainersPerEvictUnit = 1
		}
		cgc.enforceMaxContainersPerEvictUnit(evictUnits, numContainersPerEvictUnit)

		// 如果还不满足数量要求，按照容器进行删除，先删除最老的
		numContainers := evictUnits.NumContainers()
		if numContainers > gcPolicy.MaxContainers {
			flattened := make([]containerGCInfo, 0, numContainers)
			for uid := range evictUnits {
				flattened = append(flattened, evictUnits[uid]...)
			}
			sort.Sort(byCreated(flattened))

			cgc.removeOldestN(flattened, numContainers-gcPolicy.MaxContainers)
		}
	}
	
	......
	return nil
}
```

这段代码才是容器 GC 的核心逻辑，它做的事情是这样的：

- 先从正在运行的容器中找到可以被清理的，包括符合清理条件或者不被 kubelet 识别的容器
- 直接删除不能识别的容器，以及 pod 信息已经不存在的容器
- 根据配置的容器删除策略，对剩下的容器进行删除

## 镜像 GC 的代码

看过了容器 GC 的代码逻辑，我们再来看看镜像 GC 的逻辑。因为两者非常类似，这里略过参数的引用、对象的初始化和调用链分析，直接分析最核心的逻辑代码，这部分内容在 `pkg/kubelet/images/image_gc_manager.go` 文件中：

```go
type ImageGCManager interface {
	GarbageCollect() error

	// Start async garbage collection of images.
	Start() error

	GetImageList() ([]kubecontainer.Image, error)

	// Delete all unused images and returns the number of bytes freed. The number of bytes freed is always returned.
	DeleteUnusedImages() (int64, error)
}

// 镜像 GC 策略
type ImageGCPolicy struct {
	HighThresholdPercent int
	LowThresholdPercent int
	MinAge time.Duration
}

type realImageGCManager struct {
	runtime container.Runtime

	// 记录了当前使用的镜像
	imageRecords     map[string]*imageRecord
	imageRecordsLock sync.Mutex

	policy ImageGCPolicy
	
	cadvisor cadvisor.Interface
	recorder record.EventRecorder
	nodeRef *api.ObjectReference
	initialized bool

	// imageCache is the cache of latest image list.
	imageCache imageCache
}

type imageRecord struct {
	firstDetected time.Time
	lastUsed time.Time
	// Size of the image in bytes.
	size int64
}

func (im *realImageGCManager) Start() error {
	go wait.Until(func() {
		// Initial detection make detected time "unknown" in the past.
		var ts time.Time
		if im.initialized {
			ts = time.Now()
		}
		err := im.detectImages(ts)
		if err != nil {
			glog.Warningf("[imageGCManager] Failed to monitor images: %v", err)
		} else {
			im.initialized = true
		}
	}, 5*time.Minute, wait.NeverStop)

	// Start a goroutine periodically updates image cache.
	go wait.Until(func() {
		images, err := im.runtime.ListImages()
		if err != nil {
			glog.Warningf("[imageGCManager] Failed to update image list: %v", err)
		} else {
			im.imageCache.set(images)
		}
	}, 30*time.Second, wait.NeverStop)

	return nil
}

func (im *realImageGCManager) GarbageCollect() error {
	// 从 cadvisor 中获取镜像所在文件系统的信息，包括磁盘的容量和当前的使用量
	fsInfo, err := im.cadvisor.ImagesFsInfo()
	if err != nil {
		return err
	}
	capacity := int64(fsInfo.Capacity)
	available := int64(fsInfo.Available)
    ......

	// 如果镜像的磁盘使用率达到了设定的最高阈值，就进行清理工作，直到使用率
	usagePercent := 100 - int(available*100/capacity)
	if usagePercent >= im.policy.HighThresholdPercent {
		amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
		glog.Infof("[imageGCManager]: Disk usage on %q (%s) is at %d%% which is over the high threshold (%d%%). Trying to free %d bytes", fsInfo.Device, fsInfo.Mountpoint, usagePercent, im.policy.HighThresholdPercent, amountToFree)
		freed, err := im.freeSpace(amountToFree, time.Now())
		if err != nil {
			return err
		}

		if freed < amountToFree {
			err := fmt.Errorf("failed to garbage collect required amount of images. Wanted to free %d, but freed %d", amountToFree, freed)
			im.recorder.Eventf(im.nodeRef, api.EventTypeWarning, events.FreeDiskSpaceFailed, err.Error())
			return err
		}
	}

	return nil
}

func (im *realImageGCManager) DeleteUnusedImages() (int64, error) {
	return im.freeSpace(math.MaxInt64, time.Now())
}

func (im *realImageGCManager) freeSpace(bytesToFree int64, freeTime time.Time) (int64, error) {

    // 更新镜像记录列表中的数据，添加刚发现的镜像，移除已经不存在的镜像
	err := im.detectImages(freeTime)
    ......

	im.imageRecordsLock.Lock()
	defer im.imageRecordsLock.Unlock()

	// 根据镜像的最近使用时间和最近发现时间进行排序
	images := make([]evictionInfo, 0, len(im.imageRecords))
	for image, record := range im.imageRecords {
		images = append(images, evictionInfo{
			id:          image,
			imageRecord: *record,
		})
	}
	sort.Sort(byLastUsedAndDetected(images))

	// Delete unused images until we've freed up enough space.
	var deletionErrors []error
	spaceFreed := int64(0)
	for _, image := range images {
		......

		// 略过最近使用时间距离现在小于设置的 MinAge 的镜像
		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			continue
		}

		// 删除镜像并更新 imageRecords 对象中缓存的镜像信息，记录删除的镜像大小
		err := im.runtime.RemoveImage(container.ImageSpec{Image: image.id})
		if err != nil {
			deletionErrors = append(deletionErrors, err)
			continue
		}
		delete(im.imageRecords, image.id)
		spaceFreed += image.size

        // 如果删除的镜像大小满足需求，停止继续删除
		if spaceFreed >= bytesToFree {
			break
		}
	}

	if len(deletionErrors) > 0 {
		return spaceFreed, fmt.Errorf("wanted to free %d, but freed %d space with errors in image deletion: %v", bytesToFree, spaceFreed, errors.NewAggregate(deletionErrors))
	}
	return spaceFreed, nil
}
```

`realImageGCManager` 缓存了当前节点使用的镜像信息，并在 `Start()` 方法中启动两个 goroutine 周期性地去更新缓存的内容。`GarbageCollect` 的逻辑是这样的：

- 调用 cAdvisor 接口获取镜像所在磁盘的文件系统信息，根据当前的使用量和配置的 GC 策略确定是否需要进行清理
- 如果需要清理，计算需要清理的总大小，调用 `freeSpace` 进行镜像清理工作
- 把所有可以清理的镜像根据使用时间进行排序，进行逐个清理，知道清理的镜像总大小满足需求才停止

## 总结

容器和镜像 GC 的内容相对比较独立，而且逻辑也不是很复杂，源代码也很容易理解。但这部分内容却非常重要，影响到节点的正常工作状况，最后总结几点需要注意的地方：

- 默认情况下，container GC 是每分钟进行一次，image GC 是每五分钟一次，如果有不同的需要，可以通过 kubelet 的启动参数进行修改
- 不要手动清理镜像和容器，因为 kubelet 运行的时候会保存当前节点上镜像和容器的缓存，并定时更新。手动清理镜像和容器会让 kubelet 做出误判，带来不确定的问题
- 不是 kubelet 管理的容器不在 GC 的范围内，也就是说用户手动通过 docker 创建的容器不会被 kubelet 删除

**NOTE**：GC 机制后面会被 eviction 机制替代，可以在官方文档查看 [ eviction 的设计](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/kubelet-eviction.md)。

## 参考资料

- [Configuring kubelet Garbage Collection](https://kubernetes.io/docs/concepts/cluster-administration/kubelet-garbage-collection/)
- [Configuring Out Of Resource Handling](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)
- [Learn how kubelet eviction policies impact cluster rebalancing](https://blog.kublr.com/learn-how-kubelet-eviction-policies-impact-cluster-rebalancing-2e976ebc53ea)
- [谈 Kubernetes 如何控制 Node 的资源使用](https://kubernetes.cn/topics/48)
- [Kubernetes中的垃圾回收机制](http://www.cnblogs.com/openxxs/p/5275051.html)
- [Kubelet源码分析（三）：Garbage Collection](http://dockone.io/article/2134)
- [Kubelet - Eviction Policy Proposals Doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/kubelet-eviction.md)
- [Kubelet evictions - whats remaining?](https://github.com/kubernetes/kubernetes/issues/31362)