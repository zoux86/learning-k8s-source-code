* [1\. 背景](#1-背景)
* [2\. HandlePodSyncs](#2-handlepodsyncs)
  * [2\.1 dispatchWork](#21-dispatchwork)
  * [2\.2 UpdatePod](#22-updatepod)
  * [2\.3 managePodLoop](#23-managepodloop)
  * [2\.4 syncPod](#24-syncpod)
  * [2\.5 containerRuntime\.SyncPod](#25-containerruntimesyncpod)
* [3 总结](#3-总结)

### 1. 背景

本节介绍pod更新流程。本节目的是假设创建pod后。podA的容器啥的都起来了，kubelet是如何更新podA的。

在 kubelet初始化流程-下 的章节中，介绍到pleg的生产逻辑是：

pleg.Start每隔1秒，运行以此relist函数，relist的逻辑如下：

1. 记录上一次relist的时间和间隔
2. 通过runtimeApi获取所有的pod，包括exit的Pod
3. 更新pods container状态,以及记录pod数量等metrics
4. 和旧的Pod进行对比，podRecord结构体保存了旧的，和当前pod的信息，可以理解为和1s前的所有pods进行对比。对比完产生event，保存在一个map中。这里主要产生的事件为：ContainerStarted，ContainerDie, ContainerRemoved, ContainerChanged等等
5. 如果event和Pod有绑定，并且kubelet开启了cache缓存pod信息，根据最新的信息同步缓存
6. 将新的record赋值为旧的，为下一轮做准备，然后依次处理event，逻辑为：不是ContainerChanged状态的event都发送到eventChannel中去
7. 更新缓存，如果有更新失败的，记录到needsReinspection，表示下一次还需要重试

<br>

pleg的消费逻辑是:  如果pod的状态发生了改变，并且Pod还存在，调用HandlePodSyncs进行同步

本节就是分析，podA的容器都起来了，pleg 调用HandlePodSyncs做了哪些工作。

<br>

### 2. HandlePodSyncs

HandlePodSyncs调用了dispatchWork，接下里就和pod创建的流程一样了。只不过执行的函数逻辑不一样而言。

在上一节中比较详细的描述了这个调用链的函数流程。接下来只是会简单过一下用到的流程，不会贴出来每个函数的详细实现了。建议看的时候，打开上一节的内容对比看。

```
// HandlePodSyncs is the callback in the syncHandler interface for pods
// that should be dispatched to pod workers for sync.
func (kl *Kubelet) HandlePodSyncs(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodSync, mirrorPod, start)
	}
}
```

<br>

#### 2.1 dispatchWork

dispatchWork调用UpdatePod函数

```
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
```

<br>

#### 2.2 UpdatePod

UpdatePod主要是通过managePodLoop处理更新。因为kulete收到Pod创建的时候已经启动了这个协程处理。

```
p.managePodLoop(podUpdates)
```

#### 2.3 managePodLoop

managePodLoop继续往下调用syncPodFn进行同步

```
err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,   //当前最新状态
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			return err
		}()
```

#### 2.4 syncPod

因为是更新Pod status，所以执行到这个函数只会执行以下逻辑：

（1）设置pod的 apiStatus，然后更新statuManager中Pod状态

（2）创建目录，已经exist不会报错

（3）调用containerRuntime.SyncPod同步

#### 2.5 containerRuntime.SyncPod

这个函数的第一步就是调用computePodActions 判断pod状态。从这里可以得到，不要创建新的sandbox，也没有容器需要创建，也不需要删除，所有就没了，不用继续往下调用了。

### 3 总结

Pleg 每隔1s执行对比有变化的容器。然后发送到channel。对应的HandlePodSyncs会和创建Pod一样处理。只不过跳过了很多步骤。

该流程核心就是：生成最新状态，发送到apiserver