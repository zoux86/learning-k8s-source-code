Table of Contents
=================

  * [1. 背景](#1-背景)
     * [1.1 MatchVersionFlags](#11-matchversionflags)
  * [2. NewFactory](#2-newfactory)
     * [2.1 Factory接口](#21-factory接口)
     * [2.2 factoryImpl实现的函数](#22-factoryimpl实现的函数)
     * [2.3 NewBuilder](#23-newbuilder)
        * [2.3.1 builder的常见用法](#231-builder的常见用法)
        * [2.3.2 builder 功能说明](#232-builder-功能说明)
        * [2.3.3 visitorResult](#233-visitorresult)
  * [3 总结](#3-总结)

### 1. 背景

在上文了解到了kubeconfigFlags(ConfigFlags)的作用，接下来往下继续分析。

```
	// 1. configFlags
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
	
	// 2.生成matchVersionKubeConfigFlags对象。
	// --match-server-version=false: Require server version to match client version
	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
	matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())
    
    // 3.persistent意思是说这个flag能任何命令下均可使用，适合全局flag：
	cmds.PersistentFlags().AddGoFlagSet(flag.CommandLine)
    
    // 4.生成Factory
	f := cmdutil.NewFactory(matchVersionKubeConfigFlags)
```

#### 1.1 MatchVersionFlags

可以看出来MatchVersionFlags和configflags的区别就是多了一个checkMatchingServerVersion函数。

ToDiscoveryClient还是复用configflags的。

在kubectl中有一个option就是 --match-server-version=false: Require server version to match client version

可以要求kubectl 和 apiserver的版本一致。

```
func (f *MatchVersionFlags) checkMatchingServerVersion() error {
	f.checkServerVersion.Do(func() {
		if !f.RequireMatchedServerVersion {
			return
		}
		discoveryClient, err := f.Delegate.ToDiscoveryClient()
		if err != nil {
			f.matchesServerVersionErr = err
			return
		}
		f.matchesServerVersionErr = discovery.MatchesServerVersion(version.Get(), discoveryClient)
	})

	return f.matchesServerVersionErr
}

// ToRESTConfig implements RESTClientGetter.
// Returns a REST client configuration based on a provided path
// to a .kubeconfig file, loading rules, and config flag overrides.
// Expects the AddFlags method to have been called.
func (f *MatchVersionFlags) ToRESTConfig() (*rest.Config, error) {
	if err := f.checkMatchingServerVersion(); err != nil {
		return nil, err
	}
	clientConfig, err := f.Delegate.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	// TODO we should not have to do this.  It smacks of something going wrong.
	setKubernetesDefaults(clientConfig)
	return clientConfig, nil
}

func (f *MatchVersionFlags) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	return f.Delegate.ToRawKubeConfigLoader()
}

func (f *MatchVersionFlags) ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	if err := f.checkMatchingServerVersion(); err != nil {
		return nil, err
	}
	return f.Delegate.ToDiscoveryClient()
}

// ToRESTMapper returns a mapper.
func (f *MatchVersionFlags) ToRESTMapper() (meta.RESTMapper, error) {
	if err := f.checkMatchingServerVersion(); err != nil {
		return nil, err
	}
	return f.Delegate.ToRESTMapper()
}
```

### 2. NewFactory

NewFactory其实就是上面configflags的子类。factoryImpl实现了Factory的接口，所以是Factory类型。

```
f := cmdutil.NewFactory(matchVersionKubeConfigFlags)


func NewFactory(clientGetter genericclioptions.RESTClientGetter) Factory {
	if clientGetter == nil {
		panic("attempt to instantiate client_access_factory with nil clientGetter")
	}

	f := &factoryImpl{
		clientGetter: clientGetter,
	}

	return f
}
```

#### 2.1 Factory接口

```
type Factory interface {
	genericclioptions.RESTClientGetter   

	// DynamicClient returns a dynamic client ready for use
	DynamicClient() (dynamic.Interface, error)

	// KubernetesClientSet gives you back an external clientset
	KubernetesClientSet() (*kubernetes.Clientset, error)

	// Returns a RESTClient for accessing Kubernetes resources or an error.
	RESTClient() (*restclient.RESTClient, error)

	// NewBuilder returns an object that assists in loading objects from both disk and the server
	// and which implements the common patterns for CLI interactions with generic resources.
	NewBuilder() *resource.Builder

	// Returns a RESTClient for working with the specified RESTMapping or an error. This is intended
	// for working with arbitrary resources and is not guaranteed to point to a Kubernetes APIServer.
	ClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)
	// Returns a RESTClient for working with Unstructured objects.
	UnstructuredClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)

	// Returns a schema that can validate objects stored on disk.
	Validator(validate bool) (validation.Schema, error)
	// OpenAPISchema returns the schema openapi schema definition
	OpenAPISchema() (openapi.Resources, error)
}
```

<br>

#### 2.2 factoryImpl实现的函数

之前介绍了, client-go种有四种类型的客户端

- RESTClient： 是对HTTP Request进行了封装，实现了RESTful风格的API。其他客户端都是在RESTClient基础上的实现。可与用于k8s内置资源和CRD资源
- ClientSet:是对k8s内置资源对象的客户端的集合，默认情况下，不能操作CRD资源，但是通过client-gen代码生成的话，也是可以操作CRD资源的。
- DynamicClient:不仅能对K8S内置资源进行处理，还可以对CRD资源进行处理，不需要client-gen生成代码即可实现。
- DiscoveryClient：用于发现kube-apiserver所支持的资源组、资源版本、资源信息（即Group、Version、Resources）。DynamicClient内部实现了Unstructured，用于处理非结构化数据结构（即无法提前预知数据结构），这也是DynamicClient能够处理CRD自定义资源的关键。

<br>

可以看出来 factoryImpl 除了继续了configflags的函数：ToRESTConfig， ToRESTMapper， ToDiscoveryClient，ToRawKubeConfigLoader外。还有如下的函数：

 KubernetesClientSet() ：生成了KubernetesClientSet，其实也是一个 Clientset客户端。第二种类型

DynamicClient()：生成了DynamicClient，第三种类型

RESTClient()：生成了restclient。 第一种类型

ClientForMapping():  针对结构化对象，根据mapping的gvk，生成 rest路径

UnstructuredClientForMapping():  根据mapping的gvk，生成 rest路径, 和上面不同的就是，上面的是结构体话的，这个是非机构体。对应DynamicClient

OpenAPISchema():  通过discoveryClient获取k8s对象信息

Validator():  如果指定validate, 根据OpenAPISchema获得的信息，生成一个validation.schema用于进行验证

NewBuilder(): 生成一个Builder, 这里对builder还有点陌生，下一节看看Builder到底是什么。

```
// 1.复用了之前configflags的函数
func (f *factoryImpl) ToRESTConfig() (*restclient.Config, error) {
	return f.clientGetter.ToRESTConfig()
}

func (f *factoryImpl) ToRESTMapper() (meta.RESTMapper, error) {
	return f.clientGetter.ToRESTMapper()
}

func (f *factoryImpl) ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	return f.clientGetter.ToDiscoveryClient()
}

func (f *factoryImpl) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	return f.clientGetter.ToRawKubeConfigLoader()
}

// 2.生成了KubernetesClientSet，其实也是一个 Clientset客户端。第二种类型
func (f *factoryImpl) KubernetesClientSet() (*kubernetes.Clientset, error) {
	clientConfig, err := f.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	return kubernetes.NewForConfig(clientConfig)
}

//  3.生成了DynamicClient，第三种类型
func (f *factoryImpl) DynamicClient() (dynamic.Interface, error) {
	clientConfig, err := f.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	return dynamic.NewForConfig(clientConfig)
}

// 4.生成了一个builder
// NewBuilder returns a new resource builder for structured api objects.
func (f *factoryImpl) NewBuilder() *resource.Builder {
	return resource.NewBuilder(f.clientGetter)
}

// 5.生成了restclient。 第一种类型
func (f *factoryImpl) RESTClient() (*restclient.RESTClient, error) {
	clientConfig, err := f.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	setKubernetesDefaults(clientConfig)
	return restclient.RESTClientFor(clientConfig)
}

// 6.根据mapping的gvk，生成 rest路径
func (f *factoryImpl) ClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error) {
	cfg, err := f.clientGetter.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	if err := setKubernetesDefaults(cfg); err != nil {
		return nil, err
	}
	gvk := mapping.GroupVersionKind
	switch gvk.Group {
	case corev1.GroupName:
		cfg.APIPath = "/api"
	default:
		cfg.APIPath = "/apis"
	}
	gv := gvk.GroupVersion()
	cfg.GroupVersion = &gv
	return restclient.RESTClientFor(cfg)
}

// 7. 根据mapping的gvk，生成 rest路径, 和上面不同的就是，上面的是结构体话的，这个是非机构体。对应DynamicClient
func (f *factoryImpl) UnstructuredClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error) {
	cfg, err := f.clientGetter.ToRESTConfig()
	if err != nil {
		return nil, err
	}
	if err := restclient.SetKubernetesDefaults(cfg); err != nil {
		return nil, err
	}
	cfg.APIPath = "/apis"
	if mapping.GroupVersionKind.Group == corev1.GroupName {
		cfg.APIPath = "/api"
	}
	gv := mapping.GroupVersionKind.GroupVersion()
	cfg.ContentConfig = resource.UnstructuredPlusDefaultContentConfig()
	cfg.GroupVersion = &gv
	return restclient.RESTClientFor(cfg)
}

// 8.如果指定validate, 根据下面函数获得的信息，生成一个validation.schema进行验证
func (f *factoryImpl) Validator(validate bool) (validation.Schema, error) {
	if !validate {
		return validation.NullSchema{}, nil
	}

	resources, err := f.OpenAPISchema()
	if err != nil {
		return nil, err
	}

	return validation.ConjunctiveSchema{
		openapivalidation.NewSchemaValidation(resources),
		validation.NoDoubleKeySchema{},
	}, nil
}

// 9. 通过discoveryClient获取k8s对象信息
// OpenAPISchema returns metadata and structural information about Kubernetes object definitions.
func (f *factoryImpl) OpenAPISchema() (openapi.Resources, error) {
	discovery, err := f.clientGetter.ToDiscoveryClient()
	if err != nil {
		return nil, err
	}

	// Lazily initialize the OpenAPIGetter once
	f.openAPIGetter.once.Do(func() {
		// Create the caching OpenAPIGetter
		f.openAPIGetter.getter = openapi.NewOpenAPIGetter(discovery)
	})

	// Delegate to the OpenAPIGetter
	return f.openAPIGetter.getter.Get()
}
```

#### 2.3 NewBuilder

```
// NewBuilder returns a new resource builder for structured api objects.
func (f *factoryImpl) NewBuilder() *resource.Builder {
	return resource.NewBuilder(f.clientGetter)
}
```

##### 2.3.1 builder的常见用法

以kubectl create命令为例， Builder大多方法支持链式调用，最后的Do()返回一个type Result struct。这里一些列链式调用大部分都在根据传入的Cmd来设置新建Builder的属性值。

这个和rest http请求也是一样 各种函数后面.(点) 函数，最后调用一个DO发送请求。

```
r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
		

Visitor 的构建。
r 的构建跟builder属性相关，依次过一下处理Builder实例的函数流

Unstructured() : 对b.mapper 赋值，b.mapper = unstructured
Schema()： 对b.schema 赋值，b.schema = schema
ContinueOnError(): b.continueOnError 置为true。 意为遇到有错误的资源也不马上返回，跳过错误，继续处理下一个资源
NamespaceParam(cmdNamespace)：b.namespace = cmdNamespace
DefaultNamespace(): b.defaultNamespace 置为true
FilenameParam(enforceNamespace, &options.FilenameOptions) : 后边详细说
LabelSelectorParam(options.Selector)：对 b.labelSelector进行赋值
Flatten(): b.flatten 置为true,
Do()：后边详细说
```

##### 2.3.2 builder 功能说明

Builder是Kubectl命令行信息的内部载体，可以通过Builder生成Result对象。Builder 结构体保存了从命令行获取的各种参数，以及它实现了各种函数用于处理这些参数 并将其转换为一系列的resources，最终用Visitor 方法迭代处理resource。可以看出来builder的成员变量非常多。

**成员变量**

```
// Builder provides convenience functions for taking arguments and parameters
// from the command line and converting them to a list of resources to iterate
// over using the Visitor interface.
type Builder struct {
	categoryExpanderFn CategoryExpanderFunc

	// mapper is set explicitly by resource builders
	mapper *mapper

	// clientConfigFn is a function to produce a client, *if* you need one
	clientConfigFn ClientConfigFunc

	restMapperFn RESTMapperFunc

	// objectTyper is statically determinant per-command invocation based on your internal or unstructured choice
	// it does not ever need to rely upon discovery.
	objectTyper runtime.ObjectTyper

	// codecFactory describes which codecs you want to use
	negotiatedSerializer runtime.NegotiatedSerializer

	// local indicates that we cannot make server calls
	local bool

	errs []error

	paths  []Visitor
	stream bool
	dir    bool

	labelSelector     *string
	fieldSelector     *string
	selectAll         bool
	limitChunks       int64
	requestTransforms []RequestTransform

	resources []string

	namespace    string
	allNamespace bool
	names        []string

	resourceTuples []resourceTuple

	defaultNamespace bool
	requireNamespace bool

	flatten bool
	latest  bool

	requireObject bool

	singleResourceType bool
	continueOnError    bool

	singleItemImplied bool

	export bool

	schema ContentValidator

	// fakeClientFn is used for testing
	fakeClientFn FakeClientFunc
}
```

**函数说明：**

builder中大部分成员函数都是对builder进行赋值操作。这里介绍一些重要的函数。

```
k8s.io/cli-runtime/pkg/resource/builder.go

// NamespaceParam accepts the namespace that these resources should be
// considered under from - used by DefaultNamespace() and RequireNamespace()
/*
	func (b *Builder) NamespaceParam 设置b *Builder的namespace属性，
	会被 DefaultNamespace() and RequireNamespace()使用
*/
func (b *Builder) NamespaceParam(namespace string) *Builder {
	b.namespace = namespace
	return b
}

// DefaultNamespace instructs the builder to set the namespace value for any object found
// to NamespaceParam() if empty.
/*
	让builder在namespace为空的时候，找到namespace的值
*/
func (b *Builder) DefaultNamespace() *Builder {
	b.defaultNamespace = true
	return b
}

// AllNamespaces instructs the builder to use NamespaceAll as a namespace to request resources
// across all of the namespace. This overrides the namespace set by NamespaceParam().
/*
	func AllNamespaces 让builder使用NamespaceAll作为cmd的namespace，向所有的namespace请求resources。
	将重写由func (b *Builder) NamespaceParam(namespace string)设置的属性namespace
*/
func (b *Builder) AllNamespaces(allNamespace bool) *Builder {
	/*
		如果入参allNamespace bool＝true,那么重写b *Builder的namespace和allNamespace属性
			api.NamespaceAll定义在 pkg/api/v1/types.go
				==>NamespaceAll string = ""
	*/
	if allNamespace {
		b.namespace = api.NamespaceAll
	}
	b.allNamespace = allNamespace
	return b
}


// kubectl create就用到了这个，赋值了mapper和其他的函数，比如decoder
// Unstructured updates the builder so that it will request and send unstructured
// objects. Unstructured objects preserve all fields sent by the server in a map format
// based on the object's JSON structure which means no data is lost when the client
// reads and then writes an object. Use this mode in preference to Internal unless you
// are working with Go types directly.
func (b *Builder) Unstructured() *Builder {
	if b.mapper != nil {
		b.errs = append(b.errs, fmt.Errorf("another mapper was already selected, cannot use unstructured types"))
		return b
	}
	b.objectTyper = unstructuredscheme.NewUnstructuredObjectTyper()
	b.mapper = &mapper{
		localFn:      b.isLocal,
		restMapperFn: b.restMapperFn,
		clientFn:     b.getClient,
		decoder:      &metadataValidatingDecoder{unstructured.UnstructuredJSONScheme},
	}

	return b
}


// FilenameParam groups input in two categories: URLs and files (files, directories, STDIN)
// If enforceNamespace is false, namespaces in the specs will be allowed to
// override the default namespace. If it is true, namespaces that don't match
// will cause an error.
// If ContinueOnError() is set prior to this method, objects on the path that are not
// recognized will be ignored (but logged at V(2)).

译：func (b *Builder) FilenameParam以URLs and files (files, directories, STDIN)两种形式来传入参数。
		如果enforceNamespace＝false，specs中声明的namespaces将允许被重写为default namespace。
		如果enforceNamespace＝true，不匹配的namespaces将导致error。
		如果在此方法之前设置了ContinueOnError()，则路径上无法识别的objects将被忽略（记录在 V(2)级别的log中）。

func (b *Builder) FilenameParam(enforceNamespace bool, filenameOptions *FilenameOptions) *Builder {
	if errs := filenameOptions.validate(); len(errs) > 0 {
		b.errs = append(b.errs, errs...)
		return b
	}
	recursive := filenameOptions.Recursive
	paths := filenameOptions.Filenames
	for _, s := range paths {
		switch {
		case s == "-":
			b.Stdin()
		case strings.Index(s, "http://") == 0 || strings.Index(s, "https://") == 0:
			url, err := url.Parse(s)
			if err != nil {
				b.errs = append(b.errs, fmt.Errorf("the URL passed to filename %q is not valid: %v", s, err))
				continue
			}
			b.URL(defaultHttpGetAttempts, url)
		default:
			if !recursive {
				b.singleItemImplied = true
			}
			b.Path(recursive, s)
		}
	}
	if filenameOptions.Kustomize != "" {
		b.paths = append(b.paths, &KustomizeVisitor{filenameOptions.Kustomize,
			NewStreamVisitor(nil, b.mapper, filenameOptions.Kustomize, b.schema)})
	}

	if enforceNamespace {
		b.RequireNamespace()
	}

	return b
}


labelselector 和 Fieldselector
// LabelSelectorParam defines a selector that should be applied to the object types to load.
// This will not affect files loaded from disk or URL. If the parameter is empty it is
// a no-op - to select all resources invoke `b.LabelSelector(labels.Everything.String)`.
func (b *Builder) LabelSelectorParam(s string) *Builder {
	selector := strings.TrimSpace(s)
	if len(selector) == 0 {
		return b
	}
	if b.selectAll {
		b.errs = append(b.errs, fmt.Errorf("found non-empty label selector %q with previously set 'all' parameter. ", s))
		return b
	}
	return b.LabelSelector(selector)
}

// LabelSelector accepts a selector directly and will filter the resulting list by that object.
// Use LabelSelectorParam instead for user input.
func (b *Builder) LabelSelector(selector string) *Builder {
	if len(selector) == 0 {
		return b
	}

	b.labelSelector = &selector
	return b
}

// FieldSelectorParam defines a selector that should be applied to the object types to load.
// This will not affect files loaded from disk or URL. If the parameter is empty it is
// a no-op - to select all resources.
func (b *Builder) FieldSelectorParam(s string) *Builder {
	s = strings.TrimSpace(s)
	if len(s) == 0 {
		return b
	}
	if b.selectAll {
		b.errs = append(b.errs, fmt.Errorf("found non-empty field selector %q with previously set 'all' parameter. ", s))
		return b
	}
	b.fieldSelector = &s
	return b
}


// ContinueOnError will attempt to load and visit as many objects as possible, even if some visits
// return errors or some objects cannot be loaded. The default behavior is to terminate after
// the first error is returned from a VisitorFunc.
/*
	ContinueOnError将尝试加载并访问尽可能多的对象，即使某些访问返回错误或某些对象无法加载。
	默认行为是在 ‘VisitorFunc返回第一个错误之后’ 终止。
*/
func (b *Builder) ContinueOnError() *Builder {
	b.continueOnError = true
	return b
}

// Latest will fetch the latest copy of any objects loaded from URLs or files from the server.
/*
	译：func (b *Builder) Latest() 将从server端获取该URL或文件加载objects的最新副本。
*/
func (b *Builder) Latest() *Builder {
	b.latest = true
	return b
}

// Flatten will convert any objects with a field named "Items" that is an array of runtime.Object
// compatible types into individual entries and give them their own items. The original object
// is not passed to any visitors.
/*
	译：Flatten将使用一个名为“Items”的字段将任何对象转换为一个runtime.Object兼容类型的数组，并将它们分配给各自的items。
		 原始对象不会传递给任何访问者。
*/
func (b *Builder) Flatten() *Builder {
	b.flatten = true
	return b
}
```

<br>

do基本都是builder这一套中最后使用的一个函数。 

```
Do() 返回一个type Result struct，该type Result struct中含有一个visitor Visitor，visitor 能访问在Builder中定义的resources。The visitor将遵守由ContinueOnError指定的错误行为。

// Do returns a Result object with a Visitor for the resources identified by the Builder.
// The visitor will respect the error behavior specified by ContinueOnError. Note that stream
// inputs are consumed by the first execution - use Infos() or Object() on the Result to capture a list
// for further iteration.
func (b *Builder) Do() *Result {
    r := b.visitorResult()  // 第一次生成 result 实例，初始化r.visitor的值
    r.mapper = b.Mapper()
    if r.err != nil {
        return r    // 第一处return，出错时，从此处返回
    }
    if b.flatten {  // 默认为true, 在b.Flatten() 中赋值
        r.visitor = NewFlattenListVisitor(r.visitor, b.mapper) //第一次修改r.visitor 
    }
    helpers := []VisitorFunc{}
    if b.defaultNamespace {// 默认为true, 在b.DefaultNamespace() 赋值
        helpers = append(helpers, SetNamespace(b.namespace))
    }
    if b.requireNamespace { // 在FileFilenameParam() 赋值为true
        /* b.namespace 即RunCreate() 中的cmdNamespace，cmdNamespace 的值为 kubectl 命令中-- 
          namespace 参数的值*/
        helpers = append(helpers, RequireNamespace(b.namespace))
    }
    helpers = append(helpers, FilterNamespace)
    if b.requireObject {  // 默认为true，在builder 结构体初始化中赋值
        helpers = append(helpers, RetrieveLazy)
    }
    r.visitor = NewDecoratedVisitor(r.visitor, helpers...)  //  第二次修改r.visitor
    if b.continueOnError {
        r.visitor = ContinueOnErrorVisitor{r.visitor}
    }
    return r  //第二处return，一般从此处返回。
}
```

<br>

Result结构如下：

```
// Result contains helper methods for dealing with the outcome of a Builder.
type Result struct {
	err     error
	visitor Visitor

	sources            []Visitor
	singleItemImplied  bool
	targetsSingleItems bool

	mapper       *mapper
	ignoreErrors []utilerrors.Matcher

	// populated by a call to Infos
	info []*Info
} 
```

Info结构如下：

```
// Info contains temporary info to execute a REST call, or show the results
// of an already completed REST call.
type Info struct {
	// Client will only be present if this builder was not local
	Client RESTClient
	// Mapping will only be present if this builder was not local
	Mapping *meta.RESTMapping

	// Namespace will be set if the object is namespaced and has a specified value.
	Namespace string
	Name      string

	// Optional, Source is the filename or URL to template file (.json or .yaml),
	// or stdin to use to handle the resource
	Source string
	
	
	// 这个就是server端返回的对象
	// Optional, this is the most recent value returned by the server if available. It will
	// typically be in unstructured or internal forms, depending on how the Builder was
	// defined. If retrieved from the server, the Builder expects the mapping client to
	// decide the final form. Use the AsVersioned, AsUnstructured, and AsInternal helpers
	// to alter the object versions.
	Object runtime.Object
	
	// 译：可选，这是server端知道的此类resource的最新resource version。
			它可能与该object的resource version 不匹配，
			但如果设置它应该等于或新于对象的资源版本（但服务器定义资源版本）。

		简单来说，ResourceVersion的值是etcd中全局最新的Index
	*/
	// Optional, this is the most recent resource version the server knows about for
	// this type of resource. It may not match the resource version of the object,
	// but if set it should be equal to or newer than the resource version of the
	// object (however the server defines resource version).
	ResourceVersion string
	// Optional, should this resource be exported, stripped of cluster-specific and instance specific fields
	Export bool
}
```

##### 2.3.3 visitorResult

再看看Do调用的visitorResult函数，可以看出其返回值是一个type Result struct指针。 根据前面设置的参数值（或者说是命令行cmd的参数）来选择相应的Visitor

```
func (b *Builder) visitorResult() *Result {
    // 返回一，错误返回，b.errs 为一列表结构，列表长度大于0，说明之前有错误发生，在此返回。
	if len(b.errs) > 0 {
		return &Result{err: utilerrors.NewAggregate(b.errs)}
	}
    
	if b.selectAll {
		selector := labels.Everything().String()
		b.labelSelector = &selector
	}
    
    // create 命令进入此分支，例如 kubectl create -f xxx.yaml(或者URL/xxx.yaml)*/
	// visit items specified by paths
	if len(b.paths) != 0 {
		return b.visitByPaths()
	}
    
    // 
	// visit selectors
	if b.labelSelector != nil || b.fieldSelector != nil {
		return b.visitBySelector()
	}
    
    // get 某一个指定资源对象时，进入此分支
	// visit items specified by resource and name
	if len(b.resourceTuples) != 0 {
		return b.visitByResource()
	}

	// visit items specified by name
	if len(b.names) != 0 {
		return b.visitByName()
	}

	if len(b.resources) != 0 {
		for _, r := range b.resources {
			_, err := b.mappingFor(r)
			if err != nil {
				return &Result{err: err}
			}
		}
		return &Result{err: fmt.Errorf("resource(s) were provided, but no name, label selector, or --all flag specified")}
	}
	return &Result{err: missingResourceError}
}
```

综上，可以看出Do()函数主要的根据Builder设置的属性值来获取一个type Result struct。 type Result struct中最重要的数据结构是visitor Visitor和info []*Info。 然后create函数中可以看到调用createAndRefresh。

最终info就是从apiserver接受到的对象。

```
// createAndRefresh creates an object from input info and refreshes info with that object
func createAndRefresh(info *resource.Info) error {
	obj, err := resource.NewHelper(info.Client, info.Mapping).Create(info.Namespace, true, info.Object, nil)
	if err != nil {
		return err
	}
	info.Refresh(obj, true)
	return nil
}

func (m *Helper) Create(namespace string, modify bool, obj runtime.Object, options *metav1.CreateOptions) (runtime.Object, error) {
	if options == nil {
		options = &metav1.CreateOptions{}
	}
	if modify {
		// Attempt to version the object based on client logic.
		version, err := metadataAccessor.ResourceVersion(obj)
		if err != nil {
			// We don't know how to clear the version on this object, so send it to the server as is
			return m.createResource(m.RESTClient, m.Resource, namespace, obj, options)
		}
		if version != "" {
			if err := metadataAccessor.SetResourceVersion(obj, ""); err != nil {
				return nil, err
			}
		}
	}

	return m.createResource(m.RESTClient, m.Resource, namespace, obj, options)
}

func (m *Helper) createResource(c RESTClient, resource, namespace string, obj runtime.Object, options *metav1.CreateOptions) (runtime.Object, error) {
	return c.Post().
		NamespaceIfScoped(namespace, m.NamespaceScoped).
		Resource(resource).
		VersionedParams(options, metav1.ParameterCodec).
		Body(obj).
		Do().
		Get()
}
```

### 3 总结

（1）factory其实就是在之前的configflags的基础上做了一些扩展

（2）factory最重要的两个成员对象就是buider 和 vistor (builder包含了vistor)

（3）其实configflags就应该可以实现了往server端发送请求的功能了，但是kubectl功能强大，所有利用了builder+visotor机制做了封装优化。

（4）builder的具体功能就是包含了cmd和默认的所有配置。在使用的时候就是   f.NewBuilder().xx.xx.xx.Do()。在xx.xx的过程中不仅根据配置实例化了一个builder。和生成了一个与之对应的vistor

（5）vistor就是一种设计模式。kubectl中的vistor有两种，一种是产生info的vistor，另一种是处理Info的visotor。产生info的visotor在发送请求到apiserver之前对info增加某些关键字段（name, kind, spec的内容等等）。

 处理info的vistor在apiserver返回后，再对info进行处理。目前对vistor细节理解还不够，接下来进一步分析vistor。

