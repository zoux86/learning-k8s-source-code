* [1\. 背景](#1-背景)
* [2\. pleg\.Start](#2-plegstart)
* [3\.syncLoop](#3syncloop)
  * [3\.1 syncLoopIteration 相关channel介绍](#31-syncloopiteration-相关channel介绍)
    * [3\.1\.1 configCh](#311-configch)
    * [3\.1\.2 plegCh](#312-plegch)
    * [3\.1\.3 syncCh](#313-syncch)
    * [3\.1\.4 houseKeepingCh](#314-housekeepingch)
    * [3\.1\.5 livenessManager\.Updates](#315-livenessmanagerupdates)
    * [3\.1\.6 SyncHandler](#316-synchandler)
  * [3\.2 syncLoopIteration源码分析](#32-syncloopiteration源码分析)
    * [3\.2\.1 case1\-configCh](#321-case1-configch)
    * [3\.2\.2 case2\-plegCh](#322-case2-plegch)
    * [3\.2\.3 case3\-syncCh](#323-case3-syncch)
    * [3\.2\.4 case4\-livenessManager\.Updates](#324-case4-livenessmanagerupdates)
    * [3\.2\.5 housekeepingCh](#325-housekeepingch)
* [4\. 总结](#4-总结)

### 1. 背景

书接上文，在Kubelet.Run函数中，通过pleg.Start和kl.syncLoop来处理pod，所有本文从这里开始，从源码角度了解pod创建的整个过程。

```
kl.pleg.Start()
kl.syncLoop(updates, kl)
```

### 2. pleg.Start

pleg.Start每隔1秒，运行以此relist函数，relist的逻辑如下：

1. 记录上一次relist的时间和间隔
2. 通过runtimeApi获取所有的pod，包括exit的Pod
3. 更新pods container状态,以及记录pod数量等metrics
4. 和旧的Pod进行对比，podRecord结构体保存了旧的，和当前pod的信息，可以理解为和1s前的所有pods进行对比。对比完产生event，保存在一个map中。这里主要产生的事件为：ContainerStarted，ContainerDie, ContainerRemoved, ContainerChanged等等
5. 如果event和Pod有绑定，并且kubelet开启了cache缓存pod信息，根据最新的信息同步缓存
6. 将新的record赋值为旧的，为下一轮做准备，然后依次处理event，逻辑为：不是ContainerChanged状态的event都发送到eventChannel中去
7. 更新缓存，如果有更新失败的，记录到needsReinspection，表示下一次还需要重试

这里需要注意的是：查看generateEvents函数，可以发现只有当新的container状态为plegContainerUnknown，才会产生ContainerChanged。这就是发送eventa为什么会跳过ContainerChanged的原因。

```
// Start spawns a goroutine to relist periodically.
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}

每隔1s进行以此g.relist
plegRelistPeriod = time.Second * 1


// relist queries the container runtime for list of pods/containers, compare
// with the internal pods/containers, and generates events accordingly.
func (g *GenericPLEG) relist() {
	klog.V(5).Infof("GenericPLEG: Relisting")
  
  // 1.记录上一次relist的时间和间隔
	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
		metrics.DeprecatedPLEGRelistInterval.Observe(metrics.SinceInMicroseconds(lastRelistTime))
	}
  
	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
		metrics.DeprecatedPLEGRelistLatency.Observe(metrics.SinceInMicroseconds(timestamp))
	}()

	// Get all the pods.
	// 2. 通过runtimeApi获取所有的pod，包括exit的Pod
	podList, err := g.runtime.GetPods(true)
	if err != nil {
		klog.Errorf("GenericPLEG: Unable to retrieve pods: %v", err)
		return
	}

	g.updateRelistTime(timestamp)
  
  // 3.更新pods container状态,以及记录有哪些pod
	pods := kubecontainer.Pods(podList)
	// update running pod and container count
	updateRunningPodAndContainerMetrics(pods)
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	// 4.和旧的Pod进行对比，podRecord保存了旧的，和当前pod的信息。可以理解为和1s前的所有pods进行对比
	// type podRecord struct {
	//    old     *kubecontainer.Pod
	//    current *kubecontainer.Pod
  //  }
	// 这里主要产生的事件为：ContainerStarted，ContainerDie, ContainerRemoved, ContainerChanged等等，然后保存在一个map中
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	// 5.如果event和Pod有绑定，并且kubelet开启了cache缓存pod信息，根据最新的信息同步缓存
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err := g.updateCache(pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).Infof("PLEG: Ignoring events for pod %s/%s: %v", pod.Name, pod.Namespace, err)

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
			}
		}
		// Update the internal storage and send out the events.
		// 6. 将新的record赋值为旧的，为下一轮做准备，然后依次处理event，逻辑为：不是ContainerChanged状态的event都发送到eventChannel中去
		g.podRecords.update(pid)
		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.WithLabelValues().Inc()
				klog.Error("event channel is full, discard this relist() cycle event")
			}
		}
	}
  
  // 7.更新缓存，如果有更新失败的，记录到needsReinspection，表示下一次还需要重试
	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).Infof("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err := g.updateCache(pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).Infof("PLEG: pod %s/%s failed reinspection: %v", pod.Name, pod.Namespace, err)
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	g.podsToReinspect = needsReinspection
}
```

<br>

从generateEvents可以看出来，这里产生的event和k8s中的event不同。这里指的是PodLifecycleEvent。

```
func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
	if newState == oldState {
		return nil
	}

	klog.V(4).Infof("GenericPLEG: %v/%v: %v -> %v", podID, cid, oldState, newState)
	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:
			// We already reported that the container died before.
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", newState))
	}
}
```

<br>

### 3.syncLoop

从函数的注释了解到：syncLoop是核心的同步逻辑。它监听file, apiserver, and http三个channel的变化，然后进行期望状态和当前状态的同步。

syncLoop核心是调用了`syncLoopIteration`的函数来执行更具体的监控pod变化的循环。

```
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
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
  
  // for循环调用syncLoopIteration
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

		// add by gzchenyifan
		if _, err := kl.nodeLister.Get(string(kl.nodeName)); err != nil {
			klog.Errorf("skipping pod synchronization until get nodeInfo from apiserver success - %v", err)
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

#### 3.1 syncLoopIteration 相关channel介绍

`syncLoopIteration`主要通过几种`channel`来对不同类型的事件进行监听并处理。其中包括：`configCh`、`plegCh`、`syncCh`、`houseKeepingCh`、`livenessManager.Updates()`。

`syncLoopIteration`实际执行了pod的操作，此部分设置了几种不同的channel:

- `configCh`：将配置更改的pod分派给事件类型的相应处理程序回调。
- `plegCh`：更新runtime缓存，同步pod。
- `syncCh`：同步所有等待同步的pod。
- `houseKeepingCh`：触发清理pod。
- `livenessManager.Updates()`：对失败的pod或者liveness检查失败的pod进行sync操作。

<br>

##### 3.1.1 configCh

在NewMainKubelet函数中有一个重要的步骤, 就是makePodSourceConfig。

```
if kubeDeps.PodConfig == nil {
		var err error
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, bootstrapCheckpointPath)
		if err != nil {
			return nil, err
		}
	}
```

makePodSourceConfig的核心逻辑如下：

1. 如果StaticPodPath不为空，调用NewSourceFile监听该目录下定义的Pod,并且将他们发送到cfg.Channel中去

2. 如果url不为空，从url监听获取Pod, 并且将他们发送到cfg.Channel中去
3. 如果kubeclient不为空，从apiserver监听pod，并且发送都cfg.channel中去

```
// makePodSourceConfig creates a config.PodConfig from the given
// KubeletConfiguration or returns an error.
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, bootstrapCheckpointPath string) (*config.PodConfig, error) {
	manifestURLHeader := make(http.Header)
	if len(kubeCfg.StaticPodURLHeader) > 0 {
		for k, v := range kubeCfg.StaticPodURLHeader {
			for i := range v {
				manifestURLHeader.Add(k, v[i])
			}
		}
	}

	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder)
  
  // 1.如果StaticPodPath不为空，调用NewSourceFile监听该目录下定义的Pod,并且将他们发送到cfg.Channel中去
	// define file config source
	if kubeCfg.StaticPodPath != "" {
		klog.Infof("Adding pod path: %v", kubeCfg.StaticPodPath)
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(kubetypes.FileSource))
	}

	// define url config source
	// 2.如果url不为空，从url监听获取Pod, 并且将他们发送到cfg.Channel中去
	if kubeCfg.StaticPodURL != "" {
		klog.Infof("Adding pod url %q with HTTP header %v", kubeCfg.StaticPodURL, manifestURLHeader)
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(kubetypes.HTTPSource))
	}

	// Restore from the checkpoint path
	// NOTE: This MUST happen before creating the apiserver source
	// below, or the checkpoint would override the source of truth.

	var updatechannel chan<- interface{}
	if bootstrapCheckpointPath != "" {
		klog.Infof("Adding checkpoint path: %v", bootstrapCheckpointPath)
		updatechannel = cfg.Channel(kubetypes.ApiserverSource)
		err := cfg.Restore(bootstrapCheckpointPath, updatechannel)
		if err != nil {
			return nil, err
		}
	}
   
   // 3.如果kubeclient不为空，从apiserver监听pod，并且发送都cfg.channel中去
	if kubeDeps.KubeClient != nil {
		klog.Infof("Watching apiserver")
		if updatechannel == nil {
			updatechannel = cfg.Channel(kubetypes.ApiserverSource)
		}
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, updatechannel)
	}
	return cfg, nil
}
```

**所以：** update这个channel是从远端传过来的期望状态。这个channel就是configCh

##### 3.1.2 plegCh

```
plegCh := kl.pleg.Watch()

就是Pleg.Start发送的那个eventChannel
func (g *GenericPLEG) Watch() chan *PodLifecycleEvent {
	return g.eventChannel
}
```

**所以**：plegCh就是本地当前pod的状态

##### 3.1.3 syncCh

```
syncTicker := time.NewTicker(time.Second)
```

每秒一次往该channel发送数据。手动触发同步

##### 3.1.4 houseKeepingCh

默认值housekeepingPeriod=2s。每2秒一次往该channel发送数据。手动触发同步

```
housekeepingTicker := time.NewTicker(housekeepingPeriod)
```

<br>

##### 3.1.5 livenessManager.Updates

在NewMainKubelet的时候就定义了。不用想，就是livenessManager定期探测pod，有失败的将往这个channel发送。

##### 3.1.6 SyncHandler

```
// SyncHandler is an interface implemented by Kubelet, for testability
type SyncHandler interface {
	HandlePodAdditions(pods []*v1.Pod)
	HandlePodUpdates(pods []*v1.Pod)
	HandlePodRemoves(pods []*v1.Pod)
	HandlePodReconcile(pods []*v1.Pod)
	HandlePodSyncs(pods []*v1.Pod)
	HandlePodCleanups() error
}
```

SyncHandler`是一个定义Pod的不同Handler的接口，具体是实现者是`kubelet. 这里直接将kubelet对象传了进去。

```
kl.syncLoop(updates, kl)
```

具体在pkg/kubelet/kubelet.go中。

<br>

#### 3.2 syncLoopIteration源码分析

了解了syncLoopIteration函数的核心参数，接下来就很好分析syncLoopIteration的逻辑了。

syncLoopIteration整体结构是通过 select 监听每个channel，然后执行对应的操作。因为是调用者syncLoop是for循环调用的。所以syncLoopIteration会一直被调用执行。

<br>

```
// syncLoopIteration reads from various channels and dispatches pods to the
// given handler.
//
// Arguments:
// 1.  configCh:       a channel to read config events from
// 2.  handler:        the SyncHandler to dispatch pods to
// 3.  syncCh:         a channel to read periodic sync events from
// 4.  housekeepingCh: a channel to read housekeeping events from
// 5.  plegCh:         a channel to read PLEG updates from
//
// Events are also read from the kubelet liveness manager's update channel.
//
// The workflow is to read from one of the channels, handle that event, and
// update the timestamp in the sync loop monitor.
//
// Here is an appropriate place to note that despite the syntactical
// similarity to the switch statement, the case statements in a select are
// evaluated in a pseudorandom order if there are multiple channels ready to
// read from when the select is evaluated.  In other words, case statements
// are evaluated in random order, and you can not assume that the case
// statements evaluate in order if multiple channels have events.
//
// With that in mind, in truly no particular order, the different channels
// are handled as follows:
//
// * configCh: dispatch the pods for the config change to the appropriate
//             handler callback for the event type
// * plegCh: update the runtime cache; sync pod
// * syncCh: sync all pods waiting for sync
// * housekeepingCh: trigger cleanup of pods
// * liveness manager: sync pods that have failed or in which one or more
//                     containers have failed liveness checks
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletionTimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.RESTORE:
			klog.V(2).Infof("SyncLoop (RESTORE, %q): %q", u.Source, format.Pods(u.Pods))
			// These are pods restored from the checkpoint. Treat them as new
			// pods.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.Errorf("Kubelet does not support snapshot update")
		}

		if u.Op != kubetypes.RESTORE {
			// If the update type is RESTORE, it means that the update is from
			// the pod checkpoints and may be incomplete. Do not mark the
			// source as ready.

			// Mark the source ready after receiving at least one update from the
			// source. Once all the sources are marked ready, various cleanup
			// routines will start reclaiming resources. It is important that this
			// takes place only after kubelet calls the update handler to process
			// the update to ensure the internal pod cache is up-to-date.
			kl.sourcesReady.AddSource(u.Source)
		}
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.

			// We should not use the pod from livenessManager, because it is never updated after
			// initialization.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				klog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			klog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			klog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				klog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
	return true
}
```

##### 3.2.1 case1-configCh

这个channel有数据，说用pod的配置发生了改变。具体处理逻辑如下：

（1）如果是 ADD 或者RESTORE，表明是新创建了一个Pod，调用 HandlePodAdditions函数处理

（2）如果是UPDATE，调用HandlePodUpdates处理

（3）如果是REMOVE，调用HandlePodRemoves处理。remove和delete不一样，remove表示第二次删除

（4）如果是RECONCILE，调用HandlePodReconcile处理。RECONCILE表示pod有修改，但是状态还没同步, 需要协调

（5）如果是DELETE，调用HandlePodUpdates处理。因为这是第一次删除，只是赋值了Pod.deletationTimestamp而已

后面根据pod创建，删除，更新等具体的情况进行分析

```
case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletionTimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.RESTORE:
			klog.V(2).Infof("SyncLoop (RESTORE, %q): %q", u.Source, format.Pods(u.Pods))
			// These are pods restored from the checkpoint. Treat them as new
			// pods.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.Errorf("Kubelet does not support snapshot update")
		}

		if u.Op != kubetypes.RESTORE {
			// If the update type is RESTORE, it means that the update is from
			// the pod checkpoints and may be incomplete. Do not mark the
			// source as ready.

			// Mark the source ready after receiving at least one update from the
			// source. Once all the sources are marked ready, various cleanup
			// routines will start reclaiming resources. It is important that this
			// takes place only after kubelet calls the update handler to process
			// the update to ensure the internal pod cache is up-to-date.
			kl.sourcesReady.AddSource(u.Source)
		}
```

##### 3.2.2 case2-plegCh

该channel表示底层container的状态发生了改变。核心逻辑如下：

（1）如果pod的状态发生了改变，并且Pod还存在，调用HandlePodSyncs进行同步

（2）如果是ContainerDied, 清理pod的容器。具体是调用apiserver接口更新Podstatus，然后再清理这个容器

```
case e := <-plegCh:
    // return event.Type != pleg.ContainerRemoved
    // 只要不是containerRemoved,都有同步的意义
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}
   
    // 如果是ContainerDied, 清理pod的容器。具体是调用apiserver接口更新Podstatus，然后再清理这个容器
 		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
```

<br>

##### 3.2.3 case3-syncCh

这个channel是每1s调用1次。核心逻辑如下：

（1）调用getPodsToSync得到需要同步的pod列表。 （pod中有一个参数activeDeadlineSeconds 可以设置 Pod 最长的运行时间。如果pod的存活时间>activeDeadlineSeconds, 表示这个pod需要同步 ）

（2）调用HandlePodSyncs同步pod

```
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		handler.HandlePodSyncs(podsToSync)
```

<br>

##### 3.2.4 case4-livenessManager.Updates

这个channel来自于 livenessManager，核心功能也很明显。就是对探测失败的pod，调用HandlePodSyncs同步一下状态。

```
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.

			// We should not use the pod from livenessManager, because it is never updated after
			// initialization.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				klog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			klog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
```

##### 3.2.5 housekeepingCh

这个channel每2秒更新一次数据，然后调用HandlePodCleanups进行清理工作

```
case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			klog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				klog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
```

<br>

### 4. 总结

本章节主要分析了kubelet的 syncLoop函数。他的核心逻辑如下：

死循环处理一下五个channel的数据：

（1）来自configCh的数据。这个数据来源于apisever/url/file，表示pod的期望值已经更新了，然后根据不同类型的type(add ,del, update等)进行不同的处理

（2）来自plegCh的数据。这个数据来源于底层的数据，表示真实的pod状态已经发生了改变，调用HandlePodSyncs同步状态

（3）来自syncCh的数据。这个数据来源于定时数，每1秒运行一次。找出pod运行时间已经大于activeDeadlineSeconds值，需要调用HandlePodSyncs同步状态

（4）来自livenessManager.Update的数据。这个数据来源于livenessManager探针的结果。如果有pod探针失败，调用HandlePodSyncs同步数据

（5）来自housekeepingCh的数据，这个数据来源于定时数，每2秒运行1次。调用HandlePodCleanups进行数据清理

至此，kubelet的核心流程分析完了。接下来的针对每个场景进行具体分析。比如Pod的创建流程，HandlePodCleanups是如何工作的。
