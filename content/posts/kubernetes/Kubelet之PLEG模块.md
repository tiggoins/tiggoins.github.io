---
title: Kubelet | PLEG模块
date: 2024-07-02 07:30:08
tags:
 - Kubernetes
categories:
 - Kubernetes
toc: true
---
本文介绍了kubelet重要的pleg模块，引入了默认的通用PLEG`GenericPLEG`及基于事件的PLEG`EventedPLEG`。
<!--more-->

版本信息：

- kubelet：v1.20.11/v1.27.6

### PLEG是什么

PLEG的全称是`Pod Lifecycle Event Generator`，即pod生命周期事件生成器。我们知道kubelet是管理节点上pod的重要守护进程，主要完成pod实际状态与pod期望状态的reconcile(调谐循环)，为此kubelet需要对pod规格及容器状态的变更做出相应的动作。kubelet可以接收来自apiserver、http以及file方向的更新(后两者主要用于Static Pod)，当发现实际pod当前运行状态与期望状态或者pod的`specification`与现存的不一致时就会触发进一步的处理动作，而获取运行状态的方式就是通过PLEG模块来实现的，可以说PLEG是kubelet中非常核心的模块，一旦工作状态异常就会影响到kubelet的正常工作。

![pleg](https://github.com/kubernetes/community/raw/4026287dc3a2d16762353b62ca2fe4b80682960a/contributors/design-proposals/node/pleg.png)

如上图，PLEG模块位于kubelet中的位置及工作流程：

1. kubelet从apiserver、file及http中获取pod配置的更新；
2. kubelet为每个pod创建一个worker，每个worker负责处理pod的创建及删除；
3. PLEG不断调用container runtime的status API获取container状态，基于此产生事件发送给kubelet处理；
4. PLEG也可从container runtime中直接获取容器状态，如docker的[eventAPI](https://docs.docker.com/engine/api/v1.43/#tag/System/operation/SystemEvents)。这部分用于后面的`EventedPLEG`。



### PLEG的工作逻辑

PLEG的主要逻辑位于`pkg/kubelet/pleg`目录下，主要代码为`generic.go`及`pleg.go`，代码量不多。

其中在`pleg.go`中定义了Pod生命周期事件结构体及`PodLifecycleEventGenerator`接口。

```go
// PodLifeCycleEventType define the event type of pod life cycle events.
type PodLifeCycleEventType string

const (
	// ContainerStarted - event type when the new state of container is running.
	ContainerStarted PodLifeCycleEventType = "ContainerStarted"
	// ContainerDied - event type when the new state of container is exited.
	ContainerDied PodLifeCycleEventType = "ContainerDied"
	// ContainerRemoved - event type when the old state of container is exited.
	ContainerRemoved PodLifeCycleEventType = "ContainerRemoved"
	// PodSync is used to trigger syncing of a pod when the observed change of
	// the state of the pod cannot be captured by any single event above.
	PodSync PodLifeCycleEventType = "PodSync"
	// ContainerChanged - event type when the new state of container is unknown.
	ContainerChanged PodLifeCycleEventType = "ContainerChanged"
)

// PodLifecycleEvent is an event that reflects the change of the pod state.
type PodLifecycleEvent struct {
	// The pod ID.
	ID types.UID
	// The type of the event.
	Type PodLifeCycleEventType
	// The accompanied data which varies based on the event type.
	//   - ContainerStarted/ContainerStopped: the container name (string).
	//   - All other event types: unused.
	Data interface{}
}

// PodLifecycleEventGenerator contains functions for generating pod life cycle events.
type PodLifecycleEventGenerator interface {
	Start()
	Watch() chan *PodLifecycleEvent
	Healthy() (bool, error)
}
```

`generic.go`中的`GenericPLEG`实现了`PodLifecycleEventGenerator`接口。其中`GenericPLEG`中公开的方法包括`Healthy()`、`Watch()`及`Start()`，其中`Start()`是用于启动`GenericPLEG`的逻辑，`Healthy()`用于检查PLEG的工作状态，`Watch()`用于返回eventChannel事件管道用于其他位置处理。



#### Start()

```go
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```

`wait.Until`的函数原型为`func wait.Until(f func(), period time.Duration, stopCh <-chan struct{})`，即每隔一段时间执行一次f，直到stopCh中有值，该值硬编码为1s。在此处为每秒执行`g.relist`方法，并且一直执行永不返回。

如下为relist方法的逻辑如下，删除了一些非重要的逻辑：

```go
func (g *GenericPLEG) relist() {
    // 获取当前时间
	timestamp := g.clock.Now()  
	// 调用container runtime获取所有pod
	podList, err := g.runtime.GetPods(true)
    // 将当前时间更新到GenericPLEG的relistTime字段，该字段类型是`atomic.Value`
	g.updateRelistTime(timestamp)
    // 将podList转换为kubecontainer.Pods类型
	pods := kubecontainer.Pods(podList)
    // 存到podRecords的current中
	g.podRecords.setCurrent(pods)
    // 比较old及current的pods，生成事件填充到eventsByPodID的map中
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// 获取所有的container，内部使用了sets去重
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
            // 根据pod等信息生成出事件，比如pod的state是`plegContainerRunning`，那么生成出的事件的类型即为`ContainerStarted`
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}
    // 创建需要重新探测的map映射
	var needsReinspection map[types.UID]*kubecontainer.Pod
	for pid, events := range eventsByPodID {
        // 从podRecords中获取所有的pod
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache是执行探查的主逻辑
			if err := g.updateCache(pod, pid); err != nil {
                // 如果失败则将pod放到needsReinspection中
				needsReinspection[pid] = pod
				continue
			} else {
                // 如果成功则从podsToReinspect中删除该pod
				delete(g.podsToReinspect, pid)
			}
		}
		// Update the internal storage and send out the events.
		g.podRecords.update(pid)
		for i := range events {
			select {
               // 将产生的事件发送到eventChannel中，供外部使用
			case g.eventChannel <- events[i]:
			default:
                // 如果eventChannel已满，则将`pleg_discard_events`指标+1，并且产生一条错误
				metrics.PLEGDiscardEvents.Inc()
				klog.Error("event channel is full, discard this relist() cycle event")
			}
		}
	}
	if g.cacheEnabled() {
		// 如果podsToReinspect非空则遍历继续执行updateCache，如果失败仍然放到needsReinspection中
		if len(g.podsToReinspect) > 0 {
			for pid, pod := range g.podsToReinspect {
                // 对需要重新inspect的再次尝试
				if err := g.updateCache(pod, pid); err != nil {
					needsReinspection[pid] = pod
				}
			}
		}	
        // 更新缓存时间戳
		g.cache.UpdateTime(timestamp)
	}
	// 将needsReinspection赋值给genericpleg的podsToReinspect字段供下次循环使用
	g.podsToReinspect = needsReinspection
}
```

relist不断的调用`container runtime`的API获取所有的pod，然后调用`updatecache`获取各个pod的工作状态，其主要调用了`g.runtime.GetPodStatus`，而`GetPodStatus`方法如下：

```go
func (m *kubeGenericRuntimeManager) GetPodStatus(uid kubetypes.UID, name, namespace string) (*kubecontainer.PodStatus, error) {
	// 获取sandboxID
	podSandboxIDs, err := m.getSandboxIDByPodUID(uid, nil)
	sandboxStatuses := make([]*runtimeapi.PodSandboxStatus, len(podSandboxIDs))
	podIPs := []string{}
	for idx, podSandboxID := range podSandboxIDs {
        // 获取所有sandbox container的状态
		podSandboxStatus,_ := m.runtimeService.PodSandboxStatus(podSandboxID)
		sandboxStatuses[idx] = podSandboxStatus
	}
	// 获取所有container的状态
	containerStatuses, err := m.getPodContainerStatuses(uid, name, namespace)
	return &kubecontainer.PodStatus{
		ID:                uid,
		Name:              name,
		Namespace:         namespace,
		IPs:               podIPs,
		SandboxStatuses:   sandboxStatuses,
		ContainerStatuses: containerStatuses,
	}, nil
}
```

主要逻辑是调用`container runtime`获取pod的`sandbox`及所有`container`的状态，将结果包在`PodStatus`结构体中返回。



#### Healthy

```go
func (g *GenericPLEG) Healthy() (bool, error) {
	relistTime := g.getRelistTime()
	if relistTime.IsZero() {
		return false, fmt.Errorf("pleg has yet to be successful")
	}
	// Expose as metric so you can alert on `time()-pleg_last_seen_seconds > nn`
	metrics.PLEGLastSeen.Set(float64(relistTime.Unix()))
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}
```

用于检查PLEG模块的健康状态，主要流程为：

1. 获取上一次执行relist()的时间，即`GenericPLEG.relistTime`字段的值；
2. 为`pleg_last_seen_seconds`指标赋值，该指标可以通过Prometheus进行查询并配置告警；
3. 如果从上次执行relist已经过去的时间大于了`relistThreshold`，则在将健康状态设置为false并且在日志中产生一条错误信息。`relistThreshold`的默认值为3分钟：

```go
// The threshold needs to be greater than the relisting period + the
// relisting time, which can vary significantly. Set a conservative
// threshold to avoid flipping between healthy and unhealthy.
relistThreshold = 3 * time.Minute
```

结合以上，PLEG通过`Start()`方法不断地调用relist()获取pod的状态，每次执行relist前设置relistTime字段，通过Healthy()方法查询上次执行`relist`的时间`relistTime`，如果已经距离上次relist过去了3分钟，则打印一条错误日志。relist默认是每1s执行一次，但是由于节点的container数量可能较多，一次relist可能花费较多的时间，但总体上是一直不断地执行`relist`以保证pod状态是最新的。而`relist`的主要的逻辑是先找出所有的pods，然后查询出`sandboxStatuses`和`containerStatuses`汇总成`PodStatus`，可以理解为不断地执行`docker ps`找出所有的container。然后执行`docker inspect`查看各个container的状态信息。

向上分析调用`Healthy()`的调用位置，在`NewMainKubelet`初始化Kubelet逻辑中将PLEG的健康检查添加到runtimeState的healthChecks切片中。

```go
klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
klet.runtimeState = newRuntimeState(maxWaitForContainerRuntime)
klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
```

同时在`runtimeErrors()`中遍历各个healthchecks进行了调用，对其中的错误聚合后返回。

```go
func (s *runtimeState) runtimeErrors() error {
	for _, hc := range s.healthChecks {
		if ok, err := hc.fn(); !ok {
			errs = append(errs, fmt.Errorf("%s is not healthy: %v", hc.name, err))
		}
	}
	if s.runtimeError != nil {
		errs = append(errs, s.runtimeError)
	}

	return utilerrors.NewAggregate(errs)
}
```

而`runtimeErrors()`又在`syncLoop()`和`kubelet_node_status`两处位置被调用，下面分别分析这两种情况。



#### SyncLoop()

`syncLoop()`是kubelet中处理更新的主循环，代码如下：

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.Info("Starting kubelet main sync loop.")
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	// Responsible for checking limits in resolv.conf
	// The limits do not have anything to do with individual pods
	// Since this is called in syncLoop, we don't need to call it anywhere else
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}

	for {
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.Errorf("skipping pod synchronization - %v", err)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

在`syncLoop`函数中不停地检查PLEG模块的健康状态，如果某次检查时出现错误则等待默认100ms后开始下次检查，每次出错等待时间增大2倍，最大值为5s。如果检查成功则将睡眠时间重置为100ms，该函数只有在`kubetypes.PodUpdate`channel关闭时才退出。如果`runtimeErrors()`执行一直出现错误，比如某个`container`卡住无法获取其详细信息，会导致一直循环而后续的`syncLoopIteration`方法无法执行到，而`syncLoopIteration`是分发更新事件到各个处理机的方法，也就是某个container僵死可能会导致kubelet无法处理pod的更新。



#### kubelet_node_status()

而在`kubelet_node_status`中，`runtimeErrors`作为一个参数被添加到`nodestatus.ReadyCondition`函数中，该函数中检查PLEG的工作状态，如果有错误则将创建新的`NodeReadyCondition`，在此结构体中将`Status`置为`false`，

```go
if aggregatedErr := errors.NewAggregate(errs); aggregatedErr != nil {
			newNodeReadyCondition = v1.NodeCondition{
				Type:              v1.NodeReady,
				Status:            v1.ConditionFalse,
				Reason:            "KubeletNotReady",
				Message:           aggregatedErr.Error(),
				LastHeartbeatTime: currentTime,
			}
		}
```

当condition中type为ready类型的status字段变为unknown时kubelet中会产生一条事件：`status is now: NodeNotReady`，并且在其日志中输出一条warning：`Node became not ready`。而将node的status字段置为unkown是由kube-controller-manager中的`node_lifecycle_controller`完成的，当节点在默认的`40s`没有上报状态时将节点condition的各个type中的status均置为unknown。

```go
readyConditionUpdated := false
needToRecordEvent := false
for i := range node.Status.Conditions {
	if node.Status.Conditions[i].Type == v1.NodeReady {
		if node.Status.Conditions[i].Status == newNodeReadyCondition.Status {
			newNodeReadyCondition.LastTransitionTime = node.Status.Conditions[i].LastTransitionTime
		} else {
			newNodeReadyCondition.LastTransitionTime = currentTime
			needToRecordEvent = true
		}
		node.Status.Conditions[i] = newNodeReadyCondition
		readyConditionUpdated = true
		break
	}
}
if !readyConditionUpdated {
	newNodeReadyCondition.LastTransitionTime = currentTime
	node.Status.Conditions = append(node.Status.Conditions, newNodeReadyCondition)
}
if needToRecordEvent {
	if newNodeReadyCondition.Status == v1.ConditionTrue {
		recordEventFunc(v1.EventTypeNormal, events.NodeReady)
	} else {
		recordEventFunc(v1.EventTypeNormal, events.NodeNotReady)
		klog.Infof("Node became not ready: %+v", newNodeReadyCondition)
	}
}
return nil
```

节点的conditions信息参考如下：

```yaml
conditions:
  - lastHeartbeatTime: "2024-06-26T00:22:18Z"
    lastTransitionTime: "2024-06-25T06:14:04Z"
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: "2024-06-26T00:22:18Z"
    lastTransitionTime: "2024-06-25T06:14:04Z"
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: "2024-06-26T00:22:18Z"
    lastTransitionTime: "2024-06-25T06:14:04Z"
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
  - lastHeartbeatTime: "2024-06-26T00:22:18Z"
    lastTransitionTime: "2024-06-25T06:14:04Z"
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
```

继续向上分析`ReadyCondition`的调用栈，其被添加到`defaultNodeStatusFuncs()`中

```go
var setters []func(n *v1.Node) error

func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
	...
	setters = append(setters,
		nodestatus.MemoryPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderMemoryPressure, kl.recordNodeStatusEvent),
		nodestatus.DiskPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderDiskPressure, kl.recordNodeStatusEvent),
		nodestatus.PIDPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderPIDPressure, kl.recordNodeStatusEvent),
		nodestatus.ReadyCondition(kl.clock.Now, kl.runtimeState.runtimeErrors, kl.runtimeState.networkErrors, kl.runtimeState.storageErrors, validateHostFunc, kl.containerManager.Status, kl.shutdownManager.ShutdownStatus, kl.recordNodeStatusEvent),
		nodestatus.VolumesInUse(kl.volumeManager.ReconcilerStatesHasBeenSynced, kl.volumeManager.GetVolumesInUse),
		kl.recordNodeSchedulableEvent,
	)
	return setters
}
```

在kubelet的初始化方法`NewMainKubelet`中，将`defaultNodeStatusFuncs()`的执行结果赋值给`setNodeStatusFuncs`

```
klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()
```

`setNodeStatusFuncs`在`setNodeStatus`中被遍历执行，以此来判断节点的工作状态是否正常。

```go
func (kl *Kubelet) setNodeStatus(node *v1.Node) {
	for i, f := range kl.setNodeStatusFuncs {
		klog.V(5).Infof("Setting node status at position %v", i)
		if err := f(node); err != nil {
			klog.Errorf("Failed to set some node status fields: %s", err)
		}
	}
}
```

继续向上的调用栈比较简单直接列出

```
go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
      |
      |
kl.updateNodeStatus() 
      | 
      | 
kl.tryUpdateNodeStatus() [重试5次调用tryUpdateNodeStatus]
      |
      |
kl.setNodeStatus(node)
```

`nodeStatusUpdateFrequency`默认值为10s，即每隔10s执行一次`syncNodeStatus`方法。

```go
if obj.NodeStatusUpdateFrequency == zeroDuration {
	obj.NodeStatusUpdateFrequency = metav1.Duration{Duration: 10 * time.Second}
}
```

综合分析，kubelet每隔`10s`计算一次节点的工作状态，发生错误则进行重试，最多重试5次。若kubelet在默认的40s没有上报信息时，则将节点标记为Unknown，在到达默认的`300s`后对节点上运行的pod进行驱逐。



### 监控与告警

PLEG的异常工作可能会导致kubelet无法处理pod更新，节点状态NotReady导致其上的pod被驱逐引发业务的不稳定。应该将PLEG的工作状态纳入到prometheus监控中，必要的时候发出警告手动进行干预。具体的指标及含义如下:

|            指标名            |   类型    |         含义         | 阈值 |
| :--------------------------: | :-------: | :------------------: | :--: |
| pleg_relist_duration_seconds | Histogram |   执行relist的时长   |  -   |
|     pleg_discard_events      |  Counter  | pleg中丢弃的事件计数 |  >0  |
| pleg_relist_interval_seconds | Histogram |   执行relist的间隔   |  -   |
|    pleg_last_seen_seconds    |   Gauge   |  pleg上次活跃的时间  |  3m  |



### EventedPLEG

以上叙述的内容均是`GenericPLEG`的实现，可以看出为了确保能及时获取pod的状态，kubelet需要不停的执行relist()，这个过程消耗了大量的系统资源，并且随着节点的容器数量增加还在继续增长。

> Polling incurs non-negligible overhead as the number of pods/containers increases, and is exacerbated by Kubelet's parallelism -- one worker (goroutine) per pod, which queries the container runtime individually. Periodic, concurrent, large number of requests causes high CPU usage spikes (even when there is no spec/state change), poor performance, and reliability problems due to overwhelmed container runtime. Ultimately, it limits Kubelet's scalability.                                    

自1.26开始，kubelet中引入了名为`Evented PLEG`的`FeatureGate`，该[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/3386-kubelet-evented-pleg/README.md)期望通过直接从`Container Runtime`的`EventAPI`获取`Event`事件，相比于`GenericPLEG`而言不需要频繁的执行`relist()`，并且对于pod变化的感知也比`relist()`要快。自1.27版本后默认开启。



#### CRI更新

[新增了`GetContainerEvents`方法](https://github.com/kubernetes/kubernetes/pull/111642)，由kubelet发起名为`GetContainerEvents`的`RPC`调用，`Container Runtime`返回的是响应kubelet的信息流，可以理解为`Container Runtime`开启了到达`kubelet`的一个通道，可以通过这个通道持续的发送数据。

```protobuf
// GetContainerEvents gets container events from the CRI runtime
rpc  GetContainerEvents(GetEventsRequest) returns (stream ContainerEventResponse) {}

message GetEventsRequest {}

message ContainerEventResponse {
    // ID of the container
    string container_id = 1;
    // Type of the container event
    ContainerEventType container_event_type = 2;
    // Creation timestamp of this event
    int64 created_at = 3;
    // ID of the sandbox container
    PodSandboxMetadata pod_sandbox_metadata = 4;
}

enum ContainerEventType {
    // Container created
    CONTAINER_CREATED_EVENT = 0;
    // Container started
    CONTAINER_STARTED_EVENT = 1;
    // Container stopped
    CONTAINER_STOPPED_EVENT = 2;
    // Container deleted
    CONTAINER_DELETED_EVENT = 3;
}
```



#### 初始化逻辑

在kubelet的初始化方法`NewMainKubelet中`，如果启用了`EventedPLEG`的`Feature`则同时开启`Evented PLEG`及`GenericPLEG`否则只初始化`GenericPLEG`：

```go
klet.podCache = kubecontainer.NewCache()
...
if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
	// adjust Generic PLEG relisting period and threshold to higher value when Evented PLEG is turned on
	genericRelistDuration := &pleg.RelistDuration{
		RelistPeriod:    eventedPlegRelistPeriod, // 300s
		RelistThreshold: eventedPlegRelistThreshold, // 10m
	}
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, eventChannel, genericRelistDuration, klet.podCache, clock.RealClock{})
	// In case Evented PLEG has to fall back on Generic PLEG due to an error,
	// Evented PLEG should be able to reset the Generic PLEG relisting duration
	// to the default value.
	eventedRelistDuration := &pleg.RelistDuration{
		RelistPeriod:    genericPlegRelistPeriod, // 1s
		RelistThreshold: genericPlegRelistThreshold, // 3m
	}
	klet.eventedPleg = pleg.NewEventedPLEG(klet.containerRuntime, klet.runtimeService, eventChannel,
		klet.podCache, klet.pleg, eventedPlegMaxStreamRetries, eventedRelistDuration, clock.RealClock{})
} else {
	genericRelistDuration := &pleg.RelistDuration{
		RelistPeriod:    genericPlegRelistPeriod,
		RelistThreshold: genericPlegRelistThreshold,
	}
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, eventChannel, genericRelistDuration, klet.podCache, clock.RealClock{})
}
```

情况一：启用了`EventedPLEG`

- 初始化`GenericPLEG`，该配置下将每隔300s执行一次`Relist`，并且在10m没有执行relist后报错;
- `GenericPLEG`的实例作为参数传递给`EventedPLEG`执行初始化，在初始化`EventedPLEG`时也定义了一个`RelistDuration`（每秒执行一次reslit()，3分钟没有执行则产生错误），但是`EventedPLEG`不需要执行reslit，这个`RelistDuration`后面再叙述;
- 两个PLEG实例共享同一个podCache。

情况二：未启用`EventedPLEG`

- 与未改动前无异



#### Start()

```go
// Start starts the Evented PLEG
func (e *EventedPLEG) Start() {
	e.runningMu.Lock()
	defer e.runningMu.Unlock()
	if isEventedPLEGInUse() {
		return
	}
	setEventedPLEGUsage(true)
	e.stopCh = make(chan struct{})
	e.stopCacheUpdateCh = make(chan struct{})
	go wait.Until(e.watchEventsChannel, 0, e.stopCh)
	go wait.Until(e.updateGlobalCache, globalCacheUpdatePeriod, e.stopCacheUpdateCh)
}
```

可以看到`EventedPLEG`的`Satrt()`方法主要执行两个方法，其中`updateGlobalCache`每隔5s执行一次，目的是更新podCache中的全局时间，避免pod workers中pod卡在执行`GetNewerThan`方法中，不是主要方法。

```go
// In case the Evented PLEG experiences undetectable issues in the underlying
// GRPC connection there is a remote chance the pod might get stuck in a
// given state while it has progressed in its life cycle. This function will be
// called periodically to update the global timestamp of the cache so that those
// pods stuck at GetNewerThan in pod workers will get unstuck.
func (e *EventedPLEG) updateGlobalCache() {
	e.cache.UpdateTime(time.Now())
}
```

下面主要看`watchEventsChannel()`方法。



#### watchEventsChannel()

```go
func (e *EventedPLEG) watchEventsChannel() {
	containerEventsResponseCh := make(chan *runtimeapi.ContainerEventResponse, cap(e.eventChannel))
	defer close(containerEventsResponseCh)

	// Get the container events from the runtime.
	go func() {
		numAttempts := 0
		for {
			if numAttempts >= e.eventedPlegMaxStreamRetries {
				if isEventedPLEGInUse() {
					// Fall back to Generic PLEG relisting since Evented PLEG is not working.
					klog.V(4).InfoS("Fall back to Generic PLEG relisting since Evented PLEG is not working")
					e.Stop()
					e.genericPleg.Stop()       // Stop the existing Generic PLEG which runs with longer relisting period when Evented PLEG is in use.
					e.Update(e.relistDuration) // Update the relisting period to the default value for the Generic PLEG.
					e.genericPleg.Start()
					break
				}
			}

			err := e.runtimeService.GetContainerEvents(containerEventsResponseCh)
			if err != nil {
				metrics.EventedPLEGConnErr.Inc()
				numAttempts++
				e.Relist() // Force a relist to get the latest container and pods running metric.
				klog.V(4).InfoS("Evented PLEG: Failed to get container events, retrying: ", "err", err)
			}
		}
	}()

	if isEventedPLEGInUse() {
		e.processCRIEvents(containerEventsResponseCh)
	}
}
```

`watchEventsChannel()`方法的主要逻辑比较简单：

1. 创建一个`containerEventsResponseCh`管道，用于存放`Container Event`;
2. 开启Goroutine，在其中定义了`numAttempts`，默认值为0;
3. 通过`GetContainerEvents`获取`Container Event`并存储到`containerEventsResponseCh`管道中;
4. 调用`processCRIEvents`进一步处理。

- 如果在执行`GetContainerEvents`时发生错误，则将指标`evented_pleg_connection_error_count`计数器+1，`numAttempts`+1，并且强制执行一次`Relist()`以获取最新的变化。

- 当`numAttempts`达到`eventedPlegMaxStreamRetries`之后(默认值为5次)，同时关闭`EventedPLEG`和`GenericPLEG`，通过`Update()`将`GenericPLEG`中的`ReslitDuration`改为每秒执行一次，阈值3分钟，重启`GenericPLEG`。

综上，如果重试的次数超过了5次就会导致`EventedPLEG`被关闭，并且切换`GenericPLEG`到默认的频率(1s/3m)执行。不得不说这部分的设计比较简单，假设某场景下需要暂时停用`Container Runtime`就会导致`numAttempts`迅速达到5次，在此之后`EventedPLEG`就会退出工作不再启用，想要重新启用就必须重启kubelet，而PLEG处于哪种状态需要级别较低的日志才能看到。



#### cache.Set

在初始化`GenericPLEG`及`EventedPLEG`时会使用同一份podCache，可能会存在竞态，为此需要对`Set`操作加锁并且验证时间戳：传入的时间戳小于缓存的修改时间就会直接放弃直接返回false。

```go
// Set sets the PodStatus for the pod only if the data is newer than the cache
func (c *cache) Set(id types.UID, status *PodStatus, err error, timestamp time.Time) (updated bool) {
	c.lock.Lock()
	defer c.lock.Unlock()

	if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
		// Set the value in the cache only if it's not present already
		// or the timestamp in the cache is older than the current update timestamp
		if cachedVal, ok := c.pods[id]; ok && cachedVal.modified.After(timestamp) {
			return false
		}
	}

	c.pods[id] = &data{status: status, err: err, modified: timestamp}
	c.notify(id, timestamp)
	return true
}
```



#### 总结

`EventedPLEG`对原有的`GenericPLEG`依赖性较强，即使启用了`EventedPLEG`，`GenericPLEG`也会以较慢的速率(300s/10m)运行，并且在底层的gRPC连接失败时也会完全回退到`GenericPLEG`。
