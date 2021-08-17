Table of Contents
=================

  * [1. Authorization简介](#1-authorization简介)
  * [2. 6种授权机制](#2-6种授权机制)
     * [2.1 AlwaysAllow](#21-alwaysallow)
     * [2.2 AlwaysDeny授权](#22-alwaysdeny授权)
     * [2.3 ABAC授权](#23-abac授权)
     * [2.4 Webhook授权](#24-webhook授权)
     * [2.5 RBAC授权](#25-rbac授权)
     * [2.6 Node授权](#26-node授权)
  * [3. 总结](#3-总结)
  * [4. 参考](#4-参考)

kube-apiserver中与权限相关的主要有三种机制，即认证、鉴权和准入控制。这里主要记录鉴权相关的笔记。

### 1. Authorization简介

客户端请求到了apiserver端后，首先是认证，然后就是授权。apiserver同样也支持多种授权机制，并支持同时开启多个授权功能，如果开启多个授权功能，则按照顺序执行授权器，在前面的授权器具有更高的优先级来允许或拒绝请求。客户端发起一个请求，在经过授权阶段后，**只要有一个授权器通过则授权成功**。

<br>

kube-apiserver目前提供了6种授权机制，分别是AlwaysAllow、AlwaysDeny、ABAC、Webhook、RBAC、Node。

可通过kube-apiserver启动参数--authorization-mode参数设置授权机制。

目前比较常用的就是RBAC和webhook。    --authorization-mode=RBAC,Webhook

<br>

### 2. 6种授权机制

#### 2.1 AlwaysAllow

在进行AlwaysAllow授权时，直接授权成功，返回DecisionAllow决策状态。另外，AlwaysAllow的规则解析器会将资源类型的规则列表（ResourceRuleInfo）和非资源类型的规则列表（NonResourceRuleInfo）都设置为通配符（*）匹配所有资源版本、资源及资源操作方法。代码示例如下：

```
staging/src/k8s.io/apiserver/pkg/authorization/authorizerfactory/builtin.go
func (alwaysAllowAuthorizer) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	return []authorizer.ResourceRuleInfo{
			&authorizer.DefaultResourceRuleInfo{
				Verbs:     []string{"*"},
				APIGroups: []string{"*"},
				Resources: []string{"*"},
			},
		}, []authorizer.NonResourceRuleInfo{
			&authorizer.DefaultNonResourceRuleInfo{
				Verbs:           []string{"*"},
				NonResourceURLs: []string{"*"},
			},
		}, false, nil
}
```

#### 2.2 AlwaysDeny授权

AlwaysDeny授权器会阻止所有请求，该授权器很少单独使用，一般会结合其他授权器一起使用。它的应用场景是先拒绝所有请求，再允许授权过的用户请求。

--authorization-mode=AlwaysDeny,Webhook 。所以这样就做到了，只允许Webhook授权的用户通过。

<br>

在进行AlwaysDeny授权时，直接返回DecisionNoOpionion决策状态。如果存在下一个授权器，会继续执行下一个授权器；如果不存在下一个授权器，则会拒绝所有请求。这就是kube-apiserver使用AlwaysDeny的应用场景。另外，AlwaysDeny的规则解析器会将资源类型的规则列表（ResourceRuleInfo）和非资源类型的规则列表（NonResourceRuleInfo）都设置为空，代码示例如下：[插图]

```
staging/src/k8s.io/apiserver/pkg/authorization/authorizerfactory/builtin.go
func (alwaysDenyAuthorizer) Authorize(ctx context.Context, a authorizer.Attributes) (decision authorizer.Decision, reason string, err error) {
	return authorizer.DecisionNoOpinion, "Everything is forbidden.", nil
}


func (alwaysDenyAuthorizer) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	return []authorizer.ResourceRuleInfo{}, []authorizer.NonResourceRuleInfo{}, false, nil
}
```

#### 2.3 ABAC授权

ABAC授权器基于属性的访问控制（Attribute-Based Access Control，ABAC）定义了访问控制范例，其中通过将属性组合在一起的策略来向用户授予操作权限。

kube-apiserver通过指定如下参数启用ABAC授权。

●--authorization-mode=ABAC：启用ABAC授权器。

●--authorization-policy-file：基于ABAC模式，指定策略文件，该文件使用JSON格式进行描述，每一行都是一个策略对象。如下：

Alice 可以对所有somenamespace下的资源做任何事情：

```
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "somenamespace", "resource": "*", "apiGroup": "*"}}
```

**参考**：https://kubernetes.io/zh/docs/reference/access-authn-authz/abac/

在进行ABAC授权时，遍历所有的策略，通过matches函数进行匹配，如果授权成功，返回DecisionAllow决策状态。另外，ABAC的规则

解析器会根据每一个策略将资源类型的规则列表（ResourceRuleInfo）和非资源类型的规则列表（NonResourceRuleInfo）都设置为该

用户有权限操作的资源版本、资源及资源操作方法。代码示例如下：

```
pkg/auth/authorizer/abac/abac.go
// Authorize implements authorizer.Authorize
func (pl PolicyList) Authorize(ctx context.Context, a authorizer.Attributes) (authorizer.Decision, string, error) {
	for _, p := range pl {
		if matches(*p, a) {
			return authorizer.DecisionAllow, "", nil
		}
	}
	return authorizer.DecisionNoOpinion, "No policy matched.", nil
	// TODO: Benchmark how much time policy matching takes with a medium size
	// policy file, compared to other steps such as encoding/decoding.
	// Then, add Caching only if needed.
}

// RulesFor returns rules for the given user and namespace.
func (pl PolicyList) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	var (
		resourceRules    []authorizer.ResourceRuleInfo
		nonResourceRules []authorizer.NonResourceRuleInfo
	)

	for _, p := range pl {
		if subjectMatches(*p, user) {
			if p.Spec.Namespace == "*" || p.Spec.Namespace == namespace {
				if len(p.Spec.Resource) > 0 {
					r := authorizer.DefaultResourceRuleInfo{
						Verbs:     getVerbs(p.Spec.Readonly),
						APIGroups: []string{p.Spec.APIGroup},
						Resources: []string{p.Spec.Resource},
					}
					var resourceRule authorizer.ResourceRuleInfo = &r
					resourceRules = append(resourceRules, resourceRule)
				}
				if len(p.Spec.NonResourcePath) > 0 {
					r := authorizer.DefaultNonResourceRuleInfo{
						Verbs:           getVerbs(p.Spec.Readonly),
						NonResourceURLs: []string{p.Spec.NonResourcePath},
					}
					var nonResourceRule authorizer.NonResourceRuleInfo = &r
					nonResourceRules = append(nonResourceRules, nonResourceRule)
				}
			}
		}
	}
	return resourceRules, nonResourceRules, false, nil
}
```

**缺点：** 每次更新的时候，需要更新文件，并且重启kube-apiserver。

#### 2.4 Webhook授权

Webhook授权器拥有基于HTTP协议回调的机制，当用户授权时，kube-apiserver组件会查询外部的Webhook服务。该过程与WebhookTokenAuth认证相似，但其中确认用户身份的机制不一样。当客户端发送的认证请求到达kube-apiserver时，kube-apiserver回调钩子方法，将授权信息发送给远程的Webhook服务器进行认证，根据Webhook服务器返回的状态来判断是否授权成功。

<br>

kube-apiserver通过指定如下参数启用Webhook授权。

●--authorization-mode=Webhook：启用Webhook授权器。

●--authorization-webhook-config-file：使用kubeconfig格式的Webhook配置文件。Webhook授权器配置文件定义如下：

```
# Kubernetes API 版本
apiVersion: v1
# API 对象种类
kind: Config
# clusters 代表远程服务。
clusters:
  - name: name-of-remote-authz-service
    cluster:
      # 对远程服务进行身份认证的 CA。
      certificate-authority: /path/to/ca.pem
      # 远程服务的查询 URL。必须使用 'https'。
      server: https://authz.example.com/authorize

# users 代表 API 服务器的 webhook 配置
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # webhook plugin 使用 cert
      client-key: /path/to/key.pem          # cert 所对应的 key

# kubeconfig 文件必须有 context。需要提供一个给 API 服务器。
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authz-service
    user: name-of-api-server
  name: webhook
```

如上配置，文件使用kubeconfig格式。在该配置文件中，users指的是kube-apiserver本身，clusters指的是远程Webhook服务。

**参考：**https://kubernetes.io/zh/docs/reference/access-authn-authz/webhook/

<br>

在进行Webhook授权时，首先通过w.responseCache.Get函数从缓存中查找是否已有缓存的授权，如果有则直接使用该状态

（Status），如果没有则通过w.subjectAccessReview.Create（RESTClient）从远程的Webhook服务器获取授权验证，该函数发送Post

请求，并在请求体（Body）中携带授权信息。在验证Webhook服务器授权之后，返回的Status.Allowed字段为true，表示授权成功并返

回DecisionAllow决策状态。另外，Webhook的规则解析器不支持规则列表解析，因为规则是由远程的Webhook服务端进行授权的。所

以Webhook的规则解析器的资源类型的规则列表（ResourceRuleInfo）和非资源类型的规则列表（NonResourceRuleInfo）都会被设置

为空。代码示例如下：

```
staging/src/k8s.io/apiserver/plugin/pkg/authorizer/webhook/webhook.go
// Authorize makes a REST request to the remote service describing the attempted action as a JSON
// serialized api.authorization.v1beta1.SubjectAccessReview object. An example request body is
// provided below.
//
//     {
//       "apiVersion": "authorization.k8s.io/v1beta1",
//       "kind": "SubjectAccessReview",
//       "spec": {
//         "resourceAttributes": {
//           "namespace": "kittensandponies",
//           "verb": "GET",
//           "group": "group3",
//           "resource": "pods"
//         },
//         "user": "jane",
//         "group": [
//           "group1",
//           "group2"
//         ]
//       }
//     }
//
// The remote service is expected to fill the SubjectAccessReviewStatus field to either allow or
// disallow access. A permissive response would return:
//
//     {
//       "apiVersion": "authorization.k8s.io/v1beta1",
//       "kind": "SubjectAccessReview",
//       "status": {
//         "allowed": true
//       }
//     }
//
// To disallow access, the remote service would return:
//
//     {
//       "apiVersion": "authorization.k8s.io/v1beta1",
//       "kind": "SubjectAccessReview",
//       "status": {
//         "allowed": false,
//         "reason": "user does not have read access to the namespace"
//       }
//     }
//
// TODO(mikedanese): We should eventually support failing closed when we
// encounter an error. We are failing open now to preserve backwards compatible
// behavior.
func (w *WebhookAuthorizer) Authorize(ctx context.Context, attr authorizer.Attributes) (decision authorizer.Decision, reason string, err error) {
	r := &authorizationv1.SubjectAccessReview{}
	if user := attr.GetUser(); user != nil {
		r.Spec = authorizationv1.SubjectAccessReviewSpec{
			User:   user.GetName(),
			UID:    user.GetUID(),
			Groups: user.GetGroups(),
			Extra:  convertToSARExtra(user.GetExtra()),
		}
	}

	if attr.IsResourceRequest() {
		r.Spec.ResourceAttributes = &authorizationv1.ResourceAttributes{
			Namespace:   attr.GetNamespace(),
			Verb:        attr.GetVerb(),
			Group:       attr.GetAPIGroup(),
			Version:     attr.GetAPIVersion(),
			Resource:    attr.GetResource(),
			Subresource: attr.GetSubresource(),
			Name:        attr.GetName(),
		}
	} else {
		r.Spec.NonResourceAttributes = &authorizationv1.NonResourceAttributes{
			Path: attr.GetPath(),
			Verb: attr.GetVerb(),
		}
	}
	key, err := json.Marshal(r.Spec)
	if err != nil {
		return w.decisionOnError, "", err
	}
	// 先使用缓存
	if entry, ok := w.responseCache.Get(string(key)); ok {
		r.Status = entry.(authorizationv1.SubjectAccessReviewStatus)
	} else {
		var (
			result *authorizationv1.SubjectAccessReview
			err    error
		)
		webhook.WithExponentialBackoff(ctx, w.initialBackoff, func() error {
			result, err = w.subjectAccessReview.CreateContext(ctx, r)
			return err
		}, webhook.DefaultShouldRetry)
		if err != nil {
			// An error here indicates bad configuration or an outage. Log for debugging.
			klog.Errorf("Failed to make webhook authorizer request: %v", err)
			return w.decisionOnError, "", err
		}
		r.Status = result.Status
		if shouldCache(attr) {
			if r.Status.Allowed {
				w.responseCache.Add(string(key), r.Status, w.authorizedTTL)
			} else {
				w.responseCache.Add(string(key), r.Status, w.unauthorizedTTL)
			}
		}
	}
	switch {
	case r.Status.Denied && r.Status.Allowed:
		return authorizer.DecisionDeny, r.Status.Reason, fmt.Errorf("webhook subject access review returned both allow and deny response")
	case r.Status.Denied:
		return authorizer.DecisionDeny, r.Status.Reason, nil
	case r.Status.Allowed:
		return authorizer.DecisionAllow, r.Status.Reason, nil
	default:
		return authorizer.DecisionNoOpinion, r.Status.Reason, nil
	}

}

//TODO: need to finish the method to get the rules when using webhook mode
func (w *WebhookAuthorizer) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	var (
		resourceRules    []authorizer.ResourceRuleInfo
		nonResourceRules []authorizer.NonResourceRuleInfo
	)
	incomplete := true
	return resourceRules, nonResourceRules, incomplete, fmt.Errorf("webhook authorizer does not support user rule resolution")
}
```

<br>

#### 2.5 RBAC授权

RBAC授权器现实了基于角色的权限访问控制（Role-Based Access Control），其也是目前使用最为广泛的授权模型。在RBAC授权器

中，权限与角色相关联，形成了用户—角色—权限的授权模型。用户通过加入某些角色从而得到这些角色的操作权限，这极大地简化了权

限管理。

在kube-apiserver设计的RBAC授权器中，新增了角色与集群绑定的概念，也就是说，kube-apiserver可以提供4种数据类型来表达基于角色的授权，它们分别是角色（Role）、集群角色（ClusterRole）、角色绑定（RoleBinding）及集群角色绑定（ClusterRoleBinding），这4种数据类型定义在vendor/k8s.io/api/rbac/v1/types.go中。

Role <-> RoleBinding。  角色只能被授予某一个命名空间的权限。

ClusterRole <-> ClusterRoleBinding。集群角色是一组用户的集合，与规则相关联。集群角色能够被授予集群范围的权限，例如节点、非资源类型的服务端点（Endpoint）、跨所有命名空间的权限等。

<br>

```
// Role is a namespaced, logical grouping of PolicyRules that can be referenced as a unit by a RoleBinding.
type Role struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Rules holds all the PolicyRules for this Role
	// +optional
	Rules []PolicyRule `json:"rules" protobuf:"bytes,2,rep,name=rules"`
}

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// RoleBinding references a role, but does not contain it.  It can reference a Role in the same namespace or a ClusterRole in the global namespace.
// It adds who information via Subjects and namespace information by which namespace it exists in.  RoleBindings in a given
// namespace only have effect in that namespace.
type RoleBinding struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Subjects holds references to the objects the role applies to.
	// +optional
	Subjects []Subject `json:"subjects,omitempty" protobuf:"bytes,2,rep,name=subjects"`

	// RoleRef can reference a Role in the current namespace or a ClusterRole in the global namespace.
	// If the RoleRef cannot be resolved, the Authorizer must return an error.
	RoleRef RoleRef `json:"roleRef" protobuf:"bytes,3,opt,name=roleRef"`
}
```

**Role**就是相当于定义了一些规则列表。具体就是。举个例子，这个就是定义了一个 haimaxy-role 的角色。这个角色可以对 extensions.apps组下面的deploy, rs, pod执行 "get", "list", "watch", "create", "update", "patch", "delete"操作。

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: haimaxy-role
  namespace: kube-system
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # 也可以使用['*']
```

<br>

而RoleBinding就是将某个用户和角色进行绑定。

这里就是 User.haimaxy绑定了上面的haimaxy-role。这样haimaxy这个用户，就可以对 extensions.apps组下面的deploy, rs, pod执行 "get", "list", "watch", "create", "update", "patch", "delete"操作。

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: haimaxy-rolebinding
  namespace: kube-system
subjects:
- kind: User
  name: haimaxy
  apiGroup: ""
roleRef:
  kind: Role
  name: haimaxy-role
  apiGroup: ""
```

更多的使用教程，可以参考这个博客，手动创建一波就清楚了：https://www.qikqiak.com/post/use-rbac-in-k8s/

<br>

rbac的鉴权流程如下:

1. 通过`Request`获取`Attribute`包括用户，资源和对应的操作
2. `Authorize`调用`VisitRulesFor`进行具体的鉴权
3. 获取所有的ClusterRoleBindings，并对其进行遍历操作
4. 根据请求User信息，判断该是否被绑定在该ClusterRoleBinding中
5. 若在将通过函数`GetRoleReferenceRules()`获取绑定的Role所控制的访问的资源
6. 将Role所控制的访问的资源，与从API请求中提取出的资源进行比对，若比对成功，即为API请求的调用者有权访问相关资源
7. 遍历ClusterRoleBinding中，都没有获得鉴权成功的操作，将会判断提取出的信息中是否包括了namespace的信息，若包括了，将会获取该namespace下的所有RoleBindings，类似ClusterRoleBindings
8. 若在遍历了所有CluterRoleBindings，及该namespace下的所有RoleBingdings之后，仍没有对资源比对成功，则可判断该API请求的调用者没有权限访问相关资源, 鉴权失败

这里没有细看，参考了文章：https://qingwave.github.io/kube-apiserver-authorization-code/

<br>

#### 2.6 Node授权

Node授权器也被称为节点授权，是一种特殊用途的授权机制，专门授权由kubelet组件发出的API请求。

Node授权器基于RBAC授权机制实现，对kubelet组件进行基于system：node内置角色的权限控制。

system：node内置角色的权限定义在NodeRules函数中，代码示例如下：

NodeRules函数定义了system：node内置角色的权限，它拥有许多资源的操作权限，例如Configmap、Secret、Service、Pod等资源。例如，在上面的代码中，针对Pod资源的get、list、watch、create、delete等操作权限。

```
const (
	legacyGroup         = ""
	appsGroup           = "apps"
	authenticationGroup = "authentication.k8s.io"
	authorizationGroup  = "authorization.k8s.io"
	autoscalingGroup    = "autoscaling"
	batchGroup          = "batch"
	certificatesGroup   = "certificates.k8s.io"
	coordinationGroup   = "coordination.k8s.io"
	discoveryGroup      = "discovery.k8s.io"
	extensionsGroup     = "extensions"
	policyGroup         = "policy"
	rbacGroup           = "rbac.authorization.k8s.io"
	storageGroup        = "storage.k8s.io"
	resMetricsGroup     = "metrics.k8s.io"
	customMetricsGroup  = "custom.metrics.k8s.io"
	networkingGroup     = "networking.k8s.io"
	eventsGroup         = "events.k8s.io"
)

func NodeRules() []rbacv1.PolicyRule {
	nodePolicyRules := []rbacv1.PolicyRule{
		// Needed to check API access.  These creates are non-mutating
		rbacv1helpers.NewRule("create").Groups(authenticationGroup).Resources("tokenreviews").RuleOrDie(),
		rbacv1helpers.NewRule("create").Groups(authorizationGroup).Resources("subjectaccessreviews", "localsubjectaccessreviews").RuleOrDie(),

		// Needed to build serviceLister, to populate env vars for services
		rbacv1helpers.NewRule(Read...).Groups(legacyGroup).Resources("services").RuleOrDie(),

		// Nodes can register Node API objects and report status.
		// Use the NodeRestriction admission plugin to limit a node to creating/updating its own API object.
		rbacv1helpers.NewRule("create", "get", "list", "watch").Groups(legacyGroup).Resources("nodes").RuleOrDie(),
		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("nodes/status").RuleOrDie(),
		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("nodes").RuleOrDie(),

		// TODO: restrict to the bound node as creator in the NodeRestrictions admission plugin
		rbacv1helpers.NewRule("create", "update", "patch").Groups(legacyGroup).Resources("events").RuleOrDie(),

		// TODO: restrict to pods scheduled on the bound node once field selectors are supported by list/watch authorization
		rbacv1helpers.NewRule(Read...).Groups(legacyGroup).Resources("pods").RuleOrDie(),

		// Needed for the node to create/delete mirror pods.
		// Use the NodeRestriction admission plugin to limit a node to creating/deleting mirror pods bound to itself.
		rbacv1helpers.NewRule("create", "delete").Groups(legacyGroup).Resources("pods").RuleOrDie(),
		// Needed for the node to report status of pods it is running.
		// Use the NodeRestriction admission plugin to limit a node to updating status of pods bound to itself.
		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("pods/status").RuleOrDie(),
		// Needed for the node to create pod evictions.
		// Use the NodeRestriction admission plugin to limit a node to creating evictions for pods bound to itself.
		rbacv1helpers.NewRule("create").Groups(legacyGroup).Resources("pods/eviction").RuleOrDie(),

		// Needed for imagepullsecrets, rbd/ceph and secret volumes, and secrets in envs
		// Needed for configmap volume and envs
		// Use the Node authorization mode to limit a node to get secrets/configmaps referenced by pods bound to itself.
		rbacv1helpers.NewRule("get", "list", "watch").Groups(legacyGroup).Resources("secrets", "configmaps").RuleOrDie(),
		// Needed for persistent volumes
		// Use the Node authorization mode to limit a node to get pv/pvc objects referenced by pods bound to itself.
		rbacv1helpers.NewRule("get").Groups(legacyGroup).Resources("persistentvolumeclaims", "persistentvolumes").RuleOrDie(),

		// TODO: add to the Node authorizer and restrict to endpoints referenced by pods or PVs bound to the node
		// Needed for glusterfs volumes
		rbacv1helpers.NewRule("get").Groups(legacyGroup).Resources("endpoints").RuleOrDie(),
		// Used to create a certificatesigningrequest for a node-specific client certificate, and watch
		// for it to be signed. This allows the kubelet to rotate it's own certificate.
		rbacv1helpers.NewRule("create", "get", "list", "watch").Groups(certificatesGroup).Resources("certificatesigningrequests").RuleOrDie(),

		// Leases
		rbacv1helpers.NewRule("get", "create", "update", "patch", "delete").Groups("coordination.k8s.io").Resources("leases").RuleOrDie(),

		// CSI
		rbacv1helpers.NewRule("get").Groups(storageGroup).Resources("volumeattachments").RuleOrDie(),
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.ExpandPersistentVolumes) {
		// Use the Node authorization mode to limit a node to update status of pvc objects referenced by pods bound to itself.
		// Use the NodeRestriction admission plugin to limit a node to just update the status stanza.
		pvcStatusPolicyRule := rbacv1helpers.NewRule("get", "update", "patch").Groups(legacyGroup).Resources("persistentvolumeclaims/status").RuleOrDie()
		nodePolicyRules = append(nodePolicyRules, pvcStatusPolicyRule)
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) {
		// Use the Node authorization to limit a node to create tokens for service accounts running on that node
		// Use the NodeRestriction admission plugin to limit a node to create tokens bound to pods on that node
		tokenRequestRule := rbacv1helpers.NewRule("create").Groups(legacyGroup).Resources("serviceaccounts/token").RuleOrDie()
		nodePolicyRules = append(nodePolicyRules, tokenRequestRule)
	}

	// CSI
	if utilfeature.DefaultFeatureGate.Enabled(features.CSIDriverRegistry) {
		csiDriverRule := rbacv1helpers.NewRule("get", "watch", "list").Groups("storage.k8s.io").Resources("csidrivers").RuleOrDie()
		nodePolicyRules = append(nodePolicyRules, csiDriverRule)
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.CSINodeInfo) {
		csiNodeInfoRule := rbacv1helpers.NewRule("get", "create", "update", "patch", "delete").Groups("storage.k8s.io").Resources("csinodes").RuleOrDie()
		nodePolicyRules = append(nodePolicyRules, csiNodeInfoRule)
	}

	// RuntimeClass
	if utilfeature.DefaultFeatureGate.Enabled(features.RuntimeClass) {
		nodePolicyRules = append(nodePolicyRules, rbacv1helpers.NewRule("get", "list", "watch").Groups("node.k8s.io").Resources("runtimeclasses").RuleOrDie())
	}
	return nodePolicyRules
}
```

<br>

在进行Node授权时，通过r.identifier.NodeIdentity函数获取角色信息，并验证其是否为system：node内置角色，nodeName的表现形式为system：node：<nodeName>。通过rbac.RulesAllow函数进行RBAC授权，如果授权成功，返回DecisionAllow决策状态。

```
func (r *NodeAuthorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
   nodeName, isNode := r.identifier.NodeIdentity(attrs.GetUser())
   if !isNode {
      // reject requests from non-nodes
      return authorizer.DecisionNoOpinion, "", nil
   }
   if len(nodeName) == 0 {
      // reject requests from unidentifiable nodes
      klog.V(2).Infof("NODE DENY: unknown node for user %q", attrs.GetUser().GetName())
      return authorizer.DecisionNoOpinion, fmt.Sprintf("unknown node for user %q", attrs.GetUser().GetName()), nil
   }

   // subdivide access to specific resources
   if attrs.IsResourceRequest() {
      requestResource := schema.GroupResource{Group: attrs.GetAPIGroup(), Resource: attrs.GetResource()}
      switch requestResource {
      case secretResource:
         return r.authorizeReadNamespacedObject(nodeName, secretVertexType, attrs)
      case configMapResource:
         return r.authorizeReadNamespacedObject(nodeName, configMapVertexType, attrs)
      case pvcResource:
         if r.features.Enabled(features.ExpandPersistentVolumes) {
            if attrs.GetSubresource() == "status" {
               return r.authorizeStatusUpdate(nodeName, pvcVertexType, attrs)
            }
         }
         return r.authorizeGet(nodeName, pvcVertexType, attrs)
      case pvResource:
         return r.authorizeGet(nodeName, pvVertexType, attrs)
      case vaResource:
         return r.authorizeGet(nodeName, vaVertexType, attrs)
      case svcAcctResource:
         if r.features.Enabled(features.TokenRequest) {
            return r.authorizeCreateToken(nodeName, serviceAccountVertexType, attrs)
         }
         return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gate %s", features.TokenRequest), nil
      case leaseResource:
         return r.authorizeLease(nodeName, attrs)
      case csiNodeResource:
         if r.features.Enabled(features.CSINodeInfo) {
            return r.authorizeCSINode(nodeName, attrs)
         }
         return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gates %s", features.CSINodeInfo), nil
      }

   }

   // Access to other resources is not subdivided, so just evaluate against the statically defined node rules
   if rbac.RulesAllow(attrs, r.nodeRules...) {
      return authorizer.DecisionAllow, "", nil
   }
   return authorizer.DecisionNoOpinion, "", nil
}
```

<br>

### 3. 总结

（1）本文主要参考《kubernetes源码解剖》这本书籍，通过根据一些自身使用，记录一下k8s授权方面的知识。日后需要相关开发或者更深入的了解时，有一定的知识基础。

（2）针对这6种授权模式，个人认为的优缺点如下：

| 模式        | 优点                                                         | 缺点                                                         |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| AlwaysAllow | 简答，适用于自己搭建集群实践，这样就会省一些搭建环境的事情   | 非常不安全                                                   |
| AlwaysDeny  | 基本是配合使用的。AlwaysDeny放第一个，先拒绝所有请求，然后后面加webhook这些，这样的好处就是，我只允许我webhook认证过的请求。 |                                                              |
| ABAC        |                                                              | 每次需要修改文件，并且重启apiserver。而正式集群中，重启apiserver的风险是非常大的 |
| Webhook     | 可以通过webhook使用一套 定制化的权限管理系统                 |                                                              |
| RBAC        | 是目前比较流行的授权思路。有这么几个优点：<br>（1）对集群中的资源和非资源均拥有完整的覆盖<br>（2）整个RBAC完全由几个API对象完成，同其他API对象一样，可以用kubectl或API进行操作<br>（3）可以在运行时进行操作，无需重启API Server |                                                              |
| Node        | 专门针对kubelet的授权，事先专门为kubelet定义好了一组权限     |                                                              |

个人感觉目前常用的就是：RBAC，Webhook，Node这三种

###  4. 参考

书籍：kubernetes源码解剖，郑东

https://kubernetes.io/zh/docs/reference/access-authn-authz/abac/

https://kubernetes.io/zh/docs/reference/access-authn-authz/webhook/

https://www.qikqiak.com/post/use-rbac-in-k8s/

https://qingwave.github.io/kube-apiserver-authorization-code/

<br>