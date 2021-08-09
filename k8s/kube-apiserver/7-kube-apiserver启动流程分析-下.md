Table of Contents
=================

  * [1. 创建APIExtensionsServer](#1-创建apiextensionsserver)
     * [1.1  创建GenericAPIServer](#11--创建genericapiserver)
     * [1.2  实例化CustomResourceDefinitions](#12--实例化customresourcedefinitions)
     * [1.3 实例化APIGroupInfo](#13-实例化apigroupinfo)
     * [1.4 InstallAPIGroup注册APIGroup](#14-installapigroup注册apigroup)
     * [1.5 启动crdController](#15-启动crdcontroller)
  * [2. 创建KubeAPIServer](#2-创建kubeapiserver)
     * [2.1 创建GenericAPIServer](#21-创建genericapiserver)
     * [2.2 实例化Master](#22-实例化master)
     * [2.3 InstallLegacyAPI注册/api资源](#23-installlegacyapi注册api资源)
     * [2.4 InstallAPIs注册/apis资源](#24-installapis注册apis资源)
     * [2.5 路由注册总结](#25-路由注册总结)
  * [3. 创建AggregatorServer](#3-创建aggregatorserver)
  * [4. 创建GenericAPIServer](#4-创建genericapiserver)
  * [5. 启动http服务](#5-启动http服务)
  * [6. 启动https服务](#6-启动https服务)
  * [7 总结](#7-总结)
  * [8. 参考链接](#8-参考链接)

**本章重点：**

（1）kube-apiserver启动过程中，剩下的六个步骤：

* 创建APIExtensionsServer
* 创建KubeAPIServer
* 创建AggregatorServer
* 创建GenericAPIServer
* 启动HTTP服务；启动HTTPS服务。

<br>

 kube-apiserver的 9个流程分析如下：

（1）资源注册。

（2）Cobra命令行参数解析。

（3）创建APIServer通用配置。

（4）创建APIExtensionsServer。

（5）创建KubeAPIServer。

（6）创建AggregatorServer。

（7）创建GenericAPIServer。

（8）启动HTTP服务。

（9）启动HTTPS服务。

<br>

### 1. 创建APIExtensionsServer

再次回到 CreateServerChain

```go
// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*genericapiserver.GenericAPIServer, error) {
    
    // 1.创建到节点拨号连接,目的为了和节点交互。在云平台中，则需要安装本机的SSH Key到Kubernetes集群中所有节点上，可通过用户名和私钥，SSH到node节点
	nodeTunneler, proxyTransport, err := CreateNodeDialer(completedOptions)
	if err != nil {
		return nil, err
	}

    // 2. 配置API Server的Config。
	kubeAPIServerConfig, insecureServingInfo, serviceResolver, pluginInitializer, admissionPostStartHook, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	if err != nil {
		return nil, err
	}

    // 3.这里同时还配置了Extension API Server的Config，用于配置用户自己编写的API Server。
	// If additional API servers are added, they should be gated.  从这里深入挖下去
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount)
	if err != nil {
		return nil, err
	}
	// 4.创建APIExtensionsServer
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}
    
    // 5.创建kubeapiserver，这里就是定义了 /apis/groups等这些api。
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, admissionPostStartHook)
	if err != nil {
		return nil, err
	}

	// otherwise go down the normal path of standing the aggregator up in front of the API server
	// this wires up openapi
	
	// 6. kubeAPIServer prepareRun
	kubeAPIServer.GenericAPIServer.PrepareRun()

    // 7. apiExtensionsServer prepareRun
	// This will wire up openapi for extension api server
	apiExtensionsServer.GenericAPIServer.PrepareRun()

    // 8. 配置AA config，然后创建AA server。
	// aggregator comes last in the chain
	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, proxyTransport, pluginInitializer)
	if err != nil {
		return nil, err
	}
	
	// 9.创建AA server.这里传入了参数 kube-apiserver, apiExtensionServer。
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
	if err != nil {
		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
		return nil, err
	}
  
  // 10. 启动http服务
	if insecureServingInfo != nil {
		insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(aggregatorServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
		if err := insecureServingInfo.Serve(insecureHandlerChain, kubeAPIServerConfig.GenericConfig.RequestTimeout, stopCh); err != nil {
			return nil, err
		}
	}

	return aggregatorServer.GenericAPIServer, nil
}
```

<br>

**createAPIExtensionsServer的定义如下：**

```go
func createAPIExtensionsServer(apiextensionsConfig *apiextensionsapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget) (*apiextensionsapiserver.CustomResourceDefinitions, error) {
   return apiextensionsConfig.Complete().New(delegateAPIServer)
}

Complete()只是补全了配置，这里主要关心New函数


// New returns a new instance of CustomResourceDefinitions from the given config.
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
  // 1. 创建GenericAPIServer.  APIExtensionsServer的运行依赖于GenericAPIServer，通过c.GenericConfig.New函数创建名为apiextensions-apiserver的服务。
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
  
  
  // 2.实例化CustomResourceDefinitions. APIExtensionsServer（API扩展服务）通过CustomResourceDefinitions对象进行管理，实例化该对象后才能注册APIExtensionsServer下的资源。
	s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}

	apiResourceConfig := c.GenericConfig.MergedResourceConfig
	
	// 3. 实例化APIGroupInfo.  
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
	if apiResourceConfig.VersionEnabled(v1beta1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// customresourcedefinitions
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)

		apiGroupInfo.VersionedResourcesStorageMap[v1beta1.SchemeGroupVersion.Version] = storage
	}
	if apiResourceConfig.VersionEnabled(v1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// customresourcedefinitions
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)

		apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
	}
  
  
  // 4. InstallAPIGroup注册APIGroup
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	crdClient, err := internalclientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
	if err != nil {
		// it's really bad that this is leaking here, but until we can fix the test (which I'm pretty sure isn't even testing what it wants to test),
		// we need to be able to move forward
		return nil, fmt.Errorf("failed to create clientset: %v", err)
	}
	s.Informers = internalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)

	delegateHandler := delegationTarget.UnprotectedHandler()
	if delegateHandler == nil {
		delegateHandler = http.NotFoundHandler()
	}

	versionDiscoveryHandler := &versionDiscoveryHandler{
		discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
		delegate:  delegateHandler,
	}
	groupDiscoveryHandler := &groupDiscoveryHandler{
		discovery: map[string]*discovery.APIGroupHandler{},
		delegate:  delegateHandler,
	}
	establishingController := establish.NewEstablishingController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	crdHandler, err := NewCustomResourceDefinitionHandler(
		versionDiscoveryHandler,
		groupDiscoveryHandler,
		s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(),
		delegateHandler,
		c.ExtraConfig.CRDRESTOptionsGetter,
		c.GenericConfig.AdmissionControl,
		establishingController,
		c.ExtraConfig.ServiceResolver,
		c.ExtraConfig.AuthResolverWrapper,
		c.ExtraConfig.MasterCount,
		s.GenericAPIServer.Authorizer,
		c.GenericConfig.RequestTimeout,
		time.Duration(c.GenericConfig.MinRequestTimeout)*time.Second,
		apiGroupInfo.StaticOpenAPISpec,
		c.GenericConfig.MaxRequestBodyBytes,
	)
	if err != nil {
		return nil, err
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
  
  
  // 5.启动crdController
	crdController := NewDiscoveryController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler)
	namingController := status.NewNamingConditionController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	nonStructuralSchemaController := nonstructuralschema.NewConditionController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	apiApprovalController := apiapproval.NewKubernetesAPIApprovalPolicyConformantConditionController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	finalizingController := finalizer.NewCRDFinalizer(
		s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(),
		crdClient.Apiextensions(),
		crdHandler,
	)
	var openapiController *openapicontroller.Controller
	if utilfeature.DefaultFeatureGate.Enabled(apiextensionsfeatures.CustomResourcePublishOpenAPI) {
		openapiController = openapicontroller.NewController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions())
	}

	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(context genericapiserver.PostStartHookContext) error {
		s.Informers.Start(context.StopCh)
		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(context genericapiserver.PostStartHookContext) error {
		// OpenAPIVersionedService and StaticOpenAPISpec are populated in generic apiserver PrepareRun().
		// Together they serve the /openapi/v2 endpoint on a generic apiserver. A generic apiserver may
		// choose to not enable OpenAPI by having null openAPIConfig, and thus OpenAPIVersionedService
		// and StaticOpenAPISpec are both null. In that case we don't run the CRD OpenAPI controller.
		if utilfeature.DefaultFeatureGate.Enabled(apiextensionsfeatures.CustomResourcePublishOpenAPI) && s.GenericAPIServer.OpenAPIVersionedService != nil && s.GenericAPIServer.StaticOpenAPISpec != nil {
			go openapiController.Run(s.GenericAPIServer.StaticOpenAPISpec, s.GenericAPIServer.OpenAPIVersionedService, context.StopCh)
		}

		go crdController.Run(context.StopCh)
		go namingController.Run(context.StopCh)
		go establishingController.Run(context.StopCh)
		go nonStructuralSchemaController.Run(5, context.StopCh)
		go apiApprovalController.Run(5, context.StopCh)
		go finalizingController.Run(5, context.StopCh)
		return nil
	})
	// we don't want to report healthy until we can handle all CRDs that have already been registered.  Waiting for the informer
	// to sync makes sure that the lister will be valid before we begin.  There may still be races for CRDs added after startup,
	// but we won't go healthy until we can handle the ones already present.
	s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(context genericapiserver.PostStartHookContext) error {
		return wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
			return s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions().Informer().HasSynced(), nil
		}, context.StopCh)
	})

	return s, nil
}
```

<br>

#### 1.1  创建GenericAPIServer

```
  // 1. 创建GenericAPIServer.  APIExtensionsServer的运行依赖于GenericAPIServer，通过c.GenericConfig.New函数创建名为apiextensions-apiserver的服务。
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
```

在创建另外两个apiserver的时候，都用到了这个，最后再统一分析。

<br>

#### 1.2  实例化CustomResourceDefinitions

```
s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}
	

// CustomResourceDefinitions进行了另外的封装
type CustomResourceDefinitions struct {
	GenericAPIServer *genericapiserver.GenericAPIServer

	// provided for easier embedding
	Informers internalinformers.SharedInformerFactory
}
```

APIExtensionsServer（API扩展服务）通过CustomResourceDefinitions对象进行管理，实例化该对象后才能注册APIExtensionsServer下的资源。

<br>

#### 1.3 实例化APIGroupInfo

```go
// 3. 实例化APIGroupInfo.  
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
	if apiResourceConfig.VersionEnabled(v1beta1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// customresourcedefinitions
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)

		apiGroupInfo.VersionedResourcesStorageMap[v1beta1.SchemeGroupVersion.Version] = storage
	}
	if apiResourceConfig.VersionEnabled(v1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// customresourcedefinitions
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)

		apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
	}
	
	
// APIGroupInfo结构体如下所示：	
// Info about an API group.
type APIGroupInfo struct {
	PrioritizedVersions []schema.GroupVersion
	// Info about the resources in this group. It's a map from version to resource to the storage.
  // 存储资源与资源存储对象的对应关系，用于installRest的时候用
	VersionedResourcesStorageMap map[string]map[string]rest.Storage    
	// OptionsExternalVersion controls the APIVersion used for common objects in the
	// schema like api.Status, api.DeleteOptions, and metav1.ListOptions. Other implementors may
	// define a version "v1beta1" but want to use the Kubernetes "v1" internal objects.
	// If nil, defaults to groupMeta.GroupVersion.
	// TODO: Remove this when https://github.com/kubernetes/kubernetes/issues/19018 is fixed.
	OptionsExternalVersion *schema.GroupVersion
	// MetaGroupVersion defaults to "meta.k8s.io/v1" and is the scheme group version used to decode
	// common API implementations like ListOptions. Future changes will allow this to vary by group
	// version (for when the inevitable meta/v2 group emerges).
	MetaGroupVersion *schema.GroupVersion

	// Scheme includes all of the types used by this group and how to convert between them (or
	// to convert objects from outside of this group that are accepted in this API).
	// TODO: replace with interfaces
	Scheme *runtime.Scheme
	// NegotiatedSerializer controls how this group encodes and decodes data
	NegotiatedSerializer runtime.NegotiatedSerializer
	// ParameterCodec performs conversions for query parameters passed to API calls
	ParameterCodec runtime.ParameterCodec

	// StaticOpenAPISpec is the spec derived from the definitions of all resources installed together.
	// It is set during InstallAPIGroups, InstallAPIGroup, and InstallLegacyAPIGroup.
	StaticOpenAPISpec *spec.Swagger
}
```

APIGroupInfo对象用于描述资源组信息，其中该对象的VersionedResourcesStorageMap字段用于存储资源与资源存储对象的对应关系，其表现形式为map[string]map[string]rest.Storage（即<资源版本>/<资源>/<资源存储对象>）;

例如CustomResourceDefinitions资源与资源存储对象的映射关系是v1beta1/customresourcedefinitions/customResourceDefintionStorage。

<br>

在实例化APIGroupInfo对象后，完成其资源与资源存储对象的映射，APIExtensionsServer会先判断apiextensions.k8s.io/v1beta1资源

组/资源版本是否已启用，如果其已启用，则将该资源组、资源版本下的资源与资源存储对象进行映射并存储至APIGroupInfo对象的

VersionedResourcesStorageMap字段中。每个资源（包括子资源）都通过类似于NewREST的函数创建资源存储对象（即

RESTStorage）。kube-apiserver将RESTStorage封装成HTTP Handler函数，资源存储对象以RESTful的方式运行，一个RESTStorage对

象负责一个资源的增、删、改、查操作。当操作CustomResourceDefinitions资源数据时，通过对应的RESTStorage资源存储对象与

genericregistry.Store进行交互。

<br>

**提示：** 一个资源组对应一个APIGroupInfo对象，每个资源（包括子资源）对应一个资源存储对象。

admissionregistration.k8s.io就是一个group。

```
[root@k8s-master ~]# kubectl api-versions    表现形式为<group>/<version>.
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

<br>

#### 1.4 InstallAPIGroup注册APIGroup

```
  // 4. InstallAPIGroup注册APIGroup
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}
```

InstallAPIGroup注册APIGroupInfo的过程非常重要，将APIGroupInfo对象中的<资源组>/<资源版本>/<资源>/<子资源>（包括资源存储

对象）注册到APIExtensionsServerHandler函数。其过程是遍历APIGroupInfo，将<资源组>/<资源版本>/<资源名称>映射到HTTP PATH

请求路径，通过InstallREST函数将资源存储对象作为资源的Handlers方法，最后使用go-restful的ws.Route将定义好的请求路径和

Handlers方法添加路由到go-restful中。整个过程为InstallAPIGroup→s.installAPIResources→InstallREST，代码示例如下：

```

// InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
// It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
// in a slash.
func (g *APIGroupVersion) InstallREST(container *restful.Container) error {
	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
	installer := &APIInstaller{
		group:             g,
		prefix:            prefix,
		minRequestTimeout: g.MinRequestTimeout,
	}

	apiResources, ws, registrationErrors := installer.Install()
	versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
	versionDiscoveryHandler.AddToWebService(ws)
	container.Add(ws)
	return utilerrors.NewAggregate(registrationErrors)
}
```

<br>

InstallREST函数接收restful.Container指针对象。安装过程分为4步，分别介绍如下。

（1）prefix定义了HTTP PATH请求路径，其表现形式为<apiPrefix>/<group>/<version>（即/apis/apiextensions.k8s.io/v1beta1）。

（2）实例化APIInstaller安装器。

（3）在installer.Install安装器内部创建一个go-restful WebService，然后通过a.registerResourceHandlers函数，为资源注册对应的

Handlers方法（即资源存储对象Resource Storage），完成资源与资源Handlers方法的绑定并为go-restfulWebService添加该路由。

（4）最后通过container.Add函数将WebService添加到go-restful Container中。APIExtensionsServer负责管理apiextensions.k8s.io资

源组下的所有资源，该资源有v1beta1版本。通过访问http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1获得该资源/子资源的详细信

息，命令示例如下：

```
# curl http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1  ##需要替换成实际的ip,port
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apiextensions.k8s.io/v1",
  "resources": [
    {
      "name": "customresourcedefinitions",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "crd",
        "crds"
      ],
      "storageVersionHash": "jfWCUB31mvA="
    },
    {
      "name": "customresourcedefinitions/status",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

<br>

```
查看具体某个crd的信息。 khchecks.comcast.gihub.io就是一个crd
#curl http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1/customresourcedefinitions/khchecks.comcast.gihub.io
{
  "kind": "CustomResourceDefinition",
  "apiVersion": "apiextensions.k8s.io/v1",
  "metadata": {
    "name": "khchecks.comcast.github.io",
    "selfLink": "/apis/apiextensions.k8s.io/v1/customresourcedefinitions/khchecks.comcast.github.io",
    "uid": "e115a743-5c79-4aa9-8b16-016b852e9c46",
    "resourceVersion": "197092612",
    "generation": 1,
    "creationTimestamp": "2021-04-13T10:00:14Z"
  },
  "spec": {
    "group": "comcast.github.io",
    "names": {
      "plural": "khchecks",
      "singular": "khcheck",
      "shortNames": [
        "khc"
      ],
      "kind": "KuberhealthyCheck",
      "listKind": "KuberhealthyCheckList"
    },
    "scope": "Namespaced",
    "versions": [
      {
        "name": "v1",
        "served": true,
        "storage": true
      }
    ],
    "conversion": {
      "strategy": "None"
    },
    "preserveUnknownFields": true
  },
  "status": {
    "conditions": [
      {
        "type": "NamesAccepted",
        "status": "True",
        "lastTransitionTime": "2021-04-13T10:00:14Z",
        "reason": "NoConflicts",
        "message": "no conflicts found"
      },
      {
        "type": "Established",
        "status": "True",
        "lastTransitionTime": "2021-04-13T10:00:19Z",
        "reason": "InitialNamesAccepted",
        "message": "the initial names have been accepted"
      }
    ],
    "acceptedNames": {
      "plural": "khchecks",
      "singular": "khcheck",
      "shortNames": [
        "khc"
      ],
      "kind": "KuberhealthyCheck",
      "listKind": "KuberhealthyCheckList"
    },
    "storedVersions": [
      "v1"
    ]
  }
}
```

<br>

#### 1.5 启动crdController

```
go crdController.Run(context.StopCh)
```

这里和kcm中的大部分控制器一样了，runWorker -> processNextWorkItem ->  sync

这里就是当k8s运行起来后，对一个个新增的crd进行处理。

```
func (c *DiscoveryController) sync(version schema.GroupVersion) error {

	apiVersionsForDiscovery := []metav1.GroupVersionForDiscovery{}
	apiResourcesForDiscovery := []metav1.APIResource{}
	versionsForDiscoveryMap := map[metav1.GroupVersion]bool{}

	crds, err := c.crdLister.List(labels.Everything())
	if err != nil {
		return err
	}
	foundVersion := false
	foundGroup := false
	for _, crd := range crds {
		if !apiextensions.IsCRDConditionTrue(crd, apiextensions.Established) {
			continue
		}

		if crd.Spec.Group != version.Group {
			continue
		}

		foundThisVersion := false
		var storageVersionHash string
		for _, v := range crd.Spec.Versions {
			if !v.Served {
				continue
			}
			// If there is any Served version, that means the group should show up in discovery
			foundGroup = true

			gv := metav1.GroupVersion{Group: crd.Spec.Group, Version: v.Name}
			if !versionsForDiscoveryMap[gv] {
				versionsForDiscoveryMap[gv] = true
				apiVersionsForDiscovery = append(apiVersionsForDiscovery, metav1.GroupVersionForDiscovery{
					GroupVersion: crd.Spec.Group + "/" + v.Name,
					Version:      v.Name,
				})
			}
			if v.Name == version.Version {
				foundThisVersion = true
			}
			if v.Storage {
				storageVersionHash = discovery.StorageVersionHash(gv.Group, gv.Version, crd.Spec.Names.Kind)
			}
		}

		if !foundThisVersion {
			continue
		}
		foundVersion = true

		verbs := metav1.Verbs([]string{"delete", "deletecollection", "get", "list", "patch", "create", "update", "watch"})
		// if we're terminating we don't allow some verbs
		if apiextensions.IsCRDConditionTrue(crd, apiextensions.Terminating) {
			verbs = metav1.Verbs([]string{"delete", "deletecollection", "get", "list", "watch"})
		}

		apiResourcesForDiscovery = append(apiResourcesForDiscovery, metav1.APIResource{
			Name:               crd.Status.AcceptedNames.Plural,
			SingularName:       crd.Status.AcceptedNames.Singular,
			Namespaced:         crd.Spec.Scope == apiextensions.NamespaceScoped,
			Kind:               crd.Status.AcceptedNames.Kind,
			Verbs:              verbs,
			ShortNames:         crd.Status.AcceptedNames.ShortNames,
			Categories:         crd.Status.AcceptedNames.Categories,
			StorageVersionHash: storageVersionHash,
		})

		subresources, err := apiextensions.GetSubresourcesForVersion(crd, version.Version)
		if err != nil {
			return err
		}
		if subresources != nil && subresources.Status != nil {
			apiResourcesForDiscovery = append(apiResourcesForDiscovery, metav1.APIResource{
				Name:       crd.Status.AcceptedNames.Plural + "/status",
				Namespaced: crd.Spec.Scope == apiextensions.NamespaceScoped,
				Kind:       crd.Status.AcceptedNames.Kind,
				Verbs:      metav1.Verbs([]string{"get", "patch", "update"}),
			})
		}

		if subresources != nil && subresources.Scale != nil {
			apiResourcesForDiscovery = append(apiResourcesForDiscovery, metav1.APIResource{
				Group:      autoscaling.GroupName,
				Version:    "v1",
				Kind:       "Scale",
				Name:       crd.Status.AcceptedNames.Plural + "/scale",
				Namespaced: crd.Spec.Scope == apiextensions.NamespaceScoped,
				Verbs:      metav1.Verbs([]string{"get", "patch", "update"}),
			})
		}
	}

	if !foundGroup {
		c.groupHandler.unsetDiscovery(version.Group)
		c.versionHandler.unsetDiscovery(version)
		return nil
	}

	sortGroupDiscoveryByKubeAwareVersion(apiVersionsForDiscovery)

	apiGroup := metav1.APIGroup{
		Name:     version.Group,
		Versions: apiVersionsForDiscovery,
		// the preferred versions for a group is the first item in
		// apiVersionsForDiscovery after it put in the right ordered
		PreferredVersion: apiVersionsForDiscovery[0],
	}
	c.groupHandler.setDiscovery(version.Group, discovery.NewAPIGroupHandler(Codecs, apiGroup))

	if !foundVersion {
		c.versionHandler.unsetDiscovery(version)
		return nil
	}
	c.versionHandler.setDiscovery(version, discovery.NewAPIVersionHandler(Codecs, version, discovery.APIResourceListerFunc(func() []metav1.APIResource {
		return apiResourcesForDiscovery
	})))

	return nil
}
```

<br>

创建流程如下：

（1）创建GenericAPIServer

（2）实例化CustomResourceDefinitions

（3）实例化APIGroupInfo，将资源版本、资源、资源存储对象进行相互映射

（4）InstallAPIGroup注册APIGroup（apiextensions.k8s.io）

（5）启动crdController





### 2. 创建KubeAPIServer

创建KubeAPIServer的流程与创建APIExtensionsServer的流程类似，其原理都是将<资源组>/<资源版本>/<资源>与资源存储对象进行映

射并将其存储至APIGroupInfo对象的VersionedResourcesStorageMap字段中。通过installer.Install安装器为资源注册对应的Handlers

方法（即资源存储对象ResourceStorage），完成资源与资源Handlers方法的绑定并为go-restful WebService添加该路由。最后将

WebService添加到go-restful Container中。创建KubeAPIServer的流程如下所示：

（1）创建GenericAPIServer

（2）实例化Master

（3）InstallLegacyAPI注册/api资源

（4）InstallAPIs注册/apis资源。

<br>

```
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
	if err != nil {
		return nil, err
	}
	
	
	// CreateKubeAPIServer creates and wires a workable kube-apiserver
func CreateKubeAPIServer(kubeAPIServerConfig *master.Config, delegateAPIServer genericapiserver.DelegationTarget) (*master.Master, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	return kubeAPIServer, nil
}



// New returns a new instance of Master from the given config.
// Certain config fields will be set to a default value if unset.
// Certain config fields must be specified, including:
//   KubeletClientConfig
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {
	if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
	}

  // 1 创建GenericAPIServer
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

	if c.ExtraConfig.EnableLogsSupport {
		routes.Logs{}.Install(s.Handler.GoRestfulContainer)
	}

  // 2.实例化Master。KubeAPIServer（API核心服务）通过Master对象进行管理，实例化该对象后才能注册KubeAPIServer下的资源。
	m := &Master{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

  // 3.InstallLegacyAPI注册/api资源
	// install legacy rest storage
	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
		}
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}

  // 4.InstallAPIs注册/apis资源
	// The order here is preserved in discovery.
	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
	// with specific priorities.
	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
	// handlers that we have.
	restStorageProviders := []RESTStorageProvider{
		auditregistrationrest.RESTStorageProvider{},
		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		discoveryrest.StorageProvider{},
		extensionsrest.RESTStorageProvider{},
		networkingrest.RESTStorageProvider{},
		noderest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		settingsrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.RESTStorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}

	if c.ExtraConfig.Tunneler != nil {
		m.installTunneler(c.ExtraConfig.Tunneler, corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig).Nodes())
	}

	m.GenericAPIServer.AddPostStartHookOrDie("start-cluster-authentication-info-controller", func(hookContext genericapiserver.PostStartHookContext) error {
		kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
		if err != nil {
			return err
		}
		controller := clusterauthenticationtrust.NewClusterAuthenticationTrustController(m.ClusterAuthenticationInfo, kubeClient)

		// prime values and start listeners
		if m.ClusterAuthenticationInfo.ClientCA != nil {
			if notifier, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.Notifier); ok {
				notifier.AddListener(controller)
			}
			if controller, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(1, hookContext.StopCh)
			}
		}
		if m.ClusterAuthenticationInfo.RequestHeaderCA != nil {
			if notifier, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.Notifier); ok {
				notifier.AddListener(controller)
			}
			if controller, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(1, hookContext.StopCh)
			}
		}

		go controller.Run(1, hookContext.StopCh)
		return nil
	})

	return m, nil
}
```

<br>

#### 2.1 创建GenericAPIServer

等下再分析。

#### 2.2 实例化Master

```
	m := &Master{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}
	
	// Master contains state for a Kubernetes cluster master/api server.
type Master struct {
	GenericAPIServer *genericapiserver.GenericAPIServer

	ClusterAuthenticationInfo clusterauthenticationtrust.ClusterAuthenticationInfo
}
```

KubeAPIServer（API核心服务）通过Master对象进行管理，实例化该对象后才能注册KubeAPIServer下的资源。

后面的InstallLegacyAPI，InstallAPIs都需要实例化之后才能调用。

<br>

在当前的Kubernetes系统中，支持两类资源组，分别是拥有组名的资源组和没有组名的资源组。KubeAPIServer通过InstallLegacyAPI函

数将没有组名的资源组注册到/api前缀的路径下，其表现形式为/api/<version>/<resource>，例如 http://localhost:8080/api/v1/pods。

KubeAPIServer通过InstallAPIs函数将拥有组名的资源组注册到/apis前缀的路径下，其表现形式为/apis/<group>/<version>/<resource>，例如http://localhost:8080/apis/apps/v1/deployments。

#### 2.3 InstallLegacyAPI注册/api资源

代码路径：pkg/master/master.go

```
// install legacy rest storage
	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
		}
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}
```

KubeAPIServer会先判断Core Groups/v1（即核心资源组/资源版本）是否已启用，如果其已启用，则通过m.InstallLegacyAPI函数将

Core Groups/v1注册到KubeAPIServer的/api/v1下。

```go
## curl http://127.0.0.18080/api/v1/pods 这里换成实际的ip,port,可以获得所有的pods。json格式。
```

可以通过访问http://127.0.0.18080/api/v1获得Core Groups/v1下的资源与子资源信息。

<br>

InstallLegacyAPI函数的执行过程分为两步，分别介绍如下。

第1步，通过legacyRESTStorageProvider.NewLegacyRESTStorage函数实例化APIGroupInfo，APIGroupInfo对象用于描述资源组信

息，该对象的VersionedResourcesStorageMap字段用于存储资源与资源存储对象的映射关系，其表现形式为map[string]map[string]rest.Storage （即<资源版本>/<资源>/<资源存储对象>），

例如Pod资源与资源存储对象的映射关系是v1/pods/PodStorage。使Core Groups/v1下的资源与资源存储对象相互映射，代码示例如

下：代码路径：pkg/registry/core/rest/storage_core.go

这里就是将核心的资源，pod, svc等，设置好url和处理函数。

```go
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
	apiGroupInfo := genericapiserver.APIGroupInfo{
		PrioritizedVersions:          legacyscheme.Scheme.PrioritizedVersionsForGroup(""),
		VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
		Scheme:                       legacyscheme.Scheme,
		ParameterCodec:               legacyscheme.ParameterCodec,
		NegotiatedSerializer:         legacyscheme.Codecs,
	}

	var podDisruptionClient policyclient.PodDisruptionBudgetsGetter
	if policyGroupVersion := (schema.GroupVersion{Group: "policy", Version: "v1beta1"}); legacyscheme.Scheme.IsVersionRegistered(policyGroupVersion) {
		var err error
		podDisruptionClient, err = policyclient.NewForConfig(c.LoopbackClientConfig)
		if err != nil {
			return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
		}
	}
	restStorage := LegacyRESTStorage{}

	podTemplateStorage, err := podtemplatestore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	eventStorage, err := eventstore.NewREST(restOptionsGetter, uint64(c.EventTTL.Seconds()))
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
	limitRangeStorage, err := limitrangestore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	resourceQuotaStorage, resourceQuotaStatusStorage, err := resourcequotastore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
	secretStorage, err := secretstore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
	persistentVolumeStorage, persistentVolumeStatusStorage, err := pvstore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
	persistentVolumeClaimStorage, persistentVolumeClaimStatusStorage, err := pvcstore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}
	configMapStorage, err := configmapstore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	namespaceStorage, namespaceStatusStorage, namespaceFinalizeStorage, err := namespacestore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	endpointsStorage, err := endpointsstore.NewREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	nodeStorage, err := nodestore.NewStorage(restOptionsGetter, c.KubeletClientConfig, c.ProxyTransport)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	podStorage, err := podstore.NewStorage(
		restOptionsGetter,
		nodeStorage.KubeletConnectionInfo,
		c.ProxyTransport,
		podDisruptionClient,
	)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	var serviceAccountStorage *serviceaccountstore.REST
	if c.ServiceAccountIssuer != nil && utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) {
		serviceAccountStorage, err = serviceaccountstore.NewREST(restOptionsGetter, c.ServiceAccountIssuer, c.APIAudiences, c.ServiceAccountMaxExpiration, podStorage.Pod.Store, secretStorage.Store)
	} else {
		serviceAccountStorage, err = serviceaccountstore.NewREST(restOptionsGetter, nil, nil, 0, nil, nil)
	}
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	serviceRESTStorage, serviceStatusStorage, err := servicestore.NewGenericREST(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	var serviceClusterIPRegistry rangeallocation.RangeRegistry
	serviceClusterIPRange := c.ServiceIPRange
	if serviceClusterIPRange.IP == nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, fmt.Errorf("service clusterIPRange is missing")
	}

	serviceStorageConfig, err := c.StorageFactory.NewConfig(api.Resource("services"))
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	serviceClusterIPAllocator, err := ipallocator.NewAllocatorCIDRRange(&serviceClusterIPRange, func(max int, rangeSpec string) (allocator.Interface, error) {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd, err := serviceallocator.NewEtcd(mem, "/ranges/serviceips", api.Resource("serviceipallocations"), serviceStorageConfig)
		if err != nil {
			return nil, err
		}
		serviceClusterIPRegistry = etcd
		return etcd, nil
	})
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, fmt.Errorf("cannot create cluster IP allocator: %v", err)
	}
	restStorage.ServiceClusterIPAllocator = serviceClusterIPRegistry

	// allocator for secondary service ip range
	var secondaryServiceClusterIPAllocator ipallocator.Interface
	if utilfeature.DefaultFeatureGate.Enabled(features.IPv6DualStack) && c.SecondaryServiceIPRange.IP != nil {
		var secondaryServiceClusterIPRegistry rangeallocation.RangeRegistry
		secondaryServiceClusterIPAllocator, err = ipallocator.NewAllocatorCIDRRange(&c.SecondaryServiceIPRange, func(max int, rangeSpec string) (allocator.Interface, error) {
			mem := allocator.NewAllocationMap(max, rangeSpec)
			// TODO etcdallocator package to return a storage interface via the storageFactory
			etcd, err := serviceallocator.NewEtcd(mem, "/ranges/secondaryserviceips", api.Resource("serviceipallocations"), serviceStorageConfig)
			if err != nil {
				return nil, err
			}
			secondaryServiceClusterIPRegistry = etcd
			return etcd, nil
		})
		if err != nil {
			return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, fmt.Errorf("cannot create cluster secondary IP allocator: %v", err)
		}
		restStorage.SecondaryServiceClusterIPAllocator = secondaryServiceClusterIPRegistry
	}

	var serviceNodePortRegistry rangeallocation.RangeRegistry
	serviceNodePortAllocator, err := portallocator.NewPortAllocatorCustom(c.ServiceNodePortRange, func(max int, rangeSpec string) (allocator.Interface, error) {
		mem := allocator.NewAllocationMap(max, rangeSpec)
		// TODO etcdallocator package to return a storage interface via the storageFactory
		etcd, err := serviceallocator.NewEtcd(mem, "/ranges/servicenodeports", api.Resource("servicenodeportallocations"), serviceStorageConfig)
		if err != nil {
			return nil, err
		}
		serviceNodePortRegistry = etcd
		return etcd, nil
	})
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, fmt.Errorf("cannot create cluster port allocator: %v", err)
	}
	restStorage.ServiceNodePortAllocator = serviceNodePortRegistry

	controllerStorage, err := controllerstore.NewStorage(restOptionsGetter)
	if err != nil {
		return LegacyRESTStorage{}, genericapiserver.APIGroupInfo{}, err
	}

	serviceRest, serviceRestProxy := servicestore.NewREST(serviceRESTStorage,
		endpointsStorage,
		podStorage.Pod,
		serviceClusterIPAllocator,
		secondaryServiceClusterIPAllocator,
		serviceNodePortAllocator,
		c.ProxyTransport)
 
  // storage就是将ulr和处理函数进行了绑定
	restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.LegacyBinding,

		"podTemplates": podTemplateStorage,

		"replicationControllers":        controllerStorage.Controller,
		"replicationControllers/status": controllerStorage.Status,

		"services":        serviceRest,
		"services/proxy":  serviceRestProxy,
		"services/status": serviceStatusStorage,

		"endpoints": endpointsStorage,

		"nodes":        nodeStorage.Node,
		"nodes/status": nodeStorage.Status,
		"nodes/proxy":  nodeStorage.Proxy,

		"events": eventStorage,

		"limitRanges":                   limitRangeStorage,
		"resourceQuotas":                resourceQuotaStorage,
		"resourceQuotas/status":         resourceQuotaStatusStorage,
		"namespaces":                    namespaceStorage,
		"namespaces/status":             namespaceStatusStorage,
		"namespaces/finalize":           namespaceFinalizeStorage,
		"secrets":                       secretStorage,
		"serviceAccounts":               serviceAccountStorage,
		"persistentVolumes":             persistentVolumeStorage,
		"persistentVolumes/status":      persistentVolumeStatusStorage,
		"persistentVolumeClaims":        persistentVolumeClaimStorage,
		"persistentVolumeClaims/status": persistentVolumeClaimStatusStorage,
		"configMaps":                    configMapStorage,

		"componentStatuses": componentstatus.NewStorage(componentStatusStorage{c.StorageFactory}.serversToValidate),
	}
	if legacyscheme.Scheme.IsVersionRegistered(schema.GroupVersion{Group: "autoscaling", Version: "v1"}) {
		restStorageMap["replicationControllers/scale"] = controllerStorage.Scale
	}
	if legacyscheme.Scheme.IsVersionRegistered(schema.GroupVersion{Group: "policy", Version: "v1beta1"}) {
		restStorageMap["pods/eviction"] = podStorage.Eviction
	}
	if serviceAccountStorage.Token != nil {
		restStorageMap["serviceaccounts/token"] = serviceAccountStorage.Token
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.EphemeralContainers) {
		restStorageMap["pods/ephemeralcontainers"] = podStorage.EphemeralContainers
	}
	apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap

	return restStorage, apiGroupInfo, nil
}
```

每个资源（包括子资源）都通过类似于NewREST的函数创建资源存储对象（即RESTStorage）。kube-apiserver将RESTStorage封装成

HTTP Handler函数，资源存储对象以RESTful的方式运行，一个RESTStorage对象负责一个资源的增、删、改、查操作。当操作

CustomResourceDefinitions资源数据时，通过对应的RESTStorage资源存储对象与genericregistry.Store进行交互。

<br>

第2步，通过m.GenericAPIServer.InstallLegacyAPIGroup函数将APIGroupInfo对象中的<资源组>/<资源版本>/<资源>/<子资源>（包括

资源存储对象）注册到KubeAPIServer Handlers方法。其过程是遍历APIGroupInfo，将<资源组>/<资源版本>/<资源名称>映射到HTTP 

PATH请求路径，通过InstallREST函数将资源存储对象作为资源的Handlers方法。最后使用go-restful的ws.Route将定义好的请求路径和

Handlers方法添加路由到go-restful中。整个过程为InstallLegacyAPIGroup→s.installAPIResources→InstallREST，该过程与

APIExtensionsServer注册APIGroupInfo的过程类似，故不再赘述。

<br>

#### 2.4 InstallAPIs注册/apis资源

```
	// The order here is preserved in discovery.
	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
	// with specific priorities.
	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
	// handlers that we have.
	restStorageProviders := []RESTStorageProvider{
	  ...
	}
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}
```

<br>

 通过m.InstallLegacyAPI函数将拥有组名的资源组注册到KubeAPIServer的/apis下。可以通过访问http://localhost:8080/apis/apps/v1/deployments获得其下的资源与子资源信息。

  InstallLegacyAPI函数的执行过程分为两步，分别介绍如下。

第1步，实例化所有已启用的资源组的APIGroupInfo，APIGroupInfo对象用于描述资源组信息，该对象的VersionedResourcesStorageMap 字段用于存储资源与资源存储对象的映射关系，其表现形式为

map[string]map[string]rest.Storage（即<资源版本>/<资源>/<资源存储对象>），例如Deployment资源与资源存储对象的映射关系是

v1/deployments/deploymentStorage。通过restStorageBuilder.NewRESTStorage→v1Storage函数可实现apps资源组下的资源与资源存储对象的映射。代码不展示了。

每个资源（包括子资源）都通过类似于NewStorage的函数创建资源存储对象（即RESTStorage）。kube-apiserver将RESTStorage封装

成HTTP Handler函数，资源存储对象以RESTful的方式运行，一个RESTStorage对象负责一个资源的增、删、改、查操作。当操作

CustomResourceDefinitions资源数据时，通过对应的RESTStorage资源存储对象与genericregistry.Store进行交互。

<br>

第2步，通过 m.GenericAPIServer.InstallLegacyAPIGroup 函数将APIGroupInfo对象中的<资源组>/<资源版本>/<资源>/<子资源>（包括

资源存储对象）注册到KubeAPIServer Handlers方法。其过程是遍历APIGroupInfo，将<资源组>/<资源版本>/<资源名称>映射到HTTP 

PATH请求路径，通过InstallREST函数将资源存储对象作为资源的Handlers方法。最后使用go-restful的ws.Route将定义好的请求路径和

Handlers方法添加路由到go-restful中。整个过程为InstallAPIGroups→s.installAPIResources→InstallREST，该过程与

APIExtensionsServer注册APIGroupInfo的过程类似，故不再赘述。KubeAPIServer负责管理众多资源组，以apps资源组为例，通过访问

http://127.0.0.1:8080/apis/apps/v1可以获得该资源/子资源的详细信息。

<br>

#### 2.5 路由注册总结

api开头的路由通过`InstallLegacyAPI`方法添加。进入`InstallLegacyAPI`方法，通过`NewLegacyRESTStorage`方法创建各个资源的**RESTStorage**。RESTStorage是一个结构体，具体的定义在`vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go`下，结构体内主要包含`NewFunc`返回特定资源信息、`NewListFunc`返回特定资源列表、`CreateStrategy`特定资源创建时的策略、`UpdateStrategy`更新时的策略以及`DeleteStrategy`删除时的策略等重要方法。

```
// TODO: make the default exposed methods exactly match a generic RESTStorage
type Store struct {
	// NewFunc returns a new instance of the type this registry returns for a
	// GET of a single object, e.g.:
	//
	// curl GET /apis/group/version/namespaces/my-ns/myresource/name-of-object
	NewFunc func() runtime.Object

	// NewListFunc returns a new list of the type this registry; it is the
	// type returned when the resource is listed, e.g.:
	//
	// curl GET /apis/group/version/namespaces/my-ns/myresource
	NewListFunc func() runtime.Object

	// DefaultQualifiedResource is the pluralized name of the resource.
	// This field is used if there is no request info present in the context.
	// See qualifiedResourceFromContext for details.
	DefaultQualifiedResource schema.GroupResource

	// KeyRootFunc returns the root etcd key for this resource; should not
	// include trailing "/".  This is used for operations that work on the
	// entire collection (listing and watching).
	//
	// KeyRootFunc and KeyFunc must be supplied together or not at all.
	KeyRootFunc func(ctx context.Context) string

	// KeyFunc returns the key for a specific object in the collection.
	// KeyFunc is called for Create/Update/Get/Delete. Note that 'namespace'
	// can be gotten from ctx.
	//
	// KeyFunc and KeyRootFunc must be supplied together or not at all.
	KeyFunc func(ctx context.Context, name string) (string, error)

	// ObjectNameFunc returns the name of an object or an error.
	ObjectNameFunc func(obj runtime.Object) (string, error)

	// TTLFunc returns the TTL (time to live) that objects should be persisted
	// with. The existing parameter is the current TTL or the default for this
	// operation. The update parameter indicates whether this is an operation
	// against an existing object.
	//
	// Objects that are persisted with a TTL are evicted once the TTL expires.
	TTLFunc func(obj runtime.Object, existing uint64, update bool) (uint64, error)

	// PredicateFunc returns a matcher corresponding to the provided labels
	// and fields. The SelectionPredicate returned should return true if the
	// object matches the given field and label selectors.
	PredicateFunc func(label labels.Selector, field fields.Selector) storage.SelectionPredicate

	// EnableGarbageCollection affects the handling of Update and Delete
	// requests. Enabling garbage collection allows finalizers to do work to
	// finalize this object before the store deletes it.
	//
	// If any store has garbage collection enabled, it must also be enabled in
	// the kube-controller-manager.
	EnableGarbageCollection bool

	// DeleteCollectionWorkers is the maximum number of workers in a single
	// DeleteCollection call. Delete requests for the items in a collection
	// are issued in parallel.
	DeleteCollectionWorkers int

	// Decorator is an optional exit hook on an object returned from the
	// underlying storage. The returned object could be an individual object
	// (e.g. Pod) or a list type (e.g. PodList). Decorator is intended for
	// integrations that are above storage and should only be used for
	// specific cases where storage of the value is not appropriate, since
	// they cannot be watched.
	Decorator ObjectFunc
	// CreateStrategy implements resource-specific behavior during creation.
	CreateStrategy rest.RESTCreateStrategy
	// AfterCreate implements a further operation to run after a resource is
	// created and before it is decorated, optional.
	AfterCreate ObjectFunc

	// UpdateStrategy implements resource-specific behavior during updates.
	UpdateStrategy rest.RESTUpdateStrategy
	// AfterUpdate implements a further operation to run after a resource is
	// updated and before it is decorated, optional.
	AfterUpdate ObjectFunc

	// DeleteStrategy implements resource-specific behavior during deletion.
	DeleteStrategy rest.RESTDeleteStrategy
	// AfterDelete implements a further operation to run after a resource is
	// deleted and before it is decorated, optional.
	AfterDelete ObjectFunc
	// ReturnDeletedObject determines whether the Store returns the object
	// that was deleted. Otherwise, return a generic success status response.
	ReturnDeletedObject bool
	// ShouldDeleteDuringUpdate is an optional function to determine whether
	// an update from existing to obj should result in a delete.
	// If specified, this is checked in addition to standard finalizer,
	// deletionTimestamp, and deletionGracePeriodSeconds checks.
	ShouldDeleteDuringUpdate func(ctx context.Context, key string, obj, existing runtime.Object) bool
	// ExportStrategy implements resource-specific behavior during export,
	// optional. Exported objects are not decorated.
	ExportStrategy rest.RESTExportStrategy
	// TableConvertor is an optional interface for transforming items or lists
	// of items into tabular output. If unset, the default will be used.
	TableConvertor rest.TableConvertor

	// Storage is the interface for the underlying storage for the
	// resource. It is wrapped into a "DryRunnableStorage" that will
	// either pass-through or simply dry-run.
	Storage DryRunnableStorage
	// StorageVersioner outputs the <group/version/kind> an object will be
	// converted to before persisted in etcd, given a list of possible
	// kinds of the object.
	// If the StorageVersioner is nil, apiserver will leave the
	// storageVersionHash as empty in the discovery document.
	StorageVersioner runtime.GroupVersioner
	// Called to cleanup clients used by the underlying Storage; optional.
	DestroyFunc func()
}
```



 在`NewLegacyRESTStorage`内部，可以看到创建了多种资源的RESTStorage。常见的像event、secret、namespace、endpoints等，统一调用`NewREST`方法构造相应的资源。待所有资源的store创建完成之后，使用`restStorageMap`的Map类型将每个资源的路由和对应的store对应起来，方便后续去做路由的统一规划，代码如下：

```
  // storage就是将ulr和处理函数进行了绑定
	restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		....
	}
```

最终完成以api开头的所有资源的RESTStorage操作。
 创建完之后，则开始进行路由的安装，执行`InstallLegacyAPIGroup`方法，主要调用链为`InstallLegacyAPIGroup-->installAPIResources-->InstallREST-->Install-->registerResourceHandlers`，最终核心的路由构造在`registerResourceHandlers`方法内。

```
vendor/k8s.io/apiserver/pkg/endpoints/installer.go

func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
    
　　　　...

　　　　creater, isCreater := storage.(rest.Creater)
　　　　namedCreater, isNamedCreater := storage.(rest.NamedCreater)
　　　　lister, isLister := storage.(rest.Lister)
　　　　getter, isGetter := storage.(rest.Getter)
　　　　getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
　　　　gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
　　　　collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
　　　　updater, isUpdater := storage.(rest.Updater)
　　　　patcher, isPatcher := storage.(rest.Patcher)
　　　　watcher, isWatcher := storage.(rest.Watcher)
　　　　connecter, isConnecter := storage.(rest.Connecter)
　　　　storageMeta, isMetadata := storage.(rest.StorageMetadata)

　　　　if !isMetadata {
   　　　　　　storageMeta = defaultStorageMetadata{}
　　　　}
　　　　...


　　　　// Handler for standard REST verbs (GET, PUT, POST and DELETE).
　　　　// Add actions at the resource path: /api/apiVersion/resource
　　　　actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
　　　　actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
　　　　actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
　　　　
　　　　// Add actions at the item path: /api/apiVersion/resource/{name}
　　　　actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
　　　　if getSubpath {
   　　　　　　actions = appendIf(actions, action{"GET", itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
　　　　}
　　　　actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
　　　　actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
　　　　actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)

　　　　actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
　　　　actions = appendIf(actions, action{"CONNECT", itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)

　　　　...
　　　　
　　　　routes := []*restful.RouteBuilder{}

　　　　case "GET": // Get a resource.
　　　　　　　　var handler restful.RouteFunction
　　　　　　　　if isGetterWithOptions {
   　　　　　　　　　　handler = restfulGetResourceWithOptions(getterWithOptions, reqScope, isSubresource)
　　　　　　　　} else {
   　　　　　　　　　　handler = restfulGetResource(getter, exporter, reqScope)
　　　　　　　　}

　　　　　　　　if needOverride {
   　　　　　　　　　　// need change the reported verb
　　　　　　　　　　　　handler = metrics.InstrumentRouteFunc(verbOverrider.OverrideMetricsVerb(action.Verb), group, version, resource, subresource, requestScope, metrics.APIServerComponent, handler)
　　　　　　　　} else {
　　　　　　　　　　　　handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, handler)
　　　　　　　　}

　　　　　　　　if a.enableAPIResponseCompression {
   　　　　　　　　　　handler = genericfilters.RestfulWithCompression(handler)
　　　　　　　　}
　　　　　　　　doc := "read the specified " + kind
　　　　　　　　if isSubresource {
   　　　　　　　　　　doc = "read " + subresource + " of the specified " + kind
　　　　　　　　}
　　　　　　　　route := ws.GET(action.Path).To(handler).Doc(doc).
　　　　　　　　Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
　　　　　　　　Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).
　　　　　　　　Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
　　　　　　　　Returns(http.StatusOK, "OK", producedObject).Writes(producedObject)
　　　　　　　　...
　　　　　　　　routes = append(routes, route)

　　　　...

　　　　
　　　　for _, route := range routes {
   　　　　　　route.Metadata(ROUTE_META_GVK, metav1.GroupVersionKind{
      　　　　　　　　Group:   reqScope.Kind.Group,
      　　　　　　　　Version: reqScope.Kind.Version,
      　　　　　　　　Kind:    reqScope.Kind.Kind,
   　　　　　　})
   　　　　　　route.Metadata(ROUTE_META_ACTION, strings.ToLower(action.Verb))
   　　　　　　ws.Route(route)
　　　　}
　　　　...
　　　　return &apiResource, nil
}
```

**registerResourceHandlers**处理逻辑如下：

1.判断storage是否支持create、list、get等方法，并对所有支持的方法进行进一步的处理，如if !isMetadata这一块一样，内容过多不一一贴出；

2.将所有支持的方法存入actions数组中；

3.遍历actions数组，在一个switch语句中，为所有元素定义路由。如贴出的case "GET"这一块，首先创建并包装一个handler对象，然后调用WebService的一系列方法，创建一个route对象，将handler绑定到这个route上。后面还有case "PUT"、case "DELETE"等一系列case，不一一贴出。最后，将route加入routes数组中。

4.遍历routes数组，将route加入WebService中。

5.最后，返回一个APIResource结构体。

这样，Install方法就通过调用registerResourceHandlers方法，完成了WebService与APIResource的绑定。

至此，InstallLegacyAPI方法的逻辑就分析完了。总的来说，这个方法遵循了go-restful的设计模式，在/api路径下注册了WebService，并将WebService加入Container中。

<br>

这是一个非常复杂的方法，整个方法的代码在700行左右。方法的主要功能是通过上一步骤构造的RESTStorage判断该资源可以执行哪些操作（如create、update等），将其对应的操作存入到action，每一个action对应一个标准的rest操作，如create对应的action操作为POST、update对应的action操作为PUT。最终根据actions数组依次遍历，对每一个操作添加一个handler方法，注册到route中去，route注册到webservice中去，完美匹配go-restful的设计模式。

<br>

api开头的路由主要是对基础资源的路由实现，而对于其他附加的资源，如认证相关、网络相关等各种扩展的api资源，统一以apis开头命名，实现入口为`InstallAPIs`。
 `InstallAPIs`与`InstallLegacyAPIGroup`主要的区别是获取RESTStorage的方式。对于api开头的路由来说，都是/api/v1这种统一的格式；而对于apis开头路由则不一样，它包含了多种不同的格式（Kubernetes代码内叫groupName），如/apis/apps、/apis/certificates.k8s.io等各种无规律的groupName。为此，kubernetes提供了一种`RESTStorageProvider`的工厂模式的接口

```
// RESTStorageProvider is a factory type for REST storage.
type RESTStorageProvider interface {
	GroupName() string
	NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool)
}
```


 所有以apis开头的路由的资源都需要实现该接口。GroupName()方法获取到的就是类似于/apis/apps、/apis/certificates.k8s.io这样的groupName，NewRESTStorage方法获取到的是相对应的RESTStorage封装后的信息。然后的步骤和api是一样的。

<br>

**简要总结：**

（1）所有资源都必须有 restStorage。这个包含`NewFunc`返回特定资源信息、`NewListFunc`返回特定资源列表、`CreateStrategy`特定资源创建时的策略、`UpdateStrategy`更新时的策略以及`DeleteStrategy`删除时的策略等重要方法。

（2）有了restStorage，调用 `installAPIResources-->InstallREST-->Install-->registerResourceHandlers` 这一条链路，然后注册路由。



### 3. 创建AggregatorServer

AggregatorServer负责处理 `apiregistration.k8s.io` 组下的APIService资源请求，同时将来自用户的请求拦截转发给aggregated server(AA)。

这里同样，创建AggregatorServer的流程与创建APIExtensionsServer的流程类似，其原理都是将<资源组>/<资源版本>/<资源>与资源存储对

象进行映射并将其存储至APIGroupInfo对象的VersionedResourcesStorageMap字段中。通过installer.Install安装器为资源注册对应的

Handlers方法（即资源存储对象ResourceStorage），完成资源与资源Handlers方法的绑定并为go-restful WebService添加该路由。最

后将WebService添加到go-restful Container中.

创建AggregatorServer的流程说明如下。

（1）创建GenericAPIServer。

（2）实例化APIAggregator。

（3）实例化APIGroupInfo，将资源版本、资源、资源存储对象进行相互映射。

（4）installAPIGroup注册APIGroup（apiregistration.k8s.io）。

这里大体过程和上诉一致。不再啰嗦。

<br>

### 4. 创建GenericAPIServer

无论创建APIExtensionsServer、KubeAPIServer，还是AggregatorServer，它们在底层都依赖于GenericAPIServer。通过GenericAPIServer将Kubernetes资源与REST API进行映射。

例如：

```
  // 1. 创建GenericAPIServer.  APIExtensionsServer的运行依赖于GenericAPIServer，通过c.GenericConfig.New函数创建名为apiextensions-apiserver的服务。
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
```

通过c.GenericConfig.New函数创建GenericAPIServer。在NewAPIServerHandler函数的内部，通过restful.NewContainer创建restful 

Container实例，并设置Router路由。代码示例如下：

```
	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())


func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
	}

	gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(s, panicReason, httpWriter)
	})
	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
		serviceErrorHandler(s, serviceErr, request, response)
	})

	director := director{
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}

	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
}


installAPI通过routes注册GenericAPIServer的相关API。
● routes.Index：用于获取index索引页面。
● routes.Profiling：用于分析性能的可视化页面。
● routes.MetricsWithReset：用于获取metrics指标信息，一般用于Prometheus指标采集。
● routes.Version：用于获取Kubernetes系统版本信息。
```

<br>

### 5. 启动http服务

Go语言提供的HTTP标准库非常强大，Kubernetes API Server在其基础上并没有过多的封装，因为它的功能和性能已经很完善了，可直接

拿来用。在Go语言中开启HTTP服务有很多种方法，例如通过http.ListenAndServe函数可以直接启动HTTP服务，其内部实现了创建

Socket、监控端口等操作。下面看看Kubernetes APIServer通过自定义http.Server的方式创建HTTP服务的过程，代码示例如下：

```
if insecureServingInfo != nil {
		insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(aggregatorServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
		if err := insecureServingInfo.Serve(insecureHandlerChain, kubeAPIServerConfig.GenericConfig.RequestTimeout, stopCh); err != nil {
			return nil, err
		}
	}
	
	
// Serve starts an insecure http server with the given handler. It fails only if
// the initial listen call fails. It does not block.
func (s *DeprecatedInsecureServingInfo) Serve(handler http.Handler, shutdownTimeout time.Duration, stopCh <-chan struct{}) error {
	insecureServer := &http.Server{
		Addr:           s.Listener.Addr().String(),
		Handler:        handler,
		MaxHeaderBytes: 1 << 20,
	}

	if len(s.Name) > 0 {
		klog.Infof("Serving %s insecurely on %s", s.Name, s.Listener.Addr())
	} else {
		klog.Infof("Serving insecurely on %s", s.Listener.Addr())
	}
	_, err := RunServer(insecureServer, s.Listener, shutdownTimeout, stopCh)
	// NOTE: we do not handle stoppedCh returned by RunServer for graceful termination here
	return err
}


// RunServer spawns a go-routine continuously serving until the stopCh is
// closed.
// It returns a stoppedCh that is closed when all non-hijacked active requests
// have been processed.
// This function does not block
// TODO: make private when insecure serving is gone from the kube-apiserver
func RunServer(
	server *http.Server,
	ln net.Listener,
	shutDownTimeout time.Duration,
	stopCh <-chan struct{},
) (<-chan struct{}, error) {
	if ln == nil {
		return nil, fmt.Errorf("listener must not be nil")
	}

	// Shutdown server gracefully.
	stoppedCh := make(chan struct{})
	go func() {
		defer close(stoppedCh)
		<-stopCh
		ctx, cancel := context.WithTimeout(context.Background(), shutDownTimeout)
		server.Shutdown(ctx)
		cancel()
	}()

	go func() {
		defer utilruntime.HandleCrash()

		var listener net.Listener
		listener = tcpKeepAliveListener{ln.(*net.TCPListener)}
		if server.TLSConfig != nil {
			listener = tls.NewListener(listener, server.TLSConfig)
		}

		err := server.Serve(listener)

		msg := fmt.Sprintf("Stopped listening on %s", ln.Addr().String())
		select {
		case <-stopCh:
			klog.Info(msg)
		default:
			panic(fmt.Sprintf("%s due to error: %v", msg, err))
		}
	}()

	return stoppedCh, nil
}
```

在RunServer函数中，通过Go语言标准库的serverServe监听listener，并在运行过程中为每个连接创建一个goroutine。goroutine读取

请求，然后调用Handler函数来处理并响应请求。另外，在Kubernetes API Server的代码中还实现了平滑关闭HTTP服务的功能，利用Go

语言标准库的HTTP Server.Shutdown函数可以在不干扰任何活跃连接的情况下关闭服务。其原理是，首先关闭所有的监听listener，然后

关闭所有的空闲连接，接着无限期地等待所有连接变成空闲状态并关闭。如果设置带有超时的Context，将在HTTP服务关闭之前返回

Context超时错误。

<br>

### 6. 启动https服务

在NonBlockingRun函数，启动了https服务

```
// NonBlockingRun spawns the secure http server. An error is
// returned if the secure port cannot be listened on.
func (s preparedGenericAPIServer) NonBlockingRun(stopCh <-chan struct{}) error {
	// Use an stop channel to allow graceful shutdown without dropping audit events
	// after http server shutdown.
	auditStopCh := make(chan struct{})

	// Start the audit backend before any request comes in. This means we must call Backend.Run
	// before http server start serving. Otherwise the Backend.ProcessEvents call might block.
	if s.AuditBackend != nil {
		if err := s.AuditBackend.Run(auditStopCh); err != nil {
			return fmt.Errorf("failed to run the audit backend: %v", err)
		}
	}
   
  // 开启https服务
	// Use an internal stop channel to allow cleanup of the listeners on error.
	internalStopCh := make(chan struct{})
	var stoppedCh <-chan struct{}
	if s.SecureServingInfo != nil && s.Handler != nil {
		var err error
		stoppedCh, err = s.SecureServingInfo.Serve(s.Handler, s.ShutdownTimeout, internalStopCh)
		if err != nil {
			close(internalStopCh)
			close(auditStopCh)
			return err
		}
	}

	// Now that listener have bound successfully, it is the
	// responsibility of the caller to close the provided channel to
	// ensure cleanup.
	go func() {
		<-stopCh
		close(internalStopCh)
		if stoppedCh != nil {
			<-stoppedCh
		}
		s.HandlerChainWaitGroup.Wait()
		close(auditStopCh)
	}()

	s.RunPostStartHooks(stopCh)

	if _, err := systemd.SdNotify(true, "READY=1\n"); err != nil {
		klog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
	}

	return nil
}
```

HTTPS服务在http.Server上增加了TLSConfig配置，TLSConfig用于配置相关证书，可以通过命令行相关参数（--client-ca-file、--tls-private-key-file、--tls-cert-file参数）进行配置。具体过程不再赘述。

<br>

### 7 总结

Kube-apiserver启动其实就是启动一个服务。这里的关键点就是在于如何添加路由。

创建完之后，则开始进行路由的安装，执行`InstallLegacyAPIGroup`方法，主要调用链为`InstallLegacyAPIGroup-->installAPIResources-->InstallREST-->Install-->registerResourceHandlers`，最终核心的路由构造在`registerResourceHandlers`方法内。这是一个非常复杂的方法，整个方法的代码在700行左右。方法的主要功能是通过上一步骤构造的RESTStorage判断该资源可以执行哪些操作（如create、update等），将其对应的操作存入到action，每一个action对应一个标准的rest操作，如create对应的action操作为POST、update对应的action操作为PUT。最终根据actions数组依次遍历，对每一个操作添加一个handler方法，注册到route中去，route注册到webservice中去，完美匹配go-restful的设计模式。


<br>

### 8. 参考链接

kubernetes源码解剖： https://weread.qq.com/web/reader/f1e3207071eeeefaf1e138akb5332110237b53b3a3d68d2

博客： https://juejin.cn/post/6844903801934069774