* [1\. 背景](#1-背景)
* [2\. StartGarbageCollection](#2-startgarbagecollection)
* [3\. container gc处理流程](#3-container-gc处理流程)
  * [3\.1 gc 参数设置](#31-gc-参数设置)
  * [3\.2 GarbageCollect](#32-garbagecollect)
  * [3\.3 containerGC\.GarbageCollect](#33-containergcgarbagecollect)
    * [3\.2\.1 移除需要驱逐的的containers](#321-移除需要驱逐的的containers)
    * [3\.2\.2 移除sandboxes](#322-移除sandboxes)
    * [3\.2\.3 回收log Directories](#323-回收log-directories)
* [4\. Image Gc处理流程](#4-image-gc处理流程)
  * [4\.1 freeSpace](#41-freespace)
* [5 总结](#5-总结)

### 1. 背景

上文分析到，所有的容器都是stop停了，但是没有清理。这个清理工作就是GC做的。在kubelet初始化的时createAndInitKubelet函数中，开启了gc 流程。接下里看看GC流程的处理逻辑。

```
cmd/kubelet/app/server.go
createAndInitKubelet

k.StartGarbageCollection()
```

<br>

### 2. StartGarbageCollection

StartGarbageCollection逻辑如下：

（1） 开启一个协程进行container的GC处理。间隔时间1分钟，ContainerGCPeriod=1 min

（2）判断是否开了了image gc。HighThresholdPercent表示磁盘使用量超过多少开始GC

（3）如果开了了image gc，协程进行image的GC处理。间隔时间5分钟，ImageGCPeriod=5 min

可以看到，kubelet的核心就是contianer Gc 和image gc

```
// StartGarbageCollection starts garbage collection threads.
func (kl *Kubelet) StartGarbageCollection() {
	loggedContainerGCFailure := false
	
	// 1.开启一个协程进行container的GC处理。间隔时间1分钟，ContainerGCPeriod=1 min
	go wait.Until(func() {
		if err := kl.containerGC.GarbageCollect(); err != nil {
			klog.Errorf("Container garbage collection failed: %v", err)
			kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ContainerGCFailed, err.Error())
			loggedContainerGCFailure = true
		} else {
			var vLevel klog.Level = 4
			if loggedContainerGCFailure {
				vLevel = 1
				loggedContainerGCFailure = false
			}

			klog.V(vLevel).Infof("Container garbage collection succeeded")
		}
	}, ContainerGCPeriod, wait.NeverStop)
  
  
 
	// when the high threshold is set to 100, stub the image GC manager
	// 2.判断是否开了了image gc。HighThresholdPercent表示磁盘使用量超过多少开始GC。
	if kl.kubeletConfiguration.ImageGCHighThresholdPercent == 100 {
		klog.V(2).Infof("ImageGCHighThresholdPercent is set 100, Disable image GC")
		return
	}
  
  // 3.开启image gc
	prevImageGCFailed := false
	go wait.Until(func() {
		if err := kl.imageManager.GarbageCollect(); err != nil {
			if prevImageGCFailed {
				klog.Errorf("Image garbage collection failed multiple times in a row: %v", err)
				// Only create an event for repeated failures
				kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ImageGCFailed, err.Error())
			} else {
				klog.Errorf("Image garbage collection failed once. Stats initialization may not have completed yet: %v", err)
			}
			prevImageGCFailed = true
		} else {
			var vLevel klog.Level = 4
			if prevImageGCFailed {
				vLevel = 1
				prevImageGCFailed = false
			}

			klog.V(vLevel).Infof("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)
}
```

<br>

### 3. container gc处理流程

#### 3.1 gc 参数设置

ContainerGCPolicy 结构如下：

 **MinAge** 对应 kubelet的启动参数`--minimum-container-ttl-duration`， 表示已经退出的容器可以存活的最小时间，默认为 0s。

**MaxPerPodContainer** 对应kubelet的启动参数  `--maximum-dead-containers-per-container`表示一个 pod 最多可以保存多少个已经停止的容器，默认为1；

 **MaxContainers** 对应kubele启动参数   `--maximum-dead-containers`一个 node 上最多可以保留多少个已经停止的容器，默认为 -1，表示没有限制；

```
// Specified a policy for garbage collecting containers.
type ContainerGCPolicy struct {
	// Minimum age at which a container can be garbage collected, zero for no limit.
	MinAge time.Duration

	// Max number of dead containers any single pod (UID, container name) pair is
	// allowed to have, less than zero for no limit.
	MaxPerPodContainer int

	// Max number of total dead containers, less than zero for no limit.
	MaxContainers int
}
```

<br>

#### 3.2 GarbageCollect

调用链为：GarbageCollect -> runtime.GarbageCollect -> containerGC.GarbageCollect

```
func (cgc *realContainerGC) GarbageCollect() error {
	return cgc.runtime.GarbageCollect(cgc.policy, cgc.sourcesReadyProvider.AllReady(), false)
}

// GarbageCollect removes dead containers using the specified container gc policy.
func (m *kubeGenericRuntimeManager) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	return m.containerGC.GarbageCollect(gcPolicy, allSourcesReady, evictNonDeletedPods)
}
```

<br>

#### 3.3 containerGC.GarbageCollect

该函数主要逻辑为：
 （1）移除需要驱逐的的containers

* 得到需要所有需要驱逐的contaienrs，非running已经挂掉 MinAge分钟了

* 根据参数MaxPerPodContainer,保留最新的MaxPerPodContainer个containers，其他的都要驱逐

* 根据MaxContainers参数，保留最新的MaxContainers个contaeinrs，其他的按照创建时间依次驱逐

 （2）移除sandboxes

* 获取 node 上所有的 container以及所有的 sandboxes

* 收集所有 container 的 PodSandboxId， 构建 sandboxes 与 pod 的对应关系并将其保存在 sandboxesByPodUID 中

* 遍历 sandboxesByPod，若 sandboxes 所在的 pod 处于 deleted 状态，则删除该 pod 中所有的 sandboxes ；否则只保留退出时间最短的一个 sandboxes

 （3）回收log Directories

* 首先回收 deleted 状态 pod logs dir，遍历 pod logs dir /var/log/pods，/var/log/pods 为 pod logs 的默认目录，pod logs dir 的格式为 /var/log/pods/NAMESPACE_NAME_UID，解析 pod logs dir 获取 pod uid，判断 pod 是否处于 deleted 状态，若处于 deleted 状态则删除其 logs dir； 

* 回收 deleted 状态 container logs 链接目录，/var/log/containers 为 container log 的默认目录，其会软链接到 pod 的 log dir 下，例如： /var/log/containers/storage-provisioner_kube-system_storage-provisioner-acc8386e409dfb3cc01618cbd14c373d8ac6d7f0aaad9ced018746f31d0081e2.log -> /var/log/pods/kube-system_storage-provisioner_b448e496-eb5d-4d71-b93f-ff7ff77d2348/storage-provisioner/0.log

```
// GarbageCollect removes dead containers using the specified container gc policy.
// Note that gc policy is not applied to sandboxes. Sandboxes are only removed when they are
// not ready and containing no containers.
//
// GarbageCollect consists of the following steps:
// * gets evictable containers which are not active and created more than gcPolicy.MinAge ago.
// * removes oldest dead containers for each pod by enforcing gcPolicy.MaxPerPodContainer.
// * removes oldest dead containers by enforcing gcPolicy.MaxContainers.
// * gets evictable sandboxes which are not ready and contains no containers.
// * removes evictable sandboxes.
func (cgc *containerGC) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictTerminatedPods bool) error {
   errors := []error{}
   // Remove evictable containers
   if err := cgc.evictContainers(gcPolicy, allSourcesReady, evictTerminatedPods); err != nil {
      errors = append(errors, err)
   }

   // Remove sandboxes with zero containers
   if err := cgc.evictSandboxes(evictTerminatedPods); err != nil {
      errors = append(errors, err)
   }

   // Remove pod sandbox log directory
   if err := cgc.evictPodLogsDirectories(allSourcesReady); err != nil {
      errors = append(errors, err)
   }
   return utilerrors.NewAggregate(errors)
}
```

<br>

##### 3.2.1 移除需要驱逐的的containers

（1）得到需要所有需要驱逐的contaienrs，非running已经挂掉 MinAge分钟了

（2）根据参数MaxPerPodContainer,保留最新的MaxPerPodContainer个containers，其他的都要驱逐

（3）根据MaxContainers参数，保留最新的MaxContainers个contaeinrs，其他的按照创建时间依次驱逐

```
// evict all containers that are evictable
func (cgc *containerGC) evictContainers(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictTerminatedPods bool) error {
	// Separate containers by evict units.
	// 1.得到需要所有需要驱逐的contaienrs，非running已经挂掉 MinAge分钟了
	evictUnits, err := cgc.evictableContainers(gcPolicy.MinAge)
	if err != nil {
		return err
	}
  
  // 2.根据配置参数,保留最新的N个containers，其他的都要驱逐
	// Remove deleted pod containers if all sources are ready.
	if allSourcesReady {
		for key, unit := range evictUnits {
			if cgc.podStateProvider.IsPodDeleted(key.uid) || (cgc.podStateProvider.IsPodTerminated(key.uid) && evictTerminatedPods) {
				cgc.removeOldestN(unit, len(unit)) // Remove all.
				delete(evictUnits, key)
			}
		}
	}

	// Enforce max containers per evict unit.
	if gcPolicy.MaxPerPodContainer >= 0 {
		cgc.enforceMaxContainersPerEvictUnit(evictUnits, gcPolicy.MaxPerPodContainer)
	}

	// Enforce max total number of containers.
	// 3.
	if gcPolicy.MaxContainers >= 0 && evictUnits.NumContainers() > gcPolicy.MaxContainers {
		// Leave an equal number of containers per evict unit (min: 1).
		numContainersPerEvictUnit := gcPolicy.MaxContainers / evictUnits.NumEvictUnits()
		if numContainersPerEvictUnit < 1 {
			numContainersPerEvictUnit = 1
		}
		cgc.enforceMaxContainersPerEvictUnit(evictUnits, numContainersPerEvictUnit)

		// If we still need to evict, evict oldest first.
		numContainers := evictUnits.NumContainers()
		if numContainers > gcPolicy.MaxContainers {
			flattened := make([]containerGCInfo, 0, numContainers)
			for key := range evictUnits {
				flattened = append(flattened, evictUnits[key]...)
			}
			sort.Sort(byCreated(flattened))

			cgc.removeOldestN(flattened, numContainers-gcPolicy.MaxContainers)
		}
	}
	return nil
}
```

##### 3.2.2 移除sandboxes

该函数逻辑如下：

（1）获取 node 上所有的 container以及所有的 sandboxes

 （2）收集所有 container 的 PodSandboxId， 构建 sandboxes 与 pod 的对应关系并将其保存在 sandboxesByPodUID 中

 （3）遍历 sandboxesByPod，若 sandboxes 所在的 pod 处于 deleted 状态，则删除该 pod 中所有的 sandboxes ；否则只保留退出时间最短的一个 sandboxes

```
// evictSandboxes remove all evictable sandboxes. An evictable sandbox must
// meet the following requirements:
//   1. not in ready state
//   2. contains no containers.
//   3. belong to a non-existent (i.e., already removed) pod, or is not the
//      most recently created sandbox for the pod.
func (cgc *containerGC) evictSandboxes(evictTerminatedPods bool) error {
	containers, err := cgc.manager.getKubeletContainers(true)
	if err != nil {
		return err
	}

	sandboxes, err := cgc.manager.getKubeletSandboxes(true)
	if err != nil {
		return err
	}

	// collect all the PodSandboxId of container
	sandboxIDs := sets.NewString()
	for _, container := range containers {
		sandboxIDs.Insert(container.PodSandboxId)
	}

	sandboxesByPod := make(sandboxesByPodUID)
	for _, sandbox := range sandboxes {
		podUID := types.UID(sandbox.Metadata.Uid)
		sandboxInfo := sandboxGCInfo{
			id:         sandbox.Id,
			createTime: time.Unix(0, sandbox.CreatedAt),
		}

		// Set ready sandboxes to be active.
		if sandbox.State == runtimeapi.PodSandboxState_SANDBOX_READY {
			sandboxInfo.active = true
		}

		// Set sandboxes that still have containers to be active.
		if sandboxIDs.Has(sandbox.Id) {
			sandboxInfo.active = true
		}

		sandboxesByPod[podUID] = append(sandboxesByPod[podUID], sandboxInfo)
	}

	// Sort the sandboxes by age.
	for uid := range sandboxesByPod {
		sort.Sort(sandboxByCreated(sandboxesByPod[uid]))
	}

	for podUID, sandboxes := range sandboxesByPod {
		if cgc.podStateProvider.IsPodDeleted(podUID) || (cgc.podStateProvider.IsPodTerminated(podUID) && evictTerminatedPods) {
			// Remove all evictable sandboxes if the pod has been removed.
			// Note that the latest dead sandbox is also removed if there is
			// already an active one.
			cgc.removeOldestNSandboxes(sandboxes, len(sandboxes))
		} else {
			// Keep latest one if the pod still exists.
			cgc.removeOldestNSandboxes(sandboxes, len(sandboxes)-1)
		}
	}
	return nil
}
```

<br>

##### 3.2.3 回收log Directories

该方法会回收所有可回收 pod 以及 container 的 log dir，其主要逻辑为：
 （1）首先回收 deleted 状态 pod logs dir，遍历 pod logs dir /var/log/pods，/var/log/pods 为 pod logs 的默认目录，pod logs dir 的格式为 /var/log/pods/NAMESPACE_NAME_UID，解析 pod logs dir 获取 pod uid，判断 pod 是否处于 deleted 状态，若处于 deleted 状态则删除其 logs dir； 

（2）回收 deleted 状态 container logs 链接目录，/var/log/containers 为 container log 的默认目录，其会软链接到 pod 的 log dir 下，例如： /var/log/containers/storage-provisioner_kube-system_storage-provisioner-acc8386e409dfb3cc01618cbd14c373d8ac6d7f0aaad9ced018746f31d0081e2.log -> /var/log/pods/kube-system_storage-provisioner_b448e496-eb5d-4d71-b93f-ff7ff77d2348/storage-provisioner/0.log

```
// evictPodLogsDirectories evicts all evictable pod logs directories. Pod logs directories
// are evictable if there are no corresponding pods.
func (cgc *containerGC) evictPodLogsDirectories(allSourcesReady bool) error {
	osInterface := cgc.manager.osInterface
	if allSourcesReady {
		// Only remove pod logs directories when all sources are ready.
		dirs, err := osInterface.ReadDir(podLogsRootDirectory)
		if err != nil {
			return fmt.Errorf("failed to read podLogsRootDirectory %q: %v", podLogsRootDirectory, err)
		}
		for _, dir := range dirs {
			name := dir.Name()
			podUID := parsePodUIDFromLogsDirectory(name)
			if !cgc.podStateProvider.IsPodDeleted(podUID) {
				continue
			}
			err := osInterface.RemoveAll(filepath.Join(podLogsRootDirectory, name))
			if err != nil {
				klog.Errorf("Failed to remove pod logs directory %q: %v", name, err)
			}
		}
	}

	// Remove dead container log symlinks.
	// TODO(random-liu): Remove this after cluster logging supports CRI container log path.
	logSymlinks, _ := osInterface.Glob(filepath.Join(legacyContainerLogsDir, fmt.Sprintf("*.%s", legacyLogSuffix)))
	for _, logSymlink := range logSymlinks {
		if _, err := osInterface.Stat(logSymlink); os.IsNotExist(err) {
			err := osInterface.Remove(logSymlink)
			if err != nil {
				klog.Errorf("Failed to remove container log dead symlink %q: %v", logSymlink, err)
			}
		}
	}
	return nil
}
```

<br>

### 4. Image Gc处理流程

该函数逻辑

（1）获取容器镜像存储目录挂载点文件系统的磁盘信息

（2）若当前使用率大于 HighThresholdPercent，此时需要回收镜像

（3）调用 im.freeSpace 回收未使用的镜像信息

```
func (im *realImageGCManager) GarbageCollect() error {
	// Get disk usage on disk holding images.
	// 1.获取容器镜像存储目录挂载点文件系统的磁盘信息
	fsStats, err := im.statsProvider.ImageFsStats()
	if err != nil {
		return err
	}

	var capacity, available int64
	if fsStats.CapacityBytes != nil {
		capacity = int64(*fsStats.CapacityBytes)
	}
	if fsStats.AvailableBytes != nil {
		available = int64(*fsStats.AvailableBytes)
	}

	if available > capacity {
		klog.Warningf("available %d is larger than capacity %d", available, capacity)
		available = capacity
	}

	// Check valid capacity.
	if capacity == 0 {
		err := goerrors.New("invalid capacity 0 on image filesystem")
		im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.InvalidDiskCapacity, err.Error())
		return err
	}

	// If over the max threshold, free enough to place us at the lower threshold.
	// 2.若当前使用率大于 HighThresholdPercent，此时需要回收镜像
	usagePercent := 100 - int(available*100/capacity)
	if usagePercent >= im.policy.HighThresholdPercent {
		amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
		klog.Infof("[imageGCManager]: Disk usage on image filesystem is at %d%% which is over the high threshold (%d%%). Trying to free %d bytes down to the low threshold (%d%%).", usagePercent, im.policy.HighThresholdPercent, amountToFree, im.policy.LowThresholdPercent)
		// 3. 调用 im.freeSpace 回收未使用的镜像信息
		freed, err := im.freeSpace(amountToFree, time.Now())
		if err != nil {
			return err
		}

		if freed < amountToFree {
			err := fmt.Errorf("failed to garbage collect required amount of images. Wanted to free %d bytes, but freed %d bytes", amountToFree, freed)
			im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.FreeDiskSpaceFailed, err.Error())
			return err
		}
	}

	return nil
}
```

<br>

#### 4.1 freeSpace

该函数的主要逻辑：
 （1）获取已经使用的 images 列表

 （2）获取所有未使用的 images 列表

 （3）按镜像最近使用时间进行排序

 （4）从旧到新，依次回收，达到了需要释放的空间，就停止

```
// Tries to free bytesToFree worth of images on the disk.
//
// Returns the number of bytes free and an error if any occurred. The number of
// bytes freed is always returned.
// Note that error may be nil and the number of bytes free may be less
// than bytesToFree.
func (im *realImageGCManager) freeSpace(bytesToFree int64, freeTime time.Time) (int64, error) {
  // 1.获取已经使用的 images 列表
	imagesInUse, err := im.detectImages(freeTime)
	if err != nil {
		return 0, err
	}

	im.imageRecordsLock.Lock()
	defer im.imageRecordsLock.Unlock()

	// Get all images in eviction order.
	// 2.获取所有未使用的 images列表
	images := make([]evictionInfo, 0, len(im.imageRecords))
	for image, record := range im.imageRecords {
		if isImageUsed(image, imagesInUse) {
			klog.V(5).Infof("Image ID %s is being used", image)
			continue
		}
		images = append(images, evictionInfo{
			id:          image,
			imageRecord: *record,
		})
	}
	// 3.按镜像最近使用时间进行排序
	sort.Sort(byLastUsedAndDetected(images))
  
  // 4.从旧到新，依次回收，达到了需要释放的空间，就停止
	// Delete unused images until we've freed up enough space.
	var deletionErrors []error
	spaceFreed := int64(0)
	for _, image := range images {
		klog.V(5).Infof("Evaluating image ID %s for possible garbage collection", image.id)
		// Images that are currently in used were given a newer lastUsed.
		if image.lastUsed.Equal(freeTime) || image.lastUsed.After(freeTime) {
			klog.V(5).Infof("Image ID %s has lastUsed=%v which is >= freeTime=%v, not eligible for garbage collection", image.id, image.lastUsed, freeTime)
			continue
		}

		// Avoid garbage collect the image if the image is not old enough.
		// In such a case, the image may have just been pulled down, and will be used by a container right away.

		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			klog.V(5).Infof("Image ID %s has age %v which is less than the policy's minAge of %v, not eligible for garbage collection", image.id, freeTime.Sub(image.firstDetected), im.policy.MinAge)
			continue
		}

		// Remove image. Continue despite errors.
		klog.Infof("[imageGCManager]: Removing image %q to free %d bytes", image.id, image.size)
		err := im.runtime.RemoveImage(container.ImageSpec{Image: image.id})
		if err != nil {
			deletionErrors = append(deletionErrors, err)
			continue
		}
		delete(im.imageRecords, image.id)
		spaceFreed += image.size

		if spaceFreed >= bytesToFree {
			break
		}
	}

	if len(deletionErrors) > 0 {
		return spaceFreed, fmt.Errorf("wanted to free %d bytes, but freed %d bytes space with errors in image deletion: %v", bytesToFree, spaceFreed, errors.NewAggregate(deletionErrors))
	}
	return spaceFreed, nil
}
```

<br>

### 5 总结

**容器gc的清理逻辑: **
 （1）移除需要驱逐的的containers

* 得到需要所有需要驱逐的contaienrs，非running已经挂掉 MinAge分钟了

* 根据参数MaxPerPodContainer,保留最新的MaxPerPodContainer个containers，其他的都要驱逐

* 根据MaxContainers参数，保留最新的MaxContainers个contaeinrs，其他的按照创建时间依次驱逐

 （2）移除sandboxes

* 获取 node 上所有的 container以及所有的 sandboxes

* 收集所有 container 的 PodSandboxId， 构建 sandboxes 与 pod 的对应关系并将其保存在 sandboxesByPodUID 中

* 遍历 sandboxesByPod，若 sandboxes 所在的 pod 处于 deleted 状态，则删除该 pod 中所有的 sandboxes ；否则只保留退出时间最短的一个 sandboxes

 （3）回收log Directories

* 首先回收 deleted 状态 pod logs dir，遍历 pod logs dir /var/log/pods，/var/log/pods 为 pod logs 的默认目录，pod logs dir 的格式为 /var/log/pods/NAMESPACE_NAME_UID，解析 pod logs dir 获取 pod uid，判断 pod 是否处于 deleted 状态，若处于 deleted 状态则删除其 logs dir； 

* 回收 deleted 状态 container logs 链接目录，/var/log/containers 为 container log 的默认目录，其会软链接到 pod 的 log dir 下，例如： /var/log/containers/storage-provisioner_kube-system_storage-provisioner-acc8386e409dfb3cc01618cbd14c373d8ac6d7f0aaad9ced018746f31d0081e2.log -> /var/log/pods/kube-system_storage-provisioner_b448e496-eb5d-4d71-b93f-ff7ff77d2348/storage-provisioner/0.log

<br>

**镜像gc的清理逻辑: **

（1）获取容器镜像存储目录挂载点文件系统的磁盘信息

（2）若当前使用率大于 HighThresholdPercent，此时需要回收镜像

（3）调用 im.freeSpace 回收未使用的镜像信息

* 获取已经使用的 images 列表，然后过滤出来未使用的images 列表

* 将未使用的Images按镜像最近使用时间进行排序

* 从旧到新，依次回收，达到了需要释放的空间，就停止

