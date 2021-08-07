### 1. deploy基础概念

```
root@k8s-master# kubectl get deploy nginx-deployment -oyaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"            // 这个是版本号，说明这是第二个版本。
  generation: 4                                       // 这里有个 generation 
  labels:
    app: nginx
  name: nginx-deployment
  resourceVersion: "59522723"
  selfLink: /apis/apps/v1/namespaces/default/deployments/nginx-deployment
  uid: a6830e24-a479-452d-bbb2-3cb3cad82ebf
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 2                            // 这个表明只保留2个版本。
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%                          // 滚动更新的时候，不是一次就更新完了，而是一批一批的更新
      maxUnavailable: 25%                    //  升级过程中最多有多少个 pod 处于无法提供服务的状态
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 8080
          name: test1
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2020-11-28T08:35:07Z"
    lastUpdateTime: "2020-12-01T02:36:27Z"
    message: ReplicaSet "nginx-deployment-59bc6679cd" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2020-12-01T02:44:17Z"
    lastUpdateTime: "2020-12-01T02:44:17Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 4      //这里也有一个
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2
```

#### 1.1. metadata.generation & status.observedGeneration

这两个是对应的，metadata.generation 就是这个 ReplicationSet 的元配置数据被修改了多少次。这里就有个版本迭代的概念。每次我们使用 kuberctl edit 来修改 ReplicationSet 的配置文件，或者更新镜像，这个generation都会增长1，表示增加了一个版本。

这个版本迭代是配置文件只要有改动就进行版本迭代。observedGeneration就是最近观察到的可用的版本迭代。这两个只有在镜像升级的时候有可能不同，当我们使用 `kubectl rollout status` 来探测一个deployment的状态的时候，就是检查observedGeneration是否大于等于generation。

```
root@k8s-master:~# kubectl rollout status deployment kube-hpa -n kube-system
deployment "kube-hpa" successfully rolled out
```

<br>

#### 1.2. metadata.resourceVersion

每个资源在底层数据库都有版本的概念，我们可以使用 watch 来看某个资源，某个版本之后的操作。这些操作是存储在 etcd 中的。当让，并不是所有的操作都会永久存储，只会保留有限的时间的操作。这个 resourceVersion 就是这个资源对象当前的版本号。

#### 1.3 status

replicas 实际的 pod 副本数
availableReplicas 现在可用的 Pod 的副本数量，有的副本可能还处在未准备好，或者初始化状态
readyReplicas 是处于 ready 状态的 Pod 的副本数量
fullyLabeledReplicas 意思是这个 ReplicaSet 的标签 selector 对应的副本数量，不同纬度的一种统计

<br>

### 2. startDeploymentController

kcm启动时，NewControllerInitializers里面定义了所有要启动的manager，如下：

```
// NewControllerInitializers is a public map of named controller groups (you can start more than one in an init func)
// paired to their InitFunc.  This allows for structured downstream composition and subdivision.
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["endpointslice"] = startEndpointSliceController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController   //启动 deploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	controllers["csrsigning"] = startCSRSigningController
	controllers["csrapproving"] = startCSRApprovingController
	controllers["csrcleaner"] = startCSRCleanerController
	controllers["ttl"] = startTTLController
	controllers["bootstrapsigner"] = startBootstrapSignerController
	controllers["tokencleaner"] = startTokenCleanerController
	controllers["nodeipam"] = startNodeIpamController
	controllers["nodelifecycle"] = startNodeLifecycleController
	if loopMode == IncludeCloudLoops {
		controllers["service"] = startServiceController
		controllers["route"] = startRouteController
		controllers["cloud-node-lifecycle"] = startCloudNodeLifecycleController
		// TODO: volume controller into the IncludeCloudLoops only set.
	}
	controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
	controllers["attachdetach"] = startAttachDetachController
	controllers["persistentvolume-expander"] = startVolumeExpandController
	controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
	controllers["pvc-protection"] = startPVCProtectionController
	controllers["pv-protection"] = startPVProtectionController
	controllers["ttl-after-finished"] = startTTLAfterFinishedController
	controllers["root-ca-cert-publisher"] = startRootCACertPublisher

	return controllers
}
```

<br>

deployment 的本质是控制 replicaSet，replicaSet 会控制 pod，然后由 controller 驱动各个对象达到期望状态。所以deployController需要监听pod, rs, deploy三种资源的变化。

```go
cmd/kube-controller-manager/app/apps.go
func startDeploymentController(ctx ControllerContext) (http.Handler, bool, error) {
   // 判断当前是否支持deployment这种资源
   if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}] {
      return nil, false, nil
   }
   dc, err := deployment.NewDeploymentController(
      ctx.InformerFactory.Apps().V1().Deployments(),
      ctx.InformerFactory.Apps().V1().ReplicaSets(),
      ctx.InformerFactory.Core().V1().Pods(),
      ctx.ClientBuilder.ClientOrDie("deployment-controller"),
   )
   if err != nil {
      return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
   }
   go dc.Run(int(ctx.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs), ctx.Stop)
   return nil, true, nil
}
```

和其他控制器一样。new deploycontroller之后，就是run。run调用如下：

run->work->processNextItem->syncHandler 

定义时（new deploy），dc.syncHandler = dc.syncDeployment

<br>

### 3. NewDeploymentController

