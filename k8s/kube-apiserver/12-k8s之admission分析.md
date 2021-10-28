Table of Contents
=================

  * [1. 背景](#1-背景)
  * [2. 分析流程](#2-分析流程)
     * [2.1 Admission的注册](#21-admission的注册)
     * [2.2 admission的调用](#22-admission的调用)
     * [2.3  validatingwebhook, mutatingwebhook的调用](#23--validatingwebhook-mutatingwebhook的调用)
        * [2.3.1 ValidatingAdmissionWebhook调用](#231-validatingadmissionwebhook调用)
        * [2.3.2 MutatingAdmissionWebhook调用](#232-mutatingadmissionwebhook调用)
     * [2.4 动态更新webhook的原理](#24-动态更新webhook的原理)
  * [3. 总结](#3-总结)
  * [4.参考链接：](#4参考链接)

### 1. 背景


api Request -> 认证 -> 授权 -> admission -> etcd.

和initializer不同，webhook是在保存在etcd之前工作的。


经过了认证，授权之后，接下来就到了webhook这个环节了。

这篇笔记主要就是分析 `MutatingAdmissionWebhook` 和  `ValidatingAdmissionWebhook` 如何工作的。

<br>

### 2. 分析流程

#### 2.1 Admission的注册

kube-apiserver在调用NewServerRunOptions函数初始化options的时候，调用了NewAdmissionOptions去初始化了AdmissionOptions，并注册了内置的

admission插件和webhook admission插件。

```
// NewServerRunOptions creates a new ServerRunOptions object with default parameters
func NewServerRunOptions() *ServerRunOptions {
   s := ServerRunOptions{
      // 省略...
      // 初始化AdmissionOptions
      Admission:               kubeoptions.NewAdmissionOptions(), 
      Authentication:          kubeoptions.NewBuiltInAuthenticationOptions().WithAll(),
      Authorization:           kubeoptions.NewBuiltInAuthorizationOptions(),
      // 省略...
   }
   // ...
   return &s
}
```

<br>

**AdmissionOptions的一些基础概念**

```
options.AdmissionOptions
// AdmissionOptions holds the admission options.
// It is a wrap of generic AdmissionOptions.
type AdmissionOptions struct {
	// GenericAdmission holds the generic admission options.
	GenericAdmission *genericoptions.AdmissionOptions
	// DEPRECATED flag, should use EnabledAdmissionPlugins and DisabledAdmissionPlugins.
	// They are mutually exclusive, specify both will lead to an error.
	PluginNames []string
}

genericoptions.AdmissionOptions
// AdmissionOptions holds the admission options
type AdmissionOptions struct {
    // 有序的推荐插件列表集合  
	RecommendedPluginOrder []string
	// 默认禁止的插件  
	DefaultOffPlugins sets.String
	// 开启的插件列表，通过kube-apiserver 启动参数设置--enable-admission-plugins 选项  
	EnablePlugins []string
	// 禁止的插件列表，通过kube-apiserver 启动参数设置 --disable-admission-plugins 选项  
	DisablePlugins []string
	// ConfigFile is the file path with admission control configuration.
	ConfigFile string
	// 代表了所有已经注册的插件 
	Plugins *admission.Plugins
}
```

<br>

**options. NewAdmissionOptions()**

NewAdmissionOptions里面先是调用genericoptions.NewAdmissionOptions创建一个AdmissionOptions，NewAdmissionOptions同时也注册了lifecycle、validatingwebhook、mutatingwebhook这三个插件。然后再调用RegisterAllAdmissionPlugins注册内置的其他admission。

```
options. NewAdmissionOptions()
// NewAdmissionOptions creates a new instance of AdmissionOptions
// Note:
//  In addition it calls RegisterAllAdmissionPlugins to register
//  all kube-apiserver admission plugins.
//
//  Provides the list of RecommendedPluginOrder that holds sane values
//  that can be used by servers that don't care about admission chain.
//  Servers that do care can overwrite/append that field after creation.
func NewAdmissionOptions() *AdmissionOptions {
    // 这里注册了 lifecycle, initialization,validatingwebhook,mutatingwebhook 四个admission。（2.2.2 中mutating的注册函数就是这个时候调用的）
	options := genericoptions.NewAdmissionOptions()
	// 这里注册了所有的 admission, 没有上面四个 admission
	// register all admission plugins
	RegisterAllAdmissionPlugins(options.Plugins)
	// set RecommendedPluginOrder
	options.RecommendedPluginOrder = AllOrderedPlugins           // 确定了admission-plugin的相对顺序。
	// set DefaultOffPlugins
	
	// 设置默认的停用插件
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}


genericoptions.NewAdmissionOptions()
// NewAdmissionOptions creates a new instance of AdmissionOptions
// Note:
//  In addition it calls RegisterAllAdmissionPlugins to register
//  all generic admission plugins.
//
//  Provides the list of RecommendedPluginOrder that holds sane values
//  that can be used by servers that don't care about admission chain.
//  Servers that do care can overwrite/append that field after creation.
func NewAdmissionOptions() *AdmissionOptions {
	options := &AdmissionOptions{
		Plugins: admission.NewPlugins(),
		// This list is mix of mutating admission plugins and validating
		// admission plugins. The apiserver always runs the validating ones
		// after all the mutating ones, so their relative order in this list
		// doesn't matter.
		RecommendedPluginOrder: []string{lifecycle.PluginName, initialization.PluginName, mutatingwebhook.PluginName, validatingwebhook.PluginName},
		DefaultOffPlugins:      sets.NewString(initialization.PluginName),
	}
	// 注册了lifecycle、validatingwebhook、mutatingwebhook
	server.RegisterAllAdmissionPlugins(options.Plugins)
	return options
}

// validatingwebhook, mutatingwebhook 是动态的，这里应该就是注册一个总体的概念，而不是一个一个的实体。
// RegisterAllAdmissionPlugins registers all admission plugins
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	lifecycle.Register(plugins)
	initialization.Register(plugins)
	validatingwebhook.Register(plugins)
	mutatingwebhook.Register(plugins)
}
```

**AllOrderedPlugins**

```
// AllOrderedPlugins is the list of all the plugins in order.
var AllOrderedPlugins = []string{
	admit.PluginName,                        // AlwaysAdmit
	autoprovision.PluginName,                // NamespaceAutoProvision
	lifecycle.PluginName,                    // NamespaceLifecycle
	exists.PluginName,                       // NamespaceExists
	scdeny.PluginName,                       // SecurityContextDeny
	antiaffinity.PluginName,                 // LimitPodHardAntiAffinityTopology
	podpreset.PluginName,                    // PodPreset
	limitranger.PluginName,                  // LimitRanger
	serviceaccount.PluginName,               // ServiceAccount
	noderestriction.PluginName,              // NodeRestriction
	alwayspullimages.PluginName,             // AlwaysPullImages
	imagepolicy.PluginName,                  // ImagePolicyWebhook
	podsecuritypolicy.PluginName,            // PodSecurityPolicy
	podnodeselector.PluginName,              // PodNodeSelector
	podpriority.PluginName,                  // Priority
	defaulttolerationseconds.PluginName,     // DefaultTolerationSeconds
	podtolerationrestriction.PluginName,     // PodTolerationRestriction
	exec.DenyEscalatingExec,                 // DenyEscalatingExec
	exec.DenyExecOnPrivileged,               // DenyExecOnPrivileged
	eventratelimit.PluginName,               // EventRateLimit
	extendedresourcetoleration.PluginName,   // ExtendedResourceToleration
	label.PluginName,                        // PersistentVolumeLabel
	setdefault.PluginName,                   // DefaultStorageClass
	storageobjectinuseprotection.PluginName, // StorageObjectInUseProtection
	gc.PluginName,                           // OwnerReferencesPermissionEnforcement
	resize.PluginName,                       // PersistentVolumeClaimResize
	mutatingwebhook.PluginName,              // MutatingAdmissionWebhook
	initialization.PluginName,               // Initializers
	validatingwebhook.PluginName,            // ValidatingAdmissionWebhook
	resourcequota.PluginName,                // ResourceQuota
	deny.PluginName,                         // AlwaysDeny
}
```


<br>

#### 2.2 admission的调用
前面已经分析AdmissionPlugin注册到ServerRunOptions的过程， buildGenericConfig中会调用ServerRunOptions.Admission.ApplyTo生成admission chain设置到GenericConfig里面。把所有的admission plugin生成chainAdmissionHandler对象，其实就是plugin数组，这个类的Admit、Validate等方法会遍历调用每个plugin的Admit、Validate方法


```
buildGenericConfig(){
	err = s.Admission.ApplyTo(
		genericConfig,
		versionedInformers,
		kubeClientConfig,
		feature.DefaultFeatureGate,
		pluginInitializers...)
}
```

GenericConfig.AdmissionControl 又会赋值给GenericAPIServer.admissionControl


```
func (a *AdmissionOptions) ApplyTo(
   c *server.Config,
   informers informers.SharedInformerFactory,
   kubeAPIServerClientConfig *rest.Config,
   features featuregate.FeatureGate,
   pluginInitializers ...admission.PluginInitializer,
) error {
      // 省略 ...
    // 找到所有启用的plugin
   pluginNames := a.enabledPluginNames()
 
   pluginsConfigProvider, err := admission.ReadAdmissionConfiguration(pluginNames, a.ConfigFile, configScheme)
   if err != nil {
      return fmt.Errorf("failed to read plugin config: %v", err)
   }
 
   clientset, err := kubernetes.NewForConfig(kubeAPIServerClientConfig)
   if err != nil {
      return err
   }
   genericInitializer := initializer.New(clientset, informers, c.Authorization.Authorizer, features)
   initializersChain := admission.PluginInitializers{}
   pluginInitializers = append(pluginInitializers, genericInitializer)
   initializersChain = append(initializersChain, pluginInitializers...)
    // 把所有的admission plugin生成admissionChain，实际是个plugin数组
   admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
   if err != nil {
      return err
   }
    // 把admissionChain设置给GenericConfig.AdmissionControl 
   c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
   return nil
}

```

Admission Plugin是在kube-apiserver处理完前面的handler之后，在调用RESTStorage的Get、Create、Update、Delete等函数前会调用Admission Plugin。

kube-apiserver有很多的handler组成了handler链，这写handler链的最内层，是使用gorestful框架注册的WebService。每个WebService都对应一种资源的RESTStorage，比如NodeStorage（pkg/registry/core/node/storage/storage.go )，installAPIResources初始化WebService时，会把RESTStorage的Get、Create、Update等函数分别封装成Get、POST、PUT等http方法的handler注册到WebService中。 

比如把Update函数封装成http handler 作为PUT方法的handler，而在这个hanlder调用Update函数之前，会先调用Admission Plugin的Admit、Validate等函数。下面看个PUT方法的例子。

<br>

a.group.Admit是从GenericAPIServer.admissionControl取的值，就是前面ApplyTo函数生成的admissionChain。admit、updater作为参数调用restfulUpdateResource函数生成的handler



a.group.Admit是从GenericAPIServer.admissionControl取的值，就是前面ApplyTo函数生成的admissionChain。admit、updater作为参数调用restfulUpdateResource函数生成的handler





```
// staging/src/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
   admit := a.group.Admit
   // 省略 ...
   updater, isUpdater := storage.(rest.Updater)
   // 省略 ...
   switch action.Verb {
     case "GET": ...
    case "PUT": // Update a resource.
       doc := "replace the specified " + kind
       if isSubresource {
          doc = "replace " + subresource + " of the specified " + kind
       }
       // admit、updater作为参数调用restfulUpdateResource函数生成的handler
       handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, restfulUpdateResource(updater, reqScope, admit))
       route := ws.PUT(action.Path).To(handler).
          Doc(doc).
          Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
          Operation("replace"+namespaced+kind+strings.Title(subresource)+operationSuffix).
          Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
          Returns(http.StatusOK, "OK", producedObject).
          // TODO: in some cases, the API may return a v1.Status instead of the versioned object
          // but currently go-restful can't handle multiple different objects being returned.
          Returns(http.StatusCreated, "Created", producedObject).
          Reads(defaultVersionedObject).
          Writes(producedObject)
       if err := AddObjectParams(ws, route, versionedUpdateOptions); err != nil {
          return nil, err
       }
       addParams(route, action.Params)
       routes = append(routes, route)     
     case "PARTCH": ...  
     // 省略 ....
   }     
}

restfulUpdateResource调用了 handlers.UpdateResource。
func restfulUpdateResource(r rest.Updater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.UpdateResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}
```

看handlers.UpdateResource的代码实现，会先判断如果传入的admission.Interface参数是MutationInterface类型，就调用Admit，也就是调用admissionChain的Admit，最终会遍历调用每个Admission Plugin的Admit方法。而Webhook Admission是众多admission中的一个。

 

执行完Admission，后面的requestFunc 才会调用RESTStorage的Update函数。每个资源的RESTStorage最终都是要调用ETCD3Storage的Get、Update等函数。

```
// staging/src/k8s.io/apiserver/pkg/endpoints/handlers/update.go
func UpdateResource(r rest.Updater, scope *RequestScope, admit admission.Interface) http.HandlerFunc {
   return func(w http.ResponseWriter, req *http.Request) {
      // 省略 ...
      ae := request.AuditEventFrom(ctx)
      audit.LogRequestObject(ae, obj, scope.Resource, scope.Subresource, scope.Serializer)
      admit = admission.WithAudit(admit, ae)
    // 如果admit是MutationInterface类型的，就调用其Admit函数，也就是admissionChain的Admit
      if mutatingAdmission, ok := admit.(admission.MutationInterface); ok {
         transformers = append(transformers, func(ctx context.Context, newObj, oldObj runtime.Object) (runtime.Object, error) {
            isNotZeroObject, err := hasUID(oldObj)
            if err != nil {
               return nil, fmt.Errorf("unexpected error when extracting UID from oldObj: %v", err.Error())
            } else if !isNotZeroObject {
               if mutatingAdmission.Handles(admission.Create) {
                  return newObj, mutatingAdmission.Admit(ctx, admission.NewAttributesRecord(newObj, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Create, updateToCreateOptions(options), dryrun.IsDryRun(options.DryRun), userInfo), scope)
               }
            } else {
               if mutatingAdmission.Handles(admission.Update) {
                  return newObj, mutatingAdmission.Admit(ctx, admission.NewAttributesRecord(newObj, oldObj, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Update, options, dryrun.IsDryRun(options.DryRun), userInfo), scope)
               }
            }
            return newObj, nil
         })
      }
      // 省略 ...
      // 执行完MutationInterface类型的admission，这里先会执行validatingAdmission，然后才调用RESTStorage的Update函数 
      requestFunc := func() (runtime.Object, error) {
         obj, created, err := r.Update(
            ctx,
            name,
            rest.DefaultUpdatedObjectInfo(obj, transformers...),
            withAuthorization(rest.AdmissionToValidateObjectFunc(
               admit,
               admission.NewAttributesRecord(nil, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Create, updateToCreateOptions(options), dryrun.IsDryRun(options.DryRun), userInfo), scope),
               scope.Authorizer, createAuthorizerAttributes),
           // 这里调用了validatingAdmission.Validate函数
            rest.AdmissionToValidateObjectUpdateFunc(
               admit,
               admission.NewAttributesRecord(nil, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Update, options, dryrun.IsDryRun(options.DryRun), userInfo), scope),
            false,
            options,
         )
         wasCreated = created
         return obj, err
      }
      result, err := finishRequest(timeout, func() (runtime.Object, error) {
         result, err := requestFunc()
         // 省略 ...
         return result, err
      })
      // ...
      transformResponseObject(ctx, scope, trace, req, w, status, outputMediaType, result)
   }
}


// 这里调用了validatingAdmission.Validate函数
// AdmissionToValidateObjectUpdateFunc converts validating admission to a rest validate object update func
func AdmissionToValidateObjectUpdateFunc(admit admission.Interface, staticAttributes admission.Attributes, o admission.ObjectInterfaces) ValidateObjectUpdateFunc {
	validatingAdmission, ok := admit.(admission.ValidationInterface)
	if !ok {
		return func(ctx context.Context, obj, old runtime.Object) error { return nil }
	}
	return func(ctx context.Context, obj, old runtime.Object) error {
		finalAttributes := admission.NewAttributesRecord(
			obj,
			old,
			staticAttributes.GetKind(),
			staticAttributes.GetNamespace(),
			staticAttributes.GetName(),
			staticAttributes.GetResource(),
			staticAttributes.GetSubresource(),
			staticAttributes.GetOperation(),
			staticAttributes.GetOperationOptions(),
			staticAttributes.IsDryRun(),
			staticAttributes.GetUserInfo(),
		)
		if !validatingAdmission.Handles(finalAttributes.GetOperation()) {
			return nil
		}
		return validatingAdmission.Validate(ctx, finalAttributes, o)
	}
}
```

以上是PUT方法的例子，里面调用了MutationInterface和ValidationInterface。其他的方法比如POST、DELETE等也是类似。但是GET方法不会调用Admission Plugin。



#### 2.3  validatingwebhook, mutatingwebhook的调用

validatingwebhook和mutatingwebhook分别位于staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/validating/plugin.go，staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/plugin.go两个文件中。

##### 2.3.1 ValidatingAdmissionWebhook调用

（1） ValidatingAdmissionWebhook的Validate()函数实现了ValidationInterface接口，有请求到来时kube-apiserver会调用所有admission 的Validate()方法。ValidatingAdmissionWebhook持有了一个Webhook对象，Validate()会调用Webhook.Dispatch()。

（2）Webhook.Dispatch()又调用了其持有的dispatcher的Dispatch()方法。dispatcher时通过dispatcherFactory创建的，dispatcherFactory是ValidatingAdmissionWebhook创建generic.Webhook时候传入的newValidatingDispatcher函数。调用dispatcherFactory函数创建的实际上是validatingDispatcher对象，也就是Webhook.Dispatch()调用的是validatingDispatcher.Dispatch()。

（3）validatingDispatcher.Dispatch()会逐个远程调用注册的webhook plugin





NewValidatingAdmissionWebhook初始化了ValidatingAdmissionWebhook对象，内部持有了一个generic.Webhook对象，generic.Webhook是一个Validate和mutate公用的框架，创建generic.Webhook时需要一个dispatcherFactory函数，用这个函数生成dispatcher对象。

```
// staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/validating/plugin.go
// NewValidatingAdmissionWebhook returns a generic admission webhook plugin.
func NewValidatingAdmissionWebhook(configFile io.Reader) (*Plugin, error) {
   handler := admission.NewHandler(admission.Connect, admission.Create, admission.Delete, admission.Update)
   p := &Plugin{}
   var err error
   p.Webhook, err = generic.NewWebhook(handler, configFile, configuration.NewValidatingWebhookConfigurationManager, newValidatingDispatcher(p))
   if err != nil {
      return nil, err
   }
   return p, nil
}
 
// Validate makes an admission decision based on the request attributes.
func (a *Plugin) Validate(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
   return a.Webhook.Dispatch(ctx, attr, o)
}
```

调用generic.Webhook.Dispatch()时会调用dispatcher对象的Dispatch。

```
// Dispatch is called by the downstream Validate or Admit methods.
func (a *Webhook) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
   if rules.IsWebhookConfigurationResource(attr) {
      return nil
   }
   if !a.WaitForReady() {
      return admission.NewForbidden(attr, fmt.Errorf("not yet ready to handle request"))
   }
   hooks := a.hookSource.Webhooks()
   return a.dispatcher.Dispatch(ctx, attr, o, hooks)
}
```

validatingDispatcher.Dispatch遍历所有的hooks ，找到相关的webhooks，然后执行callHooks调用外部注册进来的

```go
func (d *validatingDispatcher) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces, hooks []webhook.WebhookAccessor) error {
   var relevantHooks []*generic.WebhookInvocation
   // Construct all the versions we need to call our webhooks
   versionedAttrs := map[schema.GroupVersionKind]*generic.VersionedAttributes{}
   for _, hook := range hooks {
       // 遍历所有的webhooks，根据ValidatingWebhookConfiguration中的rules是否匹配找到所有相关的hooks
      invocation, statusError := d.plugin.ShouldCallHook(hook, attr, o)
      if statusError != nil {
         return statusError
      }
      if invocation == nil {
         continue
      }
      relevantHooks = append(relevantHooks, invocation)
      // If we already have this version, continue
      if _, ok := versionedAttrs[invocation.Kind]; ok {
         continue
      }
      versionedAttr, err := generic.NewVersionedAttributes(attr, invocation.Kind, o)
      if err != nil {
         return apierrors.NewInternalError(err)
      }
      versionedAttrs[invocation.Kind] = versionedAttr
   }
 
   if len(relevantHooks) == 0 {
      // no matching hooks
      return nil
   }
 
   // Check if the request has already timed out before spawning remote calls
   select {
   case <-ctx.Done():
      // parent context is canceled or timed out, no point in continuing
      return apierrors.NewTimeoutError("request did not complete within requested timeout", 0)
   default:
   }
 
   wg := sync.WaitGroup{}
   errCh := make(chan error, len(relevantHooks))
   wg.Add(len(relevantHooks))
   for i := range relevantHooks {
      go func(invocation *generic.WebhookInvocation) {
         defer wg.Done()
         hook, ok := invocation.Webhook.GetValidatingWebhook()
         if !ok {
            utilruntime.HandleError(fmt.Errorf("validating webhook dispatch requires v1.ValidatingWebhook, but got %T", hook))
            return
         }
         versionedAttr := versionedAttrs[invocation.Kind]
         t := time.Now()
         // 启动多个go routine 并行调用注册进来的webhook plugin
         err := d.callHook(ctx, hook, invocation, versionedAttr)
         ignoreClientCallFailures := hook.FailurePolicy != nil && *hook.FailurePolicy == v1.Ignore
         rejected := false
         if err != nil {
            switch err := err.(type) {
            case *webhookutil.ErrCallingWebhook:
               if !ignoreClientCallFailures {
                  rejected = true
                  admissionmetrics.Metrics.ObserveWebhookRejection(hook.Name, "validating", string(versionedAttr.Attributes.GetOperation()), admissionmetrics.WebhookRejectionCallingWebhookError, 0)
               }
            case *webhookutil.ErrWebhookRejection:
               rejected = true
               admissionmetrics.Metrics.ObserveWebhookRejection(hook.Name, "validating", string(versionedAttr.Attributes.GetOperation()), admissionmetrics.WebhookRejectionNoError, int(err.Status.ErrStatus.Code))
            default:
               rejected = true
               admissionmetrics.Metrics.ObserveWebhookRejection(hook.Name, "validating", string(versionedAttr.Attributes.GetOperation()), admissionmetrics.WebhookRejectionAPIServerInternalError, 0)
            }
         }
         admissionmetrics.Metrics.ObserveWebhook(time.Since(t), rejected, versionedAttr.Attributes, "validating", hook.Name)
         if err == nil {
            return
         }
 
         if callErr, ok := err.(*webhookutil.ErrCallingWebhook); ok {
            if ignoreClientCallFailures {
               klog.Warningf("Failed calling webhook, failing open %v: %v", hook.Name, callErr)
               utilruntime.HandleError(callErr)
               return
            }
 
            klog.Warningf("Failed calling webhook, failing closed %v: %v", hook.Name, err)
            errCh <- apierrors.NewInternalError(err)
            return
         }
 
         if rejectionErr, ok := err.(*webhookutil.ErrWebhookRejection); ok {
            err = rejectionErr.Status
         }
         klog.Warningf("rejected by webhook %q: %#v", hook.Name, err)
         errCh <- err
      }(relevantHooks[i])
   }
   // 等待多个goroutine 执行完成
   wg.Wait()
   close(errCh)
 
   var errs []error
   for e := range errCh {
      errs = append(errs, e)
   }
   if len(errs) == 0 {
      return nil
   }
   if len(errs) > 1 {
      for i := 1; i < len(errs); i++ {
         // TODO: merge status errors; until then, just return the first one.
         utilruntime.HandleError(errs[i])
      }
   }
   return errs[0]
}
```



##### 2.3.2 MutatingAdmissionWebhook调用

看MutatingWebhook的构造函数就可以看到，MutatingWebhook和ValidatingWebhook的代码架构是一样的，只不过在创建generic.Webhook的时候传入的dispatcherFactory函数是newMutatingDispatcher，所以Webhook.Dispatch()最终调用的就是mutatingDispatcher.Dispatch(),这个和validatingDispatcher.Dispatch的实现逻辑基本是一样的，也是根据WebhookConfiguration中的rules是否匹配找到相关的webhooks，然后逐个调用。


```
// staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/mutating/plugin.go
// NewMutatingWebhook returns a generic admission webhook plugin.
func NewMutatingWebhook(configFile io.Reader) (*Plugin, error) {
   handler := admission.NewHandler(admission.Connect, admission.Create, admission.Delete, admission.Update)
   p := &Plugin{}
   var err error
   p.Webhook, err = generic.NewWebhook(handler, configFile, configuration.NewMutatingWebhookConfigurationManager, newMutatingDispatcher(p))
   if err != nil {
      return nil, err
   }
 
   return p, nil
}
 
// ValidateInitialization implements the InitializationValidator interface.
func (a *Plugin) ValidateInitialization() error {
   if err := a.Webhook.ValidateInitialization(); err != nil {
      return err
   }
   return nil
}
 
// Admit makes an admission decision based on the request attributes.
func (a *Plugin) Admit(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
   return a.Webhook.Dispatch(ctx, attr, o)
}


func (a *mutatingDispatcher) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces, hooks []webhook.WebhookAccessor) error {
	reinvokeCtx := attr.GetReinvocationContext()
	var webhookReinvokeCtx *webhookReinvokeContext
	if v := reinvokeCtx.Value(PluginName); v != nil {
		webhookReinvokeCtx = v.(*webhookReinvokeContext)
	} else {
		webhookReinvokeCtx = &webhookReinvokeContext{}
		reinvokeCtx.SetValue(PluginName, webhookReinvokeCtx)
	}

	if reinvokeCtx.IsReinvoke() && webhookReinvokeCtx.IsOutputChangedSinceLastWebhookInvocation(attr.GetObject()) {
		// If the object has changed, we know the in-tree plugin re-invocations have mutated the object,
		// and we need to reinvoke all eligible webhooks.
		webhookReinvokeCtx.RequireReinvokingPreviouslyInvokedPlugins()
	}
	defer func() {
		webhookReinvokeCtx.SetLastWebhookInvocationOutput(attr.GetObject())
	}()
	var versionedAttr *generic.VersionedAttributes
	//是一个一个执行的
	for i, hook := range hooks {
		attrForCheck := attr
		if versionedAttr != nil {
			attrForCheck = versionedAttr
		}
		invocation, statusErr := a.plugin.ShouldCallHook(hook, attrForCheck, o)
		if statusErr != nil {
			return statusErr
		}
		if invocation == nil {
			continue
		}
		hook, ok := invocation.Webhook.GetMutatingWebhook()
		if !ok {
			return fmt.Errorf("mutating webhook dispatch requires v1.MutatingWebhook, but got %T", hook)
		}
		// This means that during reinvocation, a webhook will not be
		// called for the first time. For example, if the webhook is
		// skipped in the first round because of mismatching labels,
		// even if the labels become matching, the webhook does not
		// get called during reinvocation.
		if reinvokeCtx.IsReinvoke() && !webhookReinvokeCtx.ShouldReinvokeWebhook(invocation.Webhook.GetUID()) {
			continue
		}
		

	return nil
}
```

#### 2.4 动态更新webhook的原理

我们使用的时候都是通过创建类似于创建这样的来增加一个webhook。那如果我增加了一个这个webhook, 是如何生效的呢。

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: validation-webhook-example-cfg
  labels:
    app: admission-webhook-example
webhooks:
  - name: required-labels.banzaicloud.com
    clientConfig:
      service:
        name: admission-webhook-example-webhook-svc
        namespace: default
        path: "/validate"
      caBundle: ${CA_BUNDLE}
    rules:
      - operations: [ "CREATE" ]
        apiGroups: ["apps", ""]
        apiVersions: ["v1"]
        resources: ["deployments","services"]
    namespaceSelector:
      matchLabels:
        admission-webhook-example: enabled
```

<br>

```
// Dispatch is called by the downstream Validate or Admit methods.
func (a *Webhook) Dispatch(ctx context.Context, attr admission.Attributes, o admission.ObjectInterfaces) error {
	if rules.IsWebhookConfigurationResource(attr) {
		return nil
	}
	if !a.WaitForReady() {
		return admission.NewForbidden(attr, fmt.Errorf("not yet ready to handle request"))
	}
	//这里获取了所有的webhook，然后再调用的Dispatch函数
	hooks := a.hookSource.Webhooks()
	return a.dispatcher.Dispatch(ctx, attr, o, hooks)
}


// 获得所有的validatingWebhookConfiguration
// Webhooks returns the merged ValidatingWebhookConfiguration.
func (v *validatingWebhookConfigurationManager) Webhooks() []webhook.WebhookAccessor {
	return v.configuration.Load().([]webhook.WebhookAccessor)
}

```



ValidatingWebhookConfigurationManager会维护所有的validatingWebhookConfiguration，一旦有ValidatingWebhookConfigurationManager的add, update, del都会调用updateConfiguration更新

```
pkg/admission/configuration/validating_webhook_manager.go
func NewValidatingWebhookConfigurationManager(f informers.SharedInformerFactory) generic.Source {
	informer := f.Admissionregistration().V1().ValidatingWebhookConfigurations()
	manager := &validatingWebhookConfigurationManager{
		configuration: &atomic.Value{},
		lister:        informer.Lister(),
		hasSynced:     informer.Informer().HasSynced,
	}

	// Start with an empty list
	manager.configuration.Store([]webhook.WebhookAccessor{})

	// On any change, rebuild the config
	informer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    func(_ interface{}) { manager.updateConfiguration() },
		UpdateFunc: func(_, _ interface{}) { manager.updateConfiguration() },
		DeleteFunc: func(_ interface{}) { manager.updateConfiguration() },
	})

	return manager
}

//然后上面Load的时候就回获得最新的webhook
func mergeValidatingWebhookConfigurations(configurations []*v1.ValidatingWebhookConfiguration) []webhook.WebhookAccessor {
	sort.SliceStable(configurations, ValidatingWebhookConfigurationSorter(configurations).ByName)
	accessors := []webhook.WebhookAccessor{}
	for _, c := range configurations {
		// webhook names are not validated for uniqueness, so we check for duplicates and
		// add a int suffix to distinguish between them
		names := map[string]int{}
		for i := range c.Webhooks {
			n := c.Webhooks[i].Name
			uid := fmt.Sprintf("%s/%s/%d", c.Name, n, names[n])
			names[n]++
			accessors = append(accessors, webhook.NewValidatingWebhookAccessor(uid, c.Name, &c.Webhooks[i]))
		}
	}
	return accessors
}
```



### 3. 总结

（1）webhook是通过插入在 apiserver的处理链条中，存入etcd之前生效的

（2）mutatingwebhook，ValidatingAdmission都是有对应的manager来实时更新的

（3）ValidatingAdmission，mutatingwebhook的不同在于

* 所有的请求先经过mutatingwebhook，在经过ValidatingAdmission。这也很好理解，因为ValidatingAdmission不会修改对象
* ValidatingAdmission是并行处理的，都满足后放行(可以设置超时跳过改webhook的策略)
* mutatingwebhook是一个一个串行操作的

### 4.参考链接：

 https://blog.csdn.net/u014152978/article/details/107170600
