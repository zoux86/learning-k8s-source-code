* [Table of Contents](#table-of-contents)
    * [1\. 背景回顾](#1-背景回顾)
    * [2\. 生成apiExtensionsConfig](#2-生成apiextensionsconfig)
    * [3\. createAPIExtensionsServer](#3-createapiextensionsserver)
      * [3\.1  创建GenericAPIServer](#31--创建genericapiserver)
      * [3\.2  实例化CustomResourceDefinitions](#32--实例化customresourcedefinitions)
      * [3\.3 实例化APIGroupInfo](#33-实例化apigroupinfo)
      * [3\.4 InstallAPIGroup注册APIGroup](#34-installapigroup注册apigroup)
      * [3\.5 启动crdController](#35-启动crdcontroller)
    * [4总结](#4总结)
    * [5\. 参考链接](#5-参考链接)

**本章重点：**分析第四个流程，创建APIExtensionsServer

 kube-apiserver整体启动流程如下：

（1）资源注册。

（2）Cobra命令行参数解析

（3）创建APIServer通用配置

（4）创建APIExtensionsServer

（5）创建KubeAPIServer

（6）创建AggregatorServer

（7）启动HTTP服务。

（8）启动HTTPS服务

<br>

### 1. 背景回顾

再次回到 CreateServerChain。在生成配置参数后，CreateServerChain第一个做的就是创建APIExtensionsServer。核心包含2步骤：

（1）生成apiExtensionsConfig

（2）new一个APIExtensionsServer

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

### 2. 生成apiExtensionsConfig

可以看到, 顺序为：生成kube-apisever config -> 再生成apiExtensionsConfig -> new apiExtensionsServer -> new kube-apisever。

```
kubeAPIServerConfig, insecureServingInfo, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	if err != nil {
		return nil, err
	}

	// If additional API servers are added, they should be gated.
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(proxyTransport, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig))
	if err != nil {
		return nil, err
	}
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}
```

原因其实也很好理解，因为kube-apisever config是通用的配置，apiExtensionsConfig 再此基础上多了ExtraConfig配置，核心就是如何和etcd打交道

```
apiextensionsConfig := &apiextensionsapiserver.Config{
		GenericConfig: &genericapiserver.RecommendedConfig{
			Config:                genericConfig,
			SharedInformerFactory: externalInformers,
		},
		ExtraConfig: apiextensionsapiserver.ExtraConfig{
			CRDRESTOptionsGetter: apiextensionsoptions.NewCRDRESTOptionsGetter(etcdOptions),
			MasterCount:          masterCount,
			AuthResolverWrapper:  authResolverWrapper,
			ServiceResolver:      serviceResolver,
		},
	}
	
// NewCRDRESTOptionsGetter create a RESTOptionsGetter for CustomResources.
func NewCRDRESTOptionsGetter(etcdOptions genericoptions.EtcdOptions) genericregistry.RESTOptionsGetter {
	ret := apiserver.CRDRESTOptionsGetter{
		StorageConfig:           etcdOptions.StorageConfig,
		StoragePrefix:           etcdOptions.StorageConfig.Prefix,
		EnableWatchCache:        etcdOptions.EnableWatchCache,
		DefaultWatchCacheSize:   etcdOptions.DefaultWatchCacheSize,
		EnableGarbageCollection: etcdOptions.EnableGarbageCollection,
		DeleteCollectionWorkers: etcdOptions.DeleteCollectionWorkers,
		CountMetricPollPeriod:   etcdOptions.StorageConfig.CountMetricPollPeriod,
	}
	ret.StorageConfig.Codec = unstructured.UnstructuredJSONScheme

	return ret
}
```

### 3. createAPIExtensionsServer

createAPIExtensionsServer核心就是返回一个genericServer。delegateAPIServer是个空的，一开始啥也不干，直到有CRD注册进来才开始工作。它后续将被注册为kube-apiserver的一部分。

核心步骤如下：

（1）创建GenericAPIServer.  APIExtensionsServer的运行依赖于GenericAPIServer，通过c.GenericConfig.New函数创建名为apiextensions-apiserver的服务

（2）实例化CustomResourceDefinitions. APIExtensionsServer（API扩展服务）通过CustomResourceDefinitions对象进行管理，实例化该对象后才能注册APIExtensionsServer下的资源。

（3）实例化APIGroupInfo.

（4）InstallAPIGroup注册APIGroup

（5）启动crdController，处理的CRD的创建/修改/删除

```go
func createAPIExtensionsServer(apiextensionsConfig *apiextensionsapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget) (*apiextensionsapiserver.CustomResourceDefinitions, error) {
   return apiextensionsConfig.Complete().New(delegateAPIServer)
}
Complete()就是补全了上面的APIExtensionsServer config配置，这里主要关心New函数

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

#### 3.1  创建GenericAPIServer

```
  // 1. 创建GenericAPIServer.  APIExtensionsServer的运行依赖于GenericAPIServer，通过c.GenericConfig.New函数创建名为apiextensions-apiserver的服务。
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
```

在创建另外两个apiserver的时候，都用到了这个，最后再统一分析。

<br>

#### 3.2  实例化CustomResourceDefinitions

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

#### 3.3 实例化APIGroupInfo

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

#### 3.4 InstallAPIGroup注册APIGroup

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

#### 3.5 启动crdController

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

### 4总结

（1）可以看出来APIExtensionsServer就是负载CRD资源请求的处理

（2）具体做法是，实例化APIGroupInfo，将资源版本、资源、资源存储对象进行相互映射。这里实际就是将 url 和handler函数关联起来，这个后面kube-apiserver实例化的时候再分析

<br>

### 5. 参考链接

kubernetes源码解剖： https://weread.qq.com/web/reader/f1e3207071eeeefaf1e138akb5332110237b53b3a3d68d2

博客： https://juejin.cn/post/6844903801934069774