```go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
   // 记录event
   eventBroadcaster := record.NewBroadcaster()
   eventBroadcaster.StartLogging(glog.Infof)
   eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: client.CoreV1().Events("")})

   if client != nil && client.CoreV1().RESTClient().GetRateLimiter() != nil {
      if err := metrics.RegisterMetricAndTrackRateLimiterUsage("deployment_controller", client.CoreV1().RESTClient().GetRateLimiter()); err != nil {
         return nil, err
      }
   }
   dc := &DeploymentController{
      client:        client,
      eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
      queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
   }
   dc.rsControl = controller.RealRSControl{
      KubeClient: client,
      Recorder:   dc.eventRecorder,
   }

   dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc:    dc.addDeployment,
      UpdateFunc: dc.updateDeployment,
      // This will enter the sync loop and no-op, because the deployment has been deleted from the store.
      DeleteFunc: dc.deleteDeployment,
   })
   rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc:    dc.addReplicaSet,
      UpdateFunc: dc.updateReplicaSet,
      DeleteFunc: dc.deleteReplicaSet,
   })
   
  // pod只关注删除？
   podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      DeleteFunc: dc.deletePod,
   })

   dc.syncHandler = dc.syncDeployment
   dc.enqueueDeployment = dc.enqueue

   dc.dLister = dInformer.Lister()
   dc.rsLister = rsInformer.Lister()
   dc.podLister = podInformer.Lister()
   dc.dListerSynced = dInformer.Informer().HasSynced
   dc.rsListerSynced = rsInformer.Informer().HasSynced
   dc.podListerSynced = podInformer.Informer().HasSynced
   return dc, nil
}
```

从这里看出来，这里关注：

deploy的增删改，rs的增删改，pod的删除。

接下来就是 run->works->processNextWorkItem()->syncDeployment()

<br>

### 4. 对deploy, rs, pod的处理

在之前的分析中，有addDeployment，deleteDeployment，addReplicaSet等函数。这里看一下这些函数做了什么事情。

#### 4.1 add,update, del deploy

deploy相关的变化都是入队列

```go
func (dc *DeploymentController) addDeployment(obj interface{}) {
	d := obj.(*apps.Deployment)
	glog.V(4).Infof("Adding deployment %s", d.Name)
	dc.enqueueDeployment(d)
}

func (dc *DeploymentController) updateDeployment(old, cur interface{}) {
	oldD := old.(*apps.Deployment)
	curD := cur.(*apps.Deployment)
	glog.V(4).Infof("Updating deployment %s", oldD.Name)
	dc.enqueueDeployment(curD)
}

func (dc *DeploymentController) deleteDeployment(obj interface{}) {
	d, ok := obj.(*apps.Deployment)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Couldn't get object from tombstone %#v", obj))
			return
		}
		d, ok = tombstone.Obj.(*apps.Deployment)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Tombstone contained object that is not a Deployment %#v", obj))
			return
		}
	}
	glog.V(4).Infof("Deleting deployment %s", d.Name)
	dc.enqueueDeployment(d)
}
```

<br>

#### 4.2 add,update,del ReplicaSet

```go
// addReplicaSet enqueues the deployment that manages a ReplicaSet when the ReplicaSet is created.
func (dc *DeploymentController) addReplicaSet(obj interface{}) {
	rs := obj.(*apps.ReplicaSet)
    // 1.如果是删除，删除后返回
	if rs.DeletionTimestamp != nil {
		// On a restart of the controller manager, it's possible for an object to
		// show up in a state that is already pending deletion.
		dc.deleteReplicaSet(rs)
		return
	}
    
    // 2.判断owneref是否是deploy，是的话，讲对应的rs加入队列。
	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(rs); controllerRef != nil {
		d := dc.resolveControllerRef(rs.Namespace, controllerRef)
		if d == nil {
			return
		}
		klog.V(4).Infof("ReplicaSet %s added.", rs.Name)
		dc.enqueueDeployment(d)
		return
	}

    // 3. 否则，就是孤儿rs，通过label判断 rs是否属于某个deploy
	// Otherwise, it's an orphan. Get a list of all matching Deployments and sync
	// them to see if anyone wants to adopt it.
	ds := dc.getDeploymentsForReplicaSet(rs)
	if len(ds) == 0 {
		return
	}
	klog.V(4).Infof("Orphan ReplicaSet %s added.", rs.Name)
	for _, d := range ds {
		dc.enqueueDeployment(d)
	}
}
```



```go
// updateReplicaSet figures out what deployment(s) manage a ReplicaSet when the ReplicaSet
// is updated and wake them up. If the anything of the ReplicaSets have changed, we need to
// awaken both the old and new deployments. old and cur must be *apps.ReplicaSet
// types.
func (dc *DeploymentController) updateReplicaSet(old, cur interface{}) {
	curRS := cur.(*apps.ReplicaSet)
	oldRS := old.(*apps.ReplicaSet)
	// 1. 同样的，ResourceVersion可以判断资源有没有发生改变
	if curRS.ResourceVersion == oldRS.ResourceVersion {
		// Periodic resync will send update events for all known replica sets.
		// Two different versions of the same replica set will always have different RVs.
		return
	}

	curControllerRef := metav1.GetControllerOf(curRS)
	oldControllerRef := metav1.GetControllerOf(oldRS)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	// 2.先将旧对象删除。旧对象是deploy
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if d := dc.resolveControllerRef(oldRS.Namespace, oldControllerRef); d != nil {
			dc.enqueueDeployment(d)
		}
	}
   
    // 3. 处理新对象，如果新对象还是受deploy管，加入队列
	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
		d := dc.resolveControllerRef(curRS.Namespace, curControllerRef)
		if d == nil {
			return
		}
		klog.V(4).Infof("ReplicaSet %s updated.", curRS.Name)
		dc.enqueueDeployment(d)
		return
	}
   
    // 4. 孤儿rs。因为是更新，所以如果label都没有改，肯定就不用动。
	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	labelChanged := !reflect.DeepEqual(curRS.Labels, oldRS.Labels)
	if labelChanged || controllerRefChanged {
		ds := dc.getDeploymentsForReplicaSet(curRS)
		if len(ds) == 0 {
			return
		}
		klog.V(4).Infof("Orphan ReplicaSet %s updated.", curRS.Name)
		for _, d := range ds {
			dc.enqueueDeployment(d)
		}
	}
}
```



