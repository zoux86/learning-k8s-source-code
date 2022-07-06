

k8s集群中，controller-manage、kube-proxy、kube-scheduler、kubelet等组件都会产生大量的event。这些event对查看集群对象状态或者监控告警等等都非常有用。本章写一下自己对k8s中event的理解。

### 1. event的定义

event定义在：k8s.io/api/core/v1/types.go中

```
type Event struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata" protobuf:"bytes,1,opt,name=metadata"`
    InvolvedObject ObjectReference `json:"involvedObject" protobuf:"bytes,2,opt,name=involvedObject"`
    Reason string `json:"reason,omitempty" protobuf:"bytes,3,opt,name=reason"`
    Message string `json:"message,omitempty" protobuf:"bytes,4,opt,name=message"`
    Source EventSource `json:"source,omitempty" protobuf:"bytes,5,opt,name=source"`
    FirstTimestamp metav1.Time `json:"firstTimestamp,omitempty" protobuf:"bytes,6,opt,name=firstTimestamp"`
    LastTimestamp metav1.Time `json:"lastTimestamp,omitempty" protobuf:"bytes,7,opt,name=lastTimestamp"`
    Count int32 `json:"count,omitempty" protobuf:"varint,8,opt,name=count"`
    Type string `json:"type,omitempty" protobuf:"bytes,9,opt,name=type"`
    EventTime metav1.MicroTime `json:"eventTime,omitempty" protobuf:"bytes,10,opt,name=eventTime"`
    Series *EventSeries `json:"series,omitempty" protobuf:"bytes,11,opt,name=series"`
    Action string `json:"action,omitempty" protobuf:"bytes,12,opt,name=action"`
    Related *ObjectReference `json:"related,omitempty" protobuf:"bytes,13,opt,name=related"`
    ReportingController string `json:"reportingComponent" protobuf:"bytes,14,opt,name=reportingComponent"`
    ReportingInstance string `json:"reportingInstance" protobuf:"bytes,15,opt,name=reportingInstance"`
    ReportingInstance string `json:"reportingInstance" protobuf:"bytes,15,opt,name=reportingInstance"`
}
```

Count，firstTimestamp和lasteTimestamp 表示事件重复了多少次

Message 详细的事件信息

Reason 简单的事件原因

Type  目前只支持：Normal和Warning俩种

Source 事件发出的来源

InvolvedObject 引用的另一个Kubernetes对象，例如Pod或者Deployment

<br>

### 2. kubectl自定义输出k8s事件 - （该方法适用于所有对象）

通常我们是通过kubectl  查看事件，如下：

```
root@k8s-master:~# kubectl get event
LAST SEEN   TYPE     REASON    OBJECT                        MESSAGE
40m         Normal   Pulled    pod/zx-hpa-7b56cddd95-5j6r4   Container image "busybox:latest" already present on machine
40m         Normal   Created   pod/zx-hpa-7b56cddd95-5j6r4   Created container busybox
40m         Normal   Started   pod/zx-hpa-7b56cddd95-5j6r4   Started container busybox
40m         Normal   Pulled    pod/zx-hpa-7b56cddd95-lthbz   Container image "busybox:latest" already present on machine
40m         Normal   Created   pod/zx-hpa-7b56cddd95-lthbz   Created container busybox
40m         Normal   Started   pod/zx-hpa-7b56cddd95-lthbz   Started container busybox
29m         Normal   Pulled    pod/zx-hpa-7b56cddd95-n9ft9   Container image "busybox:latest" already present on machine
29m         Normal   Created   pod/zx-hpa-7b56cddd95-n9ft9   Created container busybox
29m         Normal   Started   pod/zx-hpa-7b56cddd95-n9ft9   Started container busybox
```

补充俩点注意：

（1）event也有ns，所以kubectl get event 没有找到预期的事件，看看是否加上了 ns

（2）自定义event的输出

默认的kubectl get event只输出了五列，有时并没有我们想看到的内容，这个时候可以利用kubectl 的强大输出功能，输出自己想看到的信息。

```
根据 kubectl 操作，支持以下输出格式：

Output format	Description
-o custom-columns=<spec>	使用逗号分隔的自定义列列表打印表。
-o custom-columns-file=<filename>	使用 <filename> 文件中的自定义列模板打印表。
-o json	输出 JSON 格式的 API 对象
-o jsonpath=<template>	打印 jsonpath 表达式定义的字段
-o jsonpath-file=<filename>	打印 <filename> 文件中 jsonpath 表达式定义的字段。
-o name	仅打印资源名称而不打印任何其他内容。
-o wide	以纯文本格式输出，包含任何附加信息。对于 pod 包含节点名。
-o yaml	输出 YAML 格式的 API 对象。
```

举例：这里我们想看到 event的 Count 和 name, 以及namespaces。

**首先**，我通过 kubectl get event -oyaml查看到了event的所有字段，这里发现 count 和  metadata.name, metadata.namespace

```
- apiVersion: v1
  count: 308
  eventTime: null
  firstTimestamp: "2021-06-13T13:42:19Z"
  involvedObject:
    apiVersion: v1
    fieldPath: spec.containers{busybox}
    kind: Pod
    name: zx-hpa-7b56cddd95-n9ft9
    namespace: default
    resourceVersion: "1590656"
    uid: 379ef34e-3277-4367-a0e2-34645397590c
  kind: Event
  lastTimestamp: "2021-06-26T08:46:33Z"
  message: Container image "busybox:latest" already present on machine
  metadata:
    creationTimestamp: "2021-06-13T13:42:19Z"
    name: zx-hpa-7b56cddd95-n9ft9.16882815c5dbca52
    namespace: default
    resourceVersion: "4136244"
    selfLink: /api/v1/namespaces/default/events/zx-hpa-7b56cddd95-n9ft9.16882815c5dbca52
    uid: 9f4becb5-984d-4b3d-9308-aa6bde3e3d87
  reason: Pulled
  reportingComponent: ""
  reportingInstance: ""
  source:
    component: kubelet
    host: 192.168.0.5
  type: Normal
```

<br>

**然后** kubectl 自定义输出

```
root@k8s-master:~# kubectl get event -o custom-columns=count:count,ns:metadata.namespace,name:metadata.name
count   ns        name
51      default   test-pod2.1685b3d2de5432c9
51      default   test-pod2.1685b3d2e7bdb58c
50      default   test-pod2.1685d490dface300
309     default   zx-hpa-7b56cddd95-5j6r4.168827811e1dfc40
309     default   zx-hpa-7b56cddd95-5j6r4.168827812259f1fc
309     default   zx-hpa-7b56cddd95-5j6r4.168827812ae481ff
309     default   zx-hpa-7b56cddd95-lthbz.168827811fb9b32d
309     default   zx-hpa-7b56cddd95-lthbz.16882781231b0eaf
309     default   zx-hpa-7b56cddd95-lthbz.168827812adf08c9
309     default   zx-hpa-7b56cddd95-n9ft9.16882815c5dbca52
309     default   zx-hpa-7b56cddd95-n9ft9.16882815ca83dabb
309     default   zx-hpa-7b56cddd95-n9ft9.16882815d2c77d3b
```
