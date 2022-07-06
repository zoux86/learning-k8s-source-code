本节算是apiserver源码分析的总纲，apiserver启动流程如下所示：

（1）Pod, svc,node 等资源注册

（2）apiserver cobra命令行解析

（3）RunE运行Run(completedOptions, genericapiserver.SetupSignalHandler()) 核心函数

Run函数核心逻辑如下：

Run函数核心调用链逻辑如下：

* CreateServerChian
  * createNodeDialer
  * CreateKubeAPIServerConfig
  * createAPIExtensionsConfig
  * createAPIExtensionsServer
  * CreateKubeAPIServer
  * createAggregatorConfig
  * createAggregatorServer
  * BuildInsecureHandlerChain

* PrePareRun
  * 添加openAPI，installHealthz，installLivez，AddPreShutdownHook。可以认为是一些监控检查，swagger接口等准备工作
* Run
  * 运行NonBlockingRun函数，核心是开启审计服务，并且开启https服务

<br>

因此接下里的源码分析归纳为以下流程：

（1）资源注册。

（2）Cobra命令行参数解析

（3）创建APIServer通用配置

（4）创建APIExtensionsServer

（5）创建KubeAPIServer

（6）创建AggregatorServer

（7）启动HTTP服务。

（8）启动HTTPS服务