```go
// deleteReplicaSet enqueues the deployment that manages a ReplicaSet when
// the ReplicaSet is deleted. obj could be an *apps.ReplicaSet, or
// a DeletionFinalStateUnknown marker item.
func (dc *DeploymentController) deleteReplicaSet(obj interface{}) {
	rs, ok := obj.(*apps.ReplicaSet)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the ReplicaSet
	// changed labels the new deployment will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Couldn't get object from tombstone %#v", obj))
			return
		}
		rs, ok = tombstone.Obj.(*apps.ReplicaSet)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Tombstone contained object that is not a ReplicaSet %#v", obj))
			return
		}
	}

	controllerRef := metav1.GetControllerOf(rs)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	d := dc.resolveControllerRef(rs.Namespace, controllerRef)
	if d == nil {
		return
	}
	klog.V(4).Infof("ReplicaSet %s deleted.", rs.Name)
	// 加入队列
	dc.enqueueDeployment(d)
}
```

#### 4.3 del pod

```
// deletePod will enqueue a Recreate Deployment once all of its pods have stopped running.
func (dc *DeploymentController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the Pod
	// changed labels the new deployment will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Couldn't get object from tombstone %#v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Tombstone contained object that is not a pod %#v", obj))
			return
		}
	}
	glog.V(4).Infof("Pod %s deleted.", pod.Name)
	// 只有当pod全删除，才更新 deploy。这个判断说明是 recreate 了
	if d := dc.getDeploymentForPod(pod); d != nil && d.Spec.Strategy.Type == apps.RecreateDeploymentStrategyType {
		// Sync if this Deployment now has no more Pods.
		rsList, err := util.ListReplicaSets(d, util.RsListFromClient(dc.client.AppsV1()))
		if err != nil {
			return
		}
		podMap, err := dc.getPodMapForDeployment(d, rsList)
		if err != nil {
			return
		}
		numPods := 0
		for _, podList := range podMap {
			numPods += len(podList.Items)
		}
		if numPods == 0 {
			dc.enqueueDeployment(d)
		}
	}
}
```

deployment升级方案：

Recreate：删除所有已存在的pod,重新创建新的;

 RollingUpdate：滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。

<br>

#### 4.4 getDeploymentForPod

更加pod或得 rs，然后更加rs 或得deployment

```
// getDeploymentForPod returns the deployment managing the given Pod.
func (dc *DeploymentController) getDeploymentForPod(pod *v1.Pod) *apps.Deployment {
   // Find the owning replica set
   var rs *apps.ReplicaSet
   var err error
   controllerRef := metav1.GetControllerOf(pod)
   if controllerRef == nil {
      // No controller owns this Pod.
      return nil
   }
   if controllerRef.Kind != apps.SchemeGroupVersion.WithKind("ReplicaSet").Kind {
      // Not a pod owned by a replica set.
      return nil
   }
   rs, err = dc.rsLister.ReplicaSets(pod.Namespace).Get(controllerRef.Name)
   if err != nil || rs.UID != controllerRef.UID {
      klog.V(4).Infof("Cannot get replicaset %q for pod %q: %v", controllerRef.Name, pod.Name, err)
      return nil
   }

   // Now find the Deployment that owns that ReplicaSet.
   controllerRef = metav1.GetControllerOf(rs)
   if controllerRef == nil {
      return nil
   }
   return dc.resolveControllerRef(rs.Namespace, controllerRef)
}
```

#### 4.5 总结

从这里也可以看出来，deployment, rs的add, del, update都可能会导致deployment入队列，然后进入syncDeployment。

pod这里只关注删除，原因在于如果是recreate更新的时候，deploy等旧pod删除完才能创建新的pod。

<br>

### 5. syncDeployment

<br>

```
// syncDeployment will sync the deployment with the given key.
// This function is not meant to be invoked concurrently with the same key.
func (dc *DeploymentController) syncDeployment(key string) error {
	startTime := time.Now()
	klog.V(4).Infof("Started syncing deployment %q (%v)", key, startTime)
	defer func() {
		klog.V(4).Infof("Finished syncing deployment %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(2).Infof("Deployment %v has been deleted", key)
		return nil
	}
	if err != nil {
		return err
	}

	// Deep-copy otherwise we are mutating our cache.
	// TODO: Deep-copy only when needed.
	d := deployment.DeepCopy()
  
  // 1. 如果一个deploy的label是everything，会直接返回。（虽然会有判断是否要更新状态的说法）
	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(d)
		}
		return nil
	}
  
  // 2. 根据deploy获得rslist。以及根据rslist获得所有的pod（pod是一个map）
	// List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	// through adoption/orphaning.
	rsList, err := dc.getReplicaSetsForDeployment(d)
	if err != nil {
		return err
	}
	// List all Pods owned by this Deployment, grouped by their ReplicaSet.
	// Current uses of the podMap are:
	//
	// * check if a Pod is labeled correctly with the pod-template-hash label.
	// * check that no old Pods are running in the middle of Recreate Deployments.
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}
  
  // 3.如果是删除，则直接调用syncStatusOnly
	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(d, rsList)
	}

  // 4.检查是否处于 pause 状态
	// Update deployment conditions with an Unknown condition when pausing/resuming
	// a deployment. In this way, we can be sure that we won't timeout when a user
	// resumes a Deployment with a set progressDeadlineSeconds.
	if err = dc.checkPausedConditions(d); err != nil {
		return err
	}

  // 如果是 pause 状态,同步状态
	if d.Spec.Paused {
		return dc.sync(d, rsList)
	}

	// rollback is not re-entrant in case the underlying replica sets are updated with a new
	// revision so we should ensure that we won't proceed to update replica sets until we
	// make sure that the deployment has cleaned up its rollback spec in subsequent enqueues.
	// 5.如果annotations中有 deprecated.deployment.rollback.to 这个字段，则进行回滚
	if getRollbackTo(d) != nil {
		return dc.rollback(d, rsList)
	}

  // 6.检查 deployment 是否处于 scale 状态
	scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}

  // 7.更新deployment状态
	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```

