Table of Contents
=================

  * [1.  job简介](#1--job简介)
  * [2. job controller源码分析-初始化](#2-job-controller源码分析-初始化)
     * [2.1 startJobController](#21-startjobcontroller)
     * [2.2 NewJobController](#22-newjobcontroller)
     * [2.3 对Pod的监听事件](#23-对pod的监听事件)
        * [2.3.1 job的expectations机制](#231-job的expectations机制)
        * [2.3.2 addPod](#232-addpod)
        * [2.3.3 updatePod](#233-updatepod)
        * [2.3.4 deletePod](#234-deletepod)
        * [2.3.5 总结](#235-总结)
  * [3. 如何处理队列中的job](#3-如何处理队列中的job)
     * [3.1 sycnjob](#31-sycnjob)
     * [3.2 判断job是否完成的标准：  completed,  failed，c.Status == v1.ConditionTrue](#32-判断job是否完成的标准--completed--failedcstatus--v1conditiontrue)
     * [3.3 如何获得该job对应的pods](#33-如何获得该job对应的pods)
     * [3.4  jm.manageJob](#34--jmmanagejob)
  * [4.总结](#4总结)

### 1.  job简介

job 在 kubernetes 中主要用来处理离线任务，job 直接管理 pod，可以创建一个或多个 pod 并会确保指定数量的 pod 运行完成。

job 的一个示例如下所示：

```
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    job-name: hello-1626526800
  name: hello-1626526800
  namespace: default
spec:
  backoffLimit: 6     //标记为 failed 前的重试次数（运行多少个pod failed），默认为 6
  completions: 4      //当成功的 Pod 个数达到 .spec.completions 时，Job 被视为完成
  parallelism: 1      // 并行度。这里就是每次1个1个pod的运行，4个pod运行完后，job完成
  selector:
    matchLabels:
      controller-uid: 52f8d25f-6bbf-4439-ab6d-02876c52baea
  template:
    metadata:
      creationTimestamp: null
      labels:
        job-name: hello-1626526800
    spec:
      containers:
      - args:
        - /bin/sh
        - -c
        - date; echo "Hello, World!"
        image: busybox
        imagePullPolicy: Always
        name: hello
```

更多关于job的描述，可以参考社区介绍：https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/

<br>

### 2. job controller源码分析-初始化

#### 2.1 startJobController

这个就是 startControllers里面kcm启动时各个controller对应的init函数。

cmd\kube-controller-manager\app\batch.go

```
func startJobController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "batch", Version: "v1", Resource: "jobs"}] {
		return nil, false, nil
	}
	go job.NewJobController(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Batch().V1().Jobs(),
		ctx.ClientBuilder.ClientOrDie("job-controller"),
	).Run(int(ctx.ComponentConfig.JobController.ConcurrentJobSyncs), ctx.Stop)
	return nil, true, nil
}
```

<br>

#### 2.2 NewJobController

pkg\controller\job\job_controller.go

这里就是定义好  informer和处理函数。

可以看出来，job的add,delete, update最终都是入队列了。

```
func NewJobController(podInformer coreinformers.PodInformer, jobInformer batchinformers.JobInformer, kubeClient clientset.Interface) *JobController {
    // 1.定义event上传
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(glog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})

	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		metrics.RegisterMetricAndTrackRateLimiterUsage("job_controller", kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}
    
    
	jm := &JobController{
		kubeClient: kubeClient,
		podControl: controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "job-controller"}),
		},
		expectations: controller.NewControllerExpectations(),
		queue:        workqueue.NewNamedRateLimitingQueue(workqueue.NewItemExponentialFailureRateLimiter(DefaultJobBackOff, MaxJobBackOff), "job"),
		recorder:     eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "job-controller"}),
	}

	jobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			jm.enqueueController(obj, true)
		},
		UpdateFunc: jm.updateJob,            // 这个其实也是放入队列的，见下面的函数
		DeleteFunc: func(obj interface{}) {
			jm.enqueueController(obj, true)
		},
	})
	jm.jobLister = jobInformer.Lister()
	jm.jobStoreSynced = jobInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addPod,
		UpdateFunc: jm.updatePod,  
		DeleteFunc: jm.deletePod,
	})
	jm.podStore = podInformer.Lister()
	jm.podStoreSynced = podInformer.Informer().HasSynced

	jm.updateHandler = jm.updateJobStatus
	jm.syncHandler = jm.syncJob

	return jm
}
```

updateJob进行了一些判断，最后还是入队列了。

```
func (jm *JobController) updateJob(old, cur interface{}) {
	oldJob := old.(*batch.Job)
	curJob := cur.(*batch.Job)

	// never return error
	key, err := controller.KeyFunc(curJob)
	if err != nil {
		return
	}
	jm.enqueueController(curJob, true)
	// check if need to add a new rsync for ActiveDeadlineSeconds
	if curJob.Status.StartTime != nil {
		curADS := curJob.Spec.ActiveDeadlineSeconds
		if curADS == nil {
			return
		}
		oldADS := oldJob.Spec.ActiveDeadlineSeconds
		if oldADS == nil || *oldADS != *curADS {
			now := metav1.Now()
			start := curJob.Status.StartTime.Time
			passed := now.Time.Sub(start)
			total := time.Duration(*curADS) * time.Second
			// AddAfter will handle total < passed
			jm.queue.AddAfter(key, total-passed)
			glog.V(4).Infof("job ActiveDeadlineSeconds updated, will rsync after %d seconds", total-passed)
		}
	}
}
```

<br>

#### 2.3 对Pod的监听事件

##### 2.3.1 job的expectations机制

和rs的机制其实是一样的。更详细的可以参考rs那篇博客的介绍。

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

**SatisfiedExpectations**: 输入key, 输出bool；判断某个job是否符合预期。符合预期： add<=0 && del<=0 或者 超过了同步周期； 其他情况都是不符合预期。

**DeleteExpectations**：输入key, 无输出；从map(缓存)中删除这个key

**SetExpectations**：输入（key, add, del）; 在map中新增加一行。 **这个会更新时间，将time复制为time.Now**

**ExpectCreations**:  输入（key, add);   覆盖map中的内容，del=0， add等于函数的参数。  **这个会更新时间，将time复制为time.Now**

**ExpectDeletions**： 输入（key, del);  覆盖map中的内容，add=0， del等于函数的参数。   **这个会更新时间，将time复制为time.Now**

**CreationObserved**: 输入(key) ;  map中对应的行中 add-1

**DeletionObserved**: 输入(key);  map中对应的行中 del-1

**RaiseExpectations**:  输入(key, add, del)；  map中对应的行中 Add+add, Del+del

**LowerExpectations**: 输入(key, add, del)；  map中对应的行中 Add-add, Del-del

<br>

##### 2.3.2 addPod

（1）如果pod要删除，deletePod最终会调用DeletionObserved函数，使得这个map中对应job的del-1

（2）有ower并且是job，就将这个job在对应map，add - 1

（3）如果这个pod是孤儿，这将这个pod之前有关联的job入队列，然后通过syncJob更新

```
// When a pod is created, enqueue the controller that manages it and update it's expectations.
func (jm *JobController) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)
	if pod.DeletionTimestamp != nil {
		// on a restart of the controller controller, it's possible a new pod shows up in a state that
		// is already pending deletion. Prevent the pod from being a creation observation.
		// 1. deletePod最终会调用DeletionObserved函数。
		jm.deletePod(pod)
		return
	}

	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		job := jm.resolveControllerRef(pod.Namespace, controllerRef)
		if job == nil {
			return
		}
		jobKey, err := controller.KeyFunc(job)
		if err != nil {
			return
		}
		// 2.有ower并且是job，就将这个job在对应map，add - 1
		jm.expectations.CreationObserved(jobKey)
		jm.enqueueController(job, true)
		return
	}

	// Otherwise, it's an orphan. Get a list of all matching controllers and sync
	// them to see if anyone wants to adopt it.
	// DO NOT observe creation because no controller should be waiting for an
	// orphan.
	// 3.如果是孤儿，这将这个pod之前有关联的job入队列，然后通过syncJob更新
	for _, job := range jm.getPodJobs(pod) {
		jm.enqueueController(job, true)
	}
}
```

<br>

##### 2.3.3 updatePod

（1）如果pod要删除，deletePod最终会调用DeletionObserved函数，使得这个map中对应job的del-1

（2） 如果pod更新了owner，先是旧job加入队列，因为进入队列的job都会同步

（3）如果新owner还是job，这个job入队列

（4） 如果这个pod是孤儿，这将这个pod之前有关联的job入队列，然后通过syncJob更新

```
// When a pod is updated, figure out what job/s manage it and wake them up.
// If the labels of the pod have changed we need to awaken both the old
// and new job. old and cur must be *v1.Pod types.
func (jm *JobController) updatePod(old, cur interface{}) {
	curPod := cur.(*v1.Pod)
	oldPod := old.(*v1.Pod)
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		// Periodic resync will send update events for all known pods.
		// Two different versions of the same pod will always have different RVs.
		return
	}
	// 1.deletePod最终会调用DeletionObserved函数，使得这个map中对应job的del-1
	if curPod.DeletionTimestamp != nil {
		// when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period,
		// and after such time has passed, the kubelet actually deletes it from the store. We receive an update
		// for modification of the deletion timestamp and expect an job to create more pods asap, not wait
		// until the kubelet actually deletes the pod.
		jm.deletePod(curPod)
		return
	}

	// the only time we want the backoff to kick-in, is when the pod failed
	immediate := curPod.Status.Phase != v1.PodFailed

	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	// 2. 如果pod更新了owner，先是旧job加入队列，因为进入队列的job都会同步
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if job := jm.resolveControllerRef(oldPod.Namespace, oldControllerRef); job != nil {
			jm.enqueueController(job, immediate)
		}
	}
  
  // 3.如果新owner还是job，这个job入队列
	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
		job := jm.resolveControllerRef(curPod.Namespace, curControllerRef)
		if job == nil {
			return
		}
		jm.enqueueController(job, immediate)
		return
	}
  
  
  // 4. 如果这个pod是孤儿，这将这个pod之前有关联的job入队列，然后通过syncJob更新
	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
	if labelChanged || controllerRefChanged {
		for _, job := range jm.getPodJobs(curPod) {
			jm.enqueueController(job, immediate)
		}
	}
}
```

<br>

##### 2.3.4 deletePod

这里又出现了tombstone。

deletepod的逻辑就更简单，将map中对应job的del-1

```
// When a pod is deleted, enqueue the job that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (jm *JobController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the pod
	// changed labels the new job will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %+v", obj))
			return
		}
	}

	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	job := jm.resolveControllerRef(pod.Namespace, controllerRef)
	if job == nil {
		return
	}
	jobKey, err := controller.KeyFunc(job)
	if err != nil {
		return
	}
	jm.expectations.DeletionObserved(jobKey)
	jm.enqueueController(job, true)
}
```

<br>

##### 2.3.5 总结

* 对于pod的add, updated, del事件，核心就是维护 map中job的数据，然后就是将对应的job入队列
* 对于job的add, updated, del事件，最后都是扔进了队列

### 3. 如何处理队列中的job

```
// Run the main goroutine responsible for watching and syncing jobs.
func (jm *JobController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer jm.queue.ShutDown()

	glog.Infof("Starting job controller")
	defer glog.Infof("Shutting down job controller")

	if !controller.WaitForCacheSync("job", stopCh, jm.podStoreSynced, jm.jobStoreSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(jm.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

```
// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (jm *JobController) worker() {
	for jm.processNextWorkItem() {
	}
}

func (jm *JobController) processNextWorkItem() bool {
	key, quit := jm.queue.Get()
	if quit {
		return false
	}
	defer jm.queue.Done(key)

	forget, err := jm.syncHandler(key.(string))
	if err == nil {
		if forget {
			jm.queue.Forget(key)
		}
		return true
	}

	utilruntime.HandleError(fmt.Errorf("Error syncing job: %v", err))
	jm.queue.AddRateLimited(key)

	return true
}
```

和所有控制器一样，流程为：Run->worker->processNextWorkItem->syncHandler。

而NewJobController的时候就指定了  jm.syncHandler = jm.syncJob

<br>

#### 3.1 sycnjob

（1）判断 job 是否已经执行完成，如果完成了，直接返回。

* 当 job 的 `.status.conditions` 中有 `Complete` 或 `Failed` 的 type 且对应的 status 为 true 时表示该 job 已经执行完成

（2）获得job的重试次数，以及通过expectations判断是否需要同步,以下三种情况需要同步

- 该 job 在 map 中的 adds 和 dels 都 <= 0
- 该 job 在 map 中已经超过 5min 没有更新了；
- 该 job 在 job 中不存在，即该对象是新创建的；

（3）获取 job 关联的所有 pods，然后分为三类：active、succeeded、failed

（4）如果这个job第一次启动，设置启动时间为Now，如果还设置了ActiveDeadlineSeconds值，则等ActiveDeadlineSeconds这个时间后再入队列

（5）判断job是否失败了。有俩种情况：

* 一是 job的重试此时达到了Spec.BackoffLimit`(默认是6次)，`
*  二是 job 的运行时间达到了 `job.Spec.ActiveDeadlineSeconds` 中设定的值

（6）如果job Failed, 删除所有的pod，然后发事件说这个job已经failed, 原因是XX

（7）如果job需要同步，并且没有deletionStamp，通过manageJob调整activejob=parallelism

（8）检查 `job.Spec.Completions` 判断 job 是否已经运行完成，如果 `job.Spec.Completions` 没有设置，那只要有一个pod运行成功，就表示该 job 完成。

（9）最后通过job的状态有无变化，如果有变化，更新到 apiserver；

```go
// syncJob will sync the job with the given key if it has had its expectations fulfilled, meaning
// it did not expect to see any more of its pods created or deleted. This function is not meant to be invoked
// concurrently with the same key.
func (jm *JobController) syncJob(key string) (bool, error) {
    // 1、用于统计本次 sync 的运行时间
	startTime := time.Now()
	defer func() {
		glog.V(4).Infof("Finished syncing job %q (%v)", key, time.Since(startTime))
	}()
    
    // 2、从 lister 中获取 job 对象
	ns, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return false, err
	}
	if len(ns) == 0 || len(name) == 0 {
		return false, fmt.Errorf("invalid job key %q: either namespace or name is missing", key)
	}
	sharedJob, err := jm.jobLister.Jobs(ns).Get(name)
	if err != nil {
		if errors.IsNotFound(err) {
			glog.V(4).Infof("Job has been deleted: %v", key)
			jm.expectations.DeleteExpectations(key)
			return true, nil
		}
		return false, err
	}
	job := *sharedJob

	// if job was finished previously, we don't want to redo the termination
    // 3、判断 job 是否已经执行完成，如果完成了，直接返回
	if IsJobFinished(&job) {
		return true, nil
	}

	// retrieve the previous number of retry
	// 4、获取 job 重试的次数，这个是队列自带的函数 workqueue自己就实现了
	previousRetry := jm.queue.NumRequeues(key)

	// Check the expectations of the job before counting active pods, otherwise a new pod can sneak in
	// and update the expectations after we've retrieved active pods from the store. If a new pod enters
	// the store after we've checked the expectation, the job sync is just deferred till the next relist.
	// 5、通过Expectations，判断 job 是否能进行 sync 操作
	jobNeedsSync := jm.expectations.SatisfiedExpectations(key)

    // 6、获取 job 关联的所有 pod
	pods, err := jm.getPodsForJob(&job)
	if err != nil {
		return false, err
	}
    

   
    // 7、 获取active、succeeded、failed状态的 pod 数
	activePods := controller.FilterActivePods(pods)
	active := int32(len(activePods))
	succeeded, failed := getStatus(pods)
	conditions := len(job.Status.Conditions)
	// job first start
   // 8、判断 job 是否为首次启动
	if job.Status.StartTime == nil {
		now := metav1.Now()
		job.Status.StartTime = &now
		// enqueue a sync to check if job past ActiveDeadlineSeconds
         // 9、如果设定了 ActiveDeadlineSeconds值，这个这个事件过去了再加入队列。
		if job.Spec.ActiveDeadlineSeconds != nil {
			glog.V(4).Infof("Job %s have ActiveDeadlineSeconds will sync after %d seconds",
				key, *job.Spec.ActiveDeadlineSeconds)
			jm.queue.AddAfter(key, time.Duration(*job.Spec.ActiveDeadlineSeconds)*time.Second)
		}
	}

	var manageJobErr error
	jobFailed := false
	var failureReason string
	var failureMessage string

    // 10、通过已经失败的pod数量，判断是否超过了job运行的上限，BackoffLimit。例子中设置的为6.
	jobHaveNewFailure := failed > job.Status.Failed
	// new failures happen when status does not reflect the failures and active
	// is different than parallelism, otherwise the previous controller loop
	// failed updating status so even if we pick up failure it is not a new one
  // 因为有的pod可能在job完成前就删除了，所以需要previousRetry+1（这次错误）进行判断。
	exceedsBackoffLimit := jobHaveNewFailure && (active != *job.Spec.Parallelism) &&
		(int32(previousRetry)+1 > *job.Spec.BackoffLimit)

	if exceedsBackoffLimit || pastBackoffLimitOnFailure(&job, pods) {
		// check if the number of pod restart exceeds backoff (for restart OnFailure only)
		// OR if the number of failed jobs increased since the last syncJob
		jobFailed = true
		failureReason = "BackoffLimitExceeded"
		failureMessage = "Job has reached the specified backoff limit"
	} else if pastActiveDeadline(&job) {
		jobFailed = true
		failureReason = "DeadlineExceeded"
		failureMessage = "Job was active longer than specified deadline"
	}
    
    // 11、如果处于 failed 状态，则调用 jm.deleteJobPods 并发删除所有 active pods
	if jobFailed {
		errCh := make(chan error, active)
		jm.deleteJobPods(&job, activePods, errCh)
		select {
		case manageJobErr = <-errCh:
			if manageJobErr != nil {
				break
			}
		default:
		}

		// update status values accordingly
		failed += active
		active = 0
		job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobFailed, failureReason, failureMessage))
		jm.recorder.Event(&job, v1.EventTypeWarning, failureReason, failureMessage)
	} else {
           
        // 12、若非 failed 状态，根据 jobNeedsSync 判断是否要进行同步
		if jobNeedsSync && job.DeletionTimestamp == nil {
			active, manageJobErr = jm.manageJob(activePods, succeeded, &job)
		}
        
        // 13、检查 job.Spec.Completions 判断 job 是否已经运行完成
		completions := succeeded
		complete := false
		if job.Spec.Completions == nil {
			// This type of job is complete when any pod exits with success.
			// Each pod is capable of
			// determining whether or not the entire Job is done.  Subsequent pods are
			// not expected to fail, but if they do, the failure is ignored.  Once any
			// pod succeeds, the controller waits for remaining pods to finish, and
			// then the job is complete.
            
			if succeeded > 0 && active == 0 {
				complete = true
			}
		} else {
			// Job specifies a number of completions.  This type of job signals
			// success by having that number of successes.  Since we do not
			// start more pods than there are remaining completions, there should
			// not be any remaining active pods once this count is reached.
			if completions >= *job.Spec.Completions {
				complete = true
				if active > 0 {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManyActivePods", "Too many active pods running after completion count reached")
				}
				if completions > *job.Spec.Completions {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManySucceededPods", "Too many succeeded pods running after completion count reached")
				}
			}
		}
                
        // 14、若 job 运行完成了，则更新 job.Status.Conditions 和 job.Status.CompletionTime 字段
		if complete {
			job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobComplete, "", ""))
			now := metav1.Now()
			job.Status.CompletionTime = &now
		}
	}

	forget := false
	// Check if the number of jobs succeeded increased since the last check. If yes "forget" should be true
	// This logic is linked to the issue: https://github.com/kubernetes/kubernetes/issues/56853 that aims to
	// improve the Job backoff policy when parallelism > 1 and few Jobs failed but others succeed.
	// In this case, we should clear the backoff delay.
	if job.Status.Succeeded < succeeded {
		forget = true
	}
   
    // 15、如果 job 的 status 有变化，将 job 的 status 更新到 apiserver
	// no need to update the job if the status hasn't changed since last time
	if job.Status.Active != active || job.Status.Succeeded != succeeded || job.Status.Failed != failed || len(job.Status.Conditions) != conditions {
		job.Status.Active = active
		job.Status.Succeeded = succeeded
		job.Status.Failed = failed

		if err := jm.updateHandler(&job); err != nil {
			return forget, err
		}

		if jobHaveNewFailure && !IsJobFinished(&job) {
			// returning an error will re-enqueue Job after the backoff period
			return forget, fmt.Errorf("failed pod(s) detected for job key %q", key)
		}

		forget = true
	}

	return forget, manageJobErr
}
```



<br>

#### 3.2 判断job是否完成的标准：  completed,  failed，c.Status == v1.ConditionTrue

```
func IsJobFinished(j *batch.Job) bool {
   for _, c := range j.Status.Conditions {
      if (c.Type == batch.JobComplete || c.Type == batch.JobFailed) && c.Status == v1.ConditionTrue {
         return true
      }
   }
   return false
}
```

找两个job对比，可以看出来 completed确实有一个 status=true的字段；就是第三个判断

```
Complete的job
status:
  completionTime: "2021-01-20T07:27:11Z"
  conditions:
  - lastProbeTime: "2021-01-20T07:27:11Z"
    lastTransitionTime: "2021-01-20T07:27:11Z"
    status: "True"
    type: Complete
  startTime: "2021-01-20T07:27:04Z"
  succeeded: 1

```

```
running的job
status:
  active: 1
  startTime: "2021-01-20T07:32:06Z"

```

<br>

#### 3.3 如何获得该job对应的pods

```
// getPodsForJob returns the set of pods that this Job should manage.
// It also reconciles ControllerRef by adopting/orphaning.
// Note that the returned Pods are pointers into the cache.
func (jm *JobController) getPodsForJob(j *batch.Job) ([]*v1.Pod, error) {
   selector, err := metav1.LabelSelectorAsSelector(j.Spec.Selector)
   if err != nil {
      return nil, fmt.Errorf("couldn't convert Job selector: %v", err)
   }
   // List all pods to include those that don't match the selector anymore
   // but have a ControllerRef pointing to this controller.
   pods, err := jm.podStore.Pods(j.Namespace).List(labels.Everything())
   if err != nil {
      return nil, err
   }
   // If any adoptions are attempted, we should first recheck for deletion
   // with an uncached quorum read sometime after listing Pods (see #42639).
   canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
      fresh, err := jm.kubeClient.BatchV1().Jobs(j.Namespace).Get(j.Name, metav1.GetOptions{})
      if err != nil {
         return nil, err
      }
      if fresh.UID != j.UID {
         return nil, fmt.Errorf("original Job %v/%v is gone: got uid %v, wanted %v", j.Namespace, j.Name, fresh.UID, j.UID)
      }
      return fresh, nil
   })
   cm := controller.NewPodControllerRefManager(jm.podControl, j, selector, controllerKind, canAdoptFunc)
   return cm.ClaimPods(pods)
}
```

<br>

```
// NewPodControllerRefManager returns a PodControllerRefManager that exposes
// methods to manage the controllerRef of pods.
//
// The CanAdopt() function can be used to perform a potentially expensive check
// (such as a live GET from the API server) prior to the first adoption.
// It will only be called (at most once) if an adoption is actually attempted.
// If CanAdopt() returns a non-nil error, all adoptions will fail.
//
// NOTE: Once CanAdopt() is called, it will not be called again by the same
//       PodControllerRefManager instance. Create a new instance if it makes
//       sense to check CanAdopt() again (e.g. in a different sync pass).
func NewPodControllerRefManager(
	podControl PodControlInterface,
	controller metav1.Object,
	selector labels.Selector,
	controllerKind schema.GroupVersionKind,
	canAdopt func() error,
) *PodControllerRefManager {
	return &PodControllerRefManager{
		BaseControllerRefManager: BaseControllerRefManager{
			Controller:   controller,
			Selector:     selector,
			CanAdoptFunc: canAdopt,
		},
		controllerKind: controllerKind,
		podControl:     podControl,
	}
}
```

```
最终还是通过 labels 匹配
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

这是 job zx-testip1-1611142680产生的一个pod.

```
kind: Pod
metadata:
  labels:
    controller-uid: ecff8cf1-7523-4d90-9559-22c9e994f726  //这个是job的 uuid
    job-name: zx-testip1-1611142680
  name: zx-testip1-1611142680-4s9z8
  namespace: zx
```

<br>

#### 3.4  jm.manageJob

注意：这里进行了map的设置，初始化

`jm.manageJob` 核心工作就是根据 job的并发数来确认当前处于 active 的 pods 数量是否ok，如果不ok的话则进行调整。

具体为：

- 如果active > parallelism，说明active的pod数量太多，需要删除一些。

  删除pod的逻辑，rs那篇文章有，其实就是根据pod的运行时间，状态等信息判断pod优先级。

- 如果active < parallelism，说明active的pod数量太少，需要创建一些。

```
// manageJob is the core method responsible for managing the number of running
// pods according to what is specified in the job.Spec.
// Does NOT modify <activePods>.
func (jm *JobController) manageJob(activePods []*v1.Pod, succeeded int32, job *batch.Job) (int32, error) {
	var activeLock sync.Mutex
	active := int32(len(activePods))
	parallelism := *job.Spec.Parallelism
	jobKey, err := controller.KeyFunc(job)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for job %#v: %v", job, err))
		return 0, nil
	}

	var errCh chan error
	if active > parallelism {
		diff := active - parallelism
		errCh = make(chan error, diff)
		// 注意这里进行了map的设置
		jm.expectations.ExpectDeletions(jobKey, int(diff))
		glog.V(4).Infof("Too many pods running job %q, need %d, deleting %d", jobKey, parallelism, diff)
		// Sort the pods in the order such that not-ready < ready, unscheduled
		// < scheduled, and pending < running. This ensures that we delete pods
		// in the earlier stages whenever possible.
		sort.Sort(controller.ActivePods(activePods))

		active -= diff
		wait := sync.WaitGroup{}
		wait.Add(int(diff))
		for i := int32(0); i < diff; i++ {
			go func(ix int32) {
				defer wait.Done()
				if err := jm.podControl.DeletePod(job.Namespace, activePods[ix].Name, job); err != nil {
					defer utilruntime.HandleError(err)
					// Decrement the expected number of deletes because the informer won't observe this deletion
					glog.V(2).Infof("Failed to delete %v, decrementing expectations for job %q/%q", activePods[ix].Name, job.Namespace, job.Name)
					jm.expectations.DeletionObserved(jobKey)
					activeLock.Lock()
					active++
					activeLock.Unlock()
					errCh <- err
				}
			}(i)
		}
		wait.Wait()

	} else if active < parallelism {
		wantActive := int32(0)
		if job.Spec.Completions == nil {
			// Job does not specify a number of completions.  Therefore, number active
			// should be equal to parallelism, unless the job has seen at least
			// once success, in which leave whatever is running, running.
			if succeeded > 0 {
				wantActive = active
			} else {
				wantActive = parallelism
			}
		} else {
			// Job specifies a specific number of completions.  Therefore, number
			// active should not ever exceed number of remaining completions.
			wantActive = *job.Spec.Completions - succeeded
			if wantActive > parallelism {
				wantActive = parallelism
			}
		}
		diff := wantActive - active
		if diff < 0 {
			utilruntime.HandleError(fmt.Errorf("More active than wanted: job %q, want %d, have %d", jobKey, wantActive, active))
			diff = 0
		}
		jm.expectations.ExpectCreations(jobKey, int(diff))
		errCh = make(chan error, diff)
		glog.V(4).Infof("Too few pods running job %q, need %d, creating %d", jobKey, wantActive, diff)

		active += diff
		wait := sync.WaitGroup{}

		// Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
		// and double with each successful iteration in a kind of "slow start".
		// This handles attempts to start large numbers of pods that would
		// likely all fail with the same error. For example a project with a
		// low quota that attempts to create a large number of pods will be
		// prevented from spamming the API service with the pod create requests
		// after one of its pods fails.  Conveniently, this also prevents the
		// event spam that those failures would generate.
		for batchSize := int32(integer.IntMin(int(diff), controller.SlowStartInitialBatchSize)); diff > 0; batchSize = integer.Int32Min(2*batchSize, diff) {
			errorCount := len(errCh)
			wait.Add(int(batchSize))
			for i := int32(0); i < batchSize; i++ {
				go func() {
					defer wait.Done()
					err := jm.podControl.CreatePodsWithControllerRef(job.Namespace, &job.Spec.Template, job, metav1.NewControllerRef(job, controllerKind))
					if err != nil && errors.IsTimeout(err) {
						// Pod is created but its initialization has timed out.
						// If the initialization is successful eventually, the
						// controller will observe the creation via the informer.
						// If the initialization fails, or if the pod keeps
						// uninitialized for a long time, the informer will not
						// receive any update, and the controller will create a new
						// pod when the expectation expires.
						return
					}
					if err != nil {
						defer utilruntime.HandleError(err)
						// Decrement the expected number of creates because the informer won't observe this pod
						glog.V(2).Infof("Failed creation, decrementing expectations for job %q/%q", job.Namespace, job.Name)
						jm.expectations.CreationObserved(jobKey)
						activeLock.Lock()
						active--
						activeLock.Unlock()
						errCh <- err
					}
				}()
			}
			wait.Wait()
			// any skipped pods that we never attempted to start shouldn't be expected.
			skippedPods := diff - batchSize
			if errorCount < len(errCh) && skippedPods > 0 {
				glog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for job %q/%q", skippedPods, job.Namespace, job.Name)
				active -= skippedPods
				for i := int32(0); i < skippedPods; i++ {
					// Decrement the expected number of creates because the informer won't observe this pod
					jm.expectations.CreationObserved(jobKey)
				}
				// The skipped pods will be retried later. The next controller resync will
				// retry the slow start process.
				break
			}
			diff -= batchSize
		}
	}

	select {
	case err := <-errCh:
		// all errors have been reported before, we only need to inform the controller that there was an error and it should re-try this job once more next time.
		if err != nil {
			return active, err
		}
	default:
	}

	return active, nil
}
```

<br>

### 4.总结

（1）jobController也利用expectations机制，在每次同步计算当前active pod的数量时进行了设置。

（2）然后pod的add, update, del 对map进行了修改

举例来说，如果一个job completions=4, parallelism=2。那么当这个job创建的时候：

（1）发现map(expectations)中没有这个job，那么需要同步。

（2）通过manageJob，设置map中 add=2, del=0

（3）然后在创建了2个pod

（4）每pod创建，add-1, 创建完2个pod后，add=0, del=0，表示job需要同步了。

这个就是expectations的精髓，这2个pod没创建完之前，这个job根本不需要同步。

（5）然后同步发现，当前确实只能运行2个pod，所以等着2个pod运行完后，触发下一轮更新，再创建2个pod

（6）最后job运行完成