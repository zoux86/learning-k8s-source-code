### 1. 定义-main

cmd\kube-controller-manager\controller-manager.go

```
func main() {
	rand.Seed(time.Now().UTC().UnixNano())

	command := app.NewControllerManagerCommand()

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	pflag.CommandLine.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

<br>

```go
// NewControllerManagerCommand creates a *cobra.Command object with default parameters
func NewControllerManagerCommand() *cobra.Command {
    // 1.初始化config配置。包括每个controller的配置，例如hpacontroller的 HorizontalPodAutoscalerSyncPeriod
    // 详见 cmd\kube-controller-manager\app\options\options.go
	s, err := options.NewKubeControllerManagerOptions()
	if err != nil {
		glog.Fatalf("unable to initialize command options: %v", err)
	}

	cmd := &cobra.Command{
		Use: "kube-controller-manager",
		Long: `The Kubernetes controller manager is a daemon that embeds
the core control loops shipped with Kubernetes. In applications of robotics and
automation, a control loop is a non-terminating loop that regulates the state of
the system. In Kubernetes, a controller is a control loop that watches the shared
state of the cluster through the apiserver and makes changes attempting to move the
current state towards the desired state. Examples of controllers that ship with
Kubernetes today are the replication controller, endpoints controller, namespace
controller, and serviceaccounts controller.`,
		Run: func(cmd *cobra.Command, args []string) {
		    // 打印一些信息
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cmd.Flags())
           // 2. 实例化一个kubecontrollerconfig.Config
			c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
			if err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
            // 最关键的Run,这里是 neverStop
			if err := Run(c.Complete(), wait.NeverStop); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}
   
	fs := cmd.Flags()
     // 定义cobra的flags，这里就是定义参数的名称，默认值啥的。例如 --url --port等
	namedFlagSets := s.Flags(KnownControllers(), ControllersDisabledByDefault.List())
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}
	
	//4.设置 help, usage函数
	usageFmt := "Usage:\n  %s\n"
	cols, _, _ := apiserverflag.TerminalSize(cmd.OutOrStdout())
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine())
		apiserverflag.PrintSections(cmd.OutOrStderr(), namedFlagSets, cols)
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
		apiserverflag.PrintSections(cmd.OutOrStdout(), namedFlagSets, cols)
	})

	return cmd
}
```

<br>

这个就是 s.flags

namedFlagSets := s.Flags(KnownControllers(), ControllersDisabledByDefault.List())

```
// Flags returns flags for a specific APIServer by section name
// 依次调用其他controller-manager的flags。
func (s *KubeControllerManagerOptions) Flags(allControllers []string, disabledByDefaultControllers []string) apiserverflag.NamedFlagSets {
	fss := apiserverflag.NamedFlagSets{}
	s.Generic.AddFlags(&fss, allControllers, disabledByDefaultControllers)
	s.KubeCloudShared.AddFlags(fss.FlagSet("generic"))
	s.ServiceController.AddFlags(fss.FlagSet("service controller"))

	s.SecureServing.AddFlags(fss.FlagSet("secure serving"))
	s.InsecureServing.AddUnqualifiedFlags(fss.FlagSet("insecure serving"))
	s.Authentication.AddFlags(fss.FlagSet("authentication"))
	s.Authorization.AddFlags(fss.FlagSet("authorization"))

	s.AttachDetachController.AddFlags(fss.FlagSet("attachdetach controller"))
	s.CSRSigningController.AddFlags(fss.FlagSet("csrsigning controller"))
	s.DeploymentController.AddFlags(fss.FlagSet("deployment controller"))
	s.DaemonSetController.AddFlags(fss.FlagSet("daemonset controller"))
	s.DeprecatedFlags.AddFlags(fss.FlagSet("deprecated"))
	s.EndpointController.AddFlags(fss.FlagSet("endpoint controller"))
	s.GarbageCollectorController.AddFlags(fss.FlagSet("garbagecollector controller"))
	s.HPAController.AddFlags(fss.FlagSet("horizontalpodautoscaling controller"))
	s.JobController.AddFlags(fss.FlagSet("job controller"))
	s.NamespaceController.AddFlags(fss.FlagSet("namespace controller"))
	s.NodeIPAMController.AddFlags(fss.FlagSet("nodeipam controller"))
	s.NodeLifecycleController.AddFlags(fss.FlagSet("nodelifecycle controller"))
	s.PersistentVolumeBinderController.AddFlags(fss.FlagSet("persistentvolume-binder controller"))
	s.PodGCController.AddFlags(fss.FlagSet("podgc controller"))
	s.ReplicaSetController.AddFlags(fss.FlagSet("replicaset controller"))
	s.ReplicationController.AddFlags(fss.FlagSet("replicationcontroller"))
	s.ResourceQuotaController.AddFlags(fss.FlagSet("resourcequota controller"))
	s.SAController.AddFlags(fss.FlagSet("serviceaccount controller"))
	s.TTLAfterFinishedController.AddFlags(fss.FlagSet("ttl-after-finished controller"))

	fs := fss.FlagSet("misc")
	fs.StringVar(&s.Master, "master", s.Master, "The address of the Kubernetes API server (overrides any value in kubeconfig).")
	fs.StringVar(&s.Kubeconfig, "kubeconfig", s.Kubeconfig, "Path to kubeconfig file with authorization and master location information.")
	var dummy string
	fs.MarkDeprecated("insecure-experimental-approve-all-kubelet-csrs-for-group", "This flag does nothing.")
	fs.StringVar(&dummy, "insecure-experimental-approve-all-kubelet-csrs-for-group", "", "This flag does nothing.")
	utilfeature.DefaultFeatureGate.AddFlag(fss.FlagSet("generic"))

	return fss
}
```



```
// AddFlags adds flags related to DeploymentController for controller manager to the specified FlagSet.
func (o *DeploymentControllerOptions) AddFlags(fs *pflag.FlagSet) {
	if o == nil {
		return
	}

	fs.Int32Var(&o.ConcurrentDeploymentSyncs, "concurrent-deployment-syncs", o.ConcurrentDeploymentSyncs, "The number of deployment objects that are allowed to sync concurrently. Larger number = more responsive deployments, but more CPU (and network) load")
	fs.DurationVar(&o.DeploymentControllerSyncPeriod.Duration, "deployment-controller-sync-period", o.DeploymentControllerSyncPeriod.Duration, "Period for syncing the deployments.")
}
```

比如，以DeploymentControllerOptions.AddFlags为例，这里就是定义了concurrent-deployment-syncs，deployment-controller-sync-period这两个参数，并且赋了默认值。

参考附录，可以加深理解。

<br>

#### 1.1 NewKubeControllerManagerOptions

看起来这里是通过获取默认的参数配置，然后赋值给 KubeControllerManagerOptions

```
// NewKubeControllerManagerOptions creates a new KubeControllerManagerOptions with a default config.
func NewKubeControllerManagerOptions() (*KubeControllerManagerOptions, error) {
   componentConfig, err := NewDefaultComponentConfig(ports.InsecureKubeControllerManagerPort)
   if err != nil {
      return nil, err
   }

   s := KubeControllerManagerOptions{
      Generic:         cmoptions.NewGenericControllerManagerConfigurationOptions(componentConfig.Generic),
      KubeCloudShared: cmoptions.NewKubeCloudSharedOptions(componentConfig.KubeCloudShared),
      AttachDetachController: &AttachDetachControllerOptions{
         ReconcilerSyncLoopPeriod: componentConfig.AttachDetachController.ReconcilerSyncLoopPeriod,
      },
      CSRSigningController: &CSRSigningControllerOptions{
         ClusterSigningCertFile: componentConfig.CSRSigningController.ClusterSigningCertFile,
         ClusterSigningKeyFile:  componentConfig.CSRSigningController.ClusterSigningKeyFile,
         ClusterSigningDuration: componentConfig.CSRSigningController.ClusterSigningDuration,
      },
      DaemonSetController: &DaemonSetControllerOptions{
         ConcurrentDaemonSetSyncs: componentConfig.DaemonSetController.ConcurrentDaemonSetSyncs,
      },
      DeploymentController: &DeploymentControllerOptions{
         ConcurrentDeploymentSyncs:      componentConfig.DeploymentController.ConcurrentDeploymentSyncs,
         DeploymentControllerSyncPeriod: componentConfig.DeploymentController.DeploymentControllerSyncPeriod,
      },
      DeprecatedFlags: &DeprecatedControllerOptions{
         RegisterRetryCount: componentConfig.DeprecatedController.RegisterRetryCount,
      },
      EndpointController: &EndpointControllerOptions{
         ConcurrentEndpointSyncs: componentConfig.EndpointController.ConcurrentEndpointSyncs,
      },
      GarbageCollectorController: &GarbageCollectorControllerOptions{
         ConcurrentGCSyncs:      componentConfig.GarbageCollectorController.ConcurrentGCSyncs,
         EnableGarbageCollector: componentConfig.GarbageCollectorController.EnableGarbageCollector,
      },
      HPAController: &HPAControllerOptions{
         HorizontalPodAutoscalerSyncPeriod:                   componentConfig.HPAController.HorizontalPodAutoscalerSyncPeriod,
         HorizontalPodAutoscalerUpscaleForbiddenWindow:       componentConfig.HPAController.HorizontalPodAutoscalerUpscaleForbiddenWindow,
         HorizontalPodAutoscalerDownscaleForbiddenWindow:     componentConfig.HPAController.HorizontalPodAutoscalerDownscaleForbiddenWindow,
         HorizontalPodAutoscalerDownscaleStabilizationWindow: componentConfig.HPAController.HorizontalPodAutoscalerDownscaleStabilizationWindow,
         HorizontalPodAutoscalerCPUInitializationPeriod:      componentConfig.HPAController.HorizontalPodAutoscalerCPUInitializationPeriod,
         HorizontalPodAutoscalerInitialReadinessDelay:        componentConfig.HPAController.HorizontalPodAutoscalerInitialReadinessDelay,
         HorizontalPodAutoscalerTolerance:                    componentConfig.HPAController.HorizontalPodAutoscalerTolerance,
         HorizontalPodAutoscalerUseRESTClients:               componentConfig.HPAController.HorizontalPodAutoscalerUseRESTClients,
      },
      JobController: &JobControllerOptions{
         ConcurrentJobSyncs: componentConfig.JobController.ConcurrentJobSyncs,
      },
      NamespaceController: &NamespaceControllerOptions{
         NamespaceSyncPeriod:      componentConfig.NamespaceController.NamespaceSyncPeriod,
         ConcurrentNamespaceSyncs: componentConfig.NamespaceController.ConcurrentNamespaceSyncs,
      },
      NodeIPAMController: &NodeIPAMControllerOptions{
         NodeCIDRMaskSize: componentConfig.NodeIPAMController.NodeCIDRMaskSize,
      },
      NodeLifecycleController: &NodeLifecycleControllerOptions{
         EnableTaintManager:     componentConfig.NodeLifecycleController.EnableTaintManager,
         NodeMonitorGracePeriod: componentConfig.NodeLifecycleController.NodeMonitorGracePeriod,
         NodeStartupGracePeriod: componentConfig.NodeLifecycleController.NodeStartupGracePeriod,
         PodEvictionTimeout:     componentConfig.NodeLifecycleController.PodEvictionTimeout,
      },
      PersistentVolumeBinderController: &PersistentVolumeBinderControllerOptions{
         PVClaimBinderSyncPeriod: componentConfig.PersistentVolumeBinderController.PVClaimBinderSyncPeriod,
         VolumeConfiguration:     componentConfig.PersistentVolumeBinderController.VolumeConfiguration,
      },
      PodGCController: &PodGCControllerOptions{
         TerminatedPodGCThreshold: componentConfig.PodGCController.TerminatedPodGCThreshold,
      },
      ReplicaSetController: &ReplicaSetControllerOptions{
         ConcurrentRSSyncs: componentConfig.ReplicaSetController.ConcurrentRSSyncs,
      },
      ReplicationController: &ReplicationControllerOptions{
         ConcurrentRCSyncs: componentConfig.ReplicationController.ConcurrentRCSyncs,
      },
      ResourceQuotaController: &ResourceQuotaControllerOptions{
         ResourceQuotaSyncPeriod:      componentConfig.ResourceQuotaController.ResourceQuotaSyncPeriod,
         ConcurrentResourceQuotaSyncs: componentConfig.ResourceQuotaController.ConcurrentResourceQuotaSyncs,
      },
      SAController: &SAControllerOptions{
         ConcurrentSATokenSyncs: componentConfig.SAController.ConcurrentSATokenSyncs,
      },
      ServiceController: &cmoptions.ServiceControllerOptions{
         ConcurrentServiceSyncs: componentConfig.ServiceController.ConcurrentServiceSyncs,
      },
      TTLAfterFinishedController: &TTLAfterFinishedControllerOptions{
         ConcurrentTTLSyncs: componentConfig.TTLAfterFinishedController.ConcurrentTTLSyncs,
      },
      SecureServing: apiserveroptions.NewSecureServingOptions().WithLoopback(),
      InsecureServing: (&apiserveroptions.DeprecatedInsecureServingOptions{
         BindAddress: net.ParseIP(componentConfig.Generic.Address),
         BindPort:    int(componentConfig.Generic.Port),
         BindNetwork: "tcp",
      }).WithLoopback(),
      Authentication: apiserveroptions.NewDelegatingAuthenticationOptions(),
      Authorization:  apiserveroptions.NewDelegatingAuthorizationOptions(),
   }

   s.Authentication.RemoteKubeConfigFileOptional = true
   s.Authorization.RemoteKubeConfigFileOptional = true
   s.Authorization.AlwaysAllowPaths = []string{"/healthz"}

   s.SecureServing.ServerCert.CertDirectory = "/var/run/kubernetes"
   s.SecureServing.ServerCert.PairName = "kube-controller-manager"
   s.SecureServing.BindPort = ports.KubeControllerManagerPort

   gcIgnoredResources := make([]kubectrlmgrconfig.GroupResource, 0, len(garbagecollector.DefaultIgnoredResources()))
   for r := range garbagecollector.DefaultIgnoredResources() {
      gcIgnoredResources = append(gcIgnoredResources, kubectrlmgrconfig.GroupResource{Group: r.Group, Resource: r.Resource})
   }

   s.GarbageCollectorController.GCIgnoredResources = gcIgnoredResources

   return &s, nil
}
```



可以看出来这里的关键就是：

（1）config函数

（2）Run函数

<br>

#### 1.2 s.config  实例化一个kubecontrollerconfig.Config

这个函数的参数是：allControllers []string, disabledByDefaultControllers []string

核心就是：kubecontrollerconfig.Config

```
// Config return a controller manager config objective
func (s KubeControllerManagerOptions) Config(allControllers []string, disabledByDefaultControllers []string) (*kubecontrollerconfig.Config, error) {
	if err := s.Validate(allControllers, disabledByDefaultControllers); err != nil {
		return nil, err
	}

	if err := s.SecureServing.MaybeDefaultWithSelfSignedCerts("localhost", nil, []net.IP{net.ParseIP("127.0.0.1")}); err != nil {
		return nil, fmt.Errorf("error creating self-signed certificates: %v", err)
	}

	kubeconfig, err := clientcmd.BuildConfigFromFlags(s.Master, s.Kubeconfig)
	if err != nil {
		return nil, err
	}
	kubeconfig.ContentConfig.ContentType = s.Generic.ClientConnection.ContentType
	kubeconfig.QPS = s.Generic.ClientConnection.QPS
	kubeconfig.Burst = int(s.Generic.ClientConnection.Burst)

	client, err := clientset.NewForConfig(restclient.AddUserAgent(kubeconfig, KubeControllerManagerUserAgent))
	if err != nil {
		return nil, err
	}

	// shallow copy, do not modify the kubeconfig.Timeout.
	config := *kubeconfig
	config.Timeout = s.Generic.LeaderElection.RenewDeadline.Duration
	leaderElectionClient := clientset.NewForConfigOrDie(restclient.AddUserAgent(&config, "leader-election"))

	eventRecorder := createRecorder(client, KubeControllerManagerUserAgent)
    
    // 核心就是定义好这样一个结构体
	c := &kubecontrollerconfig.Config{
		Client:               client,                  //用于api-server通信
		Kubeconfig:           kubeconfig,              //kube-config
		EventRecorder:        eventRecorder,           //event上报
		LeaderElectionClient: leaderElectionClient,    //选举的客户端
	}
	if err := s.ApplyTo(c); err != nil {
		return nil, err
	}

	return c, nil
}
```

<br>

##### 1.2.1 s.applyTo

```
// ApplyTo fills up controller manager config with options.
func (s *KubeControllerManagerOptions) ApplyTo(c *kubecontrollerconfig.Config) error {
   if err := s.Generic.ApplyTo(&c.ComponentConfig.Generic); err != nil {
      return err
   }
   if err := s.KubeCloudShared.ApplyTo(&c.ComponentConfig.KubeCloudShared); err != nil {
      return err
   }
   if err := s.AttachDetachController.ApplyTo(&c.ComponentConfig.AttachDetachController); err != nil {
      return err
   }
   if err := s.CSRSigningController.ApplyTo(&c.ComponentConfig.CSRSigningController); err != nil {
      return err
   }
   if err := s.DaemonSetController.ApplyTo(&c.ComponentConfig.DaemonSetController); err != nil {
      return err
   }
   if err := s.DeploymentController.ApplyTo(&c.ComponentConfig.DeploymentController); err != nil {
      return err
   }
   if err := s.DeprecatedFlags.ApplyTo(&c.ComponentConfig.DeprecatedController); err != nil {
      return err
   }
   if err := s.EndpointController.ApplyTo(&c.ComponentConfig.EndpointController); err != nil {
      return err
   }
   if err := s.GarbageCollectorController.ApplyTo(&c.ComponentConfig.GarbageCollectorController); err != nil {
      return err
   }
   if err := s.HPAController.ApplyTo(&c.ComponentConfig.HPAController); err != nil {
      return err
   }
   if err := s.JobController.ApplyTo(&c.ComponentConfig.JobController); err != nil {
      return err
   }
   if err := s.NamespaceController.ApplyTo(&c.ComponentConfig.NamespaceController); err != nil {
      return err
   }
   if err := s.NodeIPAMController.ApplyTo(&c.ComponentConfig.NodeIPAMController); err != nil {
      return err
   }
   if err := s.NodeLifecycleController.ApplyTo(&c.ComponentConfig.NodeLifecycleController); err != nil {
      return err
   }
   if err := s.PersistentVolumeBinderController.ApplyTo(&c.ComponentConfig.PersistentVolumeBinderController); err != nil {
      return err
   }
   if err := s.PodGCController.ApplyTo(&c.ComponentConfig.PodGCController); err != nil {
      return err
   }
   if err := s.ReplicaSetController.ApplyTo(&c.ComponentConfig.ReplicaSetController); err != nil {
      return err
   }
   if err := s.ReplicationController.ApplyTo(&c.ComponentConfig.ReplicationController); err != nil {
      return err
   }
   if err := s.ResourceQuotaController.ApplyTo(&c.ComponentConfig.ResourceQuotaController); err != nil {
      return err
   }
   if err := s.SAController.ApplyTo(&c.ComponentConfig.SAController); err != nil {
      return err
   }
   if err := s.ServiceController.ApplyTo(&c.ComponentConfig.ServiceController); err != nil {
      return err
   }
   if err := s.TTLAfterFinishedController.ApplyTo(&c.ComponentConfig.TTLAfterFinishedController); err != nil {
      return err
   }
   if err := s.InsecureServing.ApplyTo(&c.InsecureServing, &c.LoopbackClientConfig); err != nil {
      return err
   }
   if err := s.SecureServing.ApplyTo(&c.SecureServing, &c.LoopbackClientConfig); err != nil {
      return err
   }
   if s.SecureServing.BindPort != 0 || s.SecureServing.Listener != nil {
      if err := s.Authentication.ApplyTo(&c.Authentication, c.SecureServing, nil); err != nil {
         return err
      }
      if err := s.Authorization.ApplyTo(&c.Authorization); err != nil {
         return err
      }
   }

   // sync back to component config
   // TODO: find more elegant way than syncing back the values.
   c.ComponentConfig.Generic.Port = int32(s.InsecureServing.BindPort)
   c.ComponentConfig.Generic.Address = s.InsecureServing.BindAddress.String()

   return nil
}
```

applyto 函数的逻辑就是根据KubeControllerManagerOptions，赋值给c *kubecontrollerconfig.Config。

这里随便找一个applyto具体实现看看就知道了

```
// ApplyTo fills up AttachDetachController config with options.
func (o *AttachDetachControllerOptions) ApplyTo(cfg *kubectrlmgrconfig.AttachDetachControllerConfiguration) error {
   if o == nil {
      return nil
   }

   cfg.DisableAttachDetachReconcilerSync = o.DisableAttachDetachReconcilerSync
   cfg.ReconcilerSyncLoopPeriod = o.ReconcilerSyncLoopPeriod

   return nil
}
```

##### 1.2.2 结构体定义

cmd\kube-controller-manager\app\config\config.go

ApplyTO函数的最终目的就是实例化这样一个结构体。

```
kubecontrollerconfig.Config
// Config is the main context object for the controller manager.
type Config struct {
	ComponentConfig kubectrlmgrconfig.KubeControllerManagerConfiguration    //这个是各种manager的config，如下

	SecureServing *apiserver.SecureServingInfo
	// LoopbackClientConfig is a config for a privileged loopback connection
	LoopbackClientConfig *restclient.Config

	// TODO: remove deprecated insecure serving
	InsecureServing *apiserver.DeprecatedInsecureServingInfo
	Authentication  apiserver.AuthenticationInfo
	Authorization   apiserver.AuthorizationInfo

	// the general kube client
	Client *clientset.Clientset

	// the client only used for leader election
	LeaderElectionClient *clientset.Clientset

	// the rest config for the master
	Kubeconfig *restclient.Config

	// the event sink
	EventRecorder record.EventRecorder
}
```

pkg\controller\apis\config\types.go

```
// KubeControllerManagerConfiguration contains elements describing kube-controller manager.
type KubeControllerManagerConfiguration struct {
	metav1.TypeMeta

	// Generic holds configuration for a generic controller-manager
	Generic GenericControllerManagerConfiguration
	// KubeCloudSharedConfiguration holds configuration for shared related features
	// both in cloud controller manager and kube-controller manager.
	KubeCloudShared KubeCloudSharedConfiguration

	// AttachDetachControllerConfiguration holds configuration for
	// AttachDetachController related features.
	AttachDetachController AttachDetachControllerConfiguration
	// CSRSigningControllerConfiguration holds configuration for
	// CSRSigningController related features.
	CSRSigningController CSRSigningControllerConfiguration
	// DaemonSetControllerConfiguration holds configuration for DaemonSetController
	// related features.
	DaemonSetController DaemonSetControllerConfiguration
	// DeploymentControllerConfiguration holds configuration for
	// DeploymentController related features.
	DeploymentController DeploymentControllerConfiguration
	// DeprecatedControllerConfiguration holds configuration for some deprecated
	// features.
	DeprecatedController DeprecatedControllerConfiguration
	// EndpointControllerConfiguration holds configuration for EndpointController
	// related features.
	EndpointController EndpointControllerConfiguration
	// GarbageCollectorControllerConfiguration holds configuration for
	// GarbageCollectorController related features.
	GarbageCollectorController GarbageCollectorControllerConfiguration
	// HPAControllerConfiguration holds configuration for HPAController related features.
	HPAController HPAControllerConfiguration
	// JobControllerConfiguration holds configuration for JobController related features.
	JobController JobControllerConfiguration
	// NamespaceControllerConfiguration holds configuration for NamespaceController
	// related features.
	NamespaceController NamespaceControllerConfiguration
	// NodeIPAMControllerConfiguration holds configuration for NodeIPAMController
	// related features.
	NodeIPAMController NodeIPAMControllerConfiguration
	// NodeLifecycleControllerConfiguration holds configuration for
	// NodeLifecycleController related features.
	NodeLifecycleController NodeLifecycleControllerConfiguration
	// PersistentVolumeBinderControllerConfiguration holds configuration for
	// PersistentVolumeBinderController related features.
	PersistentVolumeBinderController PersistentVolumeBinderControllerConfiguration
	// PodGCControllerConfiguration holds configuration for PodGCController
	// related features.
	PodGCController PodGCControllerConfiguration
	// ReplicaSetControllerConfiguration holds configuration for ReplicaSet related features.
	ReplicaSetController ReplicaSetControllerConfiguration
	// ReplicationControllerConfiguration holds configuration for
	// ReplicationController related features.
	ReplicationController ReplicationControllerConfiguration
	// ResourceQuotaControllerConfiguration holds configuration for
	// ResourceQuotaController related features.
	ResourceQuotaController ResourceQuotaControllerConfiguration
	// SAControllerConfiguration holds configuration for ServiceAccountController
	// related features.
	SAController SAControllerConfiguration
	// ServiceControllerConfiguration holds configuration for ServiceController
	// related features.
	ServiceController ServiceControllerConfiguration
	// TTLAfterFinishedControllerConfiguration holds configuration for
	// TTLAfterFinishedController related features.
	TTLAfterFinishedController TTLAfterFinishedControllerConfiguration
}
```

<br>

#### 1.3 Run

所以，经过1.1 config函数 c就是补全了所有的config

```
// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	glog.Infof("Version: %+v", version.Get())

	if cfgz, err := configz.New("componentconfig"); err == nil {
		cfgz.Set(c.ComponentConfig)
	} else {
		glog.Errorf("unable to register configz: %c", err)
	}
    
    // 1.开启http server。默认暴露的端口号:10252。用于controller-manager服务性能检测(如:/debug/profile)及暴露服务相关的metrics供promtheus用于监控。
	// Start the controller manager HTTP server
	// unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging)
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		if err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
	if c.InsecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging)
		insecureSuperuserAuthn := server.AuthenticationInfo{Authenticator: &server.InsecureSuperuser{}}
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, nil, &insecureSuperuserAuthn)
		if err := c.InsecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
    
    
    // 2. 定义好run函数
	run := func(ctx context.Context) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: c.Kubeconfig,
		}
		var clientBuilder controller.ControllerClientBuilder
		if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
			if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
				// It'c possible another controller process is creating the tokens for us.
				// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
				glog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
			}
			clientBuilder = controller.SAControllerClientBuilder{
				ClientConfig:         restclient.AnonymousClientConfig(c.Kubeconfig),
				CoreClient:           c.Client.CoreV1(),
				AuthenticationClient: c.Client.AuthenticationV1(),
				Namespace:            "kube-system",
			}
		} else {
			clientBuilder = rootClientBuilder
		}
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			glog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			glog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}
    
    // 3. 如果没有多个就直接run
	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		run(context.TODO())
		panic("unreachable")
	}

	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())
	rl, err := resourcelock.New(c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		"kube-system",
		"kube-controller-manager",
		c.LeaderElectionClient.CoreV1(),
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: c.EventRecorder,
		})
	if err != nil {
		glog.Fatalf("error creating lock: %v", err)
	}
		
	// 4.设置了选举
	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
		RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
		RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,               //leader 运行run函数，这个就是第二步定义的函数
			OnStoppedLeading: func() {           // 非leader就打印这个日志。
				glog.Fatalf("leaderelection lost")
			},
		},
	})
	panic("unreachable")
}
```

<br>

#### 1.4 run函数

这里就是初始化clientBuilder,然后就StartControllers。

```
run := func(ctx context.Context) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: c.Kubeconfig,
		}
		var clientBuilder controller.ControllerClientBuilder
		if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
			if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
				// It'c possible another controller process is creating the tokens for us.
				// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
				glog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
			}
			clientBuilder = controller.SAControllerClientBuilder{
				ClientConfig:         restclient.AnonymousClientConfig(c.Kubeconfig),
				CoreClient:           c.Client.CoreV1(),
				AuthenticationClient: c.Client.AuthenticationV1(),
				Namespace:            "kube-system",
			}
		} else {
			clientBuilder = rootClientBuilder
		}
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			glog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			glog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}
```

startController这里有一个参数是函数NewControllerInitializers，从这里可以看到有这么多controller

```
// NewControllerInitializers is a public map of named controller groups (you can start more than one in an init func)
// paired to their InitFunc.  This allows for structured downstream composition and subdivision.
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
   controllers := map[string]InitFunc{}
   controllers["endpoint"] = startEndpointController
   controllers["replicationcontroller"] = startReplicationController
   controllers["podgc"] = startPodGCController
   controllers["resourcequota"] = startResourceQuotaController
   controllers["namespace"] = startNamespaceController
   controllers["serviceaccount"] = startServiceAccountController
   controllers["garbagecollector"] = startGarbageCollectorController
   controllers["daemonset"] = startDaemonSetController
   controllers["job"] = startJobController
   controllers["deployment"] = startDeploymentController
   controllers["replicaset"] = startReplicaSetController
   controllers["horizontalpodautoscaling"] = startHPAController
   controllers["disruption"] = startDisruptionController
   controllers["statefulset"] = startStatefulSetController
   controllers["cronjob"] = startCronJobController
   controllers["csrsigning"] = startCSRSigningController
   controllers["csrapproving"] = startCSRApprovingController
   controllers["csrcleaner"] = startCSRCleanerController
   controllers["ttl"] = startTTLController
   controllers["bootstrapsigner"] = startBootstrapSignerController
   controllers["tokencleaner"] = startTokenCleanerController
   controllers["nodeipam"] = startNodeIpamController
   if loopMode == IncludeCloudLoops {
      controllers["service"] = startServiceController
      controllers["route"] = startRouteController
      // TODO: volume controller into the IncludeCloudLoops only set.
      // TODO: Separate cluster in cloud check from node lifecycle controller.
   }
   controllers["nodelifecycle"] = startNodeLifecycleController
   controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
   controllers["attachdetach"] = startAttachDetachController
   controllers["persistentvolume-expander"] = startVolumeExpandController
   controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
   controllers["pvc-protection"] = startPVCProtectionController
   controllers["pv-protection"] = startPVProtectionController
   controllers["ttl-after-finished"] = startTTLAfterFinishedController

   return controllers
}
```

<br>

#### 1.5 StartControllers

```
func StartControllers(ctx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc, unsecuredMux *mux.PathRecorderMux) error {
	// Always start the SA token controller first using a full-power client, since it needs to mint tokens for the rest
	// If this fails, just return here and fail since other controllers won't be able to get credentials.
	if _, _, err := startSATokenController(ctx); err != nil {
		return err
	}

	// Initialize the cloud provider with a reference to the clientBuilder only after token controller
	// has started in case the cloud provider uses the client builder.
	if ctx.Cloud != nil {
		ctx.Cloud.Initialize(ctx.ClientBuilder)
	}
	
	// 依次启动controller，这里为啥不用协程呢？
	for controllerName, initFn := range controllers {
		if !ctx.IsControllerEnabled(controllerName) {
			glog.Warningf("%q is disabled", controllerName)
			continue
		}

		time.Sleep(wait.Jitter(ctx.ComponentConfig.Generic.ControllerStartInterval.Duration, ControllerStartJitter))

		glog.V(1).Infof("Starting %q", controllerName)
		// 注意这里的 initFn就是NewControllerInitializers 中指定了。
		debugHandler, started, err := initFn(ctx)
		if err != nil {
			glog.Errorf("Error starting %q", controllerName)
			return err
		}
		if !started {
			glog.Warningf("Skipping %q", controllerName)
			continue
		}
		if debugHandler != nil && unsecuredMux != nil {
			basePath := "/debug/controllers/" + controllerName
			unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
			unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
		}
		glog.Infof("Started %q", controllerName)
	}

	return nil
}
```

<br>

#### 1.6 总结

##### 1.6.1 整体流程

(1) NewControllerManagerCommand中定义了 NewKubeControllerManagerOptions，名为s。同时调用这个，将命令行的参数，赋值给s

```
fs := cmd.Flags()
// 定义cobra的flags，这里就是定义参数的名称，默认值啥的。例如 --url --port等
namedFlagSets := s.Flags(KnownControllers(), ControllersDisabledByDefault.List())
for _, f := range namedFlagSets.FlagSets {
   fs.AddFlagSet(f)
}
```

（2）然后通过 s.config  实例化一个kubecontrollerconfig.Config, 名为c

（3）通过ApplyTo,将s的值赋值给 C。 这样C对应的每个controller都有自己的config

（4）然后就开始Run逻辑。

```
Run逻辑：
(1) 首先初始化clientBuilder
(2) 然后定义好真正运行的run函数。run函数依次运行所有的controller的init函数。这样每个controller的起点就是这个 init函数。
(3) 调用选举函数，leader运行run。非leader打印，失去leader锁的日志。
```

<br>

##### 1.6.2 一些思考

（1） 为啥参数赋值的时候，又要config，又要options，弄完弄清，直接想附录那样赋值不香吗？

这里的一个思考是：kube-controller-manage 想采用机制和策略分离原则，options参数主要是面向cmd.Flag的，用于用户启动kcm参数的接受。config面向kcm，具体来说是为了更方便kcm中各个控制器的启动。每个controller有自己的config。

这样的好像是：option和config参数分离。option 是通过AddFlags赋值。而config 则是ApplyTo赋值。

### 2. 附录

#### 2.1 cobra实践

```
package main

