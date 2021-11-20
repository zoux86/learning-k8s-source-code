Table of Contents
=================

  * [0. 四种客户端简介](#0-四种客户端简介)
  * [1.discovery](#1discovery)
     * [1.1 ServerGroups](#11-servergroups)
     * [1.2 ServerGroupsAndResources](#12-servergroupsandresources)
     * [1.3 缓存](#13-缓存)
     * [1.4 实例展示](#14-实例展示)
  * [2.restClient客户端](#2restclient客户端)
  * [3.clientSet客户端](#3clientset客户端)
     * [2.1 Clientset的定义](#21-clientset的定义)
  * [4.DynamicClient客户端](#4dynamicclient客户端)

### 0. 四种客户端简介

client-go的客户端对象有4个，作用各有不同：

- RESTClient： 是对HTTP Request进行了封装，实现了RESTful风格的API。其他客户端都是在RESTClient基础上的实现。可与用于k8s内置资源和CRD资源
- ClientSet:是对k8s内置资源对象的客户端的集合，默认情况下，不能操作CRD资源，但是通过client-gen代码生成的话，也是可以操作CRD资源的。
- DynamicClient:不仅能对K8S内置资源进行处理，还可以对CRD资源进行处理，不需要client-gen生成代码即可实现。
- DiscoveryClient：用于发现kube-apiserver所支持的资源组、资源版本、资源信息（即Group、Version、Resources）。

![client](../images/client.png)


RESTClient是最基础的客户端。RESTClient对HTTP Request进行了封装，实现了RESTful风格的API。ClientSet、DynamicClient及DiscoveryClient客户端都是基于RESTClient实现的。



ClientSet在RESTClient的基础上封装了对Resource和Version的管理方法。每一个Resource可以理解为一个客户端，而ClientSet则是多个客户端的集合，每一个Resource和Version都以函数的方式暴露给开发者。ClientSet只能够处理Kubernetes内置资源，它是通过client-gen代码生成器自动生成的。



DynamicClient与ClientSet最大的不同之处是，ClientSet仅能访问Kubernetes自带的资源（即Client集合内的资源），不能直接访问CRD自定义资源。DynamicClient能够处理Kubernetes中的所有资源对象，包括Kubernetes内置资源与CRD自定义资源。

DiscoveryClient发现客户端，用于发现kube-apiserver所支持的资源组、资源版本、资源信息（即Group、Versions、Resources）。以上4种客户端：RESTClient、ClientSet、DynamicClient、DiscoveryClient都可以通过kubeconfig配置信息连接到指定的KubernetesAPI Server。

**总结下**：RESTCLient、ClientSet和DynamicClient都可以对K8S内置资源和CRD资源进行操作。只是clientSet需要生成代码才能操作CRD资源。

而clientSet 和dynamicClient不同在于，dynamicClient可以操作任意的对象，clientset初始化是只能指定一种对象操作。

<br>


### 1.discovery

discovery包的主要作用就是提供当前k8s集群支持哪些资源以及版本信息。

Kubernetes API Server暴露出/api和/apis接口。DiscoveryClient通过RESTClient分别请求/api和/apis接口，从而获取Kubernetes API Server所支持的资源组、资源版信息。这个是通过ServerGroups函数实现的

有了group, version信息后，但是还是不够，因为还没有具体到资源。

ServerGroupsAndResources 就获得了所有的资源信息（所有的GVR资源信息），而在Resource资源的定义中，会定义好该资源支持哪些操作：list, delelte ,get等等。

所以kubectl中就使用discovery做了资源的校验。获取所有资源的版本信息，以及支持的操作。就可以判断客户端当前操作是否合理。

#### 1.1 ServerGroups

staging/src/k8s.io/client-go/discovery/discovery_client.go

```
// ServerGroups returns the supported groups, with information like supported versions and the
// preferred version.
func (d *DiscoveryClient) ServerGroups() (apiGroupList *metav1.APIGroupList, err error) {
	// Get the groupVersions exposed at /api
	v := &metav1.APIVersions{}
	// 先请求 https://192.168.0.4:6443/api，获得core下面的组
	err = d.restClient.Get().AbsPath(d.LegacyPrefix).Do().Into(v)
	apiGroup := metav1.APIGroup{}
	if err == nil && len(v.Versions) != 0 {
		apiGroup = apiVersionsToAPIGroup(v)
	}
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}

	// Get the groupVersions exposed at /apis
	apiGroupList = &metav1.APIGroupList{}
	// 再请求https://192.168.0.4:6443/api ，获得其他的组 
	err = d.restClient.Get().AbsPath("/apis").Do().Into(apiGroupList)
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}
	// to be compatible with a v1.0 server, if it's a 403 or 404, ignore and return whatever we got from /api
	if err != nil && (errors.IsNotFound(err) || errors.IsForbidden(err)) {
		apiGroupList = &metav1.APIGroupList{}
	}

	// prepend the group retrieved from /api to the list if not empty
	if len(v.Versions) != 0 {
		apiGroupList.Groups = append([]metav1.APIGroup{apiGroup}, apiGroupList.Groups...)
	}
	return apiGroupList, nil
}
```

<br>

apiGroupList 就是获取所有的 组，每个组所有的version信息

```
// APIGroupList is a list of APIGroup, to allow clients to discover the API at
// /apis.
type APIGroupList struct {
	TypeMeta `json:",inline"`
	// groups is a list of APIGroup.
	Groups []APIGroup `json:"groups" protobuf:"bytes,1,rep,name=groups"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// APIGroup contains the name, the supported versions, and the preferred version
// of a group.
type APIGroup struct {
	TypeMeta `json:",inline"`
	// name is the name of the group.
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
	// versions are the versions supported in this group.
	Versions []GroupVersionForDiscovery `json:"versions" protobuf:"bytes,2,rep,name=versions"`
	// preferredVersion is the version preferred by the API server, which
	// probably is the storage version.
	// +optional
	PreferredVersion GroupVersionForDiscovery `json:"preferredVersion,omitempty" protobuf:"bytes,3,opt,name=preferredVersion"`
	// a map of client CIDR to server address that is serving this group.
	// This is to help clients reach servers in the most network-efficient way possible.
	// Clients can use the appropriate server address as per the CIDR that they match.
	// In case of multiple matches, clients should use the longest matching CIDR.
	// The server returns only those CIDRs that it thinks that the client can match.
	// For example: the master will return an internal IP CIDR only, if the client reaches the server using an internal IP.
	// Server looks at X-Forwarded-For header or X-Real-Ip header or request.RemoteAddr (in that order) to get the client IP.
	// +optional
	ServerAddressByClientCIDRs []ServerAddressByClientCIDR `json:"serverAddressByClientCIDRs,omitempty" protobuf:"bytes,4,rep,name=serverAddressByClientCIDRs"`
}
```

直接访问 /api  /apis就能或者 gruop, version信息。

```
root@k8s-master:~# curl https://192.168.0.4:6443/api --cert /opt/kubernetes/ssl/server.pem --key /opt/kubernetes/ssl/server-key.pem --cacert /opt/kubernetes/ssl/ca.pem{
  "kind": "APIVersions",
  "versions": [  //这里省略了 gruop=core，其实core也是我们后面的称号，可以认为没有gruop的概念。
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.0.4:6443"
    }
  ]
}

root@k8s-master:~# curl https://192.168.0.4:6443/apis --cert /opt/kubernetes/ssl/server.pem --key /opt/kubernetes/ssl/server-key.pem --cacert /opt/kubernetes/ssl/ca.pem
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        },
        {
          "groupVersion": "apiregistration.k8s.io/v1beta1",
          "version": "v1beta1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
    },
    ...
}
```

#### 1.2 ServerGroupsAndResources 

```
func ServerGroupsAndResources(d DiscoveryInterface) ([]*metav1.APIGroup, []*metav1.APIResourceList, error) {
	
	...
	groupVersionResources, failedGroups := fetchGroupVersionResources(d, sgs)
    ...

}


// fetchServerResourcesForGroupVersions uses the discovery client to fetch the resources for the specified groups in parallel.
func fetchGroupVersionResources(d DiscoveryInterface, apiGroups *metav1.APIGroupList) (map[schema.GroupVersion]*metav1.APIResourceList, map[schema.GroupVersion]error) {

	for _, apiGroup := range apiGroups.Groups {
		for _, version := range apiGroup.Versions {

				apiResourceList, err := d.ServerResourcesForGroupVersion(groupVersion.String())

			
}


// ServerResourcesForGroupVersion returns the supported resources for a group and version.
func (d *DiscoveryClient) ServerResourcesForGroupVersion(groupVersion string) (resources *metav1.APIResourceList, err error) {
	url := url.URL{}
	if len(groupVersion) == 0 {
		return nil, fmt.Errorf("groupVersion shouldn't be empty")
	}
	// 如果是core v1，直接访问 curl https://192.168.0.4:6443/api/v1， 获得所有的资源
	if len(d.LegacyPrefix) > 0 && groupVersion == "v1" {
		url.Path = d.LegacyPrefix + "/" + groupVersion
	} else {
		url.Path = "/apis/" + groupVersion
	}
	resources = &metav1.APIResourceList{
		GroupVersion: groupVersion,
	}
	err = d.restClient.Get().AbsPath(url.String()).Do().Into(resources)
	if err != nil {
		// ignore 403 or 404 error to be compatible with an v1.0 server.
		if groupVersion == "v1" && (errors.IsNotFound(err) || errors.IsForbidden(err)) {
			return resources, nil
		}
		return nil, err
	}
	return resources, nil
}
```

实践：

```
root@k8s-master:~# curl https://192.168.0.4:6443/api/v1 --cert /opt/kubernetes/ssl/server.pem --key /opt/kubernetes/ssl/server-key.pem --cacert /opt/kubernetes/ssl/ca.pem
{  //省略了很多输出
  "kind": "APIResourceList",
  "groupVersion": "v1",
  "resources": [
    {
      "name": "bindings",
      "singularName": "",
      "namespaced": true,
      "kind": "Binding",
      "verbs": [
        "create"
      ]
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "Pod",
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
        "po"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "xPOwRZ+Yhw8="
 }
```

#### 1.3 缓存

DiscoveryClient可以将资源相关信息存储于本地，默认存储位置为～/.kube/cache和～/.kube/http-cache。缓存可以减轻client-go对KubernetesAPI Server的访问压力。默认每10分钟与Kubernetes API Server同步一次，同步周期较长，因为资源组、源版本、资源信息一般很少变动。本地缓存的DiscoveryClient如图5-4所示。DiscoveryClient第一次获取资源组、资源版本、资源信息时，首先会查询本地缓存，如果数据不存在（没有命中）则请求Kubernetes API Server接口（回源），Cache将Kubernetes API Server响应的数据存储在本地一份并返回给DiscoveryClient。当下一次DiscoveryClient再次获取资源信息时，会将数据直接从本地缓存返回（命中）给DiscoveryClient。本地缓存的默认存储周期为10分钟。代码示例如下：

staging/src/k8s.io/client-go/discovery/cached/disk/cached_discovery.go

```
func (d *CachedDiscoveryClient) getCachedFile(filename string) ([]byte, error) {
	// after invalidation ignore cache files not created by this process
	d.mutex.Lock()
	_, ourFile := d.ourFiles[filename]
	if d.invalidated && !ourFile {
		d.mutex.Unlock()
		return nil, errors.New("cache invalidated")
	}
	d.mutex.Unlock()

	file, err := os.Open(filename)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	fileInfo, err := file.Stat()
	if err != nil {
		return nil, err
	}

	if time.Now().After(fileInfo.ModTime().Add(d.ttl)) {
		return nil, errors.New("cache expired")
	}

	// the cache is present and its valid.  Try to read and use it.
	cachedBytes, err := ioutil.ReadAll(file)
	if err != nil {
		return nil, err
	}

	d.mutex.Lock()
	defer d.mutex.Unlock()
	d.fresh = d.fresh && ourFile

	return cachedBytes, nil
}
```

#### 1.4 实例展示

```
package main

import (
    "fmt"

    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/discovery"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载kubeconfig文件，生成config对象
    config, err := clientcmd.BuildConfigFromFlags("", "D:\\coding\\config")
    if err != nil {
        panic(err)
    }

    // discovery.NewDiscoveryClientForConfigg函数通过config实例化discoveryClient对象
    discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
    if err != nil {
        panic(err)
    }

    // discoveryClient.ServerGroupsAndResources 返回API Server所支持的资源组、资源版本、资源信息
    _, APIResourceList, err := discoveryClient.ServerGroupsAndResources()
    if err != nil {
        panic(err)
    }

    // 输出所有资源信息
    for _, list := range APIResourceList {
        gv, err := schema.ParseGroupVersion(list.GroupVersion)
        if err != nil {
            panic(err)
        }

        for _, resource := range list.APIResources {
            fmt.Printf("NAME: %v, GROUP: %v, VERSION: %v \n", resource.Name, gv.Group, gv.Version)
        }
    }
}


// 测试
 go run .\discoveryClient-example.go
NAME: bindings, GROUP: , VERSION: v1 
NAME: componentstatuses, GROUP: , VERSION: v1 
NAME: configmaps, GROUP: , VERSION: v1
NAME: endpoints, GROUP: , VERSION: v1
NAME: events, GROUP: , VERSION: v1
NAME: limitranges, GROUP: , VERSION: v1
NAME: namespaces, GROUP: , VERSION: v1
NAME: namespaces/finalize, GROUP: , VERSION: v1
NAME: namespaces/status, GROUP: , VERSION: v1
NAME: nodes, GROUP: , VERSION: v1
NAME: nodes/proxy, GROUP: , VERSION: v1
NAME: nodes/status, GROUP: , VERSION: v1
NAME: persistentvolumeclaims, GROUP: , VERSION: v1
NAME: persistentvolumeclaims/status, GROUP: , VERSION: v1
NAME: persistentvolumes, GROUP: , VERSION: v1
NAME: persistentvolumes/status, GROUP: , VERSION: v1
NAME: pods, GROUP: , VERSION: v1
NAME: pods/attach, GROUP: , VERSION: v1
NAME: pods/binding, GROUP: , VERSION: v1
NAME: pods/eviction, GROUP: , VERSION: v1
NAME: pods/exec, GROUP: , VERSION: v1
NAME: pods/log, GROUP: , VERSION: v1
NAME: pods/portforward, GROUP: , VERSION: v1
NAME: pods/proxy, GROUP: , VERSION: v1
NAME: pods/status, GROUP: , VERSION: v1
NAME: podtemplates, GROUP: , VERSION: v1
NAME: replicationcontrollers, GROUP: , VERSION: v1
NAME: replicationcontrollers/scale, GROUP: , VERSION: v1
NAME: replicationcontrollers/status, GROUP: , VERSION: v1
NAME: resourcequotas, GROUP: , VERSION: v1
NAME: resourcequotas/status, GROUP: , VERSION: v1
NAME: secrets, GROUP: , VERSION: v1
NAME: serviceaccounts, GROUP: , VERSION: v1
NAME: services, GROUP: , VERSION: v1
NAME: services/proxy, GROUP: , VERSION: v1
NAME: services/status, GROUP: , VERSION: v1
NAME: apiservices, GROUP: apiregistration.k8s.io, VERSION: v1
NAME: apiservices/status, GROUP: apiregistration.k8s.io, VERSION: v1
NAME: apiservices, GROUP: apiregistration.k8s.io, VERSION: v1beta1 
NAME: apiservices/status, GROUP: apiregistration.k8s.io, VERSION: v1beta1
NAME: ingresses, GROUP: extensions, VERSION: v1beta1
NAME: ingresses/status, GROUP: extensions, VERSION: v1beta1
NAME: controllerrevisions, GROUP: apps, VERSION: v1
NAME: daemonsets, GROUP: apps, VERSION: v1
NAME: daemonsets/status, GROUP: apps, VERSION: v1
NAME: deployments, GROUP: apps, VERSION: v1
NAME: deployments/scale, GROUP: apps, VERSION: v1
NAME: deployments/status, GROUP: apps, VERSION: v1
NAME: replicasets, GROUP: apps, VERSION: v1
NAME: replicasets/scale, GROUP: apps, VERSION: v1
NAME: replicasets/status, GROUP: apps, VERSION: v1
NAME: statefulsets, GROUP: apps, VERSION: v1
NAME: statefulsets/scale, GROUP: apps, VERSION: v1
NAME: statefulsets/status, GROUP: apps, VERSION: v1
NAME: events, GROUP: events.k8s.io, VERSION: v1beta1
NAME: tokenreviews, GROUP: authentication.k8s.io, VERSION: v1
NAME: tokenreviews, GROUP: authentication.k8s.io, VERSION: v1beta1
NAME: localsubjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1
NAME: selfsubjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1
NAME: selfsubjectrulesreviews, GROUP: authorization.k8s.io, VERSION: v1
NAME: subjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1
NAME: localsubjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1beta1
NAME: selfsubjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1beta1
NAME: selfsubjectrulesreviews, GROUP: authorization.k8s.io, VERSION: v1beta1
NAME: subjectacce***eviews, GROUP: authorization.k8s.io, VERSION: v1beta1
NAME: horizontalpodautoscalers, GROUP: autoscaling, VERSION: v1
NAME: horizontalpodautoscalers/status, GROUP: autoscaling, VERSION: v1
NAME: horizontalpodautoscalers, GROUP: autoscaling, VERSION: v2beta1
NAME: horizontalpodautoscalers/status, GROUP: autoscaling, VERSION: v2beta1
NAME: horizontalpodautoscalers, GROUP: autoscaling, VERSION: v2beta2
NAME: horizontalpodautoscalers/status, GROUP: autoscaling, VERSION: v2beta2
NAME: jobs, GROUP: batch, VERSION: v1
NAME: jobs/status, GROUP: batch, VERSION: v1
NAME: cronjobs, GROUP: batch, VERSION: v1beta1
NAME: cronjobs/status, GROUP: batch, VERSION: v1beta1
NAME: certificatesigningrequests, GROUP: certificates.k8s.io, VERSION: v1beta1
NAME: certificatesigningrequests/approval, GROUP: certificates.k8s.io, VERSION: v1beta1
NAME: certificatesigningrequests/status, GROUP: certificates.k8s.io, VERSION: v1beta1
NAME: networkpolicies, GROUP: networking.k8s.io, VERSION: v1
NAME: ingressclasses, GROUP: networking.k8s.io, VERSION: v1beta1
NAME: ingresses, GROUP: networking.k8s.io, VERSION: v1beta1
NAME: ingresses/status, GROUP: networking.k8s.io, VERSION: v1beta1
NAME: poddisruptionbudgets, GROUP: policy, VERSION: v1beta1
NAME: poddisruptionbudgets/status, GROUP: policy, VERSION: v1beta1
NAME: podsecuritypolicies, GROUP: policy, VERSION: v1beta1
NAME: clusterrolebindings, GROUP: rbac.authorization.k8s.io, VERSION: v1
NAME: clusterroles, GROUP: rbac.authorization.k8s.io, VERSION: v1
NAME: rolebindings, GROUP: rbac.authorization.k8s.io, VERSION: v1
NAME: roles, GROUP: rbac.authorization.k8s.io, VERSION: v1
NAME: clusterrolebindings, GROUP: rbac.authorization.k8s.io, VERSION: v1beta1
NAME: clusterroles, GROUP: rbac.authorization.k8s.io, VERSION: v1beta1
NAME: rolebindings, GROUP: rbac.authorization.k8s.io, VERSION: v1beta1
NAME: roles, GROUP: rbac.authorization.k8s.io, VERSION: v1beta1
NAME: csidrivers, GROUP: storage.k8s.io, VERSION: v1
NAME: csinodes, GROUP: storage.k8s.io, VERSION: v1
NAME: storageclasses, GROUP: storage.k8s.io, VERSION: v1
NAME: volumeattachments, GROUP: storage.k8s.io, VERSION: v1
NAME: volumeattachments/status, GROUP: storage.k8s.io, VERSION: v1 
NAME: csidrivers, GROUP: storage.k8s.io, VERSION: v1beta1
NAME: csinodes, GROUP: storage.k8s.io, VERSION: v1beta1
NAME: storageclasses, GROUP: storage.k8s.io, VERSION: v1beta1
NAME: volumeattachments, GROUP: storage.k8s.io, VERSION: v1beta1
NAME: mutatingwebhookconfigurations, GROUP: admissionregistration.k8s.io, VERSION: v1
NAME: validatingwebhookconfigurations, GROUP: admissionregistration.k8s.io, VERSION: v1
NAME: mutatingwebhookconfigurations, GROUP: admissionregistration.k8s.io, VERSION: v1beta1
NAME: validatingwebhookconfigurations, GROUP: admissionregistration.k8s.io, VERSION: v1beta1
NAME: customresourcedefinitions, GROUP: apiextensions.k8s.io, VERSION: v1
NAME: customresourcedefinitions/status, GROUP: apiextensions.k8s.io, VERSION: v1
NAME: customresourcedefinitions, GROUP: apiextensions.k8s.io, VERSION: v1beta1
NAME: customresourcedefinitions/status, GROUP: apiextensions.k8s.io, VERSION: v1beta1
NAME: priorityclasses, GROUP: scheduling.k8s.io, VERSION: v1
NAME: priorityclasses, GROUP: scheduling.k8s.io, VERSION: v1beta1
NAME: leases, GROUP: coordination.k8s.io, VERSION: v1
NAME: leases, GROUP: coordination.k8s.io, VERSION: v1beta1
NAME: runtimeclasses, GROUP: node.k8s.io, VERSION: v1beta1
NAME: endpointslices, GROUP: discovery.k8s.io, VERSION: v1beta1
```


### 2.restClient客户端

rest.RESTClientFor函数通过kubeconfig配置信息实例化RESTClient对象，RESTClient对象构建HTTP请求参数，例如Get函数设置请求方法为get操作，它还支持Post、Put、Delete、Patch，list, watch等请求方法。

rest由于是三个client的父类，这里介绍详细一点。

```
rest目录如下， 添加了每个文件的功能。代码就不一一展示

│  BUILD
│  client.go            初始化restClient,从初始化的过程中，可以看出来使用了令牌桶限速。同时实现了Get，put等方法，就是设置http请求的verb字段

│  client_test.go
│  config.go            处理kubeconfig的一些函数
│  config_test.go
│  OWNERS
│  plugin.go            插件，从代码中看，目前只有auth插件
│  plugin_test.go
│  request.go           处理发送http请求相关的函数, get, list等等都在这
│  request_test.go 
│  transport.go         还是处理http请求相关的函数，http中的transport
│  urlbackoff.go        处理backoff
│  urlbackoff_test.go
│  url_utils.go         处理url,定义了defaultUrl
│  url_utils_test.go   
│  zz_generated.deepcopy.go
└─watch
        BUILD
        decoder.go          对watch事件对象解码
        decoder_test.go
        encoder.go          对watch事件对象编码
        encoder_test.go     
```

restClient并没有直接调用create,get等资源的接口。它需要自己确定url，访问资源。如下的例子：

```
package main

import (
    "fmt"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载kubeconfig文件，生成config对象
    config, err := clientcmd.BuildConfigFromFlags("", "D:\\coding\\config")

    if err != nil {
        panic(err)
    }
    // 配置API路径和请求的资源组/资源版本信息
    config.APIPath = "api"
    config.GroupVersion = &corev1.SchemeGroupVersion
    config.NegotiatedSerializer = scheme.Codecs

    // 通过rest.RESTClientFor()生成RESTClient对象。 RESTClientFor通过令牌桶算法，有限制的说法。
    restClient, err := rest.RESTClientFor(config)
    if err != nil {
        panic(err)
    }

    // 通过RESTClient构建请求参数，查询default空间下所有pod资源
    result := &corev1.PodList{}
    err = restClient.Get().
        Namespace("default").
        Resource("pods").
        VersionedParams(&metav1.ListOptions{Limit: 500}, scheme.ParameterCodec).
        Do().
        Into(result)

    if err != nil {
        panic(err)
    }

    for _, d := range result.Items {
        fmt.Printf("NAMESPACE:%v \t NAME: %v \t STATUS: %v\n", d.Namespace, d.Name, d.Status.Phase)
    }
}

// 测试
go run .\restClient-example.go
NAMESPACE:default        NAME: nginx-deployment-6b474476c4-lpld7         STATUS: Running
NAMESPACE:default        NAME: nginx-deployment-6b474476c4-t6xl4         STATUS: Running
```

以这个例子为例：一般的使用就是 restClient.Get().XX.XX.Do().Into(result)。最终会回到Do 和 into函数

前面的XX例如VersionedParams函数将一些查询选项（如limit、TimeoutSeconds等）添加到请求参数中。通过Do函数执行该请求，并且获得结构。

inTO就是进行decode，然后赋值给result对象。

```
// Do formats and executes the request. Returns a Result object for easy response
// processing.
//
// Error type:
//  * If the server responds with a status: *errors.StatusError or *errors.UnexpectedObjectError
//  * http.Client.Do errors are returned directly.
func (r *Request) Do() Result {
	if err := r.tryThrottle(); err != nil {
		return Result{err: err}
	}

	var result Result
	err := r.request(func(req *http.Request, resp *http.Response) {
		result = r.transformResponse(resp, req)
	})
	if err != nil {
		return Result{err: err}
	}
	return result
}


// Into stores the result into obj, if possible. If obj is nil it is ignored.
// If the returned object is of type Status and has .Status != StatusSuccess, the
// additional information in Status will be used to enrich the error.
func (r Result) Into(obj runtime.Object) error {
	if r.err != nil {
		// Check whether the result has a Status object in the body and prefer that.
		return r.Error()
	}
	if r.decoder == nil {
		return fmt.Errorf("serializer for %s doesn't exist", r.contentType)
	}
	if len(r.body) == 0 {
		return fmt.Errorf("0-length response with status code: %d and content type: %s",
			r.statusCode, r.contentType)
	}

	out, _, err := r.decoder.Decode(r.body, nil, obj)
	if err != nil || out == obj {
		return err
	}
	// if a different object is returned, see if it is Status and avoid double decoding
	// the object.
	switch t := out.(type) {
	case *metav1.Status:
		// any status besides StatusSuccess is considered an error.
		if t.Status != metav1.StatusSuccess {
			return errors.FromObject(t)
		}
	}
	return nil
}
```

<br>

### 3.clientSet客户端

RESTClient是一种最基础的客户端，使用时需要指定Resource和Version等信息，编写代码时需要提前知道Resource所在的Group和对应的Version信息。相比RESTClient，ClientSet使用起来更加便捷，一般情况下，开发者对Kubernetes进行二次开发时通常使用ClientSet。

ClientSet对应的是   client-go/kubernetes 这个目录

这个目录结构核心目录和文件如下：

```
│  BUILD
│  clientset.go      定义和初始化clientset相关函数    
│  typed目录          里面定义了所有内置资源的get,list等等
│  scheme            
```

#### 2.1 Clientset的定义

```
// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
	*discovery.DiscoveryClient
	admissionregistrationV1      *admissionregistrationv1.AdmissionregistrationV1Client
	admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
	appsV1                       *appsv1.AppsV1Client
    ...
	coreV1                       *corev1.CoreV1Client
	...
}

CoreV1Client 其实就是一个rest client接口
// CoreV1Client is used to interact with features provided by the  group.
type CoreV1Client struct {
	restClient rest.Interface
}

只不过封装了很多额外的函数
func (c *CoreV1Client) Pods(namespace string) PodInterface {
	return newPods(c, namespace)
}
```

<br>

staging/src/k8s.io/client-go/kubernetes/typed/core/v1/pod.go

到typed目录下具体的一个资源对象文件看看, 这里以get为例。可以看出来其实就是封装了restClient的写法而已。

```
// Get takes name of the pod, and returns the corresponding pod object, and an error if there is any.
func (c *pods) Get(name string, options metav1.GetOptions) (result *v1.Pod, err error) {
	result = &v1.Pod{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do().
		Into(result)
	return
}
```

这样的好处就是每次使用的时候，简化一点。

如下的例子可见，clientSet通过 NewForConfig 实现一个客户端。用起来也方便很多。

```
package main

import (
    "fmt"

    apiv1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载kubeconfig文件，生成config对象
    config, err := clientcmd.BuildConfigFromFlags("", "D:\\coding\\config")
    if err != nil {
        panic(err)
    }

    // kubernetes.NewForConfig通过config实例化ClientSet对象
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err)
    }

    //请求core核心资源组v1资源版本下的Pods资源对象
    podClient := clientset.CoreV1().Pods(apiv1.NamespaceDefault)
    // 设置选项
    list, err := podClient.List(metav1.ListOptions{Limit: 500})
    if err != nil {
        panic(err)
    }

    for _, d := range list.Items {
        fmt.Printf("NAMESPACE: %v \t NAME:%v \t STATUS: %+v\n", d.Namespace, d.Name, d.Status.Phase)
    }
}

// 测试
go run .\clientSet-example.go

NAMESPACE: default       NAME:nginx-deployment-6b474476c4-lpld7          STATUS: Running
NAMESPACE: default       NAME:nginx-deployment-6b474476c4-t6xl4          STATUS: Running
```

<br>

### 4.DynamicClient客户端

DynamicClient是一种动态客户端，它可以对任意Kubernetes资源进行RESTful操作，包括CRD自定义资源。DynamicClient与ClientSet操作类似，同样封装了RESTClient，同样提供了Create、Update、Delete、Get、List、Watch、Patch等方法。DynamicClient与ClientSet最大的不同之处是，ClientSet仅能访问Kubernetes自带的资源（即客户端集合内的资源），不能直接访问CRD自定义资源。ClientSet需要预先实现每种Resource和Version的操作，其内部的数据都是结构化数据（即已知数据结构）。而DynamicClient内部实现了Unstructured，用于处理非结构化数据结构（即无法提前预知数据结构），这也是DynamicClient能够处理CRD自定义资源的关键。

dynamic目录结构如下：

```
│  BUILD
│  client_test.go
│  interface.go
│  scheme.go
│  simple.go                感觉叫dynamicClient.go更好，就是定义和初始化dynamic文件。然后定义update,get函数的等实现
│
├─dynamicinformer
│      BUILD
│      informer.go          定义dynamicinformer类型的Informer，其他内置资源在informer目录中都定义了
│      informer_test.go
│      interface.go
│
├─dynamiclister
│      BUILD
│      interface.go
│      lister.go            定义dynamicinformer类型的lister，其他内置资源在lister目录中都定义了
│      lister_test.go
│      shim.go
│
└─fake
        BUILD
        simple.go
        simple_test.go
```

staging/src/k8s.io/client-go/dynamic/simple.go

以Get为例，看看是如何实现的。其实和clientset是一样的。

```
func (c *dynamicResourceClient) Get(name string, opts metav1.GetOptions, subresources ...string) (*unstructured.Unstructured, error) {
   if len(name) == 0 {
      return nil, fmt.Errorf("name is required")
   }
   // 拼凑好rest url
   result := c.client.client.Get().AbsPath(append(c.makeURLSegments(name), subresources...)...).SpecificallyVersionedParams(&opts, dynamicParameterCodec, versionV1).Do()
   if err := result.Error(); err != nil {
      return nil, err
   }
   retBytes, err := result.Raw()
   if err != nil {
      return nil, err
   }
   // 都是使用unstructured.Unstructured接收返回的结果
   uncastObj, err := runtime.Decode(unstructured.UnstructuredJSONScheme, retBytes)
   if err != nil {
      return nil, err
   }
   return uncastObj.(*unstructured.Unstructured), nil
}
```

informer 和 list再下一节单独介绍

**注意：**

* DynamicClient获得的数据都是一个object类型。存的时候是 unstructured

* DynamicClient不是类型安全的，因此在访问CRD自定义资源时需要特别注意。例如，在操作指针不当的情况下可能会导致程序崩溃。

* DynamicClient如果要使用informer，必须是NewFilteredDynamicSharedInformerFactory

  ```ruby
  	f := dynamicinformer.NewFilteredDynamicSharedInformerFactory(dc, 0, v1.NamespaceAll, nil)
  ```

```
package main

import (
    "fmt"

    apiv1 "k8s.io/api/core/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    // 加载kubeconfig文件，生成config对象
    config, err := clientcmd.BuildConfigFromFlags("", "D:\\coding\\config")
    if err != nil {
        panic(err)
    }

    // dynamic.NewForConfig函数通过config实例化dynamicClient对象
    dynamicClient, err := dynamic.NewForConfig(config)
    if err != nil {
        panic(err)
    }

    // 通过schema.GroupVersionResource设置请求的资源版本和资源组，设置命名空间和请求参数,得到unstructured.UnstructuredList指针类型的PodList
    gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
    unstructObj, err := dynamicClient.Resource(gvr).Namespace(apiv1.NamespaceDefault).List(metav1.ListOptions{Limit: 500})
    if err != nil {
        panic(err)
    }

    // 通过runtime.DefaultUnstructuredConverter函数将unstructured.UnstructuredList转为PodList类型
    podList := &corev1.PodList{}
    err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj.UnstructuredContent(), podList)
    if err != nil {
        panic(err)
    }

    for _, d := range podList.Items {
        fmt.Printf("NAMESPACE: %v NAME:%v \t STATUS: %+v\n", d.Namespace, d.Name, d.Status.Phase)
    }
}

// 测试
go run .\dynamicClient-example.go
NAMESPACE: default NAME:nginx-deployment-6b474476c4-lpld7        STATUS: Running
NAMESPACE: default NAME:nginx-deployment-6b474476c4-t6xl4        STATUS: Running
```

<br>