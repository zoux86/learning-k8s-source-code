Table of Contents
=================

  * [1.背景](#1背景)
  * [2 kubectl的kubeconfig](#2-kubectl的kubeconfig)
     * [2.1 kubeConfigFlags](#21-kubeconfigflags)
        * [2.1.1 ToRESTConfig](#211-torestconfig)
        * [2.1.2 ToDiscoveryClient](#212-todiscoveryclient)
        * [2.1.3 ToRESTMapper](#213-torestmapper)
        * [2.1.4 举例说明discorvey和restMapper在创建删除资源时起到的作用](#214-举例说明discorvey和restmapper在创建删除资源时起到的作用)
  * [3.总结](#3总结)

### 1.背景

在第一篇 kubectl整体流程的分析中。kubectl在定义子命令之前，做了两件事情，就是网上经常说的Factory机制。

（1）设置kubeconfigflags，用于连接apiserver

（2）利用kubeconfigflags生成了, 一个Factory f

然后上篇文件补充了一些基础知识。介绍了client-go中连接apiserver的四种client。这篇笔记就通过源码介绍一下Factory机制。

```
  // 2.设置kubeconfigflags，用于连接apiserver
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
	kubeConfigFlags.AddFlags(flags)

  // 3.利用kubeconfigflags生成了, 一个Factory f, 这个f包含了与apiserver操作的client，每个子命令都利用这个f进行后续的操作。
	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
	matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())
	cmds.PersistentFlags().AddGoFlagSet(flag.CommandLine)

	f := cmdutil.NewFactory(matchVersionKubeConfigFlags)
```

### 2 kubectl的kubeconfig

#### 2.1 kubeConfigFlags

ConfigFlags 是生成Factory的关键，先了解一下ConfigFlags

**（1）数据结构**

```
// ConfigFlags composes the set of values necessary
// for obtaining a REST client config
type ConfigFlags struct {
	CacheDir   *string
	KubeConfig *string

	// config flags
	ClusterName      *string
	AuthInfoName     *string
	Context          *string
	Namespace        *string
	APIServer        *string
	Insecure         *bool
	CertFile         *string
	KeyFile          *string
	CAFile           *string
	BearerToken      *string
	Impersonate      *string
	ImpersonateGroup *[]string
	Username         *string
	Password         *string
	Timeout          *string

	clientConfig clientcmd.ClientConfig
	lock         sync.Mutex
	// If set to true, will use persistent client config and
	// propagate the config to the places that need it, rather than
	// loading the config multiple times
	usePersistentConfig bool
}
```

<br>

通过增加打印日志，发现使用kubectl 时默认都是空的。

```
  // 设置kubeconfigflags，用于连接apiserver
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
	kubeConfigFlags.AddFlags(flags)
	
	// 增加打印日志
	klog.Errorf("zoux Namespace is: %v,APIServer is %v,AuthInfoName is %v,BearerToken is %v,CacheDir is %v,", *kubeConfigFlags.Namespace, *kubeConfigFlags.APIServer,*kubeConfigFlags.AuthInfoName, *kubeConfigFlags.BearerToken,*kubeConfigFlags.CacheDir)
	klog.Errorf("zoux CAFile is: %v,CertFile is %v,ClusterName is %v,Context is %v,Impersonate is %v,", 	*kubeConfigFlags.CAFile,*kubeConfigFlags.CertFile,*kubeConfigFlags.ClusterName,*kubeConfigFlags.Context,*kubeConfigFlags.Impersonate)
	klog.Errorf("zoux Insecure is: %v,KeyFile is %v,KubeConfig is %v,Password is %v,Timeout is %v,Username is %v", 	*kubeConfigFlags.Insecure,*kubeConfigFlags.KeyFile,*kubeConfigFlags.KubeConfig,*kubeConfigFlags.Password,*kubeConfigFlags.Timeout,*kubeConfigFlags.Username)

	
## kubectl create 的默认输出
E1105 16:47:01.248983   13836 cmd.go:470] zoux Namespace is: ,APIServer is ,AuthInfoName is ,BearerToken is ,CacheDir is /root/.kube/http-cache,
E1105 16:47:01.249066   13836 cmd.go:471] zoux CAFile is: ,CertFile is ,ClusterName is ,Context is ,Impersonate is ,
E1105 16:47:01.249070   13836 cmd.go:472] zoux Insecure is: false,KeyFile is ,KubeConfig is ,Password is ,Timeout is 0,Username is
```

**（2）继承了RESTClientGetter接口**

```
type RESTClientGetter interface {
	// ToRESTConfig returns restconfig
	ToRESTConfig() (*rest.Config, error)        
	// ToDiscoveryClient returns discovery client
	ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error)
	// ToRESTMapper returns a restmapper
	ToRESTMapper() (meta.RESTMapper, error)
	// ToRawKubeConfigLoader return kubeconfig loader as-is
	// load kubeconfig的中间函数，ToRESTConfig回调用该函数，后面不再单独分析这个函数
	ToRawKubeConfigLoader() clientcmd.ClientConfig    
}
```

<br>

##### 2.1.1 ToRESTConfig 

返回一个restconfig, 从这里可以看出来：

（1）如果使用kubectl指定了--kubeconfig, 优先使用这个

（2）否则看环境变量的KUBECONFIG 

（3）否则使用默认的config /root/.kube/config

（4）填充其他的配置

**解析文件后，它会确定当前要使用的上下文、当前指向的集群以及当前与用户关联的所有身份验证信息。**如果用户提供了额外的参数（例如 `--username`），则这些值优先，并将覆盖 kubeconfig 中指定的值。

一旦有了上述信息， Kubectl 就会填充客户端的配置，以便它能够适当地修饰 HTTP 请求：

- x509 证书使用 `tls.TLSConfig` 发送（包括 CA 证书）；
- bearer tokens 在 HTTP 请求头 Authorization 中发送；
- 用户名和密码通过 HTTP 基础认证发送；
- OpenID 认证过程是由用户事先手动处理的，产生一个像 bearer token 一样被发送的 token。

```
// ToRESTConfig implements RESTClientGetter.
// Returns a REST client configuration based on a provided path
// to a .kubeconfig file, loading rules, and config flag overrides.
// Expects the AddFlags method to have been called.
func (f *ConfigFlags) ToRESTConfig() (*rest.Config, error) {
	return f.ToRawKubeConfigLoader().ClientConfig()
}

// ToRawKubeConfigLoader binds config flag values to config overrides
// Returns an interactive clientConfig if the password flag is enabled,
// or a non-interactive clientConfig otherwise.
func (f *ConfigFlags) ToRawKubeConfigLoader() clientcmd.ClientConfig {
	if f.usePersistentConfig {
		return f.toRawKubePersistentConfigLoader()
	}
	return f.toRawKubeConfigLoader()
}

func (f *ConfigFlags) toRawKubeConfigLoader() clientcmd.ClientConfig {
  // 1.默认的加载kubeconfig规则
	loadingRules := clientcmd.NewDefaultClientConfigLoadingRules()
	// use the standard defaults for this client command
	// DEPRECATED: remove and replace with something more accurate
	loadingRules.DefaultClientConfig = &clientcmd.DefaultClientConfig

	if f.KubeConfig != nil {
		loadingRules.ExplicitPath = *f.KubeConfig
	}

  // 2.使用命令行 --kubeconfig 指定的配置覆盖，可以看出来 --kubeconfig优先级大于默认规则
	overrides := &clientcmd.ConfigOverrides{ClusterDefaults: clientcmd.ClusterDefaults}

	...

    // 3.填充其他的配置
  	// bind auth info flag values to overrides
	if f.CertFile != nil {
		overrides.AuthInfo.ClientCertificate = *f.CertFile
	}
	if f.KeyFile != nil {
		overrides.AuthInfo.ClientKey = *f.KeyFile
	}

	return clientConfig
}


// 默认的规则是，环境变量配置的KUBECONFIG  优先级大于  默认的config文件 /root/.kube/config
// NewDefaultClientConfigLoadingRules returns a ClientConfigLoadingRules object with default fields filled in.  You are not required to
// use this constructor
func NewDefaultClientConfigLoadingRules() *ClientConfigLoadingRules {
	chain := []string{}
	warnIfAllMissing := false

	envVarFiles := os.Getenv(RecommendedConfigPathEnvVar)
	if len(envVarFiles) != 0 {
		fileList := filepath.SplitList(envVarFiles)
		// prevent the same path load multiple times
		chain = append(chain, deduplicate(fileList)...)
		warnIfAllMissing = true

	} else {
		chain = append(chain, RecommendedHomeFile)
	}

	return &ClientConfigLoadingRules{
		Precedence:       chain,
		MigrationRules:   currentMigrationRules(),
		WarnIfAllMissing: warnIfAllMissing,
	}
}
 // 默认是 /HOME/.kube/kubeconfig
	oldRecommendedHomeFile := path.Join(os.Getenv("HOME"), "/.kube/.kubeconfig")
	oldRecommendedWindowsHomeFile := path.Join(os.Getenv("HOME"), RecommendedHomeDir, RecommendedFileName)
```

##### 2.1.2 ToDiscoveryClient

这个就是基于上面的kubeconfig, 返回一个DiscoveryClient。

DiscoveryClient是发现客户端，它主要用于发现Kubernetes API Server所支持的资源组、资源版本、资源信息。Kubernetes API Server支持很多资源组、资源版本、资源信息，开发者在开发过程中很难记住所有信息，此时可以通过DiscoveryClient查看所支持的资源组、资源版本、资源信息。kubectl的api-versions和api-resources命令输出也是通过DiscoveryClient实现的。另外，DiscoveryClient同样在RESTClient的基础上进行了封装。DiscoveryClient除了可以发现Kubernetes API Server所支持的资源组、资源版本、资源信息，还可以将这些信息存储到本地，用于本地缓存（Cache），以减轻对Kubernetes API Server访问的压力。在运行Kubernetes组件的机器上，缓存信息默认存储于～/.kube/cache和～/.kube/http-cache下。

DiscoveryClient的作用和用法详见clientv-go章节。

```
// ToDiscoveryClient implements RESTClientGetter.
// Expects the AddFlags method to have been called.
// Returns a CachedDiscoveryInterface using a computed RESTConfig.
func (f *ConfigFlags) ToDiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	config, err := f.ToRESTConfig()
	if err != nil {
		return nil, err
	}

	// The more groups you have, the more discovery requests you need to make.
	// given 25 groups (our groups + a few custom resources) with one-ish version each, discovery needs to make 50 requests
	// double it just so we don't end up here again for a while.  This config is only used for discovery.
	config.Burst = 100

	// retrieve a user-provided value for the "cache-dir"
	// defaulting to ~/.kube/http-cache if no user-value is given.
	httpCacheDir := defaultCacheDir
	if f.CacheDir != nil {
		httpCacheDir = *f.CacheDir
	}

	discoveryCacheDir := computeDiscoverCacheDir(filepath.Join(homedir.HomeDir(), ".kube", "cache", "discovery"), config.Host)
	return diskcached.NewCachedDiscoveryClientForConfig(config, discoveryCacheDir, httpCacheDir, time.Duration(10*time.Minute))
}
```

**kubectl的两种缓存**

K8s 用 API group 来管理 resource API。 这是一种不同于 monolithic API（所有 API 扁平化）的 API 管理方式。

具体来说，同一资源的不同版本的 API，会放到一个 group 里面。 例如 Deployment 资源的 API group 名为 apps，最新的版本是 v1。这也是为什么 我们在创建 Deployment 时，需要在 yaml 中指定 apiVersion: apps/v1 的原因。

出于性能考虑，kubectl 会 缓存这份 OpenAPI schema， 路径是 ~/.kube/cache/discovery。想查看这个 API discovery 过程，可以删除这个文件， 然后随便执行一条 kubectl 命令，并指定足够大的日志级别（例如 kubectl get ds -v 10）,这个时候kubectl就会缓存这个。

```
root@k8s-master:~/.kube/cache/discovery/localhost_8080# ls
admissionregistration.k8s.io  batch		   node.k8s.io
apiextensions.k8s.io	      certificates.k8s.io  policy
apiregistration.k8s.io	      coordination.k8s.io  rbac.authorization.k8s.io
apps			      discovery.k8s.io	   scheduling.k8s.io
authentication.k8s.io	      events.k8s.io	   servergroups.json
authorization.k8s.io	      extensions	   storage.k8s.io
autoscaling		      networking.k8s.io    v1
root@k8s-master:~/.kube/cache/discovery/localhost_8080#
root@k8s-master:~/.kube/cache/discovery/localhost_8080# pwd
/root/.kube/cache/discovery/localhost_8080


// 缓存了resource
root@k8s-master:~/.kube/cache/discovery/localhost_8080# cd v1/
root@k8s-master:~/.kube/cache/discovery/localhost_8080/v1# cat serverresources.json
{
	"kind": "APIResourceList",
	"apiVersion": "v1",
	"groupVersion": "v1",
	"resources": [{
		"name": "bindings",
		"singularName": "",
		"namespaced": true,
		"kind": "Binding",
		"verbs": ["create"]
	}, {
		"name": "componentstatuses",
		"singularName": "",
		"namespaced": false,
		"kind": "ComponentStatus",
		"verbs": ["get", "list"],
		"shortNames": ["cs"]
	}, {
		"name": "configmaps",
		"singularName": "",
		"namespaced": true,
		"kind": "ConfigMap",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["cm"],
		"storageVersionHash": "qFsyl6wFWjQ="
	}, {
		"name": "endpoints",
		"singularName": "",
		"namespaced": true,
		"kind": "Endpoints",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["ep"],
		"storageVersionHash": "fWeeMqaN/OA="
	}, {
		"name": "events",
		"singularName": "",
		"namespaced": true,
		"kind": "Event",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["ev"],
		"storageVersionHash": "r2yiGXH7wu8="
	}, {
		"name": "limitranges",
		"singularName": "",
		"namespaced": true,
		"kind": "LimitRange",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["limits"],
		"storageVersionHash": "EBKMFVe6cwo="
	}, {
		"name": "namespaces",
		"singularName": "",
		"namespaced": false,
		"kind": "Namespace",
		"verbs": ["create", "delete", "get", "list", "patch", "update", "watch"],
		"shortNames": ["ns"],
		"storageVersionHash": "Q3oi5N2YM8M="
	}, {
		"name": "namespaces/finalize",
		"singularName": "",
		"namespaced": false,
		"kind": "Namespace",
		"verbs": ["update"]
	}, {
		"name": "namespaces/status",
		"singularName": "",
		"namespaced": false,
		"kind": "Namespace",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "nodes",
		"singularName": "",
		"namespaced": false,
		"kind": "Node",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["no"],
		"storageVersionHash": "XwShjMxG9Fs="
	}, {
		"name": "nodes/proxy",
		"singularName": "",
		"namespaced": false,
		"kind": "NodeProxyOptions",
		"verbs": ["create", "delete", "get", "patch", "update"]
	}, {
		"name": "nodes/status",
		"singularName": "",
		"namespaced": false,
		"kind": "Node",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "persistentvolumeclaims",
		"singularName": "",
		"namespaced": true,
		"kind": "PersistentVolumeClaim",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["pvc"],
		"storageVersionHash": "QWTyNDq0dC4="
	}, {
		"name": "persistentvolumeclaims/status",
		"singularName": "",
		"namespaced": true,
		"kind": "PersistentVolumeClaim",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "persistentvolumes",
		"singularName": "",
		"namespaced": false,
		"kind": "PersistentVolume",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["pv"],
		"storageVersionHash": "HN/zwEC+JgM="
	}, {
		"name": "persistentvolumes/status",
		"singularName": "",
		"namespaced": false,
		"kind": "PersistentVolume",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "pods",
		"singularName": "",
		"namespaced": true,
		"kind": "Pod",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["po"],
		"categories": ["all"],
		"storageVersionHash": "xPOwRZ+Yhw8="
	}, {
		"name": "pods/attach",
		"singularName": "",
		"namespaced": true,
		"kind": "PodAttachOptions",
		"verbs": ["create", "get"]
	}, {
		"name": "pods/binding",
		"singularName": "",
		"namespaced": true,
		"kind": "Binding",
		"verbs": ["create"]
	}, {
		"name": "pods/eviction",
		"singularName": "",
		"namespaced": true,
		"group": "policy",
		"version": "v1beta1",
		"kind": "Eviction",
		"verbs": ["create"]
	}, {
		"name": "pods/exec",
		"singularName": "",
		"namespaced": true,
		"kind": "PodExecOptions",
		"verbs": ["create", "get"]
	}, {
		"name": "pods/log",
		"singularName": "",
		"namespaced": true,
		"kind": "Pod",
		"verbs": ["get"]
	}, {
		"name": "pods/portforward",
		"singularName": "",
		"namespaced": true,
		"kind": "PodPortForwardOptions",
		"verbs": ["create", "get"]
	}, {
		"name": "pods/proxy",
		"singularName": "",
		"namespaced": true,
		"kind": "PodProxyOptions",
		"verbs": ["create", "delete", "get", "patch", "update"]
	}, {
		"name": "pods/status",
		"singularName": "",
		"namespaced": true,
		"kind": "Pod",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "podtemplates",
		"singularName": "",
		"namespaced": true,
		"kind": "PodTemplate",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"storageVersionHash": "LIXB2x4IFpk="
	}, {
		"name": "replicationcontrollers",
		"singularName": "",
		"namespaced": true,
		"kind": "ReplicationController",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["rc"],
		"categories": ["all"],
		"storageVersionHash": "Jond2If31h0="
	}, {
		"name": "replicationcontrollers/scale",
		"singularName": "",
		"namespaced": true,
		"group": "autoscaling",
		"version": "v1",
		"kind": "Scale",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "replicationcontrollers/status",
		"singularName": "",
		"namespaced": true,
		"kind": "ReplicationController",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "resourcequotas",
		"singularName": "",
		"namespaced": true,
		"kind": "ResourceQuota",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["quota"],
		"storageVersionHash": "8uhSgffRX6w="
	}, {
		"name": "resourcequotas/status",
		"singularName": "",
		"namespaced": true,
		"kind": "ResourceQuota",
		"verbs": ["get", "patch", "update"]
	}, {
		"name": "secrets",
		"singularName": "",
		"namespaced": true,
		"kind": "Secret",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"storageVersionHash": "S6u1pOWzb84="
	}, {
		"name": "serviceaccounts",
		"singularName": "",
		"namespaced": true,
		"kind": "ServiceAccount",
		"verbs": ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"],
		"shortNames": ["sa"],
		"storageVersionHash": "pbx9ZvyFpBE="
	}, {
		"name": "services",
		"singularName": "",
		"namespaced": true,
		"kind": "Service",
		"verbs": ["create", "delete", "get", "list", "patch", "update", "watch"],
		"shortNames": ["svc"],
		"categories": ["all"],
		"storageVersionHash": "0/CO1lhkEBI="
	}, {
		"name": "services/proxy",
		"singularName": "",
		"namespaced": true,
		"kind": "ServiceProxyOptions",
		"verbs": ["create", "delete", "get", "patch", "update"]
	}, {
		"name": "services/status",
		"singularName": "",
		"namespaced": true,
		"kind": "Service",
		"verbs": ["get", "patch", "update"]
	}]
}
```

##### 2.1.3 ToRESTMapper

RESTMapper用于管理所有对象的信息。外部要获取的话，直接通过version，group获取到RESTMapper，然后通过kind类型可以获取到相对应的信息。

```
// ToRESTMapper returns a mapper.
func (f *ConfigFlags) ToRESTMapper() (meta.RESTMapper, error) {
   discoveryClient, err := f.ToDiscoveryClient()
   if err != nil {
      return nil, err
   }

   mapper := restmapper.NewDeferredDiscoveryRESTMapper(discoveryClient)
   expander := restmapper.NewShortcutExpander(mapper, discoveryClient)
   return expander, nil
}

func NewShortcutExpander
type ShortcutExpander struct是可以用于Kubernetes资源的RESTMapper。 把userResources、mapper、discoveryClient封装成一个ShortcutExpander结构，可以理解为就是一个简单的封装

func NewShortcutExpander(delegate meta.RESTMapper, client discovery.DiscoveryInterface) ShortcutExpander {
	return ShortcutExpander{All: userResources, RESTMapper: delegate, discoveryClient: client}
}
```

<br>

**RESTMapper**其实主要就是kindfor, resourceFor, 实现了GVK到GVR的转化。这样的好处就是，通过yaml中的apiVersion和kind就知道要创建哪种资源

这个是资源注册到apiserver时，就知道了每种资源的gvk。

```
// RESTMapper allows clients to map resources to kind, and map kind and version
// to interfaces for manipulating those objects. It is primarily intended for
// consumers of Kubernetes compatible REST APIs as defined in docs/devel/api-conventions.md.
//
// The Kubernetes API provides versioned resources and object kinds which are scoped
// to API groups. In other words, kinds and resources should not be assumed to be
// unique across groups.
//
// TODO: split into sub-interfaces
type RESTMapper interface {
	// KindFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	KindFor(resource schema.GroupVersionResource) (schema.GroupVersionKind, error)

	// KindsFor takes a partial resource and returns the list of potential kinds in priority order
	KindsFor(resource schema.GroupVersionResource) ([]schema.GroupVersionKind, error)

	// ResourceFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	ResourceFor(input schema.GroupVersionResource) (schema.GroupVersionResource, error)

	// ResourcesFor takes a partial resource and returns the list of potential resource in priority order
	ResourcesFor(input schema.GroupVersionResource) ([]schema.GroupVersionResource, error)

	// RESTMapping identifies a preferred resource mapping for the provided group kind.
	RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error)
	// RESTMappings returns all resource mappings for the provided group kind if no
	// version search is provided. Otherwise identifies a preferred resource mapping for
	// the provided version(s).
	RESTMappings(gk schema.GroupKind, versions ...string) ([]*RESTMapping, error)

	ResourceSingularizer(resource string) (singular string, err error)
}

// RESTMapping contains the information needed to deal with objects of a specific
// resource and kind in a RESTful manner.
type RESTMapping struct {
	// Resource is the GroupVersionResource (location) for this endpoint
	Resource schema.GroupVersionResource

	// GroupVersionKind is the GroupVersionKind (data format) to submit to this endpoint
	GroupVersionKind schema.GroupVersionKind

	// Scope contains the information needed to deal with REST Resources that are in a resource hierarchy
	Scope RESTScope
}
```

##### 2.1.4 举例说明discorvey和restMapper在创建删除资源时起到的作用

**通过Go代码操作K8S 资源**

下面函数实现的功能就是：

通过operating指定操作，来操作data对应的对象。

其中data可以认为是yaml序列号后的数据。

所以这里的核心就是：

（1）生成DiscoveryClient，这样才能获取group, version, kind, resource等信息

（2）根据DiscoveryClient生成RESTMapper

（3）data中有gvk的信息，有了gvk，再根据RESTMapper就能生成一个mappering对象，就能找到gvr

（4）有了gvr就能有了restful api路径，再结合data->unstructuredObj, 就可以直接发送create/delete请求了

```
package kube

import (
	"context"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/serializer/yaml"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/discovery/cached/memory"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/restmapper"
)

func DynamicK8s(operating string, data []byte) error {
	// creates the in-cluster config
	config, err := rest.InClusterConfig()
	if err != nil {
		return err
	}

	// 1. Prepare a RESTMapper to find GVR
	dc, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		return err
	}
	mapper := restmapper.NewDeferredDiscoveryRESTMapper(memory.NewMemCacheClient(dc))
	// 2. Prepare the dynamic client
	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		return err
	}
	// 3. Decode YAML manifest into unstructured.Unstructured
	runtimeObject, gvk, err :=
		yaml.
			NewDecodingSerializer(unstructured.UnstructuredJSONScheme).
			Decode(data, nil, nil)
	if err != nil {
		return err
	}
	unstructuredObj := runtimeObject.(*unstructured.Unstructured)
	// 4. Find GVR
	mapping, err := mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
	if err != nil {
		return err
	}
	// 5. Obtain REST interface for the GVR
	var resourceREST dynamic.ResourceInterface
	if mapping.Scope.Name() == meta.RESTScopeNameNamespace {
		// namespaced resources should specify the namespace
		resourceREST = dynamicClient.Resource(mapping.Resource).Namespace(unstructuredObj.GetNamespace())
	} else {
		// for cluster-wide resources
		resourceREST = dynamicClient.Resource(mapping.Resource)
	}
	switch operating {
	case "create":
		_, err = resourceREST.Create(context.TODO(), unstructuredObj, metav1.CreateOptions{})
	case "delete":
		deletePolicy := metav1.DeletePropagationForeground
		deleteOptions := metav1.DeleteOptions{
			PropagationPolicy: &deletePolicy,
		}
		err = resourceREST.Delete(context.TODO(), unstructuredObj.GetName(), deleteOptions)
	}

	return err
}
```

### 3.总结

（1）了解到kubectl 加载kubectl的配置的优先级

（2）了解了kubectl操作 yaml中对象的大致原理