import (
	"fmt"
	"github.com/spf13/cobra"

	"flag"
)

type Config struct {
	url string
}


func main() {
	var config = &Config{}

	var rootCmd = &cobra.Command{
		Use: "test cobra",
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println(config.url)
		},
	}

	rootCmd.PersistentFlags().AddGoFlagSet(flag.CommandLine)
	rootCmd.Flags().StringVarP(&config.url, "arg-url", "", "www.baidu.com", "the url is used for connect baidu")

	rootCmd.Execute()
}

E:\goWork\src\practice>cobra.exe --arg-url aaa
aaa

E:\goWork\src\practice>cobra.exe
www.baidu.com
```

<br>

#### 2.2 k8s中的选举机制

k8s中的选举机制在Client-go包中实现。具体的做法是：多个客户端创建一起创建成功资源，哪一个goroutine 获得锁，哪一个就是主。

选择config, ep的原因在于他们被list-watcher比较少，后期由于svc, ingres等发展，现在主要是用configmap来做。

比如这个：当前kcm的锁就在k8s-master这个节点上。

```
root@k8s-master:~# kubectl get ep -n kube-system kube-controller-manager -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-master_904e0225-871d-4ff2-becc-62b58a20e3c7","leaseDurationSeconds":15,"acquireTime":"2021-07-16T21:03:58Z","renewTime":"2021-07-17T10:35:50Z","leaderTransitions":24}'
  creationTimestamp: "2021-06-05T12:50:04Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "8831710"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 5d530096-9b10-45bb-a11e-43f1f8733fa5
```