syncDeployment的大流程如下：

（1）如果一个deploy的label是everything，会直接返回。

（2）根据deploy获得rslist。以及根据deploy的label获得所有的pod，然后以rs为key，返回一个podMap（pod是一个map）

（3）如果是删除，则直接调用syncStatusOnly，并返回

（4）检查是否处于 pause 状态，如果是pause，同步状态，并返回

（5）如果需要rollback，进行rollback,然后返回

（5）检查 deployment 是否处于 scale 状态，如果是scale, 同步状态，并返回

（6）如果是滚动更新或者是recreate更新，更新deployment状态，并返回

<br>

接下来从第三步开始，具体做了什么。

#### 5.1 删除deploy

删除deploy调用了syncStatusOnly函数。

syncStatusOn函数中主要调用了 getAllReplicaSetsAndSyncRevision 和 syncDeploymentStatus 函数。

##### 5.1.1 getAllReplicaSetsAndSyncRevision

getAllReplicaSetsAndSyncRevision 就是找出来 newRS, oldRSs。

newRs 就是：**最近的**，满足  rs.spec.template =  deploy.spec.temp  的rs。 **使用最近的原因在于rs.spec.template =  deploy.spec.temp  的rs可能有多个 **

oldRss 就是所有的rs中去掉 newRs。

```
// syncStatusOnly only updates Deployments Status and doesn't take any mutating actions.
func (dc *DeploymentController) syncStatusOnly(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return err
	}
  // 这里有点没看懂。 oldRSs + newRS = rsList。 为啥又要来一个allRSs
	allRSs := append(oldRSs, newRS)
	return dc.syncDeploymentStatus(allRSs, newRS, d)
}



// rsList should come from getReplicaSetsForDeployment(d).
//
// 1. Get all old RSes this deployment targets, and calculate the max revision number among them (maxOldV).
// 2. Get new RS this deployment targets (whose pod template matches deployment's), and update new RS's revision number to (maxOldV + 1),
//    only if its revision number is smaller than (maxOldV + 1). If this step failed, we'll update it in the next deployment sync loop.
// 3. Copy new RS's revision number to deployment (update deployment's revision). If this step failed, we'll update it in the next deployment sync loop.
//
// Note that currently the deployment controller is using caches to avoid querying the server for reads.
// This may lead to stale reads of replica sets, thus incorrect deployment status.
func (dc *DeploymentController) getAllReplicaSetsAndSyncRevision(d *apps.Deployment, rsList []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, []*apps.ReplicaSet, error) {
	_, allOldRSs := deploymentutil.FindOldReplicaSets(d, rsList)

	// Get new replica set with the updated revision number
	newRS, err := dc.getNewReplicaSet(d, rsList, allOldRSs, createIfNotExisted)
	if err != nil {
		return nil, nil, err
	}

	return newRS, allOldRSs, nil
}


// FindOldReplicaSets returns the old replica sets targeted by the given Deployment, with the given slice of RSes.
// Note that the first set of old replica sets doesn't include the ones with no pods, and the second set of old replica sets include all old replica sets.
func FindOldReplicaSets(deployment *apps.Deployment, rsList []*apps.ReplicaSet) ([]*apps.ReplicaSet, []*apps.ReplicaSet) {
	var requiredRSs []*apps.ReplicaSet
	var allRSs []*apps.ReplicaSet
	newRS := FindNewReplicaSet(deployment, rsList)
	for _, rs := range rsList {
		// Filter out new replica set
		if newRS != nil && rs.UID == newRS.UID {
			continue
		}
		allRSs = append(allRSs, rs)
		if *(rs.Spec.Replicas) != 0 {
			requiredRSs = append(requiredRSs, rs)
		}
	}
	return requiredRSs, allRSs
}

// FindNewReplicaSet returns the new RS this given deployment targets (the one with the same pod template).
func FindNewReplicaSet(deployment *apps.Deployment, rsList []*apps.ReplicaSet) *apps.ReplicaSet {
	sort.Sort(controller.ReplicaSetsByCreationTimestamp(rsList))
	for i := range rsList {
		if EqualIgnoreHash(&rsList[i].Spec.Template, &deployment.Spec.Template) {
			// In rare cases, such as after cluster upgrades, Deployment may end up with
			// having more than one new ReplicaSets that have the same template as its template,
			// see https://github.com/kubernetes/kubernetes/issues/40415
			// We deterministically choose the oldest new ReplicaSet.
			return rsList[i]
		}
	}
	// new ReplicaSet does not exist.
	return nil
}
```

##### 5.1.2 syncDeploymentStatus

calculateStatus 就是根据allRSs，newRS得到deploy最新的状态。然后再更新。

