### 1. 概念介绍

 **污点（Taint）** 应用于node身上，表示该节点有污点了，如果不能忍受这个污点的pod，你就不要调度/运行到这个节点上。如果是不能运行到这个节点上，那就是污点驱逐了。

**容忍度（Toleration）** 是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。或者允许这个pod继续运行到这个节点上。

可以看出来，污点和容忍度（Toleration）相互配合，可以用来避免 Pod 被分配/运行到不合适的节点上。 每个节点上都可以应用一个或多个污点，每个pod也是可以应用一个或多个容忍度。

### 2. 污点详解

污点总共由4个字段组成：

**key, value字段**：可以任意字符。这个可以自定义。

**Effect**：NoExecute，PreferNoSchedule，NoSchedule 三选一

* NoExecute表示不能运行污点，意思是如果该节点有这种污点，但是pod没有对应的容忍度，那么这个pod是会被驱逐的
* NoSchedule表示不能调度污点，意思是如果该节点有这种污点，pod没有对应的容忍度，那么在调度的时候，这个pod是不会考虑这个节点的
* PreferNoSchedule 是NoSchedule的软化版。意思是如果该节点有这种污点，pod没有对应的容忍度，那么在调度的时候，这个pod不会优先考虑这个节点，但是如果实在没有节点可用，它还是接受调度到该节点上的。

**TimeAdded** : 这个污点是什么时候加的

```
// The node this Taint is attached to has the "effect" on
// any pod that does not tolerate the Taint.
type Taint struct {
	// Required. The taint key to be applied to a node.
	Key string `json:"key" protobuf:"bytes,1,opt,name=key"`
	// Required. The taint value corresponding to the taint key.
	// +optional
	Value string `json:"value,omitempty" protobuf:"bytes,2,opt,name=value"`
	// Required. The effect of the taint on pods
	// that do not tolerate the taint.
	// Valid effects are NoSchedule, PreferNoSchedule and NoExecute.
	Effect TaintEffect `json:"effect" protobuf:"bytes,3,opt,name=effect,casttype=TaintEffect"`
	// TimeAdded represents the time at which the taint was added.
	// It is only written for NoExecute taints.
	// +optional
	TimeAdded *metav1.Time `json:"timeAdded,omitempty" protobuf:"bytes,4,opt,name=timeAdded"`
}
```

添加污点的方式也很简单：

```
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
```

**k8s默认污点**

- node.kubernetes.io/not-ready：节点未准备好，相当于节点状态Ready的值为False。
- node.kubernetes.io/unreachable：Node Controller访问不到节点，相当于节点状态Ready的值为Unknown
- node.kubernetes.io/out-of-disk：节点磁盘耗尽
- node.kubernetes.io/memory-pressure：节点存在内存压力
- node.kubernetes.io/disk-pressure：节点存在磁盘压力
- node.kubernetes.io/network-unavailable：节点网络不可达
- node.kubernetes.io/unschedulable：节点不可调度
- node.cloudprovider.kubernetes.io/uninitialized：如果Kubelet启动时指定了一个外部的cloudprovider，它将给当前节点添加一个Taint将其标记为不可用。在cloud-controller-manager的一个controller初始化这个节点后，Kubelet将删除这个Taint

<br>

### 3. 容忍度详解

```
// Toleration represents the toleration object that can be attached to a pod.
// The pod this Toleration is attached to tolerates any taint that matches
// the triple <key,value,effect> using the matching operator <operator>.
type Toleration struct {
	// Key is the taint key that the toleration applies to. Empty means match all taint keys.
	// If the key is empty, operator must be Exists; this combination means to match all values and all keys.
	// +optional
	Key string
	// Operator represents a key's relationship to the value.
	// Valid operators are Exists and Equal. Defaults to Equal.
	// Exists is equivalent to wildcard for value, so that a pod can
	// tolerate all taints of a particular category.
	// +optional
	Operator TolerationOperator
	// Value is the taint value the toleration matches to.
	// If the operator is Exists, the value should be empty, otherwise just a regular string.
	// +optional
	Value string
	// Effect indicates the taint effect to match. Empty means match all taint effects.
	// When specified, allowed values are NoSchedule, PreferNoSchedule and NoExecute.
	// +optional
	Effect TaintEffect  // 
	// TolerationSeconds represents the period of time the toleration (which must be
	// of effect NoExecute, otherwise this field is ignored) tolerates the taint. By default,
	// it is not set, which means tolerate the taint forever (do not evict). Zero and
	// negative values will be treated as 0 (evict immediately) by the system.
	// +optional
	TolerationSeconds *int64
}
```

容忍度应用在pod身上，可以看出来，相比污点，多了2个字段：

**Operator**： string类型，Exists，Equal 二选一

`operator` 的默认值是 `Equal`。

一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

- 如果 `operator` 是 `Exists` （此时容忍度不能指定 `value`）, 例如这种

  ```
  tolerations:
  - key: "key1"
    operator: "Exists"
    effect: "NoSchedule"
  ```

- 如果 `operator` 是 `Equal` ，则它们的 `value` 应该相等。例如这种

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

**TolerationSeconds**: 容忍时间。表示在驱逐之前，我还可以忍受你这个pod运行多久。只针对NoSchedule类型生效。

<br>

**说明：**

存在两种特殊情况：

如果一个容忍度的 `key` 为空且 `operator` 为 `Exists`， 表示这个容忍度与任意的 key、value 和 effect 都匹配，即这个容忍度能容忍任何污点。

如果 `effect` 为空，则可以与所有键名 `key1` 的效果相匹配。

**TolerationSeconds：** 容忍时间。如果没有设置默认是不容忍。

### 4. 污点驱逐

污点驱逐：node在运行过程中，被设置了NoExecute的污点，但是运行的pod没有对应的容忍度。因此需要将这些pod删除。

kcm中是nodelifeController控制污点驱逐的。默认是开启的。如下参数默认是true。

```
--enable-taint-manager=true --feature-gates=TaintBasedEvictions=true
```