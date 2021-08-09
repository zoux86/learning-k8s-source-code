Table of Contents
=================

  * [1. startReplicaSetController](#1-startreplicasetcontroller)
     * [1.1 rs中的expectations机制](#11-rs中的expectations机制)
  * [2. Pod，rs变化时对应的处理逻辑](#2-podrs变化时对应的处理逻辑)
     * [2.1 addPod](#21-addpod)
     * [2.2 updatePod](#22-updatepod)
     * [2.3 deletePod](#23-deletepod)
     * [2.4 addRS](#24-addrs)
     * [2.5 updateRS](#25-updaters)
     * [2.6 deleteRS](#26-deleters)
  * [3. rs的处理逻辑](#3-rs的处理逻辑)
     * [3.1 过滤pod](#31-过滤pod)
     * [3.2 manageReplicas](#32-managereplicas)
        * [3.2.1 创建pod](#321-创建pod)
        * [3.2.2 删除pod](#322-删除pod)
     * [3.3 calculateStatus](#33-calculatestatus)
  * [4 总结](#4-总结)

### 1. startReplicaSetController

和deployController一样，kcm中定义了startReplicaSetController，startReplicaSetController和所有的控制器一样，先New一个对象，然后调用run函数。

这里可以看出来，rs控制器监听rs, 和pod的变化。

```
func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
		return nil, false, nil
	}
	go replicaset.NewReplicaSetController(
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
	return nil, true, nil
}
```

<br>

先NewReplicaSetController，再run

```
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
   // event上传
   eventBroadcaster := record.NewBroadcaster()
   eventBroadcaster.StartLogging(glog.Infof)
   eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
   return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
      apps.SchemeGroupVersion.WithKind("ReplicaSet"),
      "replicaset_controller",
      "replicaset",
      controller.RealPodControl{
         KubeClient: kubeClient,
         Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
      },
   )
}
```

```
// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
   gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
   if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
      metrics.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
   }

   rsc := &ReplicaSetController{
      GroupVersionKind: gvk,
      kubeClient:       kubeClient,
      podControl:       podControl,
      burstReplicas:    burstReplicas,
      expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
      queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
   }

   rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc:    rsc.enqueueReplicaSet,
      UpdateFunc: rsc.updateRS,
      // This will enter the sync loop and no-op, because the replica set has been deleted from the store.
      // Note that deleting a replica set immediately after scaling it to 0 will not work. The recommended
      // way of achieving this is by performing a `stop` operation on the replica set.
      DeleteFunc: rsc.enqueueReplicaSet,
   })
   rsc.rsLister = rsInformer.Lister()
   rsc.rsListerSynced = rsInformer.Informer().HasSynced

   podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc: rsc.addPod,
      // This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
      // overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
      // local storage, so it should be ok.
      UpdateFunc: rsc.updatePod,
      DeleteFunc: rsc.deletePod,
   })
   rsc.podLister = podInformer.Lister()
   rsc.podListerSynced = podInformer.Informer().HasSynced

   rsc.syncHandler = rsc.syncReplicaSet

   return rsc
}
```

这里注意一点，syncHandler函数是 syncReplicaSet。

<br>

#### 1.1 rs中的expectations机制

在介绍 rs controller如何处理rs, pod的变动之前，先介绍expectations机制。原因在于addpod, addrs, delrs等等处理函数一直用到了expectations。

expectations可以理解为一个map。举例来说，这个map可以认为有四个关键字段。

key:  有rs的ns和 rs的name组成

Add: 表示这个rs还需要增加多少个rs

del: 表示这个rs还需要删除多少个pod

Time: 表示

| Key         | Add  | Del  | Time                |
| ----------- | ---- | ---- | ------------------- |
| Default/zx1 | 0    | 0    | 2021.07.04 16:00:00 |
| zx/zx1      | 1    | 0    | 2021.07.04 16:00:00 |

<br>

**GetExpectations**:  输入是key, 输出整个map;

**SatisfiedExpectations**: 输入key, 输出bool；判断某个rs是否符合预期。符合预期： add<=0 && del<=0 或者 超过了同步周期； 其他情况都是不符合预期。

**DeleteExpectations**：输入key, 无输出；从map(缓存)中删除这个key

**SetExpectations**：输入（key, add, del）; 在map中新增加一行。 **这个会更新时间，将time复制为time.Now**

**ExpectCreations**:  输入（key, add);   覆盖map中的内容，del=0， add等于函数的参数。  **这个会更新时间，将time复制为time.Now**

**ExpectDeletions**： 输入（key, del);  覆盖map中的内容，add=0， del等于函数的参数。   **这个会更新时间，将time复制为time.Now**

**CreationObserved**: 输入(key) ;  map中对应的行中 add-1

**DeletionObserved**: 输入(key);  map中对应的行中 del-1

**RaiseExpectations**:  输入(key, add, del)；  map中对应的行中 Add+add, Del+del

**LowerExpectations**: 输入(key, add, del)；  map中对应的行中 Add-add, Del-del

```
// A TTLCache of pod creates/deletes each rc expects to see.
expectations *controller.UIDTrackingControllerExpectations


type UIDTrackingControllerExpectations struct {
	ControllerExpectationsInterface

  // 原生锁, 这里带了时间操作 sync/mutex.go
  uidStoreLock sync.Mutex

  // 缓存
	uidStore cache.Store
}

type ControllerExpectationsInterface interface {
	GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error)
	SatisfiedExpectations(controllerKey string) bool
	DeleteExpectations(controllerKey string)
	SetExpectations(controllerKey string, add, del int) error
	ExpectCreations(controllerKey string, adds int) error
	ExpectDeletions(controllerKey string, dels int) error
	CreationObserved(controllerKey string)
	DeletionObserved(controllerKey string)
	RaiseExpectations(controllerKey string, add, del int)
	LowerExpectations(controllerKey string, add, del int)
}


// ControlleeExpectations track controllee creates/deletes.
type ControlleeExpectations struct {
	// Important: Since these two int64 fields are using sync/atomic, they have to be at the top of the struct due to a bug on 32-bit platforms
	// See: https://golang.org/pkg/sync/atomic/ for more information
	add       int64
	del       int64
	key       string
	timestamp time.Time
}


// SatisfiedExpectations returns true if the required adds/dels for the given controller have been observed.
// Add/del counts are established by the controller at sync time, and updated as controllees are observed by the controller
// manager.
func (r *ControllerExpectations) SatisfiedExpectations(controllerKey string) bool {
	if exp, exists, err := r.GetExpectations(controllerKey); exists {
	  // Fulfilled就是 add<=0并且del<=0
		if exp.Fulfilled() {
			klog.V(4).Infof("Controller expectations fulfilled %#v", exp)
			return true
		} else if exp.isExpired() {
			klog.V(4).Infof("Controller expectations expired %#v", exp)
			return true
		} else {
			klog.V(4).Infof("Controller still waiting on expectations %#v", exp)
			return false
		}
	} else if err != nil {
		klog.V(2).Infof("Error encountered while checking expectations %#v, forcing sync", err)
	} else {
		// When a new controller is created, it doesn't have expectations.
		// When it doesn't see expected watch events for > TTL, the expectations expire.
		//	- In this case it wakes up, creates/deletes controllees, and sets expectations again.
		// When it has satisfied expectations and no controllees need to be created/destroyed > TTL, the expectations expire.
		//	- In this case it continues without setting expectations till it needs to create/delete controllees.
		klog.V(4).Infof("Controller %v either never recorded expectations, or the ttl expired.", controllerKey)
	}
	// Trigger a sync if we either encountered and error (which shouldn't happen since we're
	// getting from local store) or this controller hasn't established expectations.
	return true
}

// Fulfilled就是 add<=0并且del<=0
// Fulfilled returns true if this expectation has been fulfilled.
func (e *ControlleeExpectations) Fulfilled() bool {
	// TODO: think about why this line being atomic doesn't matter
	return atomic.LoadInt64(&e.add) <= 0 && atomic.LoadInt64(&e.del) <= 0
}

// 判断是否超过同步周期，同步周期是5分钟
func (exp *ControlleeExpectations) isExpired() bool {
	return clock.RealClock{}.Since(exp.timestamp) > ExpectationsTimeout
}

// 这个会覆盖之前的行，并且del=0
func (r *ControllerExpectations) ExpectCreations(controllerKey string, adds int) error {
	return r.SetExpectations(controllerKey, adds, 0)
}
```

<br>

**总结：**

（1）expectations就是通过一个类似map结构的对象，来表示所有rs期望pod和当前现状的差距

（2）

### 2. Pod，rs变化时对应的处理逻辑

#### 2.1 addPod

（1）如果pod有DeletionTimestamp，表明这个pod要被删除。将对应rs的Del+1,然后将rs加入队列。

（2） 如果pod有OwnerReference,判断OwnerReference是否是 rs。如果不是或者是rs，但是指定的rs不存在直接返回。否则rs对应的Add+1，并且将rs加入队列。因为pod数量更新了，rs也要更新。

（3）否则(pod没有OwnerReference)。所以他是一个孤儿，这个时候看有没有rs可以匹配它，如果有也可能要更新。匹配的逻辑： 判断pod的ns 和 rs的ns相等，并且 pod的labels能匹配上 rs。 找出来所有能匹配的rs，然后入队列

```
// When a pod is created, enqueue the replica set that manages it and update its expectations.
func (rsc *ReplicaSetController) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)
    
    // 1.如果pod有DeletionTimestamp，表明这个pod要被删除。
    // 2. deletePod就是将对应rs的Del+1,然后将rs加入队列。
	if pod.DeletionTimestamp != nil {
		// on a restart of the controller manager, it's possible a new pod shows up in a state that
		// is already pending deletion. Prevent the pod from being a creation observation.
		rsc.deletePod(pod)
		return
	}
     
    // 2. 如果pod有OwnerReference,判断OwnerReference是否是 rs
    // 如果不是或者是rs，但是指定的rs不存在直接返回。否则rs对应的Add+1，并且将rs加入队列。因为pod数量更新了，rs也要更新。
	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
		if rs == nil {
			return
		}
		rsKey, err := controller.KeyFunc(rs)
		if err != nil {
			return
		}
		glog.V(4).Infof("Pod %s created: %#v.", pod.Name, pod)
		// 对应rs的add+1
		rsc.expectations.CreationObserved(rsKey)
		
		rsc.enqueueReplicaSet(rs)
		return
	}
    
    
    
  // 3. 否则(pod没有OwnerReference)。所以他是一个孤儿，这个时候看有没有rs可以匹配它，如果可以也更新。
  // 匹配的逻辑： 判断pod的ns 和 rs的ns相等，并且 pod的labels能匹配上 rs
  // 找出来所有能匹配的rs，然后入队列
	// Otherwise, it's an orphan. Get a list of all matching ReplicaSets and sync
	// them to see if anyone wants to adopt it.
	// DO NOT observe creation because no controller should be waiting for an
	// orphan.
	rss := rsc.getPodReplicaSets(pod)
	if len(rss) == 0 {
		return
	}
	glog.V(4).Infof("Orphan Pod %s created: %#v.", pod.Name, pod)
	for _, rs := range rss {
		rsc.enqueueReplicaSet(rs)
	}
}
```



```
// When a pod is deleted, enqueue the replica set that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (rsc *ReplicaSetController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the pod
	// changed labels the new ReplicaSet will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %#v", obj))
			return
		}
	}

	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
	if rs == nil {
		return
	}
	// 这里keyfunc就是 ns/rsName
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}
	klog.V(4).Infof("Pod %s/%s deleted through %v, timestamp %+v: %#v.", pod.Namespace, pod.Name, utilruntime.GetCaller(), pod.DeletionTimestamp, pod)
	// 调用 expectations.DeletionObserved，然后入队列
	rsc.expectations.DeletionObserved(rsKey, controller.PodKey(pod))
	rsc.queue.Add(rsKey)
}
```

<br>

#### 2.2 updatePod

（1）ResourceVersion判断pod是否真的更新了

（2）判断pod的DeletionTimestamp是否为空，如果不为空，表明这个pod是要删除的，对应rs的Del-1

（3）如果是pod的ownerRef改变了，首先将旧rs入队，这个是肯定要更新的

（4）判断pod新的ownerRef是否是rs，如果是加入队列，如果设置了MinReadySeconds，等延迟结束再将rs添加到队列，因为到时候pod ready可能会导致rs更新。

（5）和addPod一样，判断出来pod没有OwnerReference。所以他是一个孤儿，这个时候看有没有rs可以匹配它，如果有也可能要更新。匹配的逻辑： 判断pod的ns 和 rs的ns相等，并且 pod的labels能匹配上 rs。 找出来所有能匹配的rs，然后入队列

```
// When a pod is updated, figure out what replica set/s manage it and wake them
// up. If the labels of the pod have changed we need to awaken both the old
// and new replica set. old and cur must be *v1.Pod types.
func (rsc *ReplicaSetController) updatePod(old, cur interface{}) {
	curPod := cur.(*v1.Pod)
	oldPod := old.(*v1.Pod)
	// 1.判断是否是否一致，为啥 ResourceVersion就能判断呢。参考https://fankangbest.github.io/2018/01/16/Kubernetes-resourceVersion%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		// Periodic resync will send update events for all known pods.
		// Two different versions of the same pod will always have different RVs.
		return
	}
	
	labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
	// 2.判断pod是否删除，因为删除分为两步：（1）update DeletionTimestamp, (2)删除
	if curPod.DeletionTimestamp != nil {
		// when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period,
		// and after such time has passed, the kubelet actually deletes it from the store. We receive an update
		// for modification of the deletion timestamp and expect an rs to create more replicas asap, not wait
		// until the kubelet actually deletes the pod. This is different from the Phase of a pod changing, because
		// an rs never initiates a phase change, and so is never asleep waiting for the same.
		// 从对应的rs的列表中删除pod
		rsc.deletePod(curPod)
		if labelChanged {
			// we don't need to check the oldPod.DeletionTimestamp because DeletionTimestamp cannot be unset.
			rsc.deletePod(oldPod)
		}
		return
	}

	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	// 3.如果是old rs->new rs。先将old rs进入更新队列。
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
			rsc.enqueueReplicaSet(rs)
		}
	}
    
    // 4. 如果pod有新的 ownerRef, 
	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
	    // 4.1 新的ownerRef不是 rs。啥都不干。
		rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
		if rs == nil {
			return
		}
		glog.V(4).Infof("Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		rsc.enqueueReplicaSet(rs)
		// TODO: MinReadySeconds in the Pod will generate an Available condition to be added in
		// the Pod status which in turn will trigger a requeue of the owning replica set thus
		// having its status updated with the newly available replica. For now, we can fake the
		// update by resyncing the controller MinReadySeconds after the it is requeued because
		// a Pod transitioned to Ready.
		// Note that this still suffers from #29229, we are just moving the problem one level
		// "closer" to kubelet (from the deployment to the replica set controller).
		
		// 4.2 如果oldpod ready, newpod not ready，然后设置了 MinReadySeconds延迟再添加rs到队列。
		if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
			glog.V(2).Infof("%v %q will be enqueued after %ds for availability check", rsc.Kind, rs.Name, rs.Spec.MinReadySeconds)
			// Add a second to avoid milliseconds skew in AddAfter.
			// See https://github.com/kubernetes/kubernetes/issues/39785#issuecomment-279959133 for more info.
			rsc.enqueueReplicaSetAfter(rs, (time.Duration(rs.Spec.MinReadySeconds)*time.Second)+time.Second)
		}
		return
	}
    
    // 5. 和addpod一样，判断孤儿pod。
	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	if labelChanged || controllerRefChanged {
		rss := rsc.getPodReplicaSets(curPod)
		if len(rss) == 0 {
			return
		}
		glog.V(4).Infof("Orphan Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		for _, rs := range rss {
			rsc.enqueueReplicaSet(rs)
		}
	}
}
```

spec.minReadySeconds:  新创建的Pod状态为Ready持续的时间至少为`spec.minReadySeconds`才认为Pod Available(Ready)。

<br>

#### 2.3 deletePod

deletepod就很简单：

（1）判断墓碑状态的pod是否ok。

（2）找出pod对应的rsA，从rsA中删除该pod，然后将rs加入队列。

```
// When a pod is deleted, enqueue the replica set that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (rsc *ReplicaSetController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the pod
	// changed labels the new ReplicaSet will not be woken up till the periodic resync.
	// 墓碑状态，这个是存储在etcd中，资源被删除时候的一个状态。 可以参考：https://draveness.me/etcd-introduction/
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %#v", obj))
			return
		}
	}

	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
	if rs == nil {
		return
	}
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		return
	}
	glog.V(4).Infof("Pod %s/%s deleted through %v, timestamp %+v: %#v.", pod.Namespace, pod.Name, utilruntime.GetCaller(), pod.DeletionTimestamp, pod)
	rsc.expectations.DeletionObserved(rsKey, controller.PodKey(pod))
	rsc.enqueueReplicaSet(rs)
}

// 这里会先判断是否存在
// DeletionObserved records the given deleteKey as a deletion, for the given rc.
func (u *UIDTrackingControllerExpectations) DeletionObserved(rcKey, deleteKey string) {
	u.uidStoreLock.Lock()
	defer u.uidStoreLock.Unlock()

	uids := u.GetUIDs(rcKey)
	if uids != nil && uids.Has(deleteKey) {
		klog.V(4).Infof("Controller %v received delete for pod %v", rcKey, deleteKey)
		u.ControllerExpectationsInterface.DeletionObserved(rcKey)
		uids.Delete(deleteKey)
	}
}
```

<br>

从上面可以看出来，Pod的add,update, delete都会将 rs重新加入队列。

#### 2.4 addRS

直接入队列

```
func (rsc *ReplicaSetController) addRS(obj interface{}) {
   rs := obj.(*apps.ReplicaSet)
   klog.V(4).Infof("Adding %s %s/%s", rsc.Kind, rs.Namespace, rs.Name)
   rsc.enqueueRS(rs)
}
```

#### 2.5 updateRS

判断了是否真的更新，如果是，就入队列。

```
// callback when RS is updated
func (rsc *ReplicaSetController) updateRS(old, cur interface{}) {
   oldRS := old.(*apps.ReplicaSet)
   curRS := cur.(*apps.ReplicaSet)

   // You might imagine that we only really need to enqueue the
   // replica set when Spec changes, but it is safer to sync any
   // time this function is triggered. That way a full informer
   // resync can requeue any replica set that don't yet have pods
   // but whose last attempts at creating a pod have failed (since
   // we don't block on creation of pods) instead of those
   // replica sets stalling indefinitely. Enqueueing every time
   // does result in some spurious syncs (like when Status.Replica
   // is updated and the watch notification from it retriggers
   // this function), but in general extra resyncs shouldn't be
   // that bad as ReplicaSets that haven't met expectations yet won't
   // sync, and all the listing is done using local stores.
   if *(oldRS.Spec.Replicas) != *(curRS.Spec.Replicas) {
      glog.V(4).Infof("%v %v updated. Desired pod count change: %d->%d", rsc.Kind, curRS.Name, *(oldRS.Spec.Replicas), *(curRS.Spec.Replicas))
   }
   rsc.enqueueReplicaSet(cur)
}
```

<br>

#### 2.6 deleteRS

先判断tombstone，再进行map中对应行的删除。然后入队列。

个人认为，这里每次都判断tombstone的原因在于：

k8s删除对象分为两步：（1）设置deletionTimestamp,这是个更新时间。（2）删除对象，这是个删除事件。

所以到了删除的时候，update已经做了一下处理，所以这里要通过tombstone再额外判断一次。

```
func (rsc *ReplicaSetController) deleteRS(obj interface{}) {
	rs, ok := obj.(*apps.ReplicaSet)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		rs, ok = tombstone.Obj.(*apps.ReplicaSet)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a ReplicaSet %#v", obj))
			return
		}
	}

	key, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}

	klog.V(4).Infof("Deleting %s %q", rsc.Kind, key)

	// Delete expectations for the ReplicaSet so if we create a new one with the same name it starts clean
	rsc.expectations.DeleteExpectations(key)

	rsc.queue.Add(key)
}
```

### 3. rs的处理逻辑

接下来看看rsController是如何处理队列中的对象。

```
// Run begins watching and syncing.
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	glog.Infof("Starting %v controller", controllerName)
	defer glog.Infof("Shutting down %v controller", controllerName)

	if !controller.WaitForCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

一样的套路，最后是  syncHandler。初始化NewBaseController的时候

`rsc.syncHandler = rsc.syncReplicaSet`

syncReplicaSet就是处理队列中一个一个的元素了。

```
// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem() bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("Sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

<br>

**syncReplicaSet**

（1）判断是否需要 rsNeedsSync， 如果 add<=0 && del<=0 或者 超过了同步周期，则需要同步

（2）获得所有该rs下的pod

（3）如果要同步，并且rs没有删除，调用manageReplicas对pod进行创建/删除

（4）计算当前rs的状态

（5）更新rs的状态

（6）判断是否需要将 rs 加入到延迟队列中

```
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

	startTime := time.Now()
	defer func() {
		glog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		glog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}

    // 1.判断是否需要 rsNeedsSync，这里调用了SatisfiedExpectations
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Error converting pod selector to selector: %v", err))
		return nil
	}
	
	// 2. 获得namespaces下的所有pod
	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// 2.1 过滤inactive的pods
	// Ignore inactive pods.
	var filteredPods []*v1.Pod
	for _, pod := range allPods {
		if controller.IsPodActive(pod) {
			filteredPods = append(filteredPods, pod)
		}
	}

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first.
	// 2.2 重新洗牌，获得真正属于该rs的podlist
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}
	
	// 3. 如果要同步，并且rs没有删除，调用manageReplicas对pod进行创建/删除
	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	
	// 4. 计算 rs 当前的 status
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// Always updates status as pods come up or die.
	// 5. 更新 status
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}
	
	// 6. 判断是否需要将 rs 加入到延迟队列中。这里判断的标准也是很简单： ReadyReplicas满足了，但是AvailableReplicas还没满足，那肯定还有pod在启动中
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

<br>

#### 3.1 过滤pod

（1）过滤inactivedPod: 就是pod状态不是PodSucceeded, PodFailed或者DeletionTimestamp!=nil的pod

（2）重新洗牌，获得真正属于该rs的podlist

<br>

adopt 就是根据lable（原来不匹配，现在匹配了），绑定 rs与pod

release就是根据lable（原来匹配了，现在不匹配），释放原来的绑定关系

```
func IsPodActive(p *v1.Pod) bool {
	return v1.PodSucceeded != p.Status.Phase &&
		v1.PodFailed != p.Status.Phase &&
		p.DeletionTimestamp == nil
}


func (rsc *ReplicaSetController) claimPods(rs *apps.ReplicaSet, selector labels.Selector, filteredPods []*v1.Pod) ([]*v1.Pod, error) {
	// If any adoptions are attempted, we should first recheck for deletion with
	// an uncached quorum read sometime after listing Pods (see #42639).
	canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
		fresh, err := rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace).Get(rs.Name, metav1.GetOptions{})
		if err != nil {
			return nil, err
		}
		if fresh.UID != rs.UID {
			return nil, fmt.Errorf("original %v %v/%v is gone: got uid %v, wanted %v", rsc.Kind, rs.Namespace, rs.Name, fresh.UID, rs.UID)
		}
		return fresh, nil
	})
	cm := controller.NewPodControllerRefManager(rsc.podControl, rs, selector, rsc.GroupVersionKind, canAdoptFunc)
	return cm.ClaimPods(filteredPods)
}
```

```
// ClaimPods tries to take ownership of a list of Pods.
//
// It will reconcile the following:
//   * Adopt orphans if the selector matches.
//   * Release owned objects if the selector no longer matches.
//
// Optional: If one or more filters are specified, a Pod will only be claimed if
// all filters return true.
//
// A non-nil error is returned if some form of reconciliation was attempted and
// failed. Usually, controllers should try again later in case reconciliation
// is still needed.
//
// If the error is nil, either the reconciliation succeeded, or no
// reconciliation was necessary. The list of Pods that you now own is returned.
func (m *PodControllerRefManager) ClaimPods(pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
   var claimed []*v1.Pod
   var errlist []error

   match := func(obj metav1.Object) bool {
      pod := obj.(*v1.Pod)
      // Check selector first so filters only run on potentially matching Pods.
      if !m.Selector.Matches(labels.Set(pod.Labels)) {
         return false
      }
      for _, filter := range filters {
         if !filter(pod) {
            return false
         }
      }
      return true
   }
   adopt := func(obj metav1.Object) error {
      return m.AdoptPod(obj.(*v1.Pod))
   }
   release := func(obj metav1.Object) error {
      return m.ReleasePod(obj.(*v1.Pod))
   }

   for _, pod := range pods {
      ok, err := m.ClaimObject(pod, match, adopt, release)
      if err != nil {
         errlist = append(errlist, err)
         continue
      }
      if ok {
         claimed = append(claimed, pod)
      }
   }
   return claimed, utilerrors.NewAggregate(errlist)
}
```

<br>

#### 3.2 manageReplicas

（1）计算当前pod和期望pod的数量差距

（2）进行pod的创建和删除

```
func (rsc *ReplicaSetController) manageReplicas(......) error {
    // 1.计算当前pod数量的差距
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
    rsKey, err := controller.KeyFunc(rs)
    if err != nil {
        ......
    }
    // 2.diff<0，表示需要创建 pod
    if diff < 0 {
        diff *= -1
        // 2.1 pod创建一轮的上限是500
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        // 2.2 更新map的数据，表示当前只需要创建diff个pod
        rsc.expectations.ExpectCreations(rsKey, diff)
        // 2.3 调用slowStartBatch创建pod
        successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
            err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
            if err != nil && errors.IsTimeout(err) {
                return nil
            }
            return err
        })
        // 2.3 根据创建的结果，更新map的数据
        if skippedPods := diff - successfulCreations; skippedPods > 0 {
            for i := 0; i < skippedPods; i++ {
                rsc.expectations.CreationObserved(rsKey)
            }
        }
        return err
    } else if diff > 0 {
        // 3. 如果是删除pod,同样一轮最多只能删除500个
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        // 3.1 选择需要删除的pod列表，这个是有优先级的
        podsToDelete := getPodsToDelete(filteredPods, diff)
        // 3.2 覆盖map表中的数据
        rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))
        // 3.3 进行并发删除
        errCh := make(chan error, diff)
        var wg sync.WaitGroup
        wg.Add(diff)
        for _, pod := range podsToDelete {
            go func(targetPod *v1.Pod) {
                defer wg.Done()
                if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
                    podKey := controller.PodKey(targetPod)
                    rsc.expectations.DeletionObserved(rsKey, podKey)
                    errCh <- err
                }
            }(pod)
        }
        wg.Wait()
        select {
        case err := <-errCh:
            if err != nil {
                return err
            }
        default:
        }
    }
    return nil
}
```

<br>

##### 3.2.1 创建pod

`slowStartBatch` 创建的 pod 数依次为 1，2，4，8 。以2的指数级增长，如果失败了，直接返回（当前成功创建了多少）。

```
func slowStartBatch(count int, initialBatchSize int, fn func() error) (int, error) {
    remaining := count
    successes := 0
    for batchSize := integer.IntMin(remaining, initialBatchSize); batchSize > 0; batchSize = integer.IntMin(2*batchSize, remaining) {
        errCh := make(chan error, batchSize)
        var wg sync.WaitGroup
        wg.Add(batchSize)
        for i := 0; i < batchSize; i++ {
            go func() {
                defer wg.Done()
                if err := fn(); err != nil {
                    errCh <- err
                }
            }()
        }
        wg.Wait()
        curSuccesses := batchSize - len(errCh)
        successes += curSuccesses
        if len(errCh) > 0 {
            return successes, <-errCh
        }
        remaining -= batchSize
    }
    return successes, nil
}
```

<br>

##### 3.2.2 删除pod

给pod定义优先级，从优先级最高的依次往下删，优先级越高，表示这个pod越应该删除，根据以下的条件判断优先级：

（1）没有绑定node的pod优先级比 绑定了的高

（2）pod状态是PodPending的高于PodUnknown，PodUnknown高于PodRunning

（3）pod unready的高于 ready

（4）根据运行时间排序，越短优先级越高

（5）pod中容器重启次数越多的，优先级越高

（6）pod创建时间越短，优先级越高

```
func getPodsToDelete(filteredPods []*v1.Pod, diff int) []*v1.Pod {
    if diff < len(filteredPods) {
        sort.Sort(controller.ActivePods(filteredPods))
    }
    return filteredPods[:diff]
}
```

```
type ActivePods []*v1.Pod
func (s ActivePods) Len() int      { return len(s) }
func (s ActivePods) Swap(i, j int) { s[i], s[j] = s[j], s[i] }
func (s ActivePods) Less(i, j int) bool {
    // 1.没有绑定node的pod优先级比绑定了的高
    if s[i].Spec.NodeName != s[j].Spec.NodeName && (len(s[i].Spec.NodeName) == 0 || len(s[j].Spec.NodeName) == 0) {
        return len(s[i].Spec.NodeName) == 0
    }
    // 2. pod状态是PodPending的高于PodUnknown，PodUnknown高于PodRunning
    m := map[v1.PodPhase]int{v1.PodPending: 0, v1.PodUnknown: 1, v1.PodRunning: 2}
    if m[s[i].Status.Phase] != m[s[j].Status.Phase] {
        return m[s[i].Status.Phase] < m[s[j].Status.Phase]
    }
    // 3. pod unready的高于 ready
    if podutil.IsPodReady(s[i]) != podutil.IsPodReady(s[j]) {
        return !podutil.IsPodReady(s[i])
    }
    // 4. 根据运行时间排序，越短优先级越高
    if podutil.IsPodReady(s[i]) && podutil.IsPodReady(s[j]) && !podReadyTime(s[i]).Equal(podReadyTime(s[j])) {
        return afterOrZero(podReadyTime(s[i]), podReadyTime(s[j]))
    }
    // 5. pod中容器重启次数越多的优先级越高
    if maxContainerRestarts(s[i]) != maxContainerRestarts(s[j]) {
        return maxContainerRestarts(s[i]) > maxContainerRestarts(s[j])
    }
    // 6. pod创建时间越短，优先级越高
    if !s[i].CreationTimestamp.Equal(&s[j].CreationTimestamp) {
        return afterOrZero(&s[i].CreationTimestamp, &s[j].CreationTimestamp)
    }
    return false
}
```

<br>

#### 3.3 calculateStatus

calculateStatus 会通过当前 pod 的状态计算出 rs 中 status 字段值，status 字段如下所示：

replicas 实际的 pod 副本数
availableReplicas 现在可用的 Pod 的副本数量，有的副本可能还处在未准备好，或者初始化状态
readyReplicas 是处于 ready 状态的 Pod 的副本数量
fullyLabeledReplicas 意思是这个 ReplicaSet 的标签 selector 对应的副本数量，不同纬度的一种统计

```
随便一个rs都有
status:
  availableReplicas: 1
  fullyLabeledReplicas: 1
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
```



```
func calculateStatus(rs *apps.ReplicaSet, filteredPods []*v1.Pod, manageReplicasErr error) apps.ReplicaSetStatus {
	newStatus := rs.Status
	// Count the number of pods that have labels matching the labels of the pod
	// template of the replica set, the matching pods may have more
	// labels than are in the template. Because the label of podTemplateSpec is
	// a superset of the selector of the replica set, so the possible
	// matching pods must be part of the filteredPods.
	fullyLabeledReplicasCount := 0
	readyReplicasCount := 0
	availableReplicasCount := 0
	templateLabel := labels.Set(rs.Spec.Template.Labels).AsSelectorPreValidated()
	for _, pod := range filteredPods {
		if templateLabel.Matches(labels.Set(pod.Labels)) {
			fullyLabeledReplicasCount++
		}
		if podutil.IsPodReady(pod) {
			readyReplicasCount++
			if podutil.IsPodAvailable(pod, rs.Spec.MinReadySeconds, metav1.Now()) {
				availableReplicasCount++
			}
		}
	}

	failureCond := GetCondition(rs.Status, apps.ReplicaSetReplicaFailure)
	if manageReplicasErr != nil && failureCond == nil {
		var reason string
		if diff := len(filteredPods) - int(*(rs.Spec.Replicas)); diff < 0 {
			reason = "FailedCreate"
		} else if diff > 0 {
			reason = "FailedDelete"
		}
		cond := NewReplicaSetCondition(apps.ReplicaSetReplicaFailure, v1.ConditionTrue, reason, manageReplicasErr.Error())
		SetCondition(&newStatus, cond)
	} else if manageReplicasErr == nil && failureCond != nil {
		RemoveCondition(&newStatus, apps.ReplicaSetReplicaFailure)
	}

	newStatus.Replicas = int32(len(filteredPods))
	newStatus.FullyLabeledReplicas = int32(fullyLabeledReplicasCount)
	newStatus.ReadyReplicas = int32(readyReplicasCount)
	newStatus.AvailableReplicas = int32(availableReplicasCount)
	return newStatus
}
```

<br>

### 4 总结

（1）expectations确实是一个很巧妙的方法，这种思想可以借鉴

（2）rs根本不感知deploy的存在