Table of Contents
=================

  * [1. kubectl create 命令定义](#1-kubectl-create-命令定义)
     * [1.1 editBeforeCreate](#11-editbeforecreate)
     * [1.2 AddApplyAnnotationFlags](#12-addapplyannotationflags)
  * [2.RunCreate](#2runcreate)
     * [2.1 CreateOptions](#21-createoptions)
        * [(1) 直接创建pod](#1-直接创建pod)
        * [(2) 加入--output=yaml参数](#2-加入--outputyaml参数)
        * [(3) record 会在创建对象中的annotation中记录](#3-record-会在创建对象中的annotation中记录)
        * [(4) --raw](#4---raw)
  * [2.2 RunCreate源码分析](#22-runcreate

从本篇文章尝试以kubectl  create 命令为例，梳理一些之前分析的的所有kubectl思想。

### 1. kubectl create 命令定义

create 命令 是定义在 kubectl 的 Basic Commands (Beginner) 中的。

从定义中可以看出来，kubectl create 的用法就是 kubectl create -f filename，创建资源对象。

Kubectl create主要逻辑如下：

（1）kubectl必须制定-f 或者 -k

（2）增加了很多flag, 比如dryrun , editbeforCreate等等

（3）定义了一堆subcommands，kubectl create ns/job 等等

（4）kubectl create核心命令而已，还是RunCreate函数

```
// NewCmdCreate returns new initialized instance of create sub command
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
	o := NewCreateOptions(ioStreams)

	cmd := &cobra.Command{
		Use:                   "create -f FILENAME",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Create a resource from a file or from stdin."),
		Long:                  createLong,
		Example:               createExample,
		Run: func(cmd *cobra.Command, args []string) {
		  // kubectl必须制定-f 或者 -k
			if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
			  // 报错信息
				ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
				// 没有制定的话，会执行DefaultSubCommandRun函数，其实就是提示你指定，然后打印help
				defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
				defaultRunFunc(cmd, args)
				return
			}
			// 参数补全和校验
			cmdutil.CheckErr(o.Complete(f, cmd))
			cmdutil.CheckErr(o.ValidateArgs(cmd, args))
			// RunCreate是真正的逻辑函数
			cmdutil.CheckErr(o.RunCreate(f, cmd))
		},
	}

	// bind flag structs
	o.RecordFlags.AddFlags(cmd)

	usage := "to use to create the resource"
	cmdutil.AddFilenameOptionFlags(cmd, &o.FilenameOptions, usage)
	cmdutil.AddValidateFlags(cmd)
	
	// 1.实现了editBeforeCreate，详见1.1
	cmd.Flags().BoolVar(&o.EditBeforeCreate, "edit", o.EditBeforeCreate, "Edit the API resource before creating")
	cmd.Flags().Bool("windows-line-endings", runtime.GOOS == "windows",
		"Only relevant if --edit=true. Defaults to the line ending native to your platform.")
	
	// 2.添加 AnnotationFlags flags，详见1.2
	cmdutil.AddApplyAnnotationFlags(cmd)
	// 3. 添加了dryrun, 如果dryrun的话，只会打印输出不会创建
	cmdutil.AddDryRunFlag(cmd)
	cmd.Flags().StringVarP(&o.Selector, "selector", "l", o.Selector, "Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2)")
	cmd.Flags().StringVar(&o.Raw, "raw", o.Raw, "Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.")
   
   // 3.1 绑定了printFlags
	o.PrintFlags.AddFlags(cmd)

	// create subcommands
	// 4. 定义了一堆子命令 kubectl create ns/secrete等等
	cmd.AddCommand(NewCmdCreateNamespace(f, ioStreams))
	cmd.AddCommand(NewCmdCreateQuota(f, ioStreams))
	cmd.AddCommand(NewCmdCreateSecret(f, ioStreams))
	cmd.AddCommand(NewCmdCreateConfigMap(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceAccount(f, ioStreams))
	cmd.AddCommand(NewCmdCreateService(f, ioStreams))
	cmd.AddCommand(NewCmdCreateDeployment(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePodDisruptionBudget(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePriorityClass(f, ioStreams))
	cmd.AddCommand(NewCmdCreateJob(f, ioStreams))
	cmd.AddCommand(NewCmdCreateCronJob(f, ioStreams))
	return cmd
}
```

#### 1.1 editBeforeCreate

可以看出来，editBeforeCreate，是先要你edit yaml, 然后才会创建。要是没有做出改动，是不会创建的！

```
root@k8s-master:~# kubectl create -f pod.yaml --edit=true
Edit cancelled, no changes made.
root@k8s-master:~#
root@k8s-master:~# kubectl get pod
No resources found in default namespace.
root@k8s-master:~#
root@k8s-master:~# kubectl get pod
No resources found in default namespace.
root@k8s-master:~# kubectl create -f pod.yaml --edit=true
pod/nginx1 created
root@k8s-master:~#
root@k8s-master:~# kubectl get pod
NAME     READY   STATUS    RESTARTS   AGE
nginx1   1/1     Running   0          3s
```

#### 1.2 AddApplyAnnotationFlags

```
root@k8s-master:~# kubectl create -f pod.yaml --save-config=true
pod/nginx created
root@k8s-master:~# 
root@k8s-master:~# kubectl get pod 
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4s
// annotations里面有各种信息
root@k8s-master:~# kubectl get pod nginx -oyaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"containers":[{"command":["sleep","3600"],"image":"curlimages/curl:7.75.0","name":"nginx"}],"nodeName":"k8s-node","serviceAccountName":"sa-example","terminationGracePeriodSeconds":10}}
  creationTimestamp: "2021-11-04T14:32:30Z"
  name: nginx
  namespace: default
  resourceVersion: "2416811"
  
创建是不指定
root@k8s-master:~# kubectl create -f pod.yaml
pod/nginx created
root@k8s-master:~# kubectl get pod nginx -oyaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-11-04T14:34:18Z"
  name: nginx
  namespace: default
  resourceVersion: "2417067"
  selfLink: /api/v1/namespaces/default/pods/nginx
  uid: 958b1d53-3035-4342-a048-321119d50a0b
spec:
```

### 2.RunCreate

#### 2.1 CreateOptions

k8s中经常有这样的思想。所有的flag最终封装到 Options中。例如CreateOptions，DeleteOptions等等。然后根据Options的参数进行不同的操作。

```
// CreateOptions is the commandline options for 'create' sub command
type CreateOptions struct {
   PrintFlags  *genericclioptions.PrintFlags      //以什么方式打印创建后的结果，yaml/json等等
   RecordFlags *genericclioptions.RecordFlags     //指定是否在创建后的对象记录这个create操作，详见（3）

   DryRun bool                   

   FilenameOptions  resource.FilenameOptions    //fileName相关的操作
   Selector         string
   EditBeforeCreate bool
   Raw              string                // apiserver暴露的api

   Recorder genericclioptions.Recorder 
   PrintObj func(obj kruntime.Object) error     //创建后返回的对象，默认的就是打印一句， pod nginx created

   genericclioptions.IOStreams
}
```

为例更好的了解, 在RunCreate函数之前打印了一些日志。

```
k8s.io/kubectl/pkg/cmd/create/create.go
// RunCreate performs the creation
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
	// raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
	// the validator enforces this, so
	klog.V(2).Infof("zoux CreateOptions is: %v", o)
	klog.V(2).Infof("zoux FilenameOptions is: %v,%v,%v", o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize,o.FilenameOptions.Recursive)
	klog.V(2).Infof("zoux PrintFlags is: %v, %v,%v, %v, %v, %v, %v", o.PrintFlags.JSONYamlPrintFlags, o.PrintFlags.OutputFormat, o.PrintFlags.NamePrintFlags, o.PrintFlags.OutputFlagSpecified, o.PrintFlags.OutputFormat, o.PrintFlags.TemplatePrinterFlags, o.PrintFlags.TypeSetterPrinter)
	klog.V(2).Infof("zoux PrintObj is: %v", o.PrintObj)
	klog.V(2).Infof("zoux Recorder is: %v", o.Recorder)
	klog.V(2).Infof("zoux RecordFlags is: %v", o.RecordFlags.Record)
```



##### (1) 直接创建pod

```
root@k8s-master:~# ./kubectl create -f pod.yaml -v 3
I1105 11:19:31.373990    5031 create.go:215] zoux CreateOptions is: &{0xc000334ff0 0xc0002fc0c0 false {[pod.yaml]  false}  false  {} 0x14e0d80 {0xc0000b8000 0xc0000b8008 0xc0000b8010}}
I1105 11:19:31.374118    5031 create.go:216] zoux FilenameOptions is: [pod.yaml],,false
I1105 11:19:31.374142    5031 create.go:217] zoux PrintFlags is: , &{}, &{created}, 0x129b480, 0xc0002ae2d0, &{0xc0002ae300 0xc0002ae310 0xc0002fe22c 0xc0002ae2f0},&{0xc0002fdea8 0xc000269a40}
I1105 11:19:31.374189    5031 create.go:218] zoux PrintObj is: 0x14e0d80
I1105 11:19:31.374200    5031 create.go:219] zoux Recorder is: {}
I1105 11:19:31.374212    5031 create.go:220] zoux RecordFlags is: 0xc0002fe22d
pod/nginx created
```

##### (2) 加入--output=yaml参数

```
root@k8s-master:~# ./kubectl create -f pod.yaml -v 3
I1105 11:19:31.373990    5031 create.go:215] zoux CreateOptions is: &{0xc000334ff0 0xc0002fc0c0 false {[pod.yaml]  false}  false  {} 0x14e0d80 {0xc0000b8000 0xc0000b8008 0xc0000b8010}}
I1105 11:19:31.374118    5031 create.go:216] zoux FilenameOptions is: [pod.yaml],,false
I1105 11:19:31.374142    5031 create.go:217] zoux PrintFlags is: , &{}, &{created}, 0x129b480, 0xc0002ae2d0, &{0xc0002ae300 0xc0002ae310 0xc0002fe22c 0xc0002ae2f0},&{0xc0002fdea8 0xc000269a40}
I1105 11:19:31.374189    5031 create.go:218] zoux PrintObj is: 0x14e0d80
I1105 11:19:31.374200    5031 create.go:219] zoux Recorder is: {}
I1105 11:19:31.374212    5031 create.go:220] zoux RecordFlags is: 0xc0002fe22d
pod/nginx created
root@k8s-master:~# 
root@k8s-master:~# 
root@k8s-master:~# ./kubectl create -f pod.yaml --output=yaml  -v 3
I1105 11:20:37.221485    5527 create.go:215] zoux CreateOptions is: &{0xc0003b9e00 0xc0002fa0a8 false {[pod.yaml]  false}  false  {} 0x14e0d80 {0xc0000b8000 0xc0000b8008 0xc0000b8010}}
I1105 11:20:37.221602    5527 create.go:216] zoux FilenameOptions is: [pod.yaml],,false
I1105 11:20:37.221618    5527 create.go:217] zoux PrintFlags is: yaml, &{}, &{created}, 0x129b480, 0xc00041f2a0, &{0xc00041f2d0 0xc00041f2e0 0xc00003ebf8 0xc00041f2c0},&{0xc00003fb58 0xc00003b960}
I1105 11:20:37.221648    5527 create.go:218] zoux PrintObj is: 0x14e0d80
I1105 11:20:37.221656    5527 create.go:219] zoux Recorder is: {}
I1105 11:20:37.221662    5527 create.go:220] zoux RecordFlags is: 0xc00003ebf9
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-11-05T03:20:37Z"
  name: nginx
  namespace: default
  resourceVersion: "2522253"
  selfLink: /api/v1/namespaces/default/pods/nginx
  uid: 947964b7-c65b-4e05-b7e5-a9e83a1b4e20
spec:
  containers:
  - command:
    - sleep
    - "3600"
    image: curlimages/curl:7.75.0
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: sa-example-token-lchv2
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: k8s-node
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: sa-example
  serviceAccountName: sa-example
  terminationGracePeriodSeconds: 10
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: sa-example-token-lchv2
    secret:
      defaultMode: 420
      secretName: sa-example-token-lchv2
status:
  phase: Pending
  qosClass: BestEffort
```

##### (3) record 会在创建对象中的annotation中记录

```
root@k8s-master:~# ./kubectl create -f pod.yaml --record  -v 3
I1105 11:22:41.875198    6398 create.go:215] zoux CreateOptions is: &{0xc00038ce40 0xc0002dc0a8 false {[pod.yaml]  false}  false  0xc0003ffd80 0x14e0d80 {0xc00000e010 0xc00000e018 0xc00000e020}}
I1105 11:22:41.875342    6398 create.go:216] zoux FilenameOptions is: [pod.yaml],,false
I1105 11:22:41.875369    6398 create.go:217] zoux PrintFlags is: , &{}, &{created}, 0x129b480, 0xc000448290, &{0xc0004482e0 0xc0004482f0 0xc00039379c 0xc0004482d0},&{0xc0002ddea8 0xc0002bfa40}
I1105 11:22:41.875414    6398 create.go:218] zoux PrintObj is: 0x14e0d80
I1105 11:22:41.875426    6398 create.go:219] zoux Recorder is: &{kubectl create --filename=pod.yaml --record=true --v=3}
I1105 11:22:41.875445    6398 create.go:220] zoux RecordFlags is: 0xc00039379d
pod/nginx created
root@k8s-master:~# 
root@k8s-master:~# kubectl get pod nginx -oyaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/change-cause: kubectl create --filename=pod.yaml --record=true --v=3
  creationTimestamp: "2021-11-05T03:22:42Z"
  name: nginx
  namespace: default
  resourceVersion: "2522555"
  selfLink: /api/v1/namespaces/default/pods/nginx
  uid: a03f4733-0e5e-4ab6-a7ed-4aeae41d9740
```

##### (4) --raw

```
--raw='': Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.
看起来只支持这个
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```


<br>

### 2.2 RunCreate源码分析

```
// RunCreate performs the creation
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
	// raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
	// the validator enforces this, so
	// 1.如果指定了url（apiserver暴露的restful路径）， 直接发送到这个url处理
	if len(o.Raw) > 0 {
		restClient, err := f.RESTClient()
		if err != nil {
			return err
		}
		return rawhttp.RawPost(restClient, o.IOStreams, o.Raw, o.FilenameOptions.Filenames[0])
	}
  
  // 2.是否指定了EditBeforeCreate
	if o.EditBeforeCreate {
		return RunEditOnCreate(f, o.PrintFlags, o.RecordFlags, o.IOStreams, cmd, &o.FilenameOptions)
	}
	
	// 
	schema, err := f.Validator(cmdutil.GetFlagBool(cmd, "validate"))
	if err != nil {
		return err
	}

	cmdNamespace, enforceNamespace, err := f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}
    
    
    // 3.这个是关键函数。先补充一波基础知识再额外分析
	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
	err = r.Err()
	if err != nil {
		return err
	}

	count := 0
	err = r.Visit(func(info *resource.Info, err error) error {
		if err != nil {
			return err
		}
		if err := util.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, scheme.DefaultJSONEncoder()); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}

		if err := o.Recorder.Record(info.Object); err != nil {
			klog.V(4).Infof("error recording current command: %v", err)
		}

		if !o.DryRun {
			if err := createAndRefresh(info); err != nil {
				return cmdutil.AddSourceToErr("creating", info.Source, err)
			}
		}

		count++
        
        
        // 格式化输出
		return o.PrintObj(info.Object)
	})
	if err != nil {
		return err
	}
	if count == 0 {
		return fmt.Errorf("no objects passed to create")
	}
	return nil
}
```

RunCreate函数通过`f.NewBuilder().XX.XX.Do`定义了这些visitor：DecoratedVisitor，ContinueOnErrorVisitor，FlattenListVisitor， FlattenListVisitor，EagerVisitorList FileVisitor， StreamVisitor。用于处理info

然后o.PrintObj 通过之前的printFlags定义好了printer，格式化输出创建好的obj

### 3. 总结
（1）kubectl必须制定-f 或者 -k

（2）增加了很多flag, 比如dryrun , editbeforCreate等等

（3）定义了一堆subcommands，kubectl create ns/job 等等

（4）kubectl create核心命令而已，还是RunCreate函数

（5）RunCreate函数通过`f.NewBuilder().XX.XX.Do`定义了这些visitor：DecoratedVisitor，ContinueOnErrorVisitor，FlattenListVisitor， FlattenListVisitor，EagerVisitorList FileVisitor， StreamVisitor。用于处理info

（6）然后o.PrintObj 通过之前的printFlags定义好了printer，格式化输出创建好的obj





