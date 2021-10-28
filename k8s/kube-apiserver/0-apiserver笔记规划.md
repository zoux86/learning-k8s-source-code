本章节的目标就是弄懂kube-apiserver的实现细节。从本质来说，kube-apiserver就是一个go server服务器端。

假设我要实现kube-apiserver，我想到的要考虑的以下的事情：

（1）apiserver的启动流程是怎么样的

（2）k8s这么多资源，是怎么注册的，如何进行多版本的资源管理

（3）如何和etcd存储打通

（4）一个request，经历了哪些流程

（5）认证，授权，Admission是如何实现的

（6）apiserver是如何处理create, update, delete请求的

<br>

因此这章节的目标就是弄清楚上诉的问题
