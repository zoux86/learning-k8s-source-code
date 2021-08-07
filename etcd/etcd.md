### 1. etcd的基本操作

```
 ETCDCTL_API=3 /usr/local/bin/etcdctl --cacert=/home/cld/tls/etcd/ca.pem --cert=/home/cld/tls/etcd/server.pem --key=/home/cld/tls/etcd/server-key.pem --endpoints="https://10.212.193.4:2379,https://10.212.193.5:2379" endpoint health --write-out=table
```



```
[root@k8s-master ssl]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=ca.pem --cert=server.pem  --key=server-key.pem  --endpoints="https://192.168.0.4:2379,https://192.168.0.5:2379" get / --prefix --keys-only/registry/apiregistration.k8s.io/apiservices/v1./registry/apiregistration.k8s.io/apiservices/v1.apps

/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1.autoscaling

/registry/apiregistration.k8s.io/apiservices/v1.batch

/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.apiextensions.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.apps

/registry/apiregistration.k8s.io/apiservices/v1beta1.authentication.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.authorization.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.batch

/registry/apiregistration.k8s.io/apiservices/v1beta1.certificates.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.coordination.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.events.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.extensions

/registry/apiregistration.k8s.io/apiservices/v1beta1.policy

/registry/apiregistration.k8s.io/apiservices/v1beta1.rbac.authorization.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.scheduling.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta1.storage.k8s.io

/registry/apiregistration.k8s.io/apiservices/v1beta2.apps

/registry/apiregistration.k8s.io/apiservices/v2beta1.autoscaling

/registry/apiregistration.k8s.io/apiservices/v2beta2.autoscaling

/registry/clusterrolebindings/cluster-admin

/registry/clusterrolebindings/kubelet-bootstrap

/registry/clusterrolebindings/system:aws-cloud-provider

/registry/clusterrolebindings/system:basic-user

/registry/clusterrolebindings/system:controller:attachdetach-controller

/registry/clusterrolebindings/system:controller:certificate-controller

/registry/clusterrolebindings/system:controller:clusterrole-aggregation-controller

/registry/clusterrolebindings/system:controller:cronjob-controller

/registry/clusterrolebindings/system:controller:daemon-set-controller

/registry/clusterrolebindings/system:controller:deployment-controller

/registry/clusterrolebindings/system:controller:disruption-controller

/registry/clusterrolebindings/system:controller:endpoint-controller

/registry/clusterrolebindings/system:controller:expand-controller

/registry/clusterrolebindings/system:controller:generic-garbage-collector

/registry/clusterrolebindings/system:controller:horizontal-pod-autoscaler

/registry/clusterrolebindings/system:controller:job-controller

/registry/clusterrolebindings/system:controller:namespace-controller

/registry/clusterrolebindings/system:controller:node-controller

/registry/clusterrolebindings/system:controller:persistent-volume-binder

/registry/clusterrolebindings/system:controller:pod-garbage-collector

/registry/clusterrolebindings/system:controller:pv-protection-controller

/registry/clusterrolebindings/system:controller:pvc-protection-controller

/registry/clusterrolebindings/system:controller:replicaset-controller

/registry/clusterrolebindings/system:controller:replication-controller

/registry/clusterrolebindings/system:controller:resourcequota-controller

/registry/clusterrolebindings/system:controller:route-controller

/registry/clusterrolebindings/system:controller:service-account-controller

/registry/clusterrolebindings/system:controller:service-controller

/registry/clusterrolebindings/system:controller:statefulset-controller

/registry/clusterrolebindings/system:controller:ttl-controller

/registry/clusterrolebindings/system:discovery

/registry/clusterrolebindings/system:kube-controller-manager

/registry/clusterrolebindings/system:kube-dns

/registry/clusterrolebindings/system:kube-scheduler

/registry/clusterrolebindings/system:node

/registry/clusterrolebindings/system:node-proxier

/registry/clusterrolebindings/system:volume-scheduler

/registry/clusterroles/admin

/registry/clusterroles/cluster-admin

/registry/clusterroles/edit

/registry/clusterroles/system:aggregate-to-admin

/registry/clusterroles/system:aggregate-to-edit

/registry/clusterroles/system:aggregate-to-view

/registry/clusterroles/system:auth-delegator

/registry/clusterroles/system:aws-cloud-provider

/registry/clusterroles/system:basic-user

/registry/clusterroles/system:certificates.k8s.io:certificatesigningrequests:nodeclient

/registry/clusterroles/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient

/registry/clusterroles/system:controller:attachdetach-controller

/registry/clusterroles/system:controller:certificate-controller

/registry/clusterroles/system:controller:clusterrole-aggregation-controller

/registry/clusterroles/system:controller:cronjob-controller

/registry/clusterroles/system:controller:daemon-set-controller

/registry/clusterroles/system:controller:deployment-controller

/registry/clusterroles/system:controller:disruption-controller

/registry/clusterroles/system:controller:endpoint-controller

/registry/clusterroles/system:controller:expand-controller

/registry/clusterroles/system:controller:generic-garbage-collector

/registry/clusterroles/system:controller:horizontal-pod-autoscaler

/registry/clusterroles/system:controller:job-controller

/registry/clusterroles/system:controller:namespace-controller

/registry/clusterroles/system:controller:node-controller

/registry/clusterroles/system:controller:persistent-volume-binder

/registry/clusterroles/system:controller:pod-garbage-collector

/registry/clusterroles/system:controller:pv-protection-controller

/registry/clusterroles/system:controller:pvc-protection-controller

/registry/clusterroles/system:controller:replicaset-controller

/registry/clusterroles/system:controller:replication-controller

/registry/clusterroles/system:controller:resourcequota-controller

/registry/clusterroles/system:controller:route-controller

/registry/clusterroles/system:controller:service-account-controller

/registry/clusterroles/system:controller:service-controller

/registry/clusterroles/system:controller:statefulset-controller

/registry/clusterroles/system:controller:ttl-controller

/registry/clusterroles/system:csi-external-attacher

/registry/clusterroles/system:csi-external-provisioner

/registry/clusterroles/system:discovery

/registry/clusterroles/system:heapster

/registry/clusterroles/system:kube-aggregator

/registry/clusterroles/system:kube-controller-manager

/registry/clusterroles/system:kube-dns

/registry/clusterroles/system:kube-scheduler

/registry/clusterroles/system:kubelet-api-admin

/registry/clusterroles/system:node

/registry/clusterroles/system:node-bootstrapper

/registry/clusterroles/system:node-problem-detector

/registry/clusterroles/system:node-proxier

/registry/clusterroles/system:persistent-volume-provisioner

/registry/clusterroles/system:volume-scheduler

/registry/clusterroles/view

/registry/configmaps/kube-system/extension-apiserver-authentication

/registry/deployments/default/my-nginx

/registry/events/default/my-nginx-756f645cd7-4ws6k.166a0834ed7ebb23

/registry/events/default/my-nginx-756f645cd7-4ws6k.166a083563366800

/registry/events/default/my-nginx-756f645cd7-4ws6k.166a084837c7ba70

/registry/events/default/my-nginx-756f645cd7-4ws6k.166a08483c00ad3e

/registry/events/default/my-nginx-756f645cd7-4ws6k.166a084845e099eb

/registry/events/default/my-nginx-756f645cd7-pqdgd.166a0834ed7b325b

/registry/events/default/my-nginx-756f645cd7-pqdgd.166a083562dbb392

/registry/events/default/my-nginx-756f645cd7-pqdgd.166a0846c4623629

/registry/events/default/my-nginx-756f645cd7-pqdgd.166a08470a4905f4

/registry/events/default/my-nginx-756f645cd7-pqdgd.166a08471346fd1d

/registry/events/default/my-nginx-756f645cd7.166a0834e8216608

/registry/events/default/my-nginx-756f645cd7.166a0834ea5732c8

/registry/events/default/my-nginx.166a0834e5fb8890

/registry/masterleases/192.168.0.4

/registry/minions/192.168.0.5

/registry/namespaces/default

/registry/namespaces/kube-public

/registry/namespaces/kube-system

/registry/pods/default/my-nginx-756f645cd7-4ws6k

/registry/pods/default/my-nginx-756f645cd7-pqdgd

/registry/priorityclasses/system-cluster-critical

/registry/priorityclasses/system-node-critical

/registry/ranges/serviceips

/registry/ranges/servicenodeports

/registry/replicasets/default/my-nginx-756f645cd7

/registry/rolebindings/kube-public/system:controller:bootstrap-signer

/registry/rolebindings/kube-system/system::leader-locking-kube-controller-manager

/registry/rolebindings/kube-system/system::leader-locking-kube-scheduler

/registry/rolebindings/kube-system/system:controller:bootstrap-signer

/registry/rolebindings/kube-system/system:controller:cloud-provider

/registry/rolebindings/kube-system/system:controller:token-cleaner

/registry/roles/kube-public/system:controller:bootstrap-signer

/registry/roles/kube-system/extension-apiserver-authentication-reader

/registry/roles/kube-system/system::leader-locking-kube-controller-manager

/registry/roles/kube-system/system::leader-locking-kube-scheduler

/registry/roles/kube-system/system:controller:bootstrap-signer

/registry/roles/kube-system/system:controller:cloud-provider

/registry/roles/kube-system/system:controller:token-cleaner

/registry/secrets/default/default-token-69c95

/registry/secrets/kube-public/default-token-6fqsq

/registry/secrets/kube-system/default-token-5tvm8

/registry/serviceaccounts/default/default

/registry/serviceaccounts/kube-public/default

/registry/serviceaccounts/kube-system/default

/registry/services/endpoints/default/kubernetes

/registry/services/endpoints/kube-system/kube-controller-manager

/registry/services/endpoints/kube-system/kube-scheduler

/registry/services/specs/default/kubernetes

[root@k8s-master ssl]# 
[root@k8s-master ssl]# 
[root@k8s-master ssl]# 
[root@k8s-master ssl]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=ca.pem --cert=server.pem  --key=server-key.pem  --endpoints="https://192.168.0.4:2379,https://192.168.0.5:2379" get /registry/pods/default/my-nginx-756f645cd7-4ws6k  -w=json | jq .{  "header": {    "cluster_id": 12138850119299830000,
    "member_id": 6539934570868143000,
    "revision": 7643164,
    "raft_term": 1892
  },
  "kvs": [
    {
      "key": "L3JlZ2lzdHJ5L3BvZHMvZGVmYXVsdC9teS1uZ2lueC03NTZmNjQ1Y2Q3LTR3czZr",
      "create_revision": 7642432,
      "mod_revision": 7642554,
      "version": 4,
      "value": "azhzAAoJCgJ2MRIDUG9kEtUICvoBChlteS1uZ2lueC03NTZmNjQ1Y2Q3LTR3czZrEhRteS1uZ2lueC03NTZmNjQ1Y2Q3LRoHZGVmYXVsdCIAKiRjM2U2ZDE3ZS03ZjJlLTExZWItOTY4OC1mYTI3MDAwNGIwMGQyADgAQggI99GSggYQAFofChFwb2QtdGVtcGxhdGUtaGFzaBIKNzU2ZjY0NWNkN1oPCgNydW4SCG15LW5naW54alQKClJlcGxpY2FTZXQaE215LW5naW54LTc1NmY2NDVjZDciJGMzZTE4NzlkLTdmMmUtMTFlYi05Njg4LWZhMjcwMDA0YjAwZCoHYXBwcy92MTABOAF6ABKlAwoxChNkZWZhdWx0LXRva2VuLTY5Yzk1EhoyGAoTZGVmYXVsdC10b2tlbi02OWM5NRikAxKcAQoIbXktbmdpbngSBW5naW54KgAyDQoAEAAYUCIDVENQKgBCAEpIChNkZWZhdWx0LXRva2VuLTY5Yzk1EAEaLS92YXIvcnVuL3NlY3JldHMva3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudCIAahQvZGV2L3Rlcm1pbmF0aW9uLWxvZ3IGQWx3YXlzgAEAiAEAkAEAogEERmlsZRoGQWx3YXlzIB4yDENsdXN0ZXJGaXJzdEIHZGVmYXVsdEoHZGVmYXVsdFILMTkyLjE2OC4wLjVYAGAAaAByAIIBAIoBAJoBEWRlZmF1bHQtc2NoZWR1bGVysgE2Chxub2RlLmt1YmVybmV0ZXMuaW8vbm90LXJlYWR5EgZFeGlzdHMaACIJTm9FeGVjdXRlKKwCsgE4Ch5ub2RlLmt1YmVybmV0ZXMuaW8vdW5yZWFjaGFibGUSBkV4aXN0cxoAIglOb0V4ZWN1dGUorALCAQDIAQAarQMKB1J1bm5pbmcSIwoLSW5pdGlhbGl6ZWQSBFRydWUaACIICPfRkoIGEAAqADIAEh0KBVJlYWR5EgRUcnVlGgAiCAjL0pKCBhAAKgAyABInCg9Db250YWluZXJzUmVhZHkSBFRydWUaACIICMvSkoIGEAAqADIAEiQKDFBvZFNjaGVkdWxlZBIEVHJ1ZRoAIggI99GSggYQACoAMgAaACIAKgsxOTIuMTY4LjAuNTILMTcyLjE3LjgzLjI6CAj30ZKCBhAAQtgBCghteS1uZ2lueBIMEgoKCAjK0pKCBhAAGgAgASgAMgxuZ2lueDpsYXRlc3Q6X2RvY2tlci1wdWxsYWJsZTovL25naW54QHNoYTI1NjpmMzY5M2ZlNTBkNWIxZGYxZWNkMzE1ZDU0ODEzYTc3YWZkNTZiMDI0NWE0MDQwNTVhOTQ2NTc0ZGViNmIzNGZjQklkb2NrZXI6Ly9iNDc4NTBmYWY2NGM1YjFiZWRjNjg0M2EzNzZlZTA1YTVlOGFmZmU4Y2VlZGNlMzNhOWJjNzQxY2EzNDVlOGRjSgpCZXN0RWZmb3J0WgAaACIA"
    }
  ],
  "count": 1
}
[root@k8s-master ssl]# 
[root@k8s-master ssl]# 

key 是 base64加密的
[root@k8s-master ssl]# 
[root@k8s-master ssl]# echo L3JlZ2lzdHJ5L3BvZHMvZGVmYXVsdC9teS1uZ2lueC03NTZmNjQ1Y2Q3LTR3czZr|base64 -d
/registry/pods/default/my-nginx-756f645cd7-4ws6k[root@k8s-master ssl]# ^C

但是为什么值 还有乱码
[root@k8s-master ssl]# echo azhzAAoJCgJ2MRIDUG9kEtUICvoBChlteS1uZ2lueC03NTZmNjQ1Y2Q3LTR3czZrEhRteS1uZ2lueC03NTZmNjQ1Y2Q3LRoHZGVmYXVsdCIAKiRjM2U2ZDE3ZS03ZjJlLTExZWItOTY4OC1mYTI3MDAwNGIwMGQyADgAQggI99GSggYQAFofChFwb2QtdGVtcGxhdGUtaGFzaBIKNzU2ZjY0NWNkN1oPCgNydW4SCG15LW5naW54alQKClJlcGxpY2FTZXQaE215LW5naW54LTc1NmY2NDVjZDciJGMzZTE4NzlkLTdmMmUtMTFlYi05Njg4LWZhMjcwMDA0YjAwZCoHYXBwcy92MTABOAF6ABKlAwoxChNkZWZhdWx0LXRva2VuLTY5Yzk1EhoyGAoTZGVmYXVsdC10b2tlbi02OWM5NRikAxKcAQoIbXktbmdpbngSBW5naW54KgAyDQoAEAAYUCIDVENQKgBCAEpIChNkZWZhdWx0LXRva2VuLTY5Yzk1EAEaLS92YXIvcnVuL3NlY3JldHMva3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudCIAahQvZGV2L3Rlcm1pbmF0aW9uLWxvZ3IGQWx3YXlzgAEAiAEAkAEAogEERmlsZRoGQWx3YXlzIB4yDENsdXN0ZXJGaXJzdEIHZGVmYXVsdEoHZGVmYXVsdFILMTkyLjE2OC4wLjVYAGAAaAByAIIBAIoBAJoBEWRlZmF1bHQtc2NoZWR1bGVysgE2Chxub2RlLmt1YmVybmV0ZXMuaW8vbm90LXJlYWR5EgZFeGlzdHMaACIJTm9FeGVjdXRlKKwCsgE4Ch5ub2RlLmt1YmVybmV0ZXMuaW8vdW5yZWFjaGFibGUSBkV4aXN0cxoAIglOb0V4ZWN1dGUorALCAQDIAQAarQMKB1J1bm5pbmcSIwoLSW5pdGlhbGl6ZWQSBFRydWUaACIICPfRkoIGEAAqADIAEh0KBVJlYWR5EgRUcnVlGgAiCAjL0pKCBhAAKgAyABInCg9Db250YWluZXJzUmVhZHkSBFRydWUaACIICMvSkoIGEAAqADIAEiQKDFBvZFNjaGVkdWxlZBIEVHJ1ZRoAIggI99GSggYQACoAMgAaACIAKgsxOTIuMTY4LjAuNTILMTcyLjE3LjgzLjI6CAj30ZKCBhAAQtgBCghteS1uZ2lueBIMEgoKCAjK0pKCBhAAGgAgASgAMgxuZ2lueDpsYXRlc3Q6X2RvY2tlci1wdWxsYWJsZTovL25naW54QHNoYTI1NjpmMzY5M2ZlNTBkNWIxZGYxZWNkMzE1ZDU0ODEzYTc3YWZkNTZiMDI0NWE0MDQwNTVhOTQ2NTc0ZGViNmIzNGZjQklkb2NrZXI6Ly9iNDc4NTBmYWY2NGM1YjFiZWRjNjg0M2EzNzZlZTA1YTVlOGFmZmU4Y2VlZGNlMzNhOWJjNzQxY2EzNDVlOGRjSgpCZXN0RWZmb3J0WgAaACIA|base64 -d
k8s

v1Pod�
�
my-nginx-756f645cd7-4ws6kmy-nginx-756f645cd7-default"*$c3e6d17e-7f2e-11eb-9688-fa270004b00d2�ђ�Z
pod-template-hash
756f645cd7Z
rumy-nginxjT

ReplicaSetmy-nginx-756f645cd7"$c3e1879d-7f2e-11eb-9688-fa270004b00d*apps/v108z�
1
default-token-69c952
default-token-69c95��
my-nginxnginx*2
P"TCP*BJH
default-token-69c95-/var/run/secrets/kubernetes.io/serviceaccount"j/dev/termination-logrAlways����FileAlways 2
                                                                                                              ClusterFirstBdefaultJdefaultR
          192.168.0.5X`hr���default-scheduler�6
node.kubernetes.io/not-readyExists"     NoExecute(��8
node.kubernetes.io/unreachableExists"   NoExecute(����
Running#

InitializedTru�ђ�*2
ReadyTru�Ғ�*2'
ContainersReadyTru�Ғ�*2$
```



https://github.com/openshift/origin/tree/master/tools/etcdhelper

value是乱码的原因找到了。因为使用l proto存储。有个工具可以解决这个显示问题，见上面的链接。

**参考的操作链接:**

https://jimmysong.io/kubernetes-handbook/guide/using-etcdctl-to-access-kubernetes-data.html

https://yq.aliyun.com/articles/561888

<br>