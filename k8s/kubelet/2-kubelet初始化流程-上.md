Table of Contents
=================

  * [1. kubelet入口函数](#1-kubelet入口函数)
  * [2. NewKubeletCommand函数](#2-newkubeletcommand函数)
     * [2.1 总结](#21-总结)
  * [3. Run(kubeletServer, kubeletDeps, stopCh) 函数](#3-runkubeletserver-kubeletdeps-stopch-函数)
     * [3.1. KubeletServer  函数](#31-kubeletserver--函数)
     * [3.2. Dependencies 函数](#32-dependencies-函数)
  * [4. run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh &lt;-chan struct{})](#4-runs-optionskubeletserver-kubedeps-kubeletdependencies-stopch--chan-struct)
  * [5. RunKubelet(s, kubeDeps, s.RunOnce) 函数](#5-runkubelets-kubedeps-srunonce-函数)
     * [5.1. CreateAndInitKubelet 函数](#51-createandinitkubelet-函数)
     * [5.2 NewMainKubelet](#52-newmainkubelet)
     * [5.3 startKubelet](#53-startkubelet)
  * [6. 总结](#6-总结)
        * [补充各种 Manager](#补充各种-manager)
  * [7 参考](#7-参考)



这里主要弄懂kubelet的启动流程。先上图，图片来源: https://www.bookstack.cn/read/source-code-reading-notes/kubernetes-kubelet_init.md

![kubelet-func-chanel](../images/kubelet-func-chanel.png)



### 1. kubelet入口函数

cmd\kubelet\kubelet.go

```
func main() {
   rand.Seed(time.Now().UTC().UnixNano())

   command := app.NewKubeletCommand(server.SetupSignalHandler())
   logs.InitLogs()
   defer logs.FlushLogs()

   if err := command.Execute(); err != nil {
      fmt.Fprintf(os.Stderr, "%v\n", err)
      os.Exit(1)
   }
}
```

这里还是和k8s其他组件一样，使用了Cobra框架，核心代码如下：

```go
// 初始化命令行
command := app.NewKubeletCommand(server.SetupSignalHandler())
// 执行Execute
err := command.Execute()
```

<br>

### 2. NewKubeletCommand函数

该函数整体逻辑如下：`当前不关心初始化，验证等逻辑，现在直接奔 Run(kubeletServer, kubeletDeps, stopCh) 去`

（1）初始化参数解析，初始化cleanFlagSet，kubeletFlags，kubeletConfig。这些都是初始化kubelet时要用到的

（2）定义一个cobra命令，这里核心就是 Run函数。Run函数的核心逻辑如下：

* 针对不规范的参数输入，或者help, version情况输出帮助或者版本信息
* 加载并验证 kubelet-config是否规范
* 更加kubelet-config生成kubeletServer和kubeletDeps，这个是生成kubelet的必要条件
* 运行kubelet，核心是Run函数

（3）AddFlags的具体描述如下

https://github.com/kubernetes/kubernetes/blob/0ed33881dc4355495f623c6f22e7dd0b7632b7c0/cmd/kubelet/app/options/options.go#L323

```go
cmd\kubelet\app\server.go
// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand(stopCh <-chan struct{}) *cobra.Command {
    // 初始化参数解析，初始化cleanFlagSet，kubeletFlags，kubeletConfig。这些都是初始化kubelet时要用到的
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(flag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()
	kubeletConfig, err := options.NewKubeletConfiguration()
	// programmer error
	if err != nil {
		glog.Fatal(err)
	}
	
    // 定义一个cobra命令，这里核心就是 Run函数。
	cmd := &cobra.Command{
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object
that describes a pod. The kubelet takes a set of PodSpecs that are provided through
various mechanisms (primarily through the apiserver) and ensures that the containers
described in those PodSpecs are running and healthy. The kubelet doesn't manage
containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are three ways that a container
manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. Files under this path will be monitored
periodically for updates. The monitoring period is 20s by default and is configurable
via a flag.

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).

HTTP server: The kubelet can also listen for HTTP and respond to a simple API
(underspec'd currently) to submit a new manifest.`,
		// The Kubelet has special flag parsing requirements to enforce flag precedence rules,
		// so we do all our parsing manually in Run, below.
		// DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
		// `args` arg to Run, without Cobra's interference.
		DisableFlagParsing: true,    // 没有使用Cobra框架中的默认参数解析，而是自定义参数解析。
		Run: func(cmd *cobra.Command, args []string) {
            // 接下来都是针对不规范的参数，输出使用帮助
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				cmd.Usage()
				glog.Fatal(err)
			}

			// check if there are non-flag arguments in the command line
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				cmd.Usage()
				glog.Fatalf("unknown command: %s", cmds[0])
			}
            
            // 输出help
			// short-circuit on help
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				glog.Fatal(`"help" flag is non-bool, programmer error, please correct`)
			}
			if help {
				cmd.Help()
				return
			}
            
            // 输出version
			// short-circuit on verflag
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cleanFlagSet)

            // 加载并校验kubelet config。其中包括校验初始化的kubeletFlags，并从kubeletFlags的KubeletConfigFile参数获取kubelet config的内容。
			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				glog.Fatal(err)
			}

			// validate the initial KubeletFlags
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				glog.Fatal(err)
			}

			if kubeletFlags.ContainerRuntime == "remote" && cleanFlagSet.Changed("pod-infra-container-image") {
				glog.Warning("Warning: For remote container runtime, --pod-infra-container-image is ignored in kubelet, which should be set in that remote runtime instead")
			}

			// load kubelet config file, if provided
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					glog.Fatal(err)
				}
				// We must enforce flag precedence by re-parsing the command line into the new object.
				// This is necessary to preserve backwards-compatibility across binary upgrades.
				// See issue #56171 for more details.
				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					glog.Fatal(err)
				}
				// update feature gates based on new config
				if err := utilfeature.DefaultFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					glog.Fatal(err)
				}
			}

			// We always validate the local configuration (command line + config file).
			// This is the default "last-known-good" config for dynamic config, and must always remain valid.
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig); err != nil {
				glog.Fatal(err)
			}
            
            // 设置了，就使用动态kubelet config
			// use dynamic kubelet config, if enabled
			var kubeletConfigController *dynamickubeletconfig.Controller
			if dynamicConfigDir := kubeletFlags.DynamicConfigDir.Value(); len(dynamicConfigDir) > 0 {
				var dynamicKubeletConfig *kubeletconfiginternal.KubeletConfiguration
				dynamicKubeletConfig, kubeletConfigController, err = BootstrapKubeletConfigController(dynamicConfigDir,
					func(kc *kubeletconfiginternal.KubeletConfiguration) error {
						// Here, we enforce flag precedence inside the controller, prior to the controller's validation sequence,
						// so that we get a complete validation at the same point where we can decide to reject dynamic config.
						// This fixes the flag-precedence component of issue #63305.
						// See issue #56171 for general details on flag precedence.
						return kubeletConfigFlagPrecedence(kc, args)
					})
				if err != nil {
					glog.Fatal(err)
				}
				// If we should just use our existing, local config, the controller will return a nil config
				if dynamicKubeletConfig != nil {
					kubeletConfig = dynamicKubeletConfig
					// Note: flag precedence was already enforced in the controller, prior to validation,
					// by our above transform function. Now we simply update feature gates from the new config.
					if err := utilfeature.DefaultFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
						glog.Fatal(err)
					}
				}
			}
           
            // 初始化kubeletServer和kubeletDeps
			// construct a KubeletServer from kubeletFlags and kubeletConfig
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}

			// use kubeletServer to construct the default KubeletDeps
			kubeletDeps, err := UnsecuredDependencies(kubeletServer)
			if err != nil {
				glog.Fatal(err)
			}

			// add the kubelet config controller to kubeletDeps
			kubeletDeps.KubeletConfigController = kubeletConfigController
			
            // 如果开启了docker-shim 实验特效，则执行RunDockershim，这个只是调试用的，一般是false不开的
			// start the experimental docker shim, if enabled
			if kubeletServer.KubeletFlags.ExperimentalDockershim {
				if err := RunDockershim(&kubeletServer.KubeletFlags, kubeletConfig, stopCh); err != nil {
					glog.Fatal(err)
				}
				return
			}
		 
			// run the kubelet，运行kubelet
			glog.V(5).Infof("KubeletConfiguration: %#v", kubeletServer.KubeletConfiguration)
			if err := Run(kubeletServer, kubeletDeps, stopCh); err != nil {
				glog.Fatal(err)
			}
		},
	}
		
 
	// keep cleanFlagSet separate, so Cobra doesn't pollute it with the global flags
	kubeletFlags.AddFlags(cleanFlagSet)
	options.AddKubeletConfigFlags(cleanFlagSet, kubeletConfig)
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```



<br>

### 3. Run(kubeletServer, kubeletDeps, stopCh) 函数

核心就是调用

```go
cmd\kubelet\app\server.go
// Run runs the specified KubeletServer with the given Dependencies. This should never exit.
// The kubeDeps argument may be nil - if so, it is initialized from the settings on KubeletServer.
// Otherwise, the caller is assumed to have set up the Dependencies object and a default one will
// not be generated.
func Run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())
  // 当运行环境是Windows的时候，初始化操作，但是该操作为空，只是预留。具体执行run(s, kubeDeps, stopCh)函数。
	if err := initForOS(s.KubeletFlags.WindowsService); err != nil {
		return fmt.Errorf("failed OS init: %v", err)
	}
	if err := run(s, kubeDeps, featureGate, stopCh); err != nil {
		return fmt.Errorf("failed to run Kubelet: %v", err)
	}
	return nil
}
```

<br>

这里先看一下函数参数 s-kubeletServer和kubeletDeps是什么

#### 3.1. KubeletServer 

KubeletServer  就是配置参数

基本看参数名字和注释就能知道什么意思

```
// KubeletServer encapsulates all of the parameters necessary for starting up
// a kubelet. These can either be set via command line or directly.
type KubeletServer struct {
	KubeletFlags
	kubeletconfig.KubeletConfiguration
}
```

```
// A configuration field should go in KubeletFlags instead of KubeletConfiguration if any of these are true:
// - its value will never, or cannot safely be changed during the lifetime of a node
// - its value cannot be safely shared between nodes at the same time (e.g. a hostname)
//   KubeletConfiguration is intended to be shared between nodes
// In general, please try to avoid adding flags or configuration fields,
// we already have a confusingly large amount of them.
type KubeletFlags struct {
	KubeConfig          string            // 连接kmaster集群用的
	BootstrapKubeconfig string            // 申请进入集群的config

	// Insert a probability of random errors during calls to the master.
	ChaosChance float64
	// Crash immediately, rather than eating panics.
	ReallyCrashForTesting bool

	// TODO(mtaufen): It is increasingly looking like nobody actually uses the
	//                Kubelet's runonce mode anymore, so it may be a candidate
	//                for deprecation and removal.
	// If runOnce is true, the Kubelet will check the API server once for pods,
	// run those in addition to the pods specified by static pod files, and exit.
	RunOnce bool

	// enableServer enables the Kubelet's server
	EnableServer bool

	// HostnameOverride is the hostname used to identify the kubelet instead
	// of the actual hostname.
	HostnameOverride string
	// NodeIP is IP address of the node.
	// If set, kubelet will use this IP address for the node.
	NodeIP string

	// This flag, if set, sets the unique id of the instance that an external provider (i.e. cloudprovider)
	// can use to identify a specific node
	ProviderID string
   
  // 包括了cni,cri等配置，比如cni bin目录
	// Container-runtime-specific options.
	config.ContainerRuntimeOptions

	// certDirectory is the directory where the TLS certs are located (by
	// default /var/run/kubernetes). If tlsCertFile and tlsPrivateKeyFile
	// are provided, this flag will be ignored.
	CertDirectory string

	// cloudProvider is the provider for cloud services.
	// +optional
	CloudProvider string

	// cloudConfigFile is the path to the cloud provider configuration file.
	// +optional
	CloudConfigFile string

	// rootDirectory is the directory path to place kubelet files (volume
	// mounts,etc).
	// kubelet的root目录，存放挂载, mount,etc等信息。默认是/var/lib/kubelet, 可以通过kubelet启动参数--root-dir 修改
	RootDirectory string

	// The Kubelet will use this directory for checkpointing downloaded configurations and tracking configuration health.
	// The Kubelet will create this directory if it does not already exist.
	// The path may be absolute or relative; relative paths are under the Kubelet's current working directory.
	// Providing this flag enables dynamic kubelet configuration.
	// To use this flag, the DynamicKubeletConfig feature gate must be enabled.
	DynamicConfigDir flag.StringFlag

	// The Kubelet will load its initial configuration from this file.
	// The path may be absolute or relative; relative paths are under the Kubelet's current working directory.
	// Omit this flag to use the combination of built-in default configuration values and flags.
	// 通过--config指定一个文件，里面是指定的Kubelet配置，比如maxPods=100等
	KubeletConfigFile string

	// registerNode enables automatic registration with the apiserver.
	RegisterNode bool

	// registerWithTaints are an array of taints to add to a node object when
	// the kubelet registers itself. This only takes effect when registerNode
	// is true and upon the initial registration of the node.
	RegisterWithTaints []core.Taint

	// WindowsService should be set to true if kubelet is running as a service on Windows.
	// Its corresponding flag only gets registered in Windows builds.
	WindowsService bool

	// EXPERIMENTAL FLAGS
	// Whitelist of unsafe sysctls or sysctl patterns (ending in *).
	// +optional
	AllowedUnsafeSysctls []string
	// containerized should be set to true if kubelet is running in a container.
	Containerized bool
	// remoteRuntimeEndpoint is the endpoint of remote runtime service
	// docker, containerd等容器运行时接口地址，默认是docker
	RemoteRuntimeEndpoint string
	// remoteImageEndpoint is the endpoint of remote image service
	RemoteImageEndpoint string
	// experimentalMounterPath is the path of mounter binary. Leave empty to use the default mount path
	ExperimentalMounterPath string
	// If enabled, the kubelet will integrate with the kernel memcg notification to determine if memory eviction thresholds are crossed rather than polling.
	// +optional
	ExperimentalKernelMemcgNotification bool
	// This flag, if set, enables a check prior to mount operations to verify that the required components
	// (binaries, etc.) to mount the volume are available on the underlying node. If the check is enabled
	// and fails the mount operation fails.
	ExperimentalCheckNodeCapabilitiesBeforeMount bool
	// This flag, if set, will avoid including `EvictionHard` limits while computing Node Allocatable.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node-allocatable.md) doc for more information.
	ExperimentalNodeAllocatableIgnoreEvictionThreshold bool
	// Node Labels are the node labels to add when registering the node in the cluster
	NodeLabels map[string]string
	// volumePluginDir is the full path of the directory in which to search
	// for additional third party volume plugins
	VolumePluginDir string
	// lockFilePath is the path that kubelet will use to as a lock file.
	// It uses this file as a lock to synchronize with other kubelet processes
	// that may be running.
	LockFilePath string
	// ExitOnLockContention is a flag that signifies to the kubelet that it is running
	// in "bootstrap" mode. This requires that 'LockFilePath' has been set.
	// This will cause the kubelet to listen to inotify events on the lock file,
	// releasing it and exiting when another process tries to open that file.
	ExitOnLockContention bool
	// seccompProfileRoot is the directory path for seccomp profiles.
	SeccompProfileRoot string
	// bootstrapCheckpointPath is the path to the directory containing pod checkpoints to
	// run on restore
	BootstrapCheckpointPath string
	// NodeStatusMaxImages caps the number of images reported in Node.Status.Images.
	// This is an experimental, short-term flag to help with node scalability.
	NodeStatusMaxImages int32

	// DEPRECATED FLAGS
	// minimumGCAge is the minimum age for a finished container before it is
	// garbage collected.
	MinimumGCAge metav1.Duration
	// maxPerPodContainerCount is the maximum number of old instances to
	// retain per container. Each container takes up some disk space.
	MaxPerPodContainerCount int32
	// maxContainerCount is the maximum number of old instances of containers
	// to retain globally. Each container takes up some disk space.
	MaxContainerCount int32
	// masterServiceNamespace is The namespace from which the kubernetes
	// master services should be injected into pods.
	MasterServiceNamespace string
	// registerSchedulable tells the kubelet to register the node as
	// schedulable. Won't have any effect if register-node is false.
	// DEPRECATED: use registerWithTaints instead
	RegisterSchedulable bool
	// nonMasqueradeCIDR configures masquerading: traffic to IPs outside this range will use IP masquerade.
	NonMasqueradeCIDR string
	// This flag, if set, instructs the kubelet to keep volumes from terminated pods mounted to the node.
	// This can be useful for debugging volume related issues.
	KeepTerminatedPodVolumes bool
	// allowPrivileged enables containers to request privileged mode.
	// Defaults to true.
	AllowPrivileged bool
	// hostNetworkSources is a comma-separated list of sources from which the
	// Kubelet allows pods to use of host network. Defaults to "*". Valid
	// options are "file", "http", "api", and "*" (all sources).
	HostNetworkSources []string
	// hostPIDSources is a comma-separated list of sources from which the
	// Kubelet allows pods to use the host pid namespace. Defaults to "*".
	HostPIDSources []string
	// hostIPCSources is a comma-separated list of sources from which the
	// Kubelet allows pods to use the host ipc namespace. Defaults to "*".
	HostIPCSources []string
}
```

```
// KubeletConfiguration contains the configuration for the Kubelet
type KubeletConfiguration struct {
	metav1.TypeMeta

	// staticPodPath is the path to the directory containing local (static) pods to
	// run, or the path to a single static pod file.
	StaticPodPath string
	// syncFrequency is the max period between synchronizing running
	// containers and config
	SyncFrequency metav1.Duration
	// fileCheckFrequency is the duration between checking config files for
	// new data
	FileCheckFrequency metav1.Duration
	// httpCheckFrequency is the duration between checking http for new data
	HTTPCheckFrequency metav1.Duration
	// staticPodURL is the URL for accessing static pods to run
	StaticPodURL string
	// staticPodURLHeader is a map of slices with HTTP headers to use when accessing the podURL
	StaticPodURLHeader map[string][]string
	// address is the IP address for the Kubelet to serve on (set to 0.0.0.0
	// for all interfaces)
	Address string
	// port is the port for the Kubelet to serve on.
	Port int32
	// readOnlyPort is the read-only port for the Kubelet to serve on with
	// no authentication/authorization (set to 0 to disable)
	ReadOnlyPort int32
	// tlsCertFile is the file containing x509 Certificate for HTTPS.  (CA cert,
	// if any, concatenated after server cert). If tlsCertFile and
	// tlsPrivateKeyFile are not provided, a self-signed certificate
	// and key are generated for the public address and saved to the directory
	// passed to the Kubelet's --cert-dir flag.
	TLSCertFile string
	// tlsPrivateKeyFile is the file containing x509 private key matching tlsCertFile
	TLSPrivateKeyFile string
	// TLSCipherSuites is the list of allowed cipher suites for the server.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
	TLSCipherSuites []string
	// TLSMinVersion is the minimum TLS version supported.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
	TLSMinVersion string
	// rotateCertificates enables client certificate rotation. The Kubelet will request a
	// new certificate from the certificates.k8s.io API. This requires an approver to approve the
	// certificate signing requests. The RotateKubeletClientCertificate feature
	// must be enabled.
	RotateCertificates bool
	// serverTLSBootstrap enables server certificate bootstrap. Instead of self
	// signing a serving certificate, the Kubelet will request a certificate from
	// the certificates.k8s.io API. This requires an approver to approve the
	// certificate signing requests. The RotateKubeletServerCertificate feature
	// must be enabled.
	ServerTLSBootstrap bool
	// authentication specifies how requests to the Kubelet's server are authenticated
	Authentication KubeletAuthentication
	// authorization specifies how requests to the Kubelet's server are authorized
	Authorization KubeletAuthorization
	// registryPullQPS is the limit of registry pulls per second.
	// Set to 0 for no limit.
	RegistryPullQPS int32
	// registryBurst is the maximum size of bursty pulls, temporarily allows
	// pulls to burst to this number, while still not exceeding registryPullQPS.
	// Only used if registryPullQPS > 0.
	RegistryBurst int32
	// eventRecordQPS is the maximum event creations per second. If 0, there
	// is no limit enforced.
	EventRecordQPS int32
	// eventBurst is the maximum size of a burst of event creations, temporarily
	// allows event creations to burst to this number, while still not exceeding
	// eventRecordQPS. Only used if eventRecordQPS > 0.
	EventBurst int32
	// enableDebuggingHandlers enables server endpoints for log collection
	// and local running of containers and commands
	EnableDebuggingHandlers bool
	// enableContentionProfiling enables lock contention profiling, if enableDebuggingHandlers is true.
	EnableContentionProfiling bool
	// healthzPort is the port of the localhost healthz endpoint (set to 0 to disable)
	HealthzPort int32
	// healthzBindAddress is the IP address for the healthz server to serve on
	HealthzBindAddress string
	// oomScoreAdj is The oom-score-adj value for kubelet process. Values
	// must be within the range [-1000, 1000].
	OOMScoreAdj int32
	// clusterDomain is the DNS domain for this cluster. If set, kubelet will
	// configure all containers to search this domain in addition to the
	// host's search domains.
	ClusterDomain string
	// clusterDNS is a list of IP addresses for a cluster DNS server. If set,
	// kubelet will configure all containers to use this for DNS resolution
	// instead of the host's DNS servers.
	ClusterDNS []string
	// streamingConnectionIdleTimeout is the maximum time a streaming connection
	// can be idle before the connection is automatically closed.
	StreamingConnectionIdleTimeout metav1.Duration
	// nodeStatusUpdateFrequency is the frequency that kubelet posts node
	// status to master. Note: be cautious when changing the constant, it
	// must work with nodeMonitorGracePeriod in nodecontroller.
	NodeStatusUpdateFrequency metav1.Duration
	// nodeLeaseDurationSeconds is the duration the Kubelet will set on its corresponding Lease.
	NodeLeaseDurationSeconds int32
	// imageMinimumGCAge is the minimum age for an unused image before it is
	// garbage collected.
	ImageMinimumGCAge metav1.Duration
	// imageGCHighThresholdPercent is the percent of disk usage after which
	// image garbage collection is always run. The percent is calculated as
	// this field value out of 100.
	ImageGCHighThresholdPercent int32
	// imageGCLowThresholdPercent is the percent of disk usage before which
	// image garbage collection is never run. Lowest disk usage to garbage
	// collect to. The percent is calculated as this field value out of 100.
	ImageGCLowThresholdPercent int32
	// How frequently to calculate and cache volume disk usage for all pods
	VolumeStatsAggPeriod metav1.Duration
	// KubeletCgroups is the absolute name of cgroups to isolate the kubelet in
	KubeletCgroups string
	// SystemCgroups is absolute name of cgroups in which to place
	// all non-kernel processes that are not already in a container. Empty
	// for no container. Rolling back the flag requires a reboot.
	SystemCgroups string
	// CgroupRoot is the root cgroup to use for pods.
	// If CgroupsPerQOS is enabled, this is the root of the QoS cgroup hierarchy.
	CgroupRoot string
	// Enable QoS based Cgroup hierarchy: top level cgroups for QoS Classes
	// And all Burstable and BestEffort pods are brought up under their
	// specific top level QoS cgroup.
	CgroupsPerQOS bool
	// driver that the kubelet uses to manipulate cgroups on the host (cgroupfs or systemd)
	CgroupDriver string
	// CPUManagerPolicy is the name of the policy to use.
	// Requires the CPUManager feature gate to be enabled.
	CPUManagerPolicy string
	// CPU Manager reconciliation period.
	// Requires the CPUManager feature gate to be enabled.
	CPUManagerReconcilePeriod metav1.Duration
	// Map of QoS resource reservation percentages (memory only for now).
	// Requires the QOSReserved feature gate to be enabled.
	QOSReserved map[string]string
	// runtimeRequestTimeout is the timeout for all runtime requests except long running
	// requests - pull, logs, exec and attach.
	RuntimeRequestTimeout metav1.Duration
	// hairpinMode specifies how the Kubelet should configure the container
	// bridge for hairpin packets.
	// Setting this flag allows endpoints in a Service to loadbalance back to
	// themselves if they should try to access their own Service. Values:
	//   "promiscuous-bridge": make the container bridge promiscuous.
	//   "hairpin-veth":       set the hairpin flag on container veth interfaces.
	//   "none":               do nothing.
	// Generally, one must set --hairpin-mode=hairpin-veth to achieve hairpin NAT,
	// because promiscuous-bridge assumes the existence of a container bridge named cbr0.
	HairpinMode string
	// maxPods is the number of pods that can run on this Kubelet.
	MaxPods int32
	// The CIDR to use for pod IP addresses, only used in standalone mode.
	// In cluster mode, this is obtained from the master.
	PodCIDR string
	// PodPidsLimit is the maximum number of pids in any pod.
	PodPidsLimit int64
	// ResolverConfig is the resolver configuration file used as the basis
	// for the container DNS resolution configuration.
	ResolverConfig string
	// cpuCFSQuota enables CPU CFS quota enforcement for containers that
	// specify CPU limits
	CPUCFSQuota bool
	// CPUCFSQuotaPeriod sets the CPU CFS quota period value, cpu.cfs_period_us, defaults to 100ms
	CPUCFSQuotaPeriod metav1.Duration
	// maxOpenFiles is Number of files that can be opened by Kubelet process.
	MaxOpenFiles int64
	// contentType is contentType of requests sent to apiserver.
	ContentType string
	// kubeAPIQPS is the QPS to use while talking with kubernetes apiserver
	KubeAPIQPS int32
	// kubeAPIBurst is the burst to allow while talking with kubernetes
	// apiserver
	KubeAPIBurst int32
	// serializeImagePulls when enabled, tells the Kubelet to pull images one at a time.
	SerializeImagePulls bool
	// Map of signal names to quantities that defines hard eviction thresholds. For example: {"memory.available": "300Mi"}.
	EvictionHard map[string]string
	// Map of signal names to quantities that defines soft eviction thresholds.  For example: {"memory.available": "300Mi"}.
	EvictionSoft map[string]string
	// Map of signal names to quantities that defines grace periods for each soft eviction signal. For example: {"memory.available": "30s"}.
	EvictionSoftGracePeriod map[string]string
	// Duration for which the kubelet has to wait before transitioning out of an eviction pressure condition.
	EvictionPressureTransitionPeriod metav1.Duration
	// Maximum allowed grace period (in seconds) to use when terminating pods in response to a soft eviction threshold being met.
	EvictionMaxPodGracePeriod int32
	// Map of signal names to quantities that defines minimum reclaims, which describe the minimum
	// amount of a given resource the kubelet will reclaim when performing a pod eviction while
	// that resource is under pressure. For example: {"imagefs.available": "2Gi"}
	EvictionMinimumReclaim map[string]string
	// podsPerCore is the maximum number of pods per core. Cannot exceed MaxPods.
	// If 0, this field is ignored.
	PodsPerCore int32
	// enableControllerAttachDetach enables the Attach/Detach controller to
	// manage attachment/detachment of volumes scheduled to this node, and
	// disables kubelet from executing any attach/detach operations
	EnableControllerAttachDetach bool
	// protectKernelDefaults, if true, causes the Kubelet to error if kernel
	// flags are not as it expects. Otherwise the Kubelet will attempt to modify
	// kernel flags to match its expectation.
	ProtectKernelDefaults bool
	// If true, Kubelet ensures a set of iptables rules are present on host.
	// These rules will serve as utility for various components, e.g. kube-proxy.
	// The rules will be created based on IPTablesMasqueradeBit and IPTablesDropBit.
	MakeIPTablesUtilChains bool
	// iptablesMasqueradeBit is the bit of the iptables fwmark space to mark for SNAT
	// Values must be within the range [0, 31]. Must be different from other mark bits.
	// Warning: Please match the value of the corresponding parameter in kube-proxy.
	// TODO: clean up IPTablesMasqueradeBit in kube-proxy
	IPTablesMasqueradeBit int32
	// iptablesDropBit is the bit of the iptables fwmark space to mark for dropping packets.
	// Values must be within the range [0, 31]. Must be different from other mark bits.
	IPTablesDropBit int32
	// featureGates is a map of feature names to bools that enable or disable alpha/experimental
	// features. This field modifies piecemeal the built-in default values from
	// "k8s.io/kubernetes/pkg/features/kube_features.go".
	FeatureGates map[string]bool
	// Tells the Kubelet to fail to start if swap is enabled on the node.
	FailSwapOn bool
	// A quantity defines the maximum size of the container log file before it is rotated. For example: "5Mi" or "256Ki".
	ContainerLogMaxSize string
	// Maximum number of container log files that can be present for a container.
	ContainerLogMaxFiles int32
	// ConfigMapAndSecretChangeDetectionStrategy is a mode in which config map and secret managers are running.
	ConfigMapAndSecretChangeDetectionStrategy ResourceChangeDetectionStrategy

	/* the following fields are meant for Node Allocatable */

	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G) pairs
	// that describe resources reserved for non-kubernetes components.
	// Currently only cpu and memory are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	SystemReserved map[string]string
	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G) pairs
	// that describe resources reserved for kubernetes system components.
	// Currently cpu, memory and local ephemeral storage for root file system are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	KubeReserved map[string]string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `SystemReserved` compute resource reservation for OS system daemons.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) doc for more information.
	SystemReservedCgroup string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `KubeReserved` compute resource reservation for Kubernetes node system daemons.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) doc for more information.
	KubeReservedCgroup string
	// This flag specifies the various Node Allocatable enforcements that Kubelet needs to perform.
	// This flag accepts a list of options. Acceptable options are `pods`, `system-reserved` & `kube-reserved`.
	// Refer to [Node Allocatable](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) doc for more information.
	EnforceNodeAllocatable []string
}
```

<br>

#### 3.2. Dependencies 

kubeletDeps就是运行kubelet所需要的依赖。比如docker客户端，kube客户端，csi客户端等等。

```
// Dependencies is a bin for things we might consider "injected dependencies" -- objects constructed
// at runtime that are necessary for running the Kubelet. This is a temporary solution for grouping
// these objects while we figure out a more comprehensive dependency injection story for the Kubelet.
type Dependencies struct {
	Options []Option

	// Injected Dependencies
	Auth                    server.AuthInterface
	CAdvisorInterface       cadvisor.Interface
	Cloud                   cloudprovider.Interface
	ContainerManager        cm.ContainerManager
	DockerClientConfig      *dockershim.ClientConfig
	EventClient             v1core.EventsGetter
	HeartbeatClient         clientset.Interface
	OnHeartbeatFailure      func()
	KubeClient              clientset.Interface
	CSIClient               csiclientset.Interface
	DynamicKubeClient       dynamic.Interface
	Mounter                 mount.Interface
	OOMAdjuster             *oom.OOMAdjuster
	OSInterface             kubecontainer.OSInterface
	PodConfig               *config.PodConfig
	Recorder                record.EventRecorder
	VolumePlugins           []volume.VolumePlugin
	DynamicPluginProber     volume.DynamicPluginProber
	TLSOptions              *server.TLSOptions
	KubeletConfigController *kubeletconfig.Controller
}
```

<br>

### 4. func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate, stopCh <-chan struct{}) (err error) {

再次回到run函数，它的主体逻辑如下：

kubelet  feature gates详见：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/

1. 将打开的FeatureGates用map保存
2. 对kubelet的配置进行校验，校验放在FeatureGates后的原因在于某些校验是依据FeatureGates的开关决定的
2. 如果ExitOnLockContention开启的话，就读取lockFilePath。目前很少看见用
3. 根据配置初始化kubeconfig文件
3. 判断是否为standaloneMode，如果是的话，就本地调试用的，不用连接kube-apiserver
3. 如果是使用的云厂商的服务，调用InitCloudProvider初始化
4. 如果不是standaloneMode, 初始化各种客户端，包括：
   - kubeclient
   - 事件 client：
     配置 EventRecordQPS EventBrust 参数
     调用 `k8s.io/client-go/kubernetes/typed/core/v1`中的 `NewForConfig`
   - 心跳客户端：
     配置 `QPS` `Timeout`(若开启了`NodeLease` feature，则设置 `NodeLeaseDurationSeconds` 为timeout)
   - auth client
6. 配置cgroup，kubelet在已有的cgroup上加了kubepods这一层，用来实现QOS。一般默认是 /sys/fs/cgroup/memory/kubepods
6. 初始化cadvisor
6. 初始化eventRecorder，用于上报event
6. 解析系统保留资源，这些资源是不会给pod分配的。例如：--system-reserved=cpu=2000m,memory=20000Mi
6. 设置驱逐阈值，例如--eviction-hard=memory.available<1Mi,nodefs.available<1Mi,nodefs.inodesFree<1
6. oomAdjuster设置容器进程的oom score
6. 调用RunKubelet，继续运行Kubelet核心逻辑
6. 开启监控检查服务

这里其实还是根据配置，做各种准备工作，比如客户端的初始化，实例化containerManager对象等等。接下来继续往下看看RunKubelet

```
func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate, stopCh <-chan struct{}) (err error) {
	// Set global feature gates based on the value on the initial KubeletServer
	// 1. 将打开的FeatureGates用map保存
	err = utilfeature.DefaultMutableFeatureGate.SetFromMap(s.KubeletConfiguration.FeatureGates)
	if err != nil {
		return err
	}
	// validate the initial KubeletServer (we set feature gates first, because this validation depends on feature gates)
	// 2.对kubelet的配置进行校验
	if err := options.ValidateKubeletServer(s); err != nil {
		return err
	}

	// Obtain Kubelet Lock File
	// 3. 如果ExitOnLockContention开启的话，就读取lockFilePath
	if s.ExitOnLockContention && s.LockFilePath == "" {
		return errors.New("cannot exit on lock file contention: no lock file specified")
	}
	done := make(chan struct{})
	if s.LockFilePath != "" {
		klog.Infof("acquiring file lock on %q", s.LockFilePath)
		if err := flock.Acquire(s.LockFilePath); err != nil {
			return fmt.Errorf("unable to acquire file lock on %q: %v", s.LockFilePath, err)
		}
		if s.ExitOnLockContention {
			klog.Infof("watching for inotify events for: %v", s.LockFilePath)
			if err := watchForLockfileContention(s.LockFilePath, done); err != nil {
				return err
			}
		}
	}

	// Register current configuration with /configz endpoint
	// 4.根据配置初始化kubeconfig文件
	err = initConfigz(&s.KubeletConfiguration)
	if err != nil {
		klog.Errorf("unable to register KubeletConfiguration with configz, error: %v", err)
	}
   
  // 5.判断是否为standaloneMode，如果是的话，就本地调试用的，不用连接kube-apiserver
	// About to get clients and such, detect standaloneMode
	standaloneMode := true
	if len(s.KubeConfig) > 0 {
		standaloneMode = false
	}

	if kubeDeps == nil {
		kubeDeps, err = UnsecuredDependencies(s, featureGate)
		if err != nil {
			return err
		}
	}
  
  // 6.如果是使用的云厂商的服务，调用InitCloudProvider初始化
	if kubeDeps.Cloud == nil {
		if !cloudprovider.IsExternal(s.CloudProvider) {
			cloud, err := cloudprovider.InitCloudProvider(s.CloudProvider, s.CloudConfigFile)
			if err != nil {
				return err
			}
			if cloud == nil {
				klog.V(2).Infof("No cloud provider specified: %q from the config file: %q\n", s.CloudProvider, s.CloudConfigFile)
			} else {
				klog.V(2).Infof("Successfully initialized cloud provider: %q from the config file: %q\n", s.CloudProvider, s.CloudConfigFile)
			}
			kubeDeps.Cloud = cloud
		}
	}

	hostName, err := nodeutil.GetHostname(s.HostnameOverride)
	if err != nil {
		return err
	}
	nodeName, err := getNodeName(kubeDeps.Cloud, hostName)
	if err != nil {
		return err
	}

	// if in standalone mode, indicate as much by setting all clients to nil
	// 7.如果不是standaloneMode, 初始化各种客户端
	switch {
	case standaloneMode:
		kubeDeps.KubeClient = nil
		kubeDeps.EventClient = nil
		kubeDeps.HeartbeatClient = nil
		klog.Warningf("standalone mode, no API client")

	case kubeDeps.KubeClient == nil, kubeDeps.EventClient == nil, kubeDeps.HeartbeatClient == nil:
		clientConfig, closeAllConns, err := buildKubeletClientConfig(s, nodeName)
		if err != nil {
			return err
		}
		if closeAllConns == nil {
			return errors.New("closeAllConns must be a valid function other than nil")
		}
		kubeDeps.OnHeartbeatFailure = closeAllConns

		kubeDeps.KubeClient, err = clientset.NewForConfig(clientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet client: %v", err)
		}

		// make a separate client for events
		eventClientConfig := *clientConfig
		eventClientConfig.QPS = float32(s.EventRecordQPS)
		eventClientConfig.Burst = int(s.EventBurst)
		kubeDeps.EventClient, err = v1core.NewForConfig(&eventClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet event client: %v", err)
		}

		// make a separate client for heartbeat with throttling disabled and a timeout attached
		heartbeatClientConfig := *clientConfig
		heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
		// The timeout is the minimum of the lease duration and status update frequency
		leaseTimeout := time.Duration(s.KubeletConfiguration.NodeLeaseDurationSeconds) * time.Second
		if heartbeatClientConfig.Timeout > leaseTimeout {
			heartbeatClientConfig.Timeout = leaseTimeout
		}

		heartbeatClientConfig.QPS = float32(-1)
		kubeDeps.HeartbeatClient, err = clientset.NewForConfig(&heartbeatClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet heartbeat client: %v", err)
		}
	}
  
	if kubeDeps.Auth == nil {
		auth, err := BuildAuth(nodeName, kubeDeps.KubeClient, s.KubeletConfiguration)
		if err != nil {
			return err
		}
		kubeDeps.Auth = auth
	}

	var cgroupRoots []string
  
  
  // 8.配置cgroup，kubelet在已有的cgroup上加了kubepods这一层，用来实现QOS。一般默认是 /sys/fs/cgroup/memory/kubepods
	cgroupRoots = append(cgroupRoots, cm.NodeAllocatableRoot(s.CgroupRoot, s.CgroupDriver))
	kubeletCgroup, err := cm.GetKubeletContainer(s.KubeletCgroups)
	if err != nil {
		klog.Warningf("failed to get the kubelet's cgroup: %v.  Kubelet system container metrics may be missing.", err)
	} else if kubeletCgroup != "" {
		cgroupRoots = append(cgroupRoots, kubeletCgroup)
	}
  
	runtimeCgroup, err := cm.GetRuntimeContainer(s.ContainerRuntime, s.RuntimeCgroups)
	if err != nil {
		klog.Warningf("failed to get the container runtime's cgroup: %v. Runtime system container metrics may be missing.", err)
	} else if runtimeCgroup != "" {
		// RuntimeCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, runtimeCgroup)
	}

	if s.SystemCgroups != "" {
		// SystemCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, s.SystemCgroups)
	}
  
  // 9.初始化cadvisor
	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.ContainerRuntime, s.RemoteRuntimeEndpoint)
		kubeDeps.CAdvisorInterface, err = cadvisor.New(imageFsInfoProvider, s.RootDirectory, cgroupRoots, cadvisor.UsingLegacyCadvisorStats(s.ContainerRuntime, s.RemoteRuntimeEndpoint))
		if err != nil {
			return err
		}
	}

	// Setup event recorder if required.
	// 10. 初始化eventRecorder，用于上报event
	makeEventRecorder(kubeDeps, nodeName)

	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			klog.Info("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}
    
    // 11. 解析系统保留资源，这些资源是不会给pod分配的。例如：--system-reserved=cpu=2000m,memory=20000Mi
		var reservedSystemCPUs cpuset.CPUSet
		var errParse error
		if s.ReservedSystemCPUs != "" {
			reservedSystemCPUs, errParse = cpuset.Parse(s.ReservedSystemCPUs)
			if errParse != nil {
				// invalid cpu list is provided, set reservedSystemCPUs to empty, so it won't overwrite kubeReserved/systemReserved
				klog.Infof("Invalid ReservedSystemCPUs \"%s\"", s.ReservedSystemCPUs)
				return errParse
			}
			// is it safe do use CAdvisor here ??
			machineInfo, err := kubeDeps.CAdvisorInterface.MachineInfo()
			if err != nil {
				// if can't use CAdvisor here, fall back to non-explicit cpu list behavor
				klog.Warning("Failed to get MachineInfo, set reservedSystemCPUs to empty")
				reservedSystemCPUs = cpuset.NewCPUSet()
			} else {
				reservedList := reservedSystemCPUs.ToSlice()
				first := reservedList[0]
				last := reservedList[len(reservedList)-1]
				if first < 0 || last >= machineInfo.NumCores {
					// the specified cpuset is outside of the range of what the machine has
					klog.Infof("Invalid cpuset specified by --reserved-cpus")
					return fmt.Errorf("Invalid cpuset %q specified by --reserved-cpus", s.ReservedSystemCPUs)
				}
			}
		} else {
			reservedSystemCPUs = cpuset.NewCPUSet()
		}

		if reservedSystemCPUs.Size() > 0 {
			// at cmd option valication phase it is tested either --system-reserved-cgroup or --kube-reserved-cgroup is specified, so overwrite should be ok
			klog.Infof("Option --reserved-cpus is specified, it will overwrite the cpu setting in KubeReserved=\"%v\", SystemReserved=\"%v\".", s.KubeReserved, s.SystemReserved)
			if s.KubeReserved != nil {
				delete(s.KubeReserved, "cpu")
			}
			if s.SystemReserved == nil {
				s.SystemReserved = make(map[string]string)
			}
			s.SystemReserved["cpu"] = strconv.Itoa(reservedSystemCPUs.Size())
			klog.Infof("After cpu setting is overwritten, KubeReserved=\"%v\", SystemReserved=\"%v\"", s.KubeReserved, s.SystemReserved)
		}
		// 这里会处理内存和其他资源例如pid等
		kubeReserved, err := parseResourceList(s.KubeReserved)
		if err != nil {
			return err
		}
		systemReserved, err := parseResourceList(s.SystemReserved)
		if err != nil {
			return err
		}
		
		// 12.设置驱逐阈值，例如--eviction-hard=memory.available<1Mi,nodefs.available<1Mi,nodefs.inodesFree<1
		var hardEvictionThresholds []evictionapi.Threshold
		// If the user requested to ignore eviction thresholds, then do not set valid values for hardEvictionThresholds here.
		if !s.ExperimentalNodeAllocatableIgnoreEvictionThreshold {
			hardEvictionThresholds, err = eviction.ParseThresholdConfig([]string{}, s.EvictionHard, nil, nil, nil)
			if err != nil {
				return err
			}
		}
		experimentalQOSReserved, err := cm.ParseQOSReserved(s.QOSReserved)
		if err != nil {
			return err
		}

		devicePluginEnabled := utilfeature.DefaultFeatureGate.Enabled(features.DevicePlugins)
    
    //13.利用上面的配置，实例化NewContainerManager对象
		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				ContainerRuntime:      s.ContainerRuntime,
				CgroupsPerQOS:         s.CgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				KubeletRootDir:        s.RootDirectory,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				NodeAllocatableConfig: cm.NodeAllocatableConfig{
					KubeReservedCgroupName:   s.KubeReservedCgroup,
					SystemReservedCgroupName: s.SystemReservedCgroup,
					EnforceNodeAllocatable:   sets.NewString(s.EnforceNodeAllocatable...),
					KubeReserved:             kubeReserved,
					SystemReserved:           systemReserved,
					ReservedSystemCPUs:       reservedSystemCPUs,
					HardEvictionThresholds:   hardEvictionThresholds,
				},
				QOSReserved:                           *experimentalQOSReserved,
				ExperimentalCPUManagerPolicy:          s.CPUManagerPolicy,
				ExperimentalCPUManagerReconcilePeriod: s.CPUManagerReconcilePeriod.Duration,
				ExperimentalPodPidsLimit:              s.PodPidsLimit,
				EnforceCPULimits:                      s.CPUCFSQuota,
				CPUCFSQuotaPeriod:                     s.CPUCFSQuotaPeriod.Duration,
				ExperimentalTopologyManagerPolicy:     s.TopologyManagerPolicy,
			},
			s.FailSwapOn,
			devicePluginEnabled,
			kubeDeps.Recorder)

		if err != nil {
			return err
		}
	}
 
	if err := checkPermissions(); err != nil {
		klog.Error(err)
	}

	utilruntime.ReallyCrash = s.ReallyCrashForTesting

	// TODO(vmarmol): Do this through container config.
	// 13. oomAdjuster设置容器进程的oom score
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		klog.Warning(err)
	}
  
  //14. 调用RunKubelet，继续运行Kubelet核心逻辑
	if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
		return err
	}

	// If the kubelet config controller is available, and dynamic config is enabled, start the config and status sync loops
	if utilfeature.DefaultFeatureGate.Enabled(features.DynamicKubeletConfig) && len(s.DynamicConfigDir.Value()) > 0 &&
		kubeDeps.KubeletConfigController != nil && !standaloneMode && !s.RunOnce {
		if err := kubeDeps.KubeletConfigController.StartSync(kubeDeps.KubeClient, kubeDeps.EventClient, string(nodeName)); err != nil {
			return err
		}
	}
  
  //15.开启监控检查服务
	if s.HealthzPort > 0 {
		mux := http.NewServeMux()
		healthz.InstallHandler(mux)
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), mux)
			if err != nil {
				klog.Errorf("Starting healthz server failed: %v", err)
			}
		}, 5*time.Second, wait.NeverStop)
	}
  
	if s.RunOnce {
		return nil
	}
   
	// If systemd is used, notify it that we have started
	go daemon.SdNotify(false, "READY=1")

	select {
	case <-done:
		break
	case <-stopCh:
		break
	}

	return nil
}
```

<br>

### 5. RunKubelet

RunKubelet 主要流程：

1. 获取主机名, 并建立并初始化 event recorder, 用于往apiserver发送事件
2. 设置Kubelet拥有特权
3. 设置kubelet rootdir，默认是 /var/lib/kubelet
4. 调用createAndInitKubelet函数初始化kublet对象
5. 若设置 `runonce` 参数，则只拉取一次容器组配置，并在启动容器组后退出，否则将以 server 形式保持

- 对于runonce，首先创建所需的目录，监听 pod update 信息，得到 pod 信息后，创建pod 并返回他们的状态
- 否则以 server 模式启动，调用startKubelet函数。

到这里关注两个点：

* 第一：createAndInitKubelet函数做了什么

* 第二：跟进startKubelet的逻辑

```
// RunKubelet is responsible for setting up and running a kubelet.  It is used in three different applications:
//   1 Integration tests
//   2 Kubelet binary
//   3 Standalone 'kubernetes' binary
// Eventually, #2 will be replaced with instances of #3
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
  // 1.获取主机名, 并建立并初始化 event recorder, 用于往apiserver发送事件
	hostname, err := nodeutil.GetHostname(kubeServer.HostnameOverride)
	if err != nil {
		return err
	}
	// Query the cloud provider for our node name, default to hostname if kubeDeps.Cloud == nil
	nodeName, err := getNodeName(kubeDeps.Cloud, hostname)
	if err != nil {
		return err
	}
	// Setup event recorder if required.
	makeEventRecorder(kubeDeps, nodeName)
  
  //2.设置Kubelet特权
	capabilities.Initialize(capabilities.Capabilities{
		AllowPrivileged: true,
	})
  
  // 3.设置kubelet rootdir，默认是 /var/lib/kubelet
	credentialprovider.SetPreferredDockercfgPath(kubeServer.RootDirectory)
	klog.V(2).Infof("Using root directory: %v", kubeServer.RootDirectory)

	if kubeDeps.OSInterface == nil {
		kubeDeps.OSInterface = kubecontainer.RealOS{}
	}
  
  // 4. 调用createAndInitKubelet函数初始化kublet对象
	k, err := createAndInitKubelet(&kubeServer.KubeletConfiguration,
		kubeDeps,
		&kubeServer.ContainerRuntimeOptions,
		kubeServer.ContainerRuntime,
		kubeServer.RuntimeCgroups,
		kubeServer.HostnameOverride,
		kubeServer.NodeIP,
		kubeServer.ProviderID,
		kubeServer.CloudProvider,
		kubeServer.CertDirectory,
		kubeServer.RootDirectory,
		kubeServer.RegisterNode,
		kubeServer.RegisterWithTaints,
		kubeServer.AllowedUnsafeSysctls,
		kubeServer.RemoteRuntimeEndpoint,
		kubeServer.RemoteImageEndpoint,
		kubeServer.ExperimentalMounterPath,
		kubeServer.ExperimentalKernelMemcgNotification,
		kubeServer.ExperimentalCheckNodeCapabilitiesBeforeMount,
		kubeServer.ExperimentalNodeAllocatableIgnoreEvictionThreshold,
		kubeServer.MinimumGCAge,
		kubeServer.MaxPerPodContainerCount,
		kubeServer.MaxContainerCount,
		kubeServer.MasterServiceNamespace,
		kubeServer.RegisterSchedulable,
		kubeServer.NonMasqueradeCIDR,
		kubeServer.KeepTerminatedPodVolumes,
		kubeServer.NodeLabels,
		kubeServer.SeccompProfileRoot,
		kubeServer.BootstrapCheckpointPath,
		kubeServer.NodeStatusMaxImages)
	if err != nil {
		return fmt.Errorf("failed to create kubelet: %v", err)
	}

	// NewMainKubelet should have set up a pod source config if one didn't exist
	// when the builder was run. This is just a precaution.
	if kubeDeps.PodConfig == nil {
		return fmt.Errorf("failed to create kubelet, pod source config was nil")
	}
	podCfg := kubeDeps.PodConfig

	rlimit.RlimitNumFiles(uint64(kubeServer.MaxOpenFiles))
  
  // 5.调用startKubelet运行kubelet
	// process pods and exit.
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		klog.Info("Started kubelet as runonce")
	} else {
		startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableCAdvisorJSONEndpoints, kubeServer.EnableServer)
		klog.Info("Started kubelet")
	}
	return nil
}
```

#### 5.1. CreateAndInitKubelet 函数

主要逻辑：

（1）根据各种配置生成 NewMainKubelet，并且初始化各种manager，例如livenessManager,statusManager,podManager等等

（2）BirthCry ,往apiserver发送一个启动 kubelet的事件

```
// BirthCry sends an event that the kubelet has started up.
func (kl *Kubelet) BirthCry() {
	// Make an event that kubelet restarted.
	kl.recorder.Eventf(kl.nodeRef, v1.EventTypeNormal, events.StartingKubelet, "Starting kubelet.")
}
```

（3）启动垃圾回收，具体就是后台启动多个协程，进行container, image的垃圾回收。

```
func CreateAndInitKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	kubeDeps *kubelet.Dependencies,
	crOptions *config.ContainerRuntimeOptions,
	containerRuntime string,
	runtimeCgroups string,
	hostnameOverride string,
	nodeIP string,
	providerID string,
	cloudProvider string,
	certDirectory string,
	rootDirectory string,
	registerNode bool,
	registerWithTaints []api.Taint,
	allowedUnsafeSysctls []string,
	remoteRuntimeEndpoint string,
	remoteImageEndpoint string,
	experimentalMounterPath string,
	experimentalKernelMemcgNotification bool,
	experimentalCheckNodeCapabilitiesBeforeMount bool,
	experimentalNodeAllocatableIgnoreEvictionThreshold bool,
	minimumGCAge metav1.Duration,
	maxPerPodContainerCount int32,
	maxContainerCount int32,
	masterServiceNamespace string,
	registerSchedulable bool,
	nonMasqueradeCIDR string,
	keepTerminatedPodVolumes bool,
	nodeLabels map[string]string,
	seccompProfileRoot string,
	bootstrapCheckpointPath string,
	nodeStatusMaxImages int32) (k kubelet.Bootstrap, err error) {
	// TODO: block until all sources have delivered at least one update to the channel, or break the sync loop
	// up into "per source" synchronizations

	k, err = kubelet.NewMainKubelet(kubeCfg,
		kubeDeps,
		crOptions,
		containerRuntime,
		runtimeCgroups,
		hostnameOverride,
		nodeIP,
		providerID,
		cloudProvider,
		certDirectory,
		rootDirectory,
		registerNode,
		registerWithTaints,
		allowedUnsafeSysctls,
		remoteRuntimeEndpoint,
		remoteImageEndpoint,
		experimentalMounterPath,
		experimentalKernelMemcgNotification,
		experimentalCheckNodeCapabilitiesBeforeMount,
		experimentalNodeAllocatableIgnoreEvictionThreshold,
		minimumGCAge,
		maxPerPodContainerCount,
		maxContainerCount,
		masterServiceNamespace,
		registerSchedulable,
		nonMasqueradeCIDR,
		keepTerminatedPodVolumes,
		nodeLabels,
		seccompProfileRoot,
		bootstrapCheckpointPath,
		nodeStatusMaxImages)
	if err != nil {
		return nil, err
	}

	k.BirthCry()

	k.StartGarbageCollection()

	return k, nil
}
```

<br>

#### 5.2 NewMainKubelet

`CreateAndInitKubelet`方法中执行的核心函数是`NewMainKubelet`，`NewMainKubelet`实例化一个`kubelet`对象，该部分的具体代码在`kubernetes/pkg/kubelet`中，具体参考：[kubernetes/pkg/kubelet/kubelet.go#L325](https://github.com/kubernetes/kubernetes/blob/0ed33881dc4355495f623c6f22e7dd0b7632b7c0/pkg/kubelet/kubelet.go#L325)。

**NewMainKubelet 实例化一个 kubelet 对象，并对 kubelet 内部各个 component 进行初始化工作:**

- containerGC      // 容器的垃圾回收
- statusManager  // pod 状态的管理
- imageManager  // 镜像的管路
- probeManager   // 容器健康检测
- gpuManager      // GPU 的支持
- PodCache          // Pod 缓存的管理
- secretManager   // secret 资源的管理
- configMapManager  // configMap 资源的管理
- InitNetworkPlugin     // 网络插件的初始化
- PodManager // 对 pod 的管理, e.g., CRUD
- makePodSourceConfig // pod 元数据的来源 (FILE, URL, api-server)
- diskSpaceManager  // 磁盘空间的管理
- ContainerRuntime  // 容器运行时的选择(docker 或 rkt)

<br>

#### 5.3 startKubelet

CreateAndInitKubele之后，马上就是startKubelet了，这里核心调用的了 kubelet.Run函数

主要逻辑如下：

1. 检查 logserver 以及 apiserver 是否可用

2. 如有 cloud provider 配置， 则启动`cloudResourceSyncManager`，将请求发送给 cloud provider

3. 启动 volumeManager，VolumeManager运行一组异步循环，这些循环根据在此节点上调度的Pod来确定需要附加/装入/卸载/分离的卷。

4. 调用 `kubelet.syncNodeStatus`同步 node 状态，如果从上次同步起有任何更改 或 经过了足够的时间，它将节点状态同步到主节点，并在必要时先注册kubelet。

5. 调用 `kubelet.updateRuntimeUp`，updateRuntimeUp调用容器运行时状态回调，在容器运行时首次出现时初始化依赖于运行时的模块，如果状态检查失败，则返回错误。 如果状态检查确定，则在kubelet runtimeState中更新容器运行时正常运行时间。

6. 开启循环，同步 iptables 规则（*但是在源码中无任何操作*）

7. 启动一个用于“杀死pod” 的 goroutine，如果尚未使用其他goroutine，则podKiller会从通道(`podKillingCh`)中接收到一个pod，然后启动goroutine杀死他

8. 启动 statusManager 和 probeManager （都是无限循环的同步机制），statusManager 与 apiserver 同步pod状态；probeManager 管理并接收 container 探针。

9. 启动 runtimeClass manager，默认是docker

   > runtimeClass 是 K8s 的一个 api 对象，可以通过定义 runtimeClass 实现 K8s 对接不同的 容器运行时。

10. 启动 pleg （pod lifecycle event generator），用于生成 pod 相关的 event。

至此，kubelet 整体的启动流程完毕，进入无限循环中，实时同步不同组件的状态。同时也对端口进行监听，响应 http 请求。

```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go wait.Until(func() {
		k.Run(podCfg.Updates())
	}, 0, wait.NeverStop)

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)

	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
}
```

```
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		klog.Warning("No api server defined - no node status update will be sent.")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.Fatal(err)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start syncing node status immediately, this may set up things the runtime needs to run.
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start a goroutine responsible for killing pods (that are not properly
	// handled by pod workers).
	go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

	// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```

<br>

### 6. 总结

1. kubelet采用[Cobra](https://github.com/spf13/cobra)命令行框架和[pflag](https://github.com/spf13/pflag)参数解析框架，和apiserver、scheduler、controller-manager形成统一的代码风格。
2. `kubernetes/cmd/kubelet`部分主要对运行参数进行定义和解析，初始化和构造相关的依赖组件（主要在`kubeDeps`结构体中），并没有kubelet运行的详细逻辑，该部分位于`kubernetes/pkg/kubelet`模块。
3. cmd部分调用流程如下：`Main-->NewKubeletCommand-->Run(kubeletServer, kubeletDeps, stopCh)-->run(s *options.KubeletServer, kubeDeps ..., stopCh ...)--> RunKubelet(s, kubeDeps, s.RunOnce)-->startKubelet-->k.Run(podCfg.Updates())-->pkg/kubelet`。同时`RunKubelet(s, kubeDeps, s.RunOnce)-->CreateAndInitKubelet-->kubelet.NewMainKubelet-->pkg/kubelet`。
4. 整体而言，到目前为止都是进行初始化工作。初始化kubelete各种控制器，然后运行kl.syncLoop(updates, kl)处理Pod

另一种描述：

1. 初始化模块，其实就是运行`imageManager`、`serverCertificateManager`、`oomWatcher`、`resourceAnalyzer`。
2. 运行各种manager，大部分以常驻goroutine的方式运行，其中包括`volumeManager`、`statusManager`等。
3. 执行处理变更的循环函数`syncLoop`，对pod的生命周期进行管理。

这里直接上图：图片来源: https://www.bookstack.cn/read/source-code-reading-notes/kubernetes-kubelet_init.md

![kubelet-func-chanel](../images/kubelet-func-chanel.png)





##### 补充各种 Manager

以下介绍kubelet运行时涉及到的manager的内容。

| manager                  | 说明                                               |
| ------------------------ | -------------------------------------------------- |
| imageManager             | 负责镜像垃圾回收                                   |
| serverCertificateManager | 负责处理证书                                       |
| oomWatcher               | 监控内存使用，是否发生内存耗尽即OOM                |
| resourceAnalyzer         | 监控资源使用情况                                   |
| volumeManager            | 对pod执行`attached/detached/mounted/unmounted`操作 |
| statusManager            | 使用apiserver同步pods状态; 也用作状态缓存          |
| probeManager             | 处理容器探针                                       |
| runtimeClassManager      | 同步RuntimeClasses                                 |
| podKiller                | 负责杀死pod                                        |

<br>

### 7 参考

https://www.huweihuang.com/kubernetes-notes/code-analysis/kubelet/NewKubeletCommand.html

https://blog.csdn.net/jimzbq/article/details/104282753

https://www.bookstack.cn/read/source-code-reading-notes/kubernetes-kubelet_init.md   