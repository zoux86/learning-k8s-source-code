本章节的目标就是弄懂kube-apiserver的实现细节。从本质来说，kube-apiserver就是一个go server服务器端。

假设我要实现kube-apiserver，我想到的要考虑的以下的事情，因此这章节的目标就是弄清楚这些问题。

（0）apiserver的启动流程是怎么样的。        见：第5，6，7节

（1）如何和etcd存储打通                                见：8节

（2）如何启动restful服务                                见：第7节

（3）k8s这么多资源，是怎么注册的，如何进行多版本的资源管理。     见：第4节，scheme介绍

（4）k8s的认证，授权，以及webhook

（5）为什么需要聚合apiserver，如何聚合

（6）怎样支持list-watcher

（7）crd资源是如何支持的

（8）一个request，经历了哪些流程

<br>



