Table of Contents
=================

  * [1. custom-metrics-apiserver简介](#1-custom-metrics-apiserver简介)
  * [2. 定制自己的metric server](#2-定制自己的metric-server)
     * [2.1 代码部署和编译](#21-代码部署和编译)
     * [2.2 创建 Sv and APIService](#22-创建-sv-and-apiservice)
     * [2.3 system:anonymous授权](#23-systemanonymous授权)
  * [3. 创建hpa验证是否成功](#3-创建hpa验证是否成功)
  * [4. 追踪整个过程](#4-追踪整个过程)
  * [5. 总结](#5-总结)

**本章重点：** 如何基于 custom-metrics-apiserver 项目，打造自己的 metric server

### 1. custom-metrics-apiserver简介

 项目地址： https://github.com/kubernetes-sigs/custom-metrics-apiserver/tree/master 

**自定义metric server，具体来说需要做以下几个事情：**

（1）实现  custom-metrics-apiserver 的 三个接口，如下：

```
type CustomMetricsProvider interface {
    // 定义metric。 例如 pod_cpu_used_1m
    ListAllMetrics() []CustomMetricInfo
	
	// 如何根据 metric的信息，得到具体的值
    GetMetricByName(name types.NamespacedName, info CustomMetricInfo) (*custom_metrics.MetricValue, error)
    
    // 如何根据 metric selector的信息，得到具体的值
    GetMetricBySelector(namespace string, selector labels.Selector, info CustomMetricInfo) (*custom_metrics.MetricValueList, error)
}
```

GetMetricBySelectorm, GetMetricByName 在reststorage.go被使用。

https://github.com/kubernetes-sigs/custom-metrics-apiserver/blob/master/pkg/registry/custom_metrics/reststorage.go

restful接口在installer.go中被定义。

https://github.com/kubernetes-sigs/custom-metrics-apiserver/blob/master/pkg/apiserver/installer/installer.go

**总的来说，可以认为**

（1）基于custom-metrics-apiserver这个项目，你只要实现上述三个接口就行。其他的事情这个包在你new provider的时候都自动实现了。

（2）ListAllMetrics 注册了所有的Metric，让api-server 知道有哪些自定义metric。

（3）GetMetricByName， GetMetricBySelector 都是返回具体的Metric数据。

（4）一般api server都是 调用GetMetricBySelector，因为hpa的对象基本都是deploy, GetMetricBySelector会循环调用GetMetricByName取得deploy所有pod的metric信息。

<br>

### 2. 定制自己的metric server

#### 2.1 代码部署和编译

这里我做了如下的修改。对于metric server而言，无论访问什么metric，都返回10。

```
func (p *monitorProvider) GetMetricByName(
	name types.NamespacedName,
	info provider.CustomMetricInfo,
	metricSelector labels.Selector,
) (*custom_metrics.MetricValue, error) {
	ref, err := helpers.ReferenceFor(p.mapper, name, info)
	if err != nil {
		return nil, err
	}
	return &custom_metrics.MetricValue{
		DescribedObject: ref,
		// MetricName:      info.Metric,
		Metric: custom_metrics.MetricIdentifier{
			Name: info.Metric,
		},
		Timestamp: metav1.Time{time.Unix(int64(10), 0)},
		Value:     *resource.NewMilliQuantity(int64(10*1000.0), resource.DecimalSI),
	}, nil
}
```

更详细的可以参考我的github项目。

<br>

编译生成自己的镜像：zoux/hpa:v1。然后生成一下的deployment。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kube-hpa
  name: kube-hpa
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-hpa
  template:
    metadata:
      labels:
        app: kube-hpa
      name: kube-hpa
    spec:
      hostNetwork: true
      containers:
      - name: kube-hpa
        image: zoux/hpa:v1
        imagePullPolicy: IfNotPresent
        command:
        - /metric-server
        args:
        - --master-url=XXX
        - --kube-config=/pkc/config
        - --tls-private-key-file=/pkc/server-key.pem
        - --secure-port=9997
        - --v=10
        ports:
        - containerPort: 9997
        resources:
          limits:
            cpu: 2
            memory: 2048Mi
          requests:
            cpu: 0.5
            memory: 500Mi
        volumeMounts:
        - name: pkc
          mountPath: /pkc
          readOnly: true
      volumes:
      - name: pkc
        hostPath:
          path: /opt/kubernetes/ssl
```

<br>

验证部署成功

```
root@k8s-master:~/testyaml/hpa# kubectl get pod -n kube-system -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
kube-hpa-84c884f994-gd5fl   1/1     Running   0          3d13h   192.168.0.5   192.168.0.5   <none>           <none>
```

#### 2.2 创建 Sv and APIService

上面虽然部署成功了，但是apiserver还是访问不到。

```
k8s-master:~/testyaml/hpa# kubectl get --raw "/apis/custom.metrics.k8s.io/v1
Error from server (NotFound): the server could not find the requested resource
```

原因在于，apiserver不知道如何找到kube-hpa-84c884f994-gd5fl这个pod进行访问。所以需要创建下面的svc和apiserver。

```
root@k8s-master:~/testyaml/hpa# cat tls.yaml 
apiVersion: v1
kind: Service
metadata:
  name: kube-hpa
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: https-hpa-dont-edit-it
    port: 9997
    targetPort: 9997
  selector:
    app: kube-hpa
---
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  service:
    name: kube-hpa
    namespace: kube-system
    port: 9997
  group: custom.metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

**创建完成，验证是否成功：**

```
root@k8s-master:~/testyaml/hpa# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
{"kind":"APIResourceList","apiVersion":"v1","groupVersion":"custom.metrics.k8s.io/v1beta1","resources":[{"name":"pods/pod_cpu_used_1m","singularName":"","namespaced":true,"kind":"MetricValueList","verbs":["get"]},{"name":"pods/pod_cpu_used_5m","singularName":"","namespaced":true,"kind":"MetricValueList","verbs":["get"]},{"name":"pods/container_cpu_used_1m","singularName":"","namespaced":true,"kind":"MetricValueList","verbs":...}
```

如果报错。查看该apiserver哪里报错了

```
root@k8s-master:~/testyaml/hpa# kubectl get APIService v1beta1.custom.metrics.k8s.io  -oyaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  creationTimestamp: "2021-06-13T13:22:01Z"
  name: v1beta1.custom.metrics.k8s.io
  resourceVersion: "1590641"
  selfLink: /apis/apiregistration.k8s.io/v1/apiservices/v1beta1.custom.metrics.k8s.io
  uid: d488d6a8-7e79-4311-a1e9-0b12e4591375
spec:
  group: custom.metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: kube-hpa
    namespace: kube-system
    port: 9997
  version: v1beta1
  versionPriority: 100
status:
  conditions:
  - lastTransitionTime: "2021-06-13T13:42:17Z"
    message: all checks passed
    reason: Passed
    status: "True"
    type: Available
```

或者直接curl访问：

```
curl -k  https://nodeip:9997/apis/custom.metrics.k8s.io/v1beta1
```

<br>

####  2.3 system:anonymous授权

如果没有出现类似问题，这一步直接跳过

又是会出现如下的错误 或者 上述的APIService没有运行成功。都是因为system:anonymous权限不够

```
annotations:
    autoscaling.alpha.kubernetes.io/conditions: '[{"type":"AbleToScale","status":"True","lastTransitionTime":"2021-06-13T13:33:12Z","reason":"SucceededGetScale","message":"the
      HPA controller was able to get the target''s current scale"},{"type":"ScalingActive","status":"False","lastTransitionTime":"2021-06-13T13:33:12Z","reason":"FailedGetPodsMetric","message":"the
      HPA was unable to compute the replica count: unable to get metric pod_cpu_usage_for_limit_1m:
      unable to fetch metrics from custom metrics API: pods.custom.metrics.k8s.io
      \"*\" is forbidden: User \"system:anonymous\" cannot get resource \"pods/pod_cpu_usage_for_limit_1m\"
      in API group \"custom.metrics.k8s.io\" in the namespace \"default\""}]'
    autoscaling.alpha.kubernetes.io/metrics: '[{"type":"Pods","pods":{"metricName":"pod_cpu_usage_for_limit_1m","targetAverageValue":"60"}}]'
    metric-containerName: zx-hpa
  creationTimestamp: "2021-06-13T13:32:56Z"
  name: nginx-hpa-zx-1
  namespace: default
  resourceVersion: "1589301"
  selfLink: /apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/
```

这是可以直接绑定clusterrole https://github.com/kubernetes-sigs/metrics-server/issues/81

我这里是直接给了 cluster-admin 权限，实际情况可以按照需求赋权。

```
kubectl create clusterrolebinding anonymous-role-binding --clusterrole=cluster-admin --user=system:anonymous
```

<br>

### 3. 创建hpa验证是否成功

可以看出来都是10

```
root@k8s-master:~/testyaml/hpa# kubectl get hpa
NAME             REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-zx-1   Deployment/zx-hpa   10/60     1         3         3          9m55s
root@k8s-master:~/testyaml/hpa# kubectl get hpa
NAME             REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-zx-1   Deployment/zx-hpa   10/60     1         3         3          9m57s
```

<br>

### 4. 追踪整个过程

**第一步**  Kcm（hpa controller）发送的请求。

```
I0613 23:12:36.498740    9879 httplog.go:90] GET /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m?labelSelector=app%3Dzx-hpa-test: (35.302304ms) 200 [kube-controller-manager/v1.17.4 (linux/amd64) kubernetes/8d8aa39/horizontal-pod-autoscaler 192.168.0.4:42750]
```

要在url里使用不安全字符，就需要使用转义。

%2A = *

%3D = =（等号）

<br>

**第二步**  apiserver进行了 url转换。

kcm访问的是： /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m?labelSelector=app%3Dzx-hpa-test

但是由于第二步创建了 Sv and APIService。所以访问这个url会被转换为：

https://192.168.0.5:9997/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m?labelSelector=app%3Dzx-hpa-test

192.168.0.5是pod kube-hpa-84c884f994-gd5fl 所在的节点ip也是podia(hostNetwork模式)。 9997是定义的端口。

<br>

**第三步：**  访问 https://192.168.0.5:9997/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m?labelSelector=app%3Dzx-hpa-test

直接在master节点上（masterip=192.168.0.4）通过curl模拟

```
root@k8s-master:~/testyaml/hpa# curl -k  https://192.168.0.5:9997/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m?labelSelector=app%3Dzx-hpa-test
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/pod_aa_100m"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "zx-hpa-7b56cddd95-5j6r4|",
        "apiVersion": "/v1"
      },
      "metricName": "pod_aa_100m",
      "timestamp": "1970-01-01T00:00:10Z",
      "value": "10",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "zx-hpa-7b56cddd95-lthbz|",
        "apiVersion": "/v1"
      },
      "metricName": "pod_aa_100m",
      "timestamp": "1970-01-01T00:00:10Z",
      "value": "10",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "zx-hpa-7b56cddd95-n9ft9|",
        "apiVersion": "/v1"
      },
      "metricName": "pod_aa_100m",
      "timestamp": "1970-01-01T00:00:10Z",
      "value": "10",
      "selector": null
    }
  ]
}
```

<br>

### 5. 总结

（1）如何定制自己的metric-server，包括代码编写和环境搭建

（2）Kubernetes 里的 Custom Metrics 机制，也是借助 Aggregator APIServer 扩展机制来实现的。这里的具体原理是，当你把 Custom Metrics APIServer 启动之后，Kubernetes 里就会出现一个叫作custom.metrics.k8s.io的 API。而当你访问这个 URL 时，Aggregator 就会把你的请求转发给 Custom Metrics APIServer 。

这里一定要注意： kube-apiserver启动参数一定要包含： -enable-swagger-ui=true  

（3）ListAllMetrics()并没有将metric注册到apiserver。 apiserver并没有对metric进行验证。上文中，我metric server的ListAllMetrics()并没有注册  pod_aa_100m这个metric，但是可以正常使用。

原因：apiserver并没有进行验证，apiserver只进行url转发，如果有返回数据，apiserver就认为这个metric是正确的。所以这一点可以用来自定义metric。