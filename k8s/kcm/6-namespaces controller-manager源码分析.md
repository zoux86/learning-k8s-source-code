Table of Contents
=================

  * [1. startNamespaceController](#1-startnamespacecontroller)
  * [2. NewNamespaceController](#2-newnamespacecontroller)
  * [3. Run](#3-run)
     * [3.1 syncNamespaceFromKey](#31-syncnamespacefromkey)
     * [3.2. deleteAllContent](#32-deleteallcontent)
  * [4 总结](#4-总结)

和其他kcm控制器组件一样，nsController还是在NewControllerInitializers定义然后启动。

### 1. startNamespaceController

cmd\kube-controller-manager\app\core.go

这里有很多的startController函数。这个就是 startControllers里面各个controller对应的init函数。

ns也是用了令牌桶进行限速。

```
func startNamespaceController(ctx ControllerContext) (http.Handler, bool, error) {
	// the namespace cleanup controller is very chatty.  It makes lots of discovery calls and then it makes lots of delete calls
	// the ratelimiter negatively affects its speed.  Deleting 100 total items in a namespace (that's only a few of each resource
	// including events), takes ~10 seconds by default.
	nsKubeconfig := ctx.ClientBuilder.ConfigOrDie("namespace-controller")
	nsKubeconfig.QPS *= 20
	nsKubeconfig.Burst *= 100
	namespaceKubeClient := clientset.NewForConfigOrDie(nsKubeconfig)
	return startModifiedNamespaceController(ctx, namespaceKubeClient, nsKubeconfig)
}


func startModifiedNamespaceController(ctx ControllerContext, namespaceKubeClient clientset.Interface, nsKubeconfig *restclient.Config) (http.Handler, bool, error) {

	metadataClient, err := metadata.NewForConfig(nsKubeconfig)
	if err != nil {
		return nil, true, err
	}

	discoverResourcesFn := namespaceKubeClient.Discovery().ServerPreferredNamespacedResources

	namespaceController := namespacecontroller.NewNamespaceController(
		namespaceKubeClient,
		metadataClient,
		discoverResourcesFn,
		ctx.InformerFactory.Core().V1().Namespaces(),
		ctx.ComponentConfig.NamespaceController.NamespaceSyncPeriod.Duration,
		v1.FinalizerKubernetes,
	)
	go namespaceController.Run(int(ctx.ComponentConfig.NamespaceController.ConcurrentNamespaceSyncs), ctx.Stop)

	return nil, true, nil
}
```

（1）NewNamespaceController

（2）nsController.Run

<br>

### 2. NewNamespaceController

```
// NewNamespaceController creates a new NamespaceController
func NewNamespaceController(
	kubeClient clientset.Interface,
	metadataClient metadata.Interface,
	discoverResourcesFn func() ([]*metav1.APIResourceList, error),
	namespaceInformer coreinformers.NamespaceInformer,
	resyncPeriod time.Duration,
	finalizerToken v1.FinalizerName) *NamespaceController {

	// create the controller so we can inject the enqueue function
	namespaceController := &NamespaceController{
		queue:                      workqueue.NewNamedRateLimitingQueue(nsControllerRateLimiter(), "namespace"),
		namespacedResourcesDeleter: deletion.NewNamespacedResourcesDeleter(kubeClient.CoreV1().Namespaces(), metadataClient, kubeClient.CoreV1(), discoverResourcesFn, finalizerToken),
	}

	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		ratelimiter.RegisterMetricAndTrackRateLimiterUsage("namespace_controller", kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	// configure the namespace informer event handlers
	namespaceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				namespace := obj.(*v1.Namespace)
				namespaceController.enqueueNamespace(namespace)
			},
			UpdateFunc: func(oldObj, newObj interface{}) {
				namespace := newObj.(*v1.Namespace)
				namespaceController.enqueueNamespace(namespace)
			},
		},
		resyncPeriod,
	)
	namespaceController.lister = namespaceInformer.Lister()
	namespaceController.listerSynced = namespaceInformer.Informer().HasSynced

	return namespaceController
}
```

NewNamespaceController就是定义了kubeconfig，然后定义好需要监听对象的处理函数。这里这监听addFunc, updateFunc。

Q：delete为啥不监听？

A：因为ns都delete掉了，监听没什么意义。ns 有deletionStamp，是update事件。

<br>

### 3. Run

定义好控制器之后就开始运行了。

```go
func (nm *NamespaceController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer nm.queue.ShutDown()

	klog.Infof("Starting namespace controller")
	defer klog.Infof("Shutting down namespace controller")

	if !cache.WaitForNamedCacheSync("namespace", stopCh, nm.listerSynced) {
		return
	}

	klog.V(5).Info("Starting workers of namespace controller")
	for i := 0; i < workers; i++ {
		go wait.Until(nm.worker, time.Second, stopCh)
	}
	<-stopCh
}
```

还是一样，run完之后调用 worker,并发处理队列中的namespace。这里并没有processNextItem()

可以看出来，work函数会一直循环处理一个ns。知道这个ns从队列中被移除。

```
// worker processes the queue of namespace objects.
// Each namespace can be in the queue at most once.
// The system ensures that no two workers can process
// the same namespace at the same time.
func (nm *NamespaceController) worker() {
	workFunc := func() bool {
		key, quit := nm.queue.Get()
		if quit {
			return true
		}
		defer nm.queue.Done(key)

		err := nm.syncNamespaceFromKey(key.(string))
		if err == nil {
			// no error, forget this entry and return
			nm.queue.Forget(key)
			return false
		}

		if estimate, ok := err.(*deletion.ResourcesRemainingError); ok {
			t := estimate.Estimate/2 + 1
			klog.V(4).Infof("Content remaining in namespace %s, waiting %d seconds", key, t)
			nm.queue.AddAfter(key, time.Duration(t)*time.Second)
		} else {
			// rather than wait for a full resync, re-add the namespace to the queue to be processed
			nm.queue.AddRateLimited(key)
			utilruntime.HandleError(fmt.Errorf("deletion of namespace %v failed: %v", key, err))
		}
		return false
	}

	for {
		quit := workFunc()

		if quit {
			return
		}
	}
}
```

<br>

#### 3.1 syncNamespaceFromKey

syncNamespaceFromKey主要调用了nm.namespacedResourcesDeleter.Delete，它们的逻辑如下：

（1）如果namespace不存在，返回nil

（2）如果namespace没有DeletionTimestamp字段，返回nil

（3）可以删除的话，先删除namespaces下所以的资源，如果某一个资源删除需要等待，返回一个ResourcesRemainingError

（4）所有的资源删除完后，删除namespace。

```
// syncNamespaceFromKey looks for a namespace with the specified key in its store and synchronizes it
func (nm *NamespaceController) syncNamespaceFromKey(key string) (err error) {
	startTime := time.Now()
	defer func() {
		glog.V(4).Infof("Finished syncing namespace %q (%v)", key, time.Since(startTime))
	}()
  
  // 1.如果namespace不存在，返回nil
	namespace, err := nm.lister.Get(key)
	if errors.IsNotFound(err) {
		glog.Infof("Namespace has been deleted %v", key)
		return nil
	}
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Unable to retrieve namespace %v from store: %v", key, err))
		return err
	}
	return nm.namespacedResourcesDeleter.Delete(namespace.Name)
}
```

Delete函数的主要逻辑就是：如果ns不需要删除就返回，需要删除就先删除资源，再删除ns。

具体为：

（1）ns没有DeletionTimestamp不做任何操作

（2）如果没有Finalizers,也不处理

（3）

```
// Delete deletes all resources in the given namespace.
// Before deleting resources:
// * It ensures that deletion timestamp is set on the
//   namespace (does nothing if deletion timestamp is missing).
// * Verifies that the namespace is in the "terminating" phase
//   (updates the namespace phase if it is not yet marked terminating)
// After deleting the resources:
// * It removes finalizer token from the given namespace.
//
// Returns an error if any of those steps fail.
// Returns ResourcesRemainingError if it deleted some resources but needs
// to wait for them to go away.
// Caller is expected to keep calling this until it succeeds.
func (d *namespacedResourcesDeleter) Delete(nsName string) error {
	// Multiple controllers may edit a namespace during termination
	// first get the latest state of the namespace before proceeding
	// if the namespace was deleted already, don't do anything
	namespace, err := d.nsClient.Get(nsName, metav1.GetOptions{})
	if err != nil {
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}
	// 1.ns没有DeletionTimestamp不做任何操作。
	if namespace.DeletionTimestamp == nil {
		return nil
	}

	klog.V(5).Infof("namespace controller - syncNamespace - namespace: %s, finalizerToken: %s", namespace.Name, d.finalizerToken)

	// ensure that the status is up to date on the namespace
	// if we get a not found error, we assume the namespace is truly gone
	// 2. 对ns的状态进行修改。retryOnConflictError是一个通用的函数，updateNamespaceStatusFunc是实际修改的    // 函数，这里就是修改ns的 phase为Terminating
	namespace, err = d.retryOnConflictError(namespace, d.updateNamespaceStatusFunc)
	if err != nil {
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}

	// the latest view of the namespace asserts that namespace is no longer deleting..
	if namespace.DeletionTimestamp.IsZero() {
		return nil
	}
  
  
  // 2.如果没有Finalizers,也不处理
	// return if it is already finalized.
	if finalized(namespace) {
		return nil
	}
  
  // 3.开始删除ns下的所有资源，estimate表示有多少个资源删除不了
	// there may still be content for us to remove
	estimate, err := d.deleteAllContent(namespace)
	if err != nil {
		return err
	}
	if estimate > 0 {
		return &ResourcesRemainingError{estimate}
	}
   
   // 移除finalize，然后apiserver能够删除
	// we have removed content, so mark it finalized by us
	_, err = d.retryOnConflictError(namespace, d.finalizeNamespace)
	if err != nil {
		// in normal practice, this should not be possible, but if a deployment is running
		// two controllers to do namespace deletion that share a common finalizer token it's
		// possible that a not found could occur since the other controller would have finished the delete.
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}
	return nil
}
```

<br>

#### 3.2. deleteAllContent

这里是用了 dynamic client 一个一个的删除所有的对象, 这里看起来就是并行的

```
// deleteAllContent will use the dynamic client to delete each resource identified in groupVersionResources.
// It returns an estimate of the time remaining before the remaining resources are deleted.
// If estimate > 0, not all resources are guaranteed to be gone.
func (d *namespacedResourcesDeleter) deleteAllContent(ns *v1.Namespace) (int64, error) {
	namespace := ns.Name
	namespaceDeletedAt := *ns.DeletionTimestamp
	var errs []error
	conditionUpdater := namespaceConditionUpdater{}
	estimate := int64(0)
	klog.V(4).Infof("namespace controller - deleteAllContent - namespace: %s", namespace)

	resources, err := d.discoverResourcesFn()
	if err != nil {
		// discovery errors are not fatal.  We often have some set of resources we can operate against even if we don't have a complete list
		errs = append(errs, err)
		conditionUpdater.ProcessDiscoverResourcesErr(err)
	}
	// TODO(sttts): get rid of opCache and pass the verbs (especially "deletecollection") down into the deleter
	deletableResources := discovery.FilteredBy(discovery.SupportsAllVerbs{Verbs: []string{"delete"}}, resources)
	groupVersionResources, err := discovery.GroupVersionResources(deletableResources)
	if err != nil {
		// discovery errors are not fatal.  We often have some set of resources we can operate against even if we don't have a complete list
		errs = append(errs, err)
		conditionUpdater.ProcessGroupVersionErr(err)
	}

	numRemainingTotals := allGVRDeletionMetadata{
		gvrToNumRemaining:        map[schema.GroupVersionResource]int{},
		finalizersToNumRemaining: map[string]int{},
	}
	for gvr := range groupVersionResources {
		gvrDeletionMetadata, err := d.deleteAllContentForGroupVersionResource(gvr, namespace, namespaceDeletedAt)
		if err != nil {
			// If there is an error, hold on to it but proceed with all the remaining
			// groupVersionResources.
			errs = append(errs, err)
			conditionUpdater.ProcessDeleteContentErr(err)
		}
		if gvrDeletionMetadata.finalizerEstimateSeconds > estimate {
			estimate = gvrDeletionMetadata.finalizerEstimateSeconds
		}
		if gvrDeletionMetadata.numRemaining > 0 {
			numRemainingTotals.gvrToNumRemaining[gvr] = gvrDeletionMetadata.numRemaining
			for finalizer, numRemaining := range gvrDeletionMetadata.finalizersToNumRemaining {
				if numRemaining == 0 {
					continue
				}
				numRemainingTotals.finalizersToNumRemaining[finalizer] = numRemainingTotals.finalizersToNumRemaining[finalizer] + numRemaining
			}
		}
	}
	conditionUpdater.ProcessContentTotals(numRemainingTotals)

	// we always want to update the conditions because if we have set a condition to "it worked" after it was previously, "it didn't work",
	// we need to reflect that information.  Recall that additional finalizers can be set on namespaces, so this finalizer may clear itself and
	// NOT remove the resource instance.
	if hasChanged := conditionUpdater.Update(ns); hasChanged {
		if _, err = d.nsClient.UpdateStatus(ns); err != nil {
			utilruntime.HandleError(fmt.Errorf("couldn't update status condition for namespace %q: %v", namespace, err))
		}
	}

	// if len(errs)==0, NewAggregate returns nil.
	klog.V(4).Infof("namespace controller - deleteAllContent - namespace: %s, estimate: %v, errors: %v", namespace, estimate, utilerrors.NewAggregate(errs))
	return estimate, utilerrors.NewAggregate(errs)
}
```

<br>

### 4 总结

（1）ns只对add, update的ns进行处理，而且只针对设置里deletionStamp的ns进行处理。

（2）如果ns的有deletionStamp，nsController做的操作为：

* 第一，修改ns的状态为删除中
* 第二，移除ns的finalizer

<br>

为什么会这样，这就得补充一下ns的基本特征：

（1）ns在删除之前，有一个finalizers：kubernetes。并且phase: Active

finalizers：kubernetes的作用就是，删除ns的时候会卡住，得等nsController删除里该命名空间下的所有资源才会移除

```
//删除前
root@k8s-master:~/testyaml# kubectl get ns zoux -oyaml -w
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2021-07-20T03:18:44Z"
  name: zoux
  resourceVersion: "9449396"
  selfLink: /api/v1/namespaces/zoux
  uid: 12fab759-0cda-4d98-97db-330cbf407e15
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
  
  
//删除中  
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2021-07-20T03:18:44Z"
  deletionTimestamp: "2021-07-20T03:20:07Z"
  name: zoux
  resourceVersion: "9449629"
  selfLink: /api/v1/namespaces/zoux
  uid: 12fab759-0cda-4d98-97db-330cbf407e15
spec:
  finalizers:
  - kubernetes
status:
  phase: Terminating
```