* [1\.背景](#1背景)
* [2\. HandlePodUpdates](#2-handlepodupdates)
  * [2\.1 syncPodFn](#21-syncpodfn)
  * [2\.2 kl\.killPod(pod, nil, podStatus, nil)](#22-klkillpodpod-nil-podstatus-nil)
  * [2\.3  kl\.containerRuntime\.KillPod(pod, p, gracePeriodOverride)](#23--klcontainerruntimekillpodpod-p-graceperiodoverride)
  * [2\.4 killContainersWithSyncResult 删除业务容器](#24-killcontainerswithsyncresult-删除业务容器)
    * [2\.4\.1 StopContainer](#241-stopcontainer)
  * [2\.5 StopPodSandbox](#25-stoppodsandbox)
  * [2\.6 总结](#26-总结)
* [3\. Pod是如何被删除的](#3-pod是如何被删除的)
  * [3\.1 SetPodStatus](#31-setpodstatus)
  * [3\.2 updateStatusInternal](#32-updatestatusinternal)
  * [3\.3 statusManager\.start](#33-statusmanagerstart)
  * [3\.4 m\.syncPod(syncRequest\.podUID, syncRequest\.status)](#34-msyncpodsyncrequestpoduid-syncrequeststatus)
  * [3\.4 总结](#34-总结)
* [4 kubelet监听到删除pod操作后做了什么操作](#4-kubelet监听到删除pod操作后做了什么操作)
  * [4\.1 HandlePodRemoves](#41-handlepodremoves)
  * [4\.2  kl\.deletePod](#42--kldeletepod)
  * [4\.3 podKiller处理 podKillingCh](#43-podkiller处理-podkillingch)
* [5\. 总结](#5-总结)

### 1.背景

当一个pod删除时，client端向apiserver发送请求，apiserver将pod的deletionTimestamp打上时间。kubelet watch到该事件，开始处理。所以一开始 kubele监听到的其实是 update事件。

所以通过分析kubelet delete其实也是分析了 apiserver更新pod流程。所以不单独写 apiserver更新pod，kubelet的流程处理了。

<br>

### 2. HandlePodUpdates

由于在分析pod创建流程的时候，将函数的流程都说明了。这里就不重复说明了，还是建议对比着看。

HandlePodUpdates也是调用了dispatchWork进行处理

dispatchWork调用UpdatePod

UpdatePod调用managePodLoop

managePodLoop调用syncPodFn

```
// dispatchWork调用UpdatePod
// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
				metrics.PodWorkerDuration.WithLabelValues(syncType.String()).Observe(metrics.SinceInSeconds(start))
				metrics.DeprecatedPodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	
	// UpdatePod调用managePodLoop
	p.managePodLoop(podUpdates)
	
	// managePodLoop调用syncPodFn进行处理
	err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			return err
```

<br>

这里注意一个点就是：在dispatchWork函数中, kubelet是不会执行kl.statusManager.TerminatePod(pod) 函数的。

因为podIsTerminated 判断pod是否terminated，并不是有了DeletionTimestamp就会认为是Terminated状态，而是有DeletionTimestamp且所有的容器不在运行了。

这个时候kubelet收到了pod update事件，并且有DeletionTimestamp，但是它container还是running的。

```
// dispatchWork starts the asynchronous sync of the pod in a pod worker.
// If the pod is terminated, dispatchWork
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
    
	if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			// If the pod is in a terminated state, there is no pod worker to
			// handle the work item. Check if the DeletionTimestamp has been
			// set, and force a status update to trigger a pod deletion request
			// to the apiserver.
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
				metrics.PodWorkerDuration.WithLabelValues(syncType.String()).Observe(metrics.SinceInSeconds(start))
				metrics.DeprecatedPodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	// Note the number of containers for new pods.
	if syncType == kubetypes.SyncPodCreate {
		metrics.ContainersPerPodCount.Observe(float64(len(pod.Spec.Containers)))
	}
}


并不是有了DeletionTimestamp就会认为是Terminated状态，而是有DeletionTimestamp且所有的容器不在运行了
// podIsTerminated returns true if pod is in the terminated state ("Failed" or "Succeeded").
func (kl *Kubelet) podIsTerminated(pod *v1.Pod) bool {
	// Check the cached pod status which was set after the last sync.
	status, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok {
		// If there is no cached status, use the status from the
		// apiserver. This is useful if kubelet has recently been
		// restarted.
		status = pod.Status
	}
	return status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(status.ContainerStatuses))
}
```

<br>

####  2.1 syncPodFn

```
在初始化syncPodFn函数的时候指定了函数为：syncPod。该函数的具体逻辑如下：

（1）如果是要kill pod，调用SetPodStatus设置状态，并且调用killPod

（2）如果是创建pod，先纪录pod的 firstSeenTime时间

（3）设置pod的 apiStatus，kubelet 监听得到的pod是没有status的。所以第一次创建的时候，kubelet会根据spec的内容，创建status，例如hostip, podid等等。

（4）如果pod已经running了，记录pod从 firstSeenTime到running的时间

（5）判断该Pod能否运行在这个节点上，如果不行给出原因，比如pid不够等原因

（6）更新statusManager中该pod status。查看下去最终就是调用了apisever client更新了pod状态

（7）如果pod不能运行在该node上，或者有DeletionTimestamp，或者Pod状态为failed，调用killpod函数删除该pod

（8）pod使用的不是hostNetwork，并且如果网络插件没有准备好，报错后返回

（9）如果pod没有被设置删除，并且是第一次出现，更新pod的cgroup。这里最终会调用func (m *qosContainerManagerImpl) setCPUCgroupConfig 函数设置 cpu/mem等cgruop

（10）如果是staticpod，创建mirrorpod

（11）为pod创建data目录。会在root-dir目录下创建 pods,以及pods/volume等目录。 默认的root-dir为 /var/lib/kubelet，可以通过--root-dir修改

（12）如果pod要删除，attach mount

（13）获取pod的secrets，并且调用containerRuntime.SyncPod同步
```

到这里了，还是不要被欺骗了，这里的updateType还是update，而不是kill。所以会跳过第一步。而是执行第7步。

```
	// Kill pod if it should not be running
	if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
		var syncErr error
		if err := kl.killPod(pod, nil, podStatus, nil); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
			syncErr = fmt.Errorf("error killing pod: %v", err)
			utilruntime.HandleError(syncErr)
		} else {
			if !runnable.Admit {
				// There was no error killing the pod, but the pod cannot be run.
				// Return an error to signal that the sync loop should back off.
				syncErr = fmt.Errorf("pod cannot be run: %s", runnable.Message)
			}
		}
		return syncErr
	}
```

<br>

#### 2.2 kl.killPod(pod, nil, podStatus, nil)

参数介绍：

pod： apiserver传下的将要删除的pod

runningPod:  nil

status: kubelet从runtime获得的真实pod状态

gracePeriodOverride： nil

该函数核心就是调用 kl.containerRuntime.KillPod(pod, p, gracePeriodOverride) 函数

```
// One of the following arguments must be non-nil: runningPod, status.
// TODO: Modify containerRuntime.KillPod() to accept the right arguments.
func (kl *Kubelet) killPod(pod *v1.Pod, runningPod *kubecontainer.Pod, status *kubecontainer.PodStatus, gracePeriodOverride *int64) error {
	var p kubecontainer.Pod
	if runningPod != nil {
		p = *runningPod
	} else if status != nil {
		p = kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), status)
	} else {
		return fmt.Errorf("one of the two arguments must be non-nil: runningPod, status")
	}

	// Call the container runtime KillPod method which stops all running containers of the pod
	if err := kl.containerRuntime.KillPod(pod, p, gracePeriodOverride); err != nil {
		return err
	}
	if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
		klog.V(2).Infof("Failed to update QoS cgroups while killing pod: %v", err)
	}
	return nil
}
```

#### 2.3  kl.containerRuntime.KillPod(pod, p, gracePeriodOverride)

KillPod 直接调用的是 killPodWithSyncResult。注意gracePeriodOverride=nil。 表示这一次是优雅删除，不是强制的。

```
// KillPod kills all the containers of a pod. Pod may be nil, running pod must not be.
// gracePeriodOverride if specified allows the caller to override the pod default grace period.
// only hard kill paths are allowed to specify a gracePeriodOverride in the kubelet in order to not corrupt user data.
// it is useful when doing SIGKILL for hard eviction scenarios, or max grace period during soft eviction scenarios.
func (m *kubeGenericRuntimeManager) KillPod(pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) error {
	err := m.killPodWithSyncResult(pod, runningPod, gracePeriodOverride)
	return err.Error()
}
```

<br>

killPodWithSyncResult 核心是：

（1）调用killContainersWithSyncResult 函数删除业务容器

（2）调用StopPodSandbox 函数停止sandbox容器。这里有一点，这里只是停止sandbox，说清理工作会由gc做

```
// killPodWithSyncResult kills a runningPod and returns SyncResult.
// Note: The pod passed in could be *nil* when kubelet restarted.
func (m *kubeGenericRuntimeManager) killPodWithSyncResult(pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) (result kubecontainer.PodSyncResult) {
	killContainerResults := m.killContainersWithSyncResult(pod, runningPod, gracePeriodOverride)
	for _, containerResult := range killContainerResults {
		result.AddSyncResult(containerResult)
	}

	// stop sandbox, the sandbox will be removed in GarbageCollect
	killSandboxResult := kubecontainer.NewSyncResult(kubecontainer.KillPodSandbox, runningPod.ID)
	result.AddSyncResult(killSandboxResult)
	// Stop all sandboxes belongs to same pod
	for _, podSandbox := range runningPod.Sandboxes {
		if err := m.runtimeService.StopPodSandbox(podSandbox.ID.ID); err != nil {
			killSandboxResult.Fail(kubecontainer.ErrKillPodSandbox, err.Error())
			klog.Errorf("Failed to stop sandbox %q", podSandbox.ID)
		}
	}

	return
}
```

<br>

#### 2.4 killContainersWithSyncResult 删除业务容器

这里就是异步调用m.killContainer，kill掉所有容器

```
// killContainersWithSyncResult kills all pod's containers with sync results.
func (m *kubeGenericRuntimeManager) killContainersWithSyncResult(pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) (syncResults []*kubecontainer.SyncResult) {
	containerResults := make(chan *kubecontainer.SyncResult, len(runningPod.Containers))
	wg := sync.WaitGroup{}

	wg.Add(len(runningPod.Containers))
	for _, container := range runningPod.Containers {
		go func(container *kubecontainer.Container) {
			defer utilruntime.HandleCrash()
			defer wg.Done()

			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, container.Name)
			if err := m.killContainer(pod, container.ID, container.Name, "", gracePeriodOverride); err != nil {
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
			}
			containerResults <- killContainerResult
		}(container)
	}
	wg.Wait()
	close(containerResults)

	for containerResult := range containerResults {
		syncResults = append(syncResults, containerResult)
	}
	return
}
```

<br>

killContainer的逻辑也很简单：

（1）在stop container之前运行 pre-stop的hooks命令

（2）stop container。如果gracePeriodOverride!=nil， 这里会将gracePeriod带到stop container函数

```
// killContainer kills a container through the following steps:
// * Run the pre-stop lifecycle hooks (if applicable).
// * Stop the container.
func (m *kubeGenericRuntimeManager) killContainer(pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, message string, gracePeriodOverride *int64) error {
	var containerSpec *v1.Container
	if pod != nil {
		if containerSpec = kubecontainer.GetContainerSpec(pod, containerName); containerSpec == nil {
			return fmt.Errorf("failed to get containerSpec %q(id=%q) in pod %q when killing container for reason %q",
				containerName, containerID.String(), format.Pod(pod), message)
		}
	} else {
		// Restore necessary information if one of the specs is nil.
		restoredPod, restoredContainer, err := m.restoreSpecsFromContainerLabels(containerID)
		if err != nil {
			return err
		}
		pod, containerSpec = restoredPod, restoredContainer
	}

	// From this point, pod and container must be non-nil.
	gracePeriod := int64(minimumGracePeriodInSeconds)
	switch {
	case pod.DeletionGracePeriodSeconds != nil:
		gracePeriod = *pod.DeletionGracePeriodSeconds
	case pod.Spec.TerminationGracePeriodSeconds != nil:
		gracePeriod = *pod.Spec.TerminationGracePeriodSeconds
	}

	if len(message) == 0 {
		message = fmt.Sprintf("Stopping container %s", containerSpec.Name)
	}
	m.recordContainerEvent(pod, containerSpec, containerID.ID, v1.EventTypeNormal, events.KillingContainer, message)

	// Run internal pre-stop lifecycle hook
	if err := m.internalLifecycle.PreStopContainer(containerID.ID); err != nil {
		return err
	}

	// Run the pre-stop lifecycle hooks if applicable and if there is enough time to run it
	if containerSpec.Lifecycle != nil && containerSpec.Lifecycle.PreStop != nil && gracePeriod > 0 {
		gracePeriod = gracePeriod - m.executePreStopHook(pod, containerID, containerSpec, gracePeriod)
	}
	// always give containers a minimal shutdown window to avoid unnecessary SIGKILLs
	if gracePeriod < minimumGracePeriodInSeconds {
		gracePeriod = minimumGracePeriodInSeconds
	}
	if gracePeriodOverride != nil {
		gracePeriod = *gracePeriodOverride
		klog.V(3).Infof("Killing container %q, but using %d second grace period override", containerID, gracePeriod)
	}

	klog.V(2).Infof("Killing container %q with %d second grace period", containerID.String(), gracePeriod)

	err := m.runtimeService.StopContainer(containerID.ID, gracePeriod)
	if err != nil {
		klog.Errorf("Container %q termination failed with gracePeriod %d: %v", containerID.String(), gracePeriod, err)
	} else {
		klog.V(3).Infof("Container %q exited normally", containerID.String())
	}

	m.containerRefManager.ClearRef(containerID)

	return err
}
```

<br>

##### 2.4.1 StopContainer

最终是调用了docker stop Container停止了容器。前面的gracePeriodOverride就是这里的超时参数。如果为nil，上面传入的默认超时是2s。

```
// StopContainer stops a running container with a grace period (i.e., timeout).
func (r *RemoteRuntimeService) StopContainer(containerID string, timeout int64) error {
	// Use timeout + default timeout (2 minutes) as timeout to leave extra time
	// for SIGKILL container and request latency.
	t := r.timeout + time.Duration(timeout)*time.Second
	ctx, cancel := getContextWithTimeout(t)
	defer cancel()

	r.logReduction.ClearID(containerID)
	_, err := r.runtimeClient.StopContainer(ctx, &runtimeapi.StopContainerRequest{
		ContainerId: containerID,
		Timeout:     timeout,
	})
	if err != nil {
		klog.Errorf("StopContainer %q from runtime service failed: %v", containerID, err)
		return err
	}

	return nil
}



// StopContainer stops a running container with a grace period (i.e., timeout).
func (ds *dockerService) StopContainer(_ context.Context, r *runtimeapi.StopContainerRequest) (*runtimeapi.StopContainerResponse, error) {
	err := ds.client.StopContainer(r.ContainerId, time.Duration(r.Timeout)*time.Second)
	if err != nil {
		return nil, err
	}
	return &runtimeapi.StopContainerResponse{}, nil
}
```

<br>

####  2.5 StopPodSandbox

StopPodSandbox最终调用的是 dockershim的StopPodSandbox。该函数逻辑如下：、

（1）docker inspect 获取元数据

（2）通过TearDownPod 清理网络，ip，网桥啥的

（3）stop sandboxcontainer

```
pkg/kubelet/dockershim/docker_sandbox.go
// StopPodSandbox stops the sandbox. If there are any running containers in the
// sandbox, they should be force terminated.
// TODO: This function blocks sandbox teardown on networking teardown. Is it
// better to cut our losses assuming an out of band GC routine will cleanup
// after us?
func (ds *dockerService) StopPodSandbox(ctx context.Context, r *runtimeapi.StopPodSandboxRequest) (*runtimeapi.StopPodSandboxResponse, error) {
	var namespace, name string
	var hostNetwork bool

	podSandboxID := r.PodSandboxId
	resp := &runtimeapi.StopPodSandboxResponse{}

	// Try to retrieve minimal sandbox information from docker daemon or sandbox checkpoint.
	// 1.docker inspect 获取元数据。这里看到了checkpoint的作用了，如果失败，还可以通过checkpint获取
	inspectResult, metadata, statusErr := ds.getPodSandboxDetails(podSandboxID)
	if statusErr == nil {
		namespace = metadata.Namespace
		name = metadata.Name
		hostNetwork = (networkNamespaceMode(inspectResult) == runtimeapi.NamespaceMode_NODE)
	} else {
		checkpoint := NewPodSandboxCheckpoint("", "", &CheckpointData{})
		checkpointErr := ds.checkpointManager.GetCheckpoint(podSandboxID, checkpoint)

		// Proceed if both sandbox container and checkpoint could not be found. This means that following
		// actions will only have sandbox ID and not have pod namespace and name information.
		// Return error if encounter any unexpected error.
		if checkpointErr != nil {
			if checkpointErr != errors.ErrCheckpointNotFound {
				err := ds.checkpointManager.RemoveCheckpoint(podSandboxID)
				if err != nil {
					klog.Errorf("Failed to delete corrupt checkpoint for sandbox %q: %v", podSandboxID, err)
				}
			}
			if libdocker.IsContainerNotFoundError(statusErr) {
				klog.Warningf("Both sandbox container and checkpoint for id %q could not be found. "+
					"Proceed without further sandbox information.", podSandboxID)
			} else {
				return nil, utilerrors.NewAggregate([]error{
					fmt.Errorf("failed to get checkpoint for sandbox %q: %v", podSandboxID, checkpointErr),
					fmt.Errorf("failed to get sandbox status: %v", statusErr)})
			}
		} else {
			_, name, namespace, _, hostNetwork = checkpoint.GetData()
		}
	}

	// WARNING: The following operations made the following assumption:
	// 1. kubelet will retry on any error returned by StopPodSandbox.
	// 2. tearing down network and stopping sandbox container can succeed in any sequence.
	// This depends on the implementation detail of network plugin and proper error handling.
	// For kubenet, if tearing down network failed and sandbox container is stopped, kubelet
	// will retry. On retry, kubenet will not be able to retrieve network namespace of the sandbox
	// since it is stopped. With empty network namespcae, CNI bridge plugin will conduct best
	// effort clean up and will not return error.
	errList := []error{}
	ready, ok := ds.getNetworkReady(podSandboxID)
	if !hostNetwork && (ready || !ok) {
		// Only tear down the pod network if we haven't done so already
		cID := kubecontainer.BuildContainerID(runtimeName, podSandboxID)
		err := ds.network.TearDownPod(namespace, name, cID)
		if err == nil {
			ds.setNetworkReady(podSandboxID, false)
		} else {
			errList = append(errList, err)
		}
	}
	if err := ds.client.StopContainer(podSandboxID, defaultSandboxGracePeriod); err != nil {
		// Do not return error if the container does not exist
		if !libdocker.IsContainerNotFoundError(err) {
			klog.Errorf("Failed to stop sandbox %q: %v", podSandboxID, err)
			errList = append(errList, err)
		} else {
			// remove the checkpoint for any sandbox that is not found in the runtime
			ds.checkpointManager.RemoveCheckpoint(podSandboxID)
		}
	}

	if len(errList) == 0 {
		return resp, nil
	}

	// TODO: Stop all running containers in the sandbox.
	return nil, utilerrors.NewAggregate(errList)
}
```

<br>

#### 2.6 总结

客户端删除（优雅删除）的情况下，apiserver只是update了deleteTimestamp。然后kubelete监听到了这个事件后停止了所有容器，清理了网络。

### 3. Pod是如何被删除的

从上面了解到客户端删除pod（一般是优雅删除），其实是apiserver给他打上了deleteTimestamp，这其实是一个update操作。

然后kubelet收到这个update后，就会进行上面的操作。stop了所有容器，清理了网络。

那pod是如何彻底被清除的呢？

<br>

在pleg的更新流程，我们可以知道。当所有容器被stop的时候，其实也会触发pleg的一次update操作。这个其实也会调用dispatchWork->UpdatePod->managePodLoop->syncPod。

而syncPod的逻辑有一个很重要的步骤SetPodStatus。详细流程可以参考 pod创建流程那一章节。

SetPodStatus核心是更新statusManger中Pod状态，但是如果pod deletetimeStamp!=nil 可能会调用apiserver删除pod操作。

```
（6）更新statusManager中该pod status，如果pod deletetimeStamp!=nil 可能会调用apiserver删除pod操作。
// Update status in the status manager
	kl.statusManager.SetPodStatus(pod, apiPodStatus)
```

<br>

#### 3.1 SetPodStatus

这里核心调用updateStatusInternal函数

```
func (m *manager) SetPodStatus(pod *v1.Pod, status v1.PodStatus) {
	m.podStatusesLock.Lock()
	defer m.podStatusesLock.Unlock()

	for _, c := range pod.Status.Conditions {
		if !kubetypes.PodConditionByKubelet(c.Type) {
			klog.Errorf("Kubelet is trying to update pod condition %q for pod %q. "+
				"But it is not owned by kubelet.", string(c.Type), format.Pod(pod))
		}
	}
	// Make sure we're caching a deep copy.
	status = *status.DeepCopy()

	// Force a status update if deletion timestamp is set. This is necessary
	// because if the pod is in the non-running state, the pod worker still
	// needs to be able to trigger an update and/or deletion.
	m.updateStatusInternal(pod, status, pod.DeletionTimestamp != nil)
}
```

#### 3.2 updateStatusInternal

updateStatusInternal的核心就是更新podStatus,然后往这个podStatusChannel发送状态

```
// updateStatusInternal updates the internal status cache, and queues an update to the api server if
// necessary. Returns whether an update was triggered.
// This method IS NOT THREAD SAFE and must be called from a locked function.
func (m *manager) updateStatusInternal(pod *v1.Pod, status v1.PodStatus, forceUpdate bool) bool {
	var oldStatus v1.PodStatus
	...
	m.podStatuses[pod.UID] = newStatus
  
  // 核心是往这个podStatusChannel发送状态
	select {
	case m.podStatusChannel <- podStatusSyncRequest{pod.UID, newStatus}:
		klog.V(5).Infof("Status Manager: adding pod: %q, with status: (%d, %v) to podStatusChannel",
			pod.UID, newStatus.version, newStatus.status)
		return true
	default:
		// Let the periodic syncBatch handle the update if the channel is full.
		// We can't block, since we hold the mutex lock.
		klog.V(4).Infof("Skipping the status update for pod %q for now because the channel is full; status: %+v",
			format.Pod(pod), status)
		return false
	}
}
```

<br>

#### 3.3 statusManager.start

在Kubelet.Run的时候, statusManager.start了起来

```
// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()
```

<br>

Start函数的核心就是用一个协程处理podStatusChannel的数据。

```
func (m *manager) Start() {
	// Don't start the status manager if we don't have a client. This will happen
	// on the master, where the kubelet is responsible for bootstrapping the pods
	// of the master components.
	if m.kubeClient == nil {
		klog.Infof("Kubernetes client is nil, not starting status manager.")
		return
	}

	klog.Info("Starting to sync pod status with apiserver")
	//lint:ignore SA1015 Ticker can link since this is only called once and doesn't handle termination.
	syncTicker := time.Tick(syncPeriod)
	// syncPod and syncBatch share the same go routine to avoid sync races.
	go wait.Forever(func() {
		select {
		case syncRequest := <-m.podStatusChannel:
			klog.V(5).Infof("Status Manager: syncing pod: %q, with status: (%d, %v) from podStatusChannel",
				syncRequest.podUID, syncRequest.status.version, syncRequest.status.status)
			m.syncPod(syncRequest.podUID, syncRequest.status)
		case <-syncTicker:
			m.syncBatch()
		}
	}, 0)
}
```

#### 3.4 m.syncPod(syncRequest.podUID, syncRequest.status)

m.syncPod核心逻辑如下：

（1）根据resourceVerison判断是否要更新

（2）从apiserver获得pod的最新数据

（3）调用apiserver接口更新podstatus

（4）如果pod canBeDeleted, 调用delete删除pod。只有这里的NewDeleteOptions(0)表示不要优雅删除

canBeDeleted的逻辑是同时满足以下条件：

* pod不能是mirrorPod，并且有DeletionTimestamp

* 没有容器运行

* volumes已经被清除

* cgroup已经被清除

```
// syncPod syncs the given status with the API server. The caller must not hold the lock.
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
  // 1.根据resourceVerison判断是否要更新
	if !m.needsUpdate(uid, status) {
		klog.V(1).Infof("Status for pod %q is up-to-date; skipping", uid)
		return
	}
  
  // 2.从apiserver获得pod的最新数据
	// TODO: make me easier to express from client code
	pod, err := m.kubeClient.CoreV1().Pods(status.podNamespace).Get(status.podName, metav1.GetOptions{})
	if errors.IsNotFound(err) {
		klog.V(3).Infof("Pod %q does not exist on the server", format.PodDesc(status.podName, status.podNamespace, uid))
		// If the Pod is deleted the status will be cleared in
		// RemoveOrphanedStatuses, so we just ignore the update here.
		return
	}
	if err != nil {
		klog.Warningf("Failed to get status for pod %q: %v", format.PodDesc(status.podName, status.podNamespace, uid), err)
		return
	}

	translatedUID := m.podManager.TranslatePodUID(pod.UID)
	// Type convert original uid just for the purpose of comparison.
	if len(translatedUID) > 0 && translatedUID != kubetypes.ResolvedPodUID(uid) {
		klog.V(2).Infof("Pod %q was deleted and then recreated, skipping status update; old UID %q, new UID %q", format.Pod(pod), uid, translatedUID)
		m.deletePodStatus(uid)
		return
	}
  
  // 3.调用apiserver接口更新podstatus
	oldStatus := pod.Status.DeepCopy()
	newPod, patchBytes, err := statusutil.PatchPodStatus(m.kubeClient, pod.Namespace, pod.Name, pod.UID, *oldStatus, mergePodStatus(*oldStatus, status.status))
	klog.V(3).Infof("Patch status for pod %q with %q", format.Pod(pod), patchBytes)
	if err != nil {
		klog.Warningf("Failed to update status for pod %q: %v", format.Pod(pod), err)
		return
	}
	pod = newPod

	klog.V(3).Infof("Status for pod %q updated successfully: (%d, %+v)", format.Pod(pod), status.version, status.status)
	m.apiStatusVersions[kubetypes.MirrorPodUID(pod.UID)] = status.version
  
  // 4.如果pod canBeDeleted, 调用delete删除pod。只有这里的NewDeleteOptions(0)表示不要优雅删除
	// We don't handle graceful deletion of mirror pods.
	if m.canBeDeleted(pod, status.status) {
		deleteOptions := metav1.NewDeleteOptions(0)
		// Use the pod UID as the precondition for deletion to prevent deleting a newly created pod with the same name and namespace.
		deleteOptions.Preconditions = metav1.NewUIDPreconditions(string(pod.UID))
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(pod.Name, deleteOptions)
		if err != nil {
			klog.Warningf("Failed to delete status for pod %q: %v", format.Pod(pod), err)
			return
		}
		klog.V(3).Infof("Pod %q fully terminated and removed from etcd", format.Pod(pod))
		m.deletePodStatus(uid)
	}
}
```

而canBeDeleted的逻辑是什么样子的呢？

```
func (m *manager) canBeDeleted(pod *v1.Pod, status v1.PodStatus) bool {
	if pod.DeletionTimestamp == nil || kubetypes.IsMirrorPod(pod) {
		return false
	}
	return m.podDeletionSafety.PodResourcesAreReclaimed(pod, status)
}
```

可以看出来canBeDeleted的逻辑是同时满足以下条件：

（1）pod不能是mirrorPod，并且有DeletionTimestamp

（2）没有容器运行

（3)   volumes已经被清除

（4）cgroup已经被清除

```
// PodResourcesAreReclaimed returns true if all required node-level resources that a pod was consuming have
// been reclaimed by the kubelet.  Reclaiming resources is a prerequisite to deleting a pod from the API server.
func (kl *Kubelet) PodResourcesAreReclaimed(pod *v1.Pod, status v1.PodStatus) bool {
   if !notRunning(status.ContainerStatuses) {
      // We shouldn't delete pods that still have running containers
      klog.V(3).Infof("Pod %q is terminated, but some containers are still running", format.Pod(pod))
      return false
   }
   // pod's containers should be deleted
   runtimeStatus, err := kl.podCache.Get(pod.UID)
   if err != nil {
      klog.V(3).Infof("Pod %q is terminated, Error getting runtimeStatus from the podCache: %s", format.Pod(pod), err)
      return false
   }
   if len(runtimeStatus.ContainerStatuses) > 0 {
      var statusStr string
      for _, status := range runtimeStatus.ContainerStatuses {
         statusStr += fmt.Sprintf("%+v ", *status)
      }
      klog.V(3).Infof("Pod %q is terminated, but some containers have not been cleaned up: %s", format.Pod(pod), statusStr)
      return false
   }
   if kl.podVolumesExist(pod.UID) && !kl.keepTerminatedPodVolumes {
      // We shouldn't delete pods whose volumes have not been cleaned up if we are not keeping terminated pod volumes
      klog.V(3).Infof("Pod %q is terminated, but some volumes have not been cleaned up", format.Pod(pod))
      return false
   }
   if kl.kubeletConfiguration.CgroupsPerQOS {
      pcm := kl.containerManager.NewPodContainerManager()
      if pcm.Exists(pod) {
         klog.V(3).Infof("Pod %q is terminated, but pod cgroup sandbox has not been cleaned up", format.Pod(pod))
         return false
      }
   }
   return true
}
```

#### 3.4 总结

等容器清理完后，通过pleg的同步，在判断容器停止，volume清理后，最终会往apiserver发送一个 强制删除pod的请求。这个时候apiserver才会往etcd调用删除pod的操作。

### 4 kubelet监听到删除pod操作后做了什么操作

参考 kubelet监听pod变化那一章节。可以知道apiserver 从etcd删除了pod数据。kubelet最终会受到一个remove的update。这个对应HandlePodRemoves函数

#### 4.1 HandlePodRemoves

核心逻辑：

（1）调用podManager.DeletePod，主要是从secretManager, configMapManager，checkpointManager等等地方删除这个pod，告诉他们这个pod真的不在了

（2）调用kl.deletePod

（3）从probeManager中移除该pod

```
// HandlePodRemoves is the callback in the SyncHandler interface for pods
// being removed from a config source.
func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.DeletePod(pod)
		if kubetypes.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}
		// Deletion is allowed to fail because the periodic cleanup routine
		// will trigger deletion again.
		if err := kl.deletePod(pod); err != nil {
			klog.V(2).Infof("Failed to delete pod %q, err: %v", format.Pod(pod), err)
		}
		kl.probeManager.RemovePod(pod)
	}
}
```

<br>

#### 4.2  kl.deletePod

该函数核心就是：

（1）停掉对应的PodWorker

（2）发送到 kl.podKillingCh <- &podPair

```
// deletePod deletes the pod from the internal state of the kubelet by:
// 1.  stopping the associated pod worker asynchronously
// 2.  signaling to kill the pod by sending on the podKillingCh channel
//
// deletePod returns an error if not all sources are ready or the pod is not
// found in the runtime cache.
func (kl *Kubelet) deletePod(pod *v1.Pod) error {
	...
	// 1.停掉对应的PodWorker
	kl.podWorkers.ForgetWorker(pod.UID)

  
  ...
	// 2.发送到podKillingCh
	podPair := kubecontainer.PodPair{APIPod: pod, RunningPod: &runningPod}
  
  
	kl.podKillingCh <- &podPair
	// TODO: delete the mirror pod here?

	// We leave the volume/directory cleanup to the periodic cleanup routine.
	return nil
}
```

<br>

#### 4.3 podKiller处理 podKillingCh

podKiller就是调用killpod函数处理。这个和2节是一样的。做的是删除容器等操作。就不再展开了。

如果pod 强制删除，那其实是没有delete操作，而是直接remove。所以这一步也是要的。

```
// podKiller launches a goroutine to kill a pod received from the channel if
// another goroutine isn't already in action.
func (kl *Kubelet) podKiller() {
	killing := sets.NewString()
	// guard for the killing set
	lock := sync.Mutex{}
	for podPair := range kl.podKillingCh {
		runningPod := podPair.RunningPod
		apiPod := podPair.APIPod

		lock.Lock()
		exists := killing.Has(string(runningPod.ID))
		if !exists {
			killing.Insert(string(runningPod.ID))
		}
		lock.Unlock()

		if !exists {
			go func(apiPod *v1.Pod, runningPod *kubecontainer.Pod) {
				klog.V(2).Infof("Killing unwanted pod %q", runningPod.Name)
				// 调用killpod函数处理
				err := kl.killPod(apiPod, runningPod, nil, nil)
				if err != nil {
					klog.Errorf("Failed killing the pod %q: %v", runningPod.Name, err)
				}
				lock.Lock()
				killing.Delete(string(runningPod.ID))
				lock.Unlock()
			}(apiPod, runningPod)
		}
	}
}
```

### 5. 总结

Kubelet 其实会进行2次删除。第一次是pod更新时间，有deleteTimeStamp，对应了 delete

第二次是pod被删除了，执行了remove操作。

通过一个实例来总结整个的删除过程：下面是删除1个pod对应的kubelet日志和分析过程

```
root:/home/zoux# tail -f /home/zoux/log/kubelet.stderr.log | grep 'SyncLoop' | grep test-pod2

// 1、有了deleteTime所以是删除事件
I0310 11:44:02.640238  731714 kubelet.go:1932] SyncLoop (DELETE, "api"): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)"

// 2. SYNC 定时触发了一次同步
I0310 11:44:03.768615  731714 kubelet.go:1980] SyncLoop (SYNC): 1 pods; test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)

// 3. 应该是业务容器died，然后触发了pleg的同步
I0310 11:44:33.496057  731714 kubelet.go:1961] SyncLoop (PLEG): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)", event: &pleg.PodLifecycleEvent{ID:"9cf9dae8-5c99-43e1-8ff8-78e766e176db", Type:"ContainerDied", Data:"9a0d86b06f0558b73633e3e05efebc1bbd23f6c227e13e276552f96387aa2357"}

// 4. 应该是sandbox died，然后触发了pleg的同步
I0310 11:44:33.496180  731714 kubelet.go:1961] SyncLoop (PLEG): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)", event: &pleg.PodLifecycleEvent{ID:"9cf9dae8-5c99-43e1-8ff8-78e766e176db", Type:"ContainerDied", Data:"4a65fcda2151b053f61ae08688fcf5adacc041ef672bf2d0fb31284882823e88"}

// 5.apiserver 触发了第一次同步，应该是1的更新status导致 （status还不同步）
I0310 11:44:33.504168  731714 kubelet.go:1929] SyncLoop (RECONCILE, "api"): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)"

// 6. apiserver 触发了第2次同步，应该是2的更新status导致 （status还不同步）
I0310 11:44:33.512140  731714 kubelet.go:1929] SyncLoop (RECONCILE, "api"): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)"

// 7.应该是容器都died了，这个时候status也同步了，但是有deleteTImeStamp，所有又是一次delete
I0310 11:44:34.533461  731714 kubelet.go:1932] SyncLoop (DELETE, "api"): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)"

// 8.etcd没有这个数据了，所以remove
I0310 11:44:34.535965  731714 kubelet.go:1926] SyncLoop (REMOVE, "api"): "test-pod2_default(9cf9dae8-5c99-43e1-8ff8-78e766e176db)"
```