```
// syncDeploymentStatus checks if the status is up-to-date and sync it if necessary
func (dc *DeploymentController) syncDeploymentStatus(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, d *apps.Deployment) error {
	newStatus := calculateStatus(allRSs, newRS, d)

	if reflect.DeepEqual(d.Status, newStatus) {
		return nil
	}

	newDeployment := d
	newDeployment.Status = newStatus
	_, err := dc.client.AppsV1().Deployments(newDeployment.Namespace).UpdateStatus(newDeployment)
	return err
}


// calculateStatus calculates the latest status for the provided deployment by looking into the provided replica sets.
func calculateStatus(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) apps.DeploymentStatus {
	availableReplicas := deploymentutil.GetAvailableReplicaCountForReplicaSets(allRSs)
	totalReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
	unavailableReplicas := totalReplicas - availableReplicas
	// If unavailableReplicas is negative, then that means the Deployment has more available replicas running than
	// desired, e.g. whenever it scales down. In such a case we should simply default unavailableReplicas to zero.
	if unavailableReplicas < 0 {
		unavailableReplicas = 0
	}

	status := apps.DeploymentStatus{
		// TODO: Ensure that if we start retrying status updates, we won't pick up a new Generation value.
		ObservedGeneration:  deployment.Generation,
		Replicas:            deploymentutil.GetActualReplicaCountForReplicaSets(allRSs),
		UpdatedReplicas:     deploymentutil.GetActualReplicaCountForReplicaSets([]*apps.ReplicaSet{newRS}),
		ReadyReplicas:       deploymentutil.GetReadyReplicaCountForReplicaSets(allRSs),
		AvailableReplicas:   availableReplicas,
		UnavailableReplicas: unavailableReplicas,
		CollisionCount:      deployment.Status.CollisionCount,
	}

	// Copy conditions one by one so we won't mutate the original object.
	conditions := deployment.Status.Conditions
	for i := range conditions {
		status.Conditions = append(status.Conditions, conditions[i])
	}

	if availableReplicas >= *(deployment.Spec.Replicas)-deploymentutil.MaxUnavailable(*deployment) {
		minAvailability := deploymentutil.NewDeploymentCondition(apps.DeploymentAvailable, v1.ConditionTrue, deploymentutil.MinimumReplicasAvailable, "Deployment has minimum availability.")
		deploymentutil.SetDeploymentCondition(&status, *minAvailability)
	} else {
		noMinAvailability := deploymentutil.NewDeploymentCondition(apps.DeploymentAvailable, v1.ConditionFalse, deploymentutil.MinimumReplicasUnavailable, "Deployment does not have minimum availability.")
		deploymentutil.SetDeploymentCondition(&status, *noMinAvailability)
	}

	return status
}
```

<br>

##### 5.1.3 总结

deploy 根据DeletionTimestamp判断该pod是否需要删除，deploy controller并没有进行deploy的删除，而是仅仅更新了状态。deploy的删除时gc来做的。后文在详细分析。

这里注意一个问题就是deploy的DeletionTimestamp到底是谁加上去的。

答案是：APIsever。 当kubectl delete 的时候。最终kubelet调用了会调用到 store里面的DELETE函数，这里的的操作就是给DeletionTimestamp赋值。

<br>

#### 5.2 pause操作

目前比较少用到。暂时忽略。

#### 5.3 Rollback操作

（1）判断deploy的annotations中是否有"deprecated.deployment.rollback.to" 字段的key，如果有需要rollback

（2）获取deprecated.deployment.rollback.to对应的value, 这个就是表示是需要rollback到哪个rs

（3）将rs.sepc.template 赋值给 deployment.spec.template

（4）更新deploy, 删除annotations中 deprecated.deployment.rollback.to字段

特殊情况：如果value=0，则更新到最近的版本。如果value不存在，则忽略。

```
if getRollbackTo(d) != nil {
   return dc.rollback(d, rsList)
}

// getRollbackTo 就是判断deploy的annotations中是否有"deprecated.deployment.rollback.to" 字段的key
// TODO: Remove this when extensions/v1beta1 and apps/v1beta1 Deployment are dropped.
func getRollbackTo(d *apps.Deployment) *extensions.RollbackConfig {
	// Extract the annotation used for round-tripping the deprecated RollbackTo field.
	revision := d.Annotations[apps.DeprecatedRollbackTo]
	if revision == "" {
		return nil
	}
	revision64, err := strconv.ParseInt(revision, 10, 64)
	if err != nil {
		// If it's invalid, ignore it.
		return nil
	}
	return &extensions.RollbackConfig{
		Revision: revision64,
	}
}

// 这里的核心思想就是找到对应版本的rs。然后将rs.sepc.template 赋值给 deployment.spec.template
// 然后更新deploy, 删除annotations中 deprecated.deployment.rollback.to
// rollback the deployment to the specified revision. In any case cleanup the rollback spec.
func (dc *DeploymentController) rollback(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, allOldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
	if err != nil {
		return err
	}

	allRSs := append(allOldRSs, newRS)
	rollbackTo := getRollbackTo(d)
	// If rollback revision is 0, rollback to the last revision
	if rollbackTo.Revision == 0 {
		if rollbackTo.Revision = deploymentutil.LastRevision(allRSs); rollbackTo.Revision == 0 {
			// If we still can't find the last revision, gives up rollback
			dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find last revision.")
			// Gives up rollback
			return dc.updateDeploymentAndClearRollbackTo(d)
		}
	}
	for _, rs := range allRSs {
		v, err := deploymentutil.Revision(rs)
		if err != nil {
			klog.V(4).Infof("Unable to extract revision from deployment's replica set %q: %v", rs.Name, err)
			continue
		}
		if v == rollbackTo.Revision {
			klog.V(4).Infof("Found replica set %q with desired revision %d", rs.Name, v)
			// rollback by copying podTemplate.Spec from the replica set
			// revision number will be incremented during the next getAllReplicaSetsAndSyncRevision call
			// no-op if the spec matches current deployment's podTemplate.Spec
			performedRollback, err := dc.rollbackToTemplate(d, rs)
			if performedRollback && err == nil {
				dc.emitRollbackNormalEvent(d, fmt.Sprintf("Rolled back deployment %q to revision %d", d.Name, rollbackTo.Revision))
			}
			return err
		}
	}
	dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find the revision to rollback to.")
	// Gives up rollback
	return dc.updateDeploymentAndClearRollbackTo(d)
}
```

