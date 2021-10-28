Table of Contents
=================

  * [1. 什么是serviceaccount](#1-什么是serviceaccount)
  * [2、Service account与User account区别](#2service-account与user-account区别)
  * [3、默认Service Account](#3默认service-account)
     * [3.1 默认sa的权限测试](#31-默认sa的权限测试)
     * [3.2 自定义sa的权限测试](#32-自定义sa的权限测试)
  * [4. 如何通过client-go使用sa](#4-如何通过client-go使用sa)

### 1. 什么是serviceaccount

k8s中提供了良好的多租户认证管理机制，如RBAC、ServiceAccount还有各种Policy等。

当用户访问集群（例如使用kubectl命令）时，apiserver 会将用户认证为一个特定的 User Account（目前通常是admin，除非系统管理员自定义了集群配置）。

Pod 容器中的进程也可以与 apiserver 联系。 当它们在联系 apiserver 的时候，它们会被认证为一个特定的 Service Account（例如default）。

<br>

**使用场景**

Service Account它并不是给kubernetes集群的用户使用的，而是给pod里面的进程使用的，它为pod提供必要的身份认证。

<br>

Service Account包含3个主要内容，分别介绍如下：

* NameSpace: 指定了Pod所在的命名空间
* CA： kube-apiserver组件的CA公钥证书，是Pod中的进程对kube-apiserver进程验证的证书
* Token：用作身份验证，通过kube-apiserver私钥签发经过Base64b编码的Bearer Token

### 2、Service account与User account区别

1. User account是为人设计的，而service account则是为Pod中的进程调用Kubernetes API或其他外部服务而设计的
2. User account是跨namespace的，而service account则是仅局限它所在的namespace；
3. 每个namespace都会自动创建一个default service account
4. Token controller检测service account的创建，并为它们创建secret
5. 开启ServiceAccount Admission Controller后:

 5.1 每个Pod在创建后都会自动设置spec.serviceAccount为default（除非指定了其他ServiceAccout）
​ 5.2 验证Pod引用的service account已经存在，否则拒绝创建
​ 5.3 如果Pod没有指定ImagePullSecrets，则把service account的ImagePullSecrets加到Pod中
​ 5.4 每个container启动后都会挂载该service account的token和ca.crt到/var/run/secrets/kubernetes.io/serviceaccount/

```bash
# kubectl exec nginx-3137573019-md1u2 ls /run/secrets/kubernetes.io/serviceaccount
 ca.crt namespace token 
```

**查看系统的config配置**

这里用到的token就是被授权过的SeviceAccount账户的token,集群利用token来使用ServiceAccount账户

```text
[root@master yaml]#  cat /root/.kube/config
```

### 3、默认Service Account

默认在 pod 中使用自动挂载的 service account 凭证来访问 API，如 Accessing the Cluster（[https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod](https://link.zhihu.com/?target=https%3A//kubernetes.io/docs/tasks/access-application-cluster/access-cluster/%23accessing-the-api-from-a-pod)） 中所描述。

当创建 pod 的时候，如果没有指定一个 service account，系统会自动在与该pod 相同的 namespace 下为其指派一个default service account，并且使用默认的 Service Account 访问 API server。

例如：

获取刚创建的 pod 的原始 json 或 yaml 信息，将看到spec.serviceAccountName字段已经被设置为 default。

```
root@k8s-master:~# kubectl get sa
NAME      SECRETS   AGE
default   1         2d4h
root@k8s-master:~# kubectl get sa default -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-10-23T09:04:02Z"
  name: default
  namespace: default
  resourceVersion: "231"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 5953ce17-9e38-4768-9d61-e7066f838b0d
secrets:
- name: default-token-f8snr
root@k8s-master:~#
root@k8s-master:~#
root@k8s-master:~#
root@k8s-master:~# kubectl get secret default-token-f8snr -oyaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR2akNDQXFhZ0F3SUJBZ0lVZVNKWlB2SmZGangyOVBrU2NHdmw1eEFOQ2lZd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1pURUxNQWtHQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbAphV3BwYm1jeEREQUtCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUk13RVFZRFZRUURFd3ByCmRXSmxjbTVsZEdWek1CNFhEVEl4TVRBeU16QTRNakl3TUZvWERUSTJNVEF5TWpBNE1qSXdNRm93WlRFTE1Ba0cKQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbGFXcHBibWN4RERBSwpCZ05WQkFvVEEyczRjekVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUk13RVFZRFZRUURFd3ByZFdKbGNtNWxkR1Z6Ck1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBdDVPTVlLUG4xS3ZOY3FoaGxqdVQKei9pUDFiTGdWOUhFNGhZVmV0VDkralNTVTQzd20wWExqWlliT0oxZktDWkV5NU14ZUlXb1c2bFVhMDRLc2VZNAovSFdGM255VGVQVmx2citBbm9kNFZ2TWZxRXpBcmplcS85aElOcGxZdFFOMDBSanNpdHA3bDRRT1licEhTWUFNCnhXSmFPZG5lK2FNbmQrUkFaM1d0bGV1aXd5REZzVXI0NUhqeGJoeGR1YUNURUQwanNPYy9zbEQwRTFGZTRHOWoKOXpjK0xMb2ZTWHQ1N1B3Z1g5MVlwbnJUNmtTRUs0SGpMcjczMzRYTmRYbjBkektBc1A0RURzNkdibDEyZ1JiUQpuV3g2cHpSUmpkUXlua1Z0dkMzTXMrUVIrcUswb3RMMDVMTStPdy9VY2M4cXBFTUtWUVBRVFkyWGljLzZsa3IvCkV3SURBUUFCbzJZd1pEQU9CZ05WSFE4QkFmOEVCQU1DQVFZd0VnWURWUjBUQVFIL0JBZ3dCZ0VCL3dJQkFqQWQKQmdOVkhRNEVGZ1FVSjNiTDE3UGlVd0g5WDNhekp2VFVNbU1iUlgwd0h3WURWUjBqQkJnd0ZvQVVKM2JMMTdQaQpVd0g5WDNhekp2VFVNbU1iUlgwd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLSXpSSXpSMmp3UG0vU25LSXRBCjIyMUJFdnJTWEh4UE13VTJQbjgybmhQWjBaOFc0K0x3ZjBFcExlZ0xWaVgzMEJrTU5INkRkTkNUbEdrSnRSZW4KSHdMNVNnZkVnaTA0V0tXenVpT25jd2dnWkNOTXpyZGhGcFFqLzNOOWhqWUM0V050UXZlaWVmYjlZOGtpbUUvVAp6STh1MXpZTFRreG5FU3pHTE8yUGNtZXQ2TmtCb0NBTU1vc3R0ZC92RlN0b250TVk1OXBiMlpnejN1MXZuZkt5CmlpbzZVM1VtbWt2NGMzdnYwbzEwTVlMVElLR2ZiRVllSkROdjFhZ3NvSWlBQklNbEhGeUh1TUZIZmp5RExiamkKOHo0TTBmKzFkNXdqc2NHVFNsQng5anJXTzk3WFNFeU9BdDNkbkE0OU5sNUJjTDZXZWlhbGlQT0F4QWVPcUROZAp2elE9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrTXhUV2hJVVdKWWJtNU9OemRoTW5GV1FYVnpURjlWTkdSbmIzcDZNVVUxUTBGTlVGOTFlVFJ2UW5jaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJsWm1GMWJIUXRkRzlyWlc0dFpqaHpibklpTENKcmRXSmxjbTVsZEdWekxtbHZMM05sY25acFkyVmhZMk52ZFc1MEwzTmxjblpwWTJVdFlXTmpiM1Z1ZEM1dVlXMWxJam9pWkdWbVlYVnNkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVnlkbWxqWlMxaFkyTnZkVzUwTG5WcFpDSTZJalU1TlROalpURTNMVGxsTXpndE5EYzJPQzA1WkRZeExXVTNNRFkyWmpnek9HSXdaQ0lzSW5OMVlpSTZJbk41YzNSbGJUcHpaWEoyYVdObFlXTmpiM1Z1ZERwa1pXWmhkV3gwT21SbFptRjFiSFFpZlEub09hNkZ6SDVhTEIzRnZYWW9ZTHNRWUNTOHl2ZTRXdWRBbGtjNjFwTVd0UEFBRy1URUJ5WjNvN3FzSU0yRmNkTW9VbXFCOEFoakx0QlZjeVhOMVFfd0dDNE9oLUdRQnpJZ3JPZTRDUm5QWkpGX2F0ZW15LXlsazI1aldJSG9VOWU1azAxMHExYjhMU0RJekVwSFd0UzZlZC01ZkQxdG5lSHdtU09LYTJtdTZ2QWVsUW9ydmFoeHU3UWxHSWFUcWRQaVk3ZWRyUFpKSUFGWUNMeFAtMklFV0ZRbFJMUkRxcVN0ckpBbTFDUFFoeFh4ZUgtSFJoTzhnQnB4bHV0VUdSOU5LNFdoMnRFYWIyaGV1YUZUQkp0dVIxeTlJbVZFQzFpaTFlT2NGeGJRRi1zWnRlZGEwWFBTbE1rZ1BHYmNUT3VPOEdvZHBZTzA5TnFZRW5WR29pQWtn
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 5953ce17-9e38-4768-9d61-e7066f838b0d
  creationTimestamp: "2021-10-23T09:04:02Z"
  name: default-token-f8snr
  namespace: default
  resourceVersion: "229"
  selfLink: /api/v1/namespaces/default/secrets/default-token-f8snr
  uid: 11cfe3f0-ad48-458f-8959-fcc3adccacd3
type: kubernetes.io/service-account-token
```

<br>

**默认的Sa作用：** 目前看起来就是给pod塞了一个sa，没有任何的权限绑定。

#### 3.1 默认sa的权限测试

（1）kubectl get role没看见有role和 default绑定

（2）进入一个pod后, 执行以下的命令发现这个sa没有权限

```
/ $ export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/ $ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
/ $
/ $ curl -H "Authorization: Bearer $TOKEN" https://kubernetes

curl: (6) Could not resolve host: kubernetes

// 这个就是没有权限
/ $ curl -H "Authorization: Bearer $TOKEN" https://192.168.0.4:6443/api/v1/namespaces/default/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:default:default\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}/ $
/ $

// 这个就是没有权限
/ $ curl -H "Authorization: Bearer $TOKEN" https://10.0.0.1:443/api/v1/namespaces/default/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}/ $
/ $
/ $ exit
```

#### 3.2 自定义sa的权限测试

（1）创建sa

```
root@k8s-master:~# kubectl create serviceaccount sa-example
serviceaccount/sa-example created

root@k8s-master:~# kubectl get sa sa-example -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-10-25T13:51:23Z"
  name: sa-example
  namespace: default
  resourceVersion: "434232"
  selfLink: /api/v1/namespaces/default/serviceaccounts/sa-example
  uid: 42654626-8b42-4c5e-83de-fb836acfc934
secrets:
- name: sa-example-token-lchv2
```

(2) 创建role 

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default                          # 命名空间
  name: role-example
rules:
- apiGroups: [""]
  resources: ["pods"]                         # 可以访问pod
  verbs: ["get", "list"]                      # 可以执行GET、LIST操作
```

(3) 创建rolebinding

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rolebinding-example
  namespace: default
subjects:                                
- kind: User                              
  name: user-example
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount                    
  name: sa-example
  namespace: default
roleRef:                                  
  kind: Role
  name: role-example
  apiGroup: rbac.authorization.k8s.io

```

(4) 将pod设置自定义sa

```
root@k8s-master:~# cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  serviceAccountName: sa-example
  nodeName: k8s-node
  containers:
  - name: nginx
    image: curlimages/curl:7.75.0
    command:
      - sleep
      - "3600"
```

(5) 执行上诉命令

```
root@k8s-master:~# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   2d5h
```



```
root@k8s-master:~# kubectl exec -it nginx sh

/ $  export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/ $ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

/ $ curl -H "Authorization: Bearer $TOKEN" https://192.168.0.4:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:default:sa-example\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}/ $

//有get pod的权限
/ $ curl -H "Authorization: Bearer $TOKEN" https://10.0.0.1:443/api/v1/namespace
s/default/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "435185"
  },
  "items": [
    {
      "metadata": {
        "name": "nginx",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/pods/nginx",
        "uid": "0ceadb16-588f-40ae-a8c1-4d3cfb34df20",
        "resourceVersion": "435049",
        "creationTimestamp": "2021-10-25T13:57:11Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"nginx\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"3600\"],\"image\":\"curlimages/curl:7.75.0\",\"name\":\"nginx\"}],\"nodeName\":\"k8s-node\",\"serviceAccountName\":\"sa-example\"}}\n"
        }
      },
      "spec": {
        "volumes": [
          {
            "name": "sa-example-token-lchv2",
            "secret": {
              "secretName": "sa-example-token-lchv2",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "nginx",
            "image": "curlimages/curl:7.75.0",
            "command": [
              "sleep",
              "3600"
            ],
            "resources": {

            },
            "volumeMounts": [
              {
                "name": "sa-example-token-lchv2",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "sa-example",
        "serviceAccount": "sa-example",
        "nodeName": "k8s-node",
        "securityContext": {

        },
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:11Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:13Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:13Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:11Z"
          }
        ],
        "hostIP": "192.168.0.5",
        "podIP": "10.244.1.7",
        "podIPs": [
          {
            "ip": "10.244.1.7"
          }
        ],
        "startTime": "2021-10-25T13:57:11Z",
        "containerStatuses": [
          {
            "name": "nginx",
            "state": {
              "running": {
                "startedAt": "2021-10-25T13:57:12Z"
              }
            },
            "lastState": {

            },
            "ready": true,
            "restartCount": 0,
            "image": "curlimages/curl:7.75.0",
            "imageID": "docker-pullable://curlimages/curl@sha256:28ec2dae8001949f657dbb36141508d65572f382dbd587f868289e2ceb0d47dd",
            "containerID": "docker://d6e4cc4acfa4b3093d3ee82286cf67da117f7f6ce23fd47254ee64a79d8ff29f",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    }
  ]
}/ $



//使用 apiserver的ip:端口也是可以的
/ $ curl -H "Authorization: Bearer $TOKEN" https://192.168.0.4:6443/api/v1/names
paces/default/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "435286"
  },
  "items": [
    {
      "metadata": {
        "name": "nginx",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/pods/nginx",
        "uid": "0ceadb16-588f-40ae-a8c1-4d3cfb34df20",
        "resourceVersion": "435049",
        "creationTimestamp": "2021-10-25T13:57:11Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"nginx\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"3600\"],\"image\":\"curlimages/curl:7.75.0\",\"name\":\"nginx\"}],\"nodeName\":\"k8s-node\",\"serviceAccountName\":\"sa-example\"}}\n"
        }
      },
      "spec": {
        "volumes": [
          {
            "name": "sa-example-token-lchv2",
            "secret": {
              "secretName": "sa-example-token-lchv2",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "nginx",
            "image": "curlimages/curl:7.75.0",
            "command": [
              "sleep",
              "3600"
            ],
            "resources": {

            },
            "volumeMounts": [
              {
                "name": "sa-example-token-lchv2",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "sa-example",
        "serviceAccount": "sa-example",
        "nodeName": "k8s-node",
        "securityContext": {

        },
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:11Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:13Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:13Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-10-25T13:57:11Z"
          }
        ],
        "hostIP": "192.168.0.5",
        "podIP": "10.244.1.7",
        "podIPs": [
          {
            "ip": "10.244.1.7"
          }
        ],
        "startTime": "2021-10-25T13:57:11Z",
        "containerStatuses": [
          {
            "name": "nginx",
            "state": {
              "running": {
                "startedAt": "2021-10-25T13:57:12Z"
              }
            },
            "lastState": {

            },
            "ready": true,
            "restartCount": 0,
            "image": "curlimages/curl:7.75.0",
            "imageID": "docker-pullable://curlimages/curl@sha256:28ec2dae8001949f657dbb36141508d65572f382dbd587f868289e2ceb0d47dd",
            "containerID": "docker://d6e4cc4acfa4b3093d3ee82286cf67da117f7f6ce23fd47254ee64a79d8ff29f",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    }
  ]
}/ $
```

### 4. 如何通过client-go使用sa

直接调用client-go/rest的InClusterConfig

```
    // creates the in-cluster config
    config, err := rest.InClusterConfig()
    if err != nil {
        panic(err.Error())
    }
    // creates the clientset
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err.Error())
    }
```

InClusterConfig的源码分析，这里定义了tokenFile和rootCAFile

```
// InClusterConfig returns a config object which uses the service account
// kubernetes gives to pods. It's intended for clients that expect to be
// running inside a pod running on kubernetes. It will return ErrNotInCluster
// if called from a process not running in a kubernetes environment.
func InClusterConfig() (*Config, error) {
	const (
		tokenFile  = "/var/run/secrets/kubernetes.io/serviceaccount/token"
		rootCAFile = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	)
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, ErrNotInCluster
	}

	token, err := ioutil.ReadFile(tokenFile)
	if err != nil {
		return nil, err
	}

	tlsClientConfig := TLSClientConfig{}

	if _, err := certutil.NewPool(rootCAFile); err != nil {
		klog.Errorf("Expected to load root CA config from %s, but got err: %v", rootCAFile, err)
	} else {
		tlsClientConfig.CAFile = rootCAFile
	}

	return &Config{
		// TODO: switch to using cluster DNS.
		Host:            "https://" + net.JoinHostPort(host, port),
		TLSClientConfig: tlsClientConfig,
		BearerToken:     string(token),
		BearerTokenFile: tokenFile,
	}, nil
}
```