<br>

#### 5.4 scale操作

（1）判断是否需要scale, 这里通过deploy.spec.Replicas是否等于 rs中的annotations中的desired来判断，不相等就要scale

（2）调用scale进行扩缩

```
scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}
	
// 这里就是判断 deploy.spec.Replicas是否等于 rs中的annotations中的desired，不相等就要scale
// isScalingEvent checks whether the provided deployment has been updated with a scaling event
// by looking at the desired-replicas annotation in the active replica sets of the deployment.
//
// rsList should come from getReplicaSetsForDeployment(d).
// podMap should come from getPodMapForDeployment(d, rsList).
func (dc *DeploymentController) isScalingEvent(d *apps.Deployment, rsList []*apps.ReplicaSet) (bool, error) {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return false, err
	}
	allRSs := append(oldRSs, newRS)
	for _, rs := range controller.FilterActiveReplicaSets(allRSs) {
		desired, ok := deploymentutil.GetDesiredReplicasAnnotation(rs)
		if !ok {
			continue
		}
		if desired != *(d.Spec.Replicas) {
			return true, nil
		}
	}
	return false, nil
}

//以一个rs为例，annotations中确实有desired-replicas
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "1"
    deployment.kubernetes.io/max-replicas: "2"
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-06-12T14:47:22Z"
```

<br>

调用sync->scale 进行扩缩，主要逻辑如下：

（1） 获得最新的一个activeRs，进行扩缩容

（2）如果newRS已经是期望状态，将所有的oldRS缩到0

（3）如果是滚动更新，根据MaxSurge等字段，一步一步的更新，oldRs和newRs。最终的状态是newrs是期望状态，oldrs都是0。

这里如果是recreate更新，则什么都不会做，等到旧pod删除完了之后，自然会进入（1），就直接扩缩容就行了。

```
// sync is responsible for reconciling deployments on scaling events or when they
// are paused.
func (dc *DeploymentController) sync(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return err
	}
	if err := dc.scale(d, newRS, oldRSs); err != nil {
		// If we get an error while trying to scale, the deployment will be requeued
		// so we can abort this resync
		return err
	}

	// Clean up the deployment when it's paused and no rollback is in flight.
	if d.Spec.Paused && getRollbackTo(d) == nil {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	allRSs := append(oldRSs, newRS)
	return dc.syncDeploymentStatus(allRSs, newRS, d)
}


// scale scales proportionally in order to mitigate risk. Otherwise, scaling up can increase the size
// of the new replica set and scaling down can decrease the sizes of the old ones, both of which would
// have the effect of hastening the rollout progress, which could produce a higher proportion of unavailable
// replicas in the event of a problem with the rolled out template. Should run only on scaling events or
// when a deployment is paused and not during the normal rollout process.
func (dc *DeploymentController) scale(deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
	// If there is only one active replica set then we should scale that up to the full count of the
	// deployment. If there is no active replica set, then we should scale up the newest replica set.
	// 1. 获得最新的一个activeRs，进行扩缩容
	if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
		if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
			return nil
		}
		_, _, err := dc.scaleReplicaSetAndRecordEvent(activeOrLatest, *(deployment.Spec.Replicas), deployment)
		return err
	}
  
  // 2. 如果newRS已经是期望状态，将所有的oldRS缩到0
	// If the new replica set is saturated, old replica sets should be fully scaled down.
	// This case handles replica set adoption during a saturated new replica set.
	if deploymentutil.IsSaturated(deployment, newRS) {
		for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
			if _, _, err := dc.scaleReplicaSetAndRecordEvent(old, 0, deployment); err != nil {
				return err
			}
		}
		return nil
	}

  // 3. 如果是滚动更新，根据MaxSurge等字段，一步一步的更新，oldRs和newRs。最终的状态是newrs是期望状态，oldrs都是0。
	// There are old replica sets with pods and the new replica set is not saturated.
	// We need to proportionally scale all replica sets (new and old) in case of a
	// rolling deployment.
	if deploymentutil.IsRollingUpdate(deployment) {
		allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
		allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

		allowedSize := int32(0)
		if *(deployment.Spec.Replicas) > 0 {
			allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
		}

		// Number of additional replicas that can be either added or removed from the total
		// replicas count. These replicas should be distributed proportionally to the active
		// replica sets.
		deploymentReplicasToAdd := allowedSize - allRSsReplicas

		// The additional replicas should be distributed proportionally amongst the active
		// replica sets from the larger to the smaller in size replica set. Scaling direction
		// drives what happens in case we are trying to scale replica sets of the same size.
		// In such a case when scaling up, we should scale up newer replica sets first, and
		// when scaling down, we should scale down older replica sets first.
		var scalingOperation string
		switch {
		case deploymentReplicasToAdd > 0:
			sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
			scalingOperation = "up"

		case deploymentReplicasToAdd < 0:
			sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
			scalingOperation = "down"
		}

		// Iterate over all active replica sets and estimate proportions for each of them.
		// The absolute value of deploymentReplicasAdded should never exceed the absolute
		// value of deploymentReplicasToAdd.
		deploymentReplicasAdded := int32(0)
		nameToSize := make(map[string]int32)
		for i := range allRSs {
			rs := allRSs[i]

			// Estimate proportions if we have replicas to add, otherwise simply populate
			// nameToSize with the current sizes for each replica set.
			if deploymentReplicasToAdd != 0 {
				proportion := deploymentutil.GetProportion(rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)

				nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
				deploymentReplicasAdded += proportion
			} else {
				nameToSize[rs.Name] = *(rs.Spec.Replicas)
			}
		}

		// Update all replica sets
		for i := range allRSs {
			rs := allRSs[i]

			// Add/remove any leftovers to the largest replica set.
			if i == 0 && deploymentReplicasToAdd != 0 {
				leftover := deploymentReplicasToAdd - deploymentReplicasAdded
				nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
				if nameToSize[rs.Name] < 0 {
					nameToSize[rs.Name] = 0
				}
			}

			// TODO: Use transactions when we have them.
			if _, _, err := dc.scaleReplicaSet(rs, nameToSize[rs.Name], deployment, scalingOperation); err != nil {
				// Return as soon as we fail, the deployment is requeued
				return err
			}
		}
	}
	return nil
}
```

<br>

##### 5.4.1 获得最新的一个activeRs

从这里可以看出来：activeRs 就是 rs.Spec.Replica>0 的rs

这里的逻辑就是：

* 如果没有一个rs是active的，那就当newRS是当前要扩缩容的。newRs 就是：**最近的**，满足  rs.spec.template =  deploy.spec.temp  的rs。
* 如果有一个active的rs。那么当其作为要扩缩容的。
* 如果找到多个active的rs, 那么表示这个可能是滚动更新等复杂情况，走后面的逻辑。
* 扩缩容直接调用了scaleReplicaSetAndRecordEvent函数，这个最后分析。

```
	if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
		if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
			return nil
		}
		_, _, err := dc.scaleReplicaSetAndRecordEvent(activeOrLatest, *(deployment.Spec.Replicas), deployment)
		return err
	}
	
	
// FindActiveOrLatest returns the only active or the latest replica set in case there is at most one active
// replica set. If there are more active replica sets, then we should proportionally scale them.
func FindActiveOrLatest(newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) *apps.ReplicaSet {
	if newRS == nil && len(oldRSs) == 0 {
		return nil
	}

	sort.Sort(sort.Reverse(controller.ReplicaSetsByCreationTimestamp(oldRSs)))
	allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))

	switch len(allRSs) {
	case 0:
		// If there is no active replica set then we should return the newest.
		if newRS != nil {
			return newRS
		}
		return oldRSs[0]
	case 1:
		return allRSs[0]
	default:
		return nil
	}
}


// FilterActiveReplicaSets returns replica sets that have (or at least ought to have) pods.
func FilterActiveReplicaSets(replicaSets []*apps.ReplicaSet) []*apps.ReplicaSet {
	activeFilter := func(rs *apps.ReplicaSet) bool {
		return rs != nil && *(rs.Spec.Replicas) > 0
	}
	return FilterReplicaSets(replicaSets, activeFilter)
}

type filterRS func(rs *apps.ReplicaSet) bool

// FilterReplicaSets returns replica sets that are filtered by filterFn (all returned ones should match filterFn).
func FilterReplicaSets(RSes []*apps.ReplicaSet, filterFn filterRS) []*apps.ReplicaSet {
	var filtered []*apps.ReplicaSet
	for i := range RSes {
		if filterFn(RSes[i]) {
			filtered = append(filtered, RSes[i])
		}
	}
	return filtered
}
```

<br>

##### 5.4.2 如果newRS已经是期望状态，将所有的oldRS缩到0

从这里很直观就可以看出来

```
	// If the new replica set is saturated, old replica sets should be fully scaled down.
	// This case handles replica set adoption during a saturated new replica set.
	if deploymentutil.IsSaturated(deployment, newRS) {
		for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
			if _, _, err := dc.scaleReplicaSetAndRecordEvent(old, 0, deployment); err != nil {
				return err
			}
		}
		return nil
	}
	
	// IsSaturated checks if the new replica set is saturated by comparing its size with its deployment size.
// Both the deployment and the replica set have to believe this replica set can own all of the desired
// replicas in the deployment and the annotation helps in achieving that. All pods of the ReplicaSet
// need to be available.
func IsSaturated(deployment *apps.Deployment, rs *apps.ReplicaSet) bool {
	if rs == nil {
		return false
	}
	desiredString := rs.Annotations[DesiredReplicasAnnotation]
	desired, err := strconv.Atoi(desiredString)
	if err != nil {
		return false
	}
	return *(rs.Spec.Replicas) == *(deployment.Spec.Replicas) &&
		int32(desired) == *(deployment.Spec.Replicas) &&
		rs.Status.AvailableReplicas == *(deployment.Spec.Replicas)
}	
```

<br>

#### 5.5 recreate更新

这种策略就非常简单。先将所有旧rs scaledown到0。然后再将newRs扩到期望值。这里需要注意的是，如果旧rs还有pod running，这这个时候是再次同步，也就是说新的rs是等所有旧pod全部删除完了之后，才会开始创建。

```
// rolloutRecreate implements the logic for recreating a replica set.
func (dc *DeploymentController) rolloutRecreate(d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
	// Don't create a new RS if not already existed, so that we avoid scaling up before scaling down.
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)
	activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)

	// scale down old replica sets.
	scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(activeOldRSs, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// Update DeploymentStatus.
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

   // 如果旧rs还有pod running，这个时候是再次同步。
	// Do not process a deployment when it has old pods running.
	if oldPodsRunning(newRS, oldRSs, podMap) {
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// If we need to create a new RS, create it now.
	if newRS == nil {
		newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
		if err != nil {
			return err
		}
		allRSs = append(oldRSs, newRS)
	}

	// scale up new replica set.
	if _, err := dc.scaleUpNewReplicaSetForRecreate(newRS, d); err != nil {
		return err
	}

	if util.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// Sync deployment status.
	return dc.syncRolloutStatus(allRSs, newRS, d)
}



// scaleDownOldReplicaSetsForRecreate scales down old replica sets when deployment strategy is "Recreate".
func (dc *DeploymentController) scaleDownOldReplicaSetsForRecreate(oldRSs []*apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
	scaled := false
	for i := range oldRSs {
		rs := oldRSs[i]
		// Scaling not required.
		if *(rs.Spec.Replicas) == 0 {
			continue
		}
		scaledRS, updatedRS, err := dc.scaleReplicaSetAndRecordEvent(rs, 0, deployment)
		if err != nil {
			return false, err
		}
		if scaledRS {
			oldRSs[i] = updatedRS
			scaled = true
		}
	}
	return scaled, nil
}
```

<br>

#### 5.6 rolloutRolling更新

（1）获得newRS, oldRSs

（2）如果是scaledUp，返回 syncRolloutStatus

（3）如果是scaledDown，返回syncRolloutStatus

（4）如果到了这里，说明不是scaledUp也不是scaledDown，那说明可能是达到了期望值，通过DeploymentComplete判断一下

（5）同步状态

```
// rolloutRolling implements the logic for rolling a new replica set.
func (dc *DeploymentController) rolloutRolling(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
   newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
   if err != nil {
      return err
   }
   allRSs := append(oldRSs, newRS)

   // Scale up, if we can.
   scaledUp, err := dc.reconcileNewReplicaSet(allRSs, newRS, d)
   if err != nil {
      return err
   }
   if scaledUp {
      // Update DeploymentStatus
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   // Scale down, if we can.
   scaledDown, err := dc.reconcileOldReplicaSets(allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
   if err != nil {
      return err
   }
   if scaledDown {
      // Update DeploymentStatus
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   if deploymentutil.DeploymentComplete(d, &d.Status) {
      if err := dc.cleanupDeployment(oldRSs, d); err != nil {
         return err
      }
   }

   // Sync deployment status
   return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

<br>

##### 5.6.1 如果是scaledUp（针对news），返回 syncRolloutStatus

这里就是判断是否是scaleup，如果是，还计算了一下需要扩容的副本数。这里更加了更新策略，以及MaxSurge等因素来计算。

然后通过scaleReplicaSetAndRecordEvent来修改rs并发送事件。

```
func (dc *DeploymentController) reconcileNewReplicaSet(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
	if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
		// Scaling not required.
		return false, nil
	}
	if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
		// Scale down.
		scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, *(deployment.Spec.Replicas), deployment)
		return scaled, err
	}
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
	if err != nil {
		return false, err
	}
	scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, newReplicasCount, deployment)
	return scaled, err
}


// NewRSNewReplicas calculates the number of replicas a deployment's new RS should have.
// When one of the followings is true, we're rolling out the deployment; otherwise, we're scaling it.
// 1) The new RS is saturated: newRS's replicas == deployment's replicas
// 2) Max number of pods allowed is reached: deployment's replicas + maxSurge == all RSs' replicas
func NewRSNewReplicas(deployment *apps.Deployment, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet) (int32, error) {
	switch deployment.Spec.Strategy.Type {
	case apps.RollingUpdateDeploymentStrategyType:
		// Check if we can scale up.
		maxSurge, err := intstrutil.GetValueFromIntOrPercent(deployment.Spec.Strategy.RollingUpdate.MaxSurge, int(*(deployment.Spec.Replicas)), true)
		if err != nil {
			return 0, err
		}
		// Find the total number of pods
		currentPodCount := GetReplicaCountForReplicaSets(allRSs)
		maxTotalPods := *(deployment.Spec.Replicas) + int32(maxSurge)
		if currentPodCount >= maxTotalPods {
			// Cannot scale up.
			return *(newRS.Spec.Replicas), nil
		}
		// Scale up.
		scaleUpCount := maxTotalPods - currentPodCount
		// Do not exceed the number of desired replicas.
		scaleUpCount = int32(integer.IntMin(int(scaleUpCount), int(*(deployment.Spec.Replicas)-*(newRS.Spec.Replicas))))
		return *(newRS.Spec.Replicas) + scaleUpCount, nil
	case apps.RecreateDeploymentStrategyType:
		return *(deployment.Spec.Replicas), nil
	default:
		return 0, fmt.Errorf("deployment type %v isn't supported", deployment.Spec.Strategy.Type)
	}
}
```

<br>

scaledown同样也是差不多的逻辑。计算的是当前旧rs应该减少的部分。

<br>

#### 5.7 scaleReplicaSetAndRecordEvent

这个函数作用和名字一样。通过restful 对rs进行 scale。 然后用事件记录。

```
func (dc *DeploymentController) scaleReplicaSetAndRecordEvent(rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment) (bool, *apps.ReplicaSet, error) {
	// No need to scale
	if *(rs.Spec.Replicas) == newScale {
		return false, rs, nil
	}
	var scalingOperation string
	if *(rs.Spec.Replicas) < newScale {
		scalingOperation = "up"
	} else {
		scalingOperation = "down"
	}
	scaled, newRS, err := dc.scaleReplicaSet(rs, newScale, deployment, scalingOperation)
	return scaled, newRS, err
}


func (dc *DeploymentController) scaleReplicaSet(rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment, scalingOperation string) (bool, *apps.ReplicaSet, error) {

	sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale

	annotationsNeedUpdate := deploymentutil.ReplicasAnnotationsNeedUpdate(rs, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))

	scaled := false
	var err error
	if sizeNeedsUpdate || annotationsNeedUpdate {
		rsCopy := rs.DeepCopy()
		*(rsCopy.Spec.Replicas) = newScale
		deploymentutil.SetReplicasAnnotations(rsCopy, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
		rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(rsCopy)
		if err == nil && sizeNeedsUpdate {
			scaled = true
			dc.eventRecorder.Eventf(deployment, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled %s replica set %s to %d", scalingOperation, rs.Name, newScale)
		}
	}
	return scaled, rs, err
}
```

<br>

