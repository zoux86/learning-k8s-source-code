Table of Contents
=================

  * [1. cmd/kubectl/kubectl.go](#1-cmdkubectlkubectlgo)
  * [2. NewDefaultKubectlCommand](#2-newdefaultkubectlcommand)
     * [2.1 如何设置kubectl自动补全](#21-如何设置kubectl自动补全)
     * [2.2 kubectl config](#22-kubectl-config)
     * [2.3 kubectl  api-versions](#23-kubectl--api-versions)
     * [2.4 kubectl api-resources](#24-kubectl-api-resources)
     * [2.5 kubectl options](#25-kubectl-options)
  * [3. 总结](#3-总结)

k8s源码一般的目录就是  cm/kubectl  下是启动的主函数

然后进入 pkg/kubectl  运行着真正的逻辑

### 1. cmd/kubectl/kubectl.go

```
cmd/kubectl/kubectl.go
func main() {
	rand.Seed(time.Now().UnixNano())
    
    
    // 1. 主要的函数就是 NewDefaultKubectlCommand
	command := cmd.NewDefaultKubectlCommand()

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// cliflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	// 1.设置统一化函数
	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	// 2.同时使用pflag和flag
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	// cliflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

SetNormalizeFunc的作用是：

如果我们创建了名称为 --des-detail 的参数，但是用户却在传参时写成了 --des_detail 或 --des.detail 会怎么样？默认情况下程序会报错退出，但是我们可以通过 pflag 提供的 SetNormalizeFunc 功能轻松的解决这个问题。

<br>

### 2. NewDefaultKubectlCommand

pkg/kubectl/cmd/cmd.go 

这里调用链路如下：

NewDefaultKubectlCommand -> NewDefaultKubectlCommandWithArgs -> NewKubectlCommand 

NewKubectlCommand 只是注册一些命令。具体步骤如下：

1. 定义kubectl 命令。从这里可以看出来，针对kubectl而言不会有任何参数，只会输出kubectl的使用帮助
2. 设置kubeconfigflags，用于连接apiserver
3. 利用kubeconfigflags生成了, 一个Factory f, 这个f包含了与apiserver操作的client，每个子命令都利用这个f进行后续的操作。(接下来单独分析这一步)
4. kubectl 子命令分为了以下这几类: Basic Commands (Beginner), Basic Commands (Intermediate), Deploy Commands, Cluster Management Commands, Troubleshooting and Debugging Commands, Advanced Commands , Settings Commands
5. 注册上述的command
6. 注册没有分类的一些命令，例如，kubectl pulgin ， kubectl version等等。

到这里，kubectl的大体逻辑就是定义好一堆的子命令。接下来具体看一个子命令，kubectl create -f 命令。

```
// NewKubectlCommand creates the `kubectl` command and its nested children.
func NewKubectlCommand(in io.Reader, out, err io.Writer) *cobra.Command {

     // 1.定义kubectl 命令。从这里可以看出来，针对kubectl而言不会有任何参数，只会输出kubectl的使用帮助
	// Parent command to which all subcommands are added.
	cmds := &cobra.Command{
		Use:   "kubectl",
		Short: i18n.T("kubectl controls the Kubernetes cluster manager"),
		Long: templates.LongDesc(`
      kubectl controls the Kubernetes cluster manager.

      Find more information at:
            https://kubernetes.io/docs/reference/kubectl/overview/`),
		Run: runHelp,
		// Hook before and after Run initialize and write profiles to disk,
		// respectively.
		PersistentPreRunE: func(*cobra.Command, []string) error {
			return initProfiling()
		},
		PersistentPostRunE: func(*cobra.Command, []string) error {
			return flushProfiling()
		},
		BashCompletionFunction: bashCompletionFunc,
	}
 

  flags := cmds.PersistentFlags()
	flags.SetNormalizeFunc(cliflag.WarnWordSepNormalizeFunc) // Warn for "_" flags
  // Normalize all flags that are coming from other packages or pre-configurations
	// a.k.a. change all "_" to "-". e.g. glog package
	flags.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	addProfilingFlags(flags)

  // 2.设置kubeconfigflags，用于连接apiserver
	kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
	kubeConfigFlags.AddFlags(flags)

  // 3.利用kubeconfigflags生成了, 一个Factory f, 这个f包含了与apiserver操作的client，每个子命令都利用这个f进行后续的操作。
	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
	matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())
	cmds.PersistentFlags().AddGoFlagSet(flag.CommandLine)

	f := cmdutil.NewFactory(matchVersionKubeConfigFlags)
     
     // 4. kubectl子命令分为了以下这几类
	 	groups := templates.CommandGroups{
		{
			Message: "Basic Commands (Beginner):",
			Commands: []*cobra.Command{
				create.NewCmdCreate(f, ioStreams),
				expose.NewCmdExposeService(f, ioStreams),
				run.NewCmdRun(f, ioStreams),
				set.NewCmdSet(f, ioStreams),
			},
		},
		{
			Message: "Basic Commands (Intermediate):",
			Commands: []*cobra.Command{
				explain.NewCmdExplain("kubectl", f, ioStreams),
				get.NewCmdGet("kubectl", f, ioStreams),
				edit.NewCmdEdit(f, ioStreams),
				delete.NewCmdDelete(f, ioStreams),
			},
		},
		{
			Message: "Deploy Commands:",
			Commands: []*cobra.Command{
				rollout.NewCmdRollout(f, ioStreams),
				rollingupdate.NewCmdRollingUpdate(f, ioStreams),
				scale.NewCmdScale(f, ioStreams),
				autoscale.NewCmdAutoscale(f, ioStreams),
			},
		},
		{
			Message: "Cluster Management Commands:",
			Commands: []*cobra.Command{
				certificates.NewCmdCertificate(f, ioStreams),
				clusterinfo.NewCmdClusterInfo(f, ioStreams),
				top.NewCmdTop(f, ioStreams),
				drain.NewCmdCordon(f, ioStreams),
				drain.NewCmdUncordon(f, ioStreams),
				drain.NewCmdDrain(f, ioStreams),
				taint.NewCmdTaint(f, ioStreams),
			},
		},
		{
			Message: "Troubleshooting and Debugging Commands:",
			Commands: []*cobra.Command{
				describe.NewCmdDescribe("kubectl", f, ioStreams),
				logs.NewCmdLogs(f, ioStreams),
				attach.NewCmdAttach(f, ioStreams),
				cmdexec.NewCmdExec(f, ioStreams),
				portforward.NewCmdPortForward(f, ioStreams),
				proxy.NewCmdProxy(f, ioStreams),
				cp.NewCmdCp(f, ioStreams),
				auth.NewCmdAuth(f, ioStreams),
			},
		},
		{
			Message: "Advanced Commands:",
			Commands: []*cobra.Command{
				diff.NewCmdDiff(f, ioStreams),
				apply.NewCmdApply("kubectl", f, ioStreams),
				patch.NewCmdPatch(f, ioStreams),
				replace.NewCmdReplace(f, ioStreams),
				wait.NewCmdWait(f, ioStreams),
				convert.NewCmdConvert(f, ioStreams),
				kustomize.NewCmdKustomize(ioStreams),
			},
		},
		{
			Message: "Settings Commands:",
			Commands: []*cobra.Command{
				label.NewCmdLabel(f, ioStreams),
				annotate.NewCmdAnnotate("kubectl", f, ioStreams),
				completion.NewCmdCompletion(ioStreams.Out, ""),
			},
		},
	}
	
    // 4.1.注册上面的子命令
	groups.Add(cmds)

	filters := []string{"options"}
    
    // 4.2 直接使用kubectl alpha 命令可以查看有哪些命令是alpha 阶段的。
	// Hide the "alpha" subcommand if there are no alpha commands in this build.
	alpha := cmdpkg.NewCmdAlpha(f, ioStreams)
	if !alpha.HasSubCommands() {
		filters = append(filters, alpha.Name())
	}

	templates.ActsAsRootCommand(cmds, filters, groups...)
    
    // 4.3 kubectl代码自动补全
	for name, completion := range bashCompletionFlags {
		if cmds.Flag(name) != nil {
			if cmds.Flag(name).Annotations == nil {
				cmds.Flag(name).Annotations = map[string][]string{}
			}
			cmds.Flag(name).Annotations[cobra.BashCompCustom] = append(
				cmds.Flag(name).Annotations[cobra.BashCompCustom],
				completion,
			)
		}
	}
    
    // 4.4 添加一些不在默认分组的子命令
    // 1.alpha命令
	cmds.AddCommand(alpha)
	// kubectl config命令，见2.2
	cmds.AddCommand(cmdconfig.NewCmdConfig(f, clientcmd.NewDefaultPathOptions(), ioStreams))
	// 配置kubectl 插件。 例如kubectl debug就是一个插件。  详见：https://kubernetes.io/zh/docs/tasks/extend-kubectl/kubectl-plugins/
	cmds.AddCommand(plugin.NewCmdPlugin(f, ioStreams))
	
    // kubectl version输出版本信息
	cmds.AddCommand(version.NewCmdVersion(f, ioStreams))
	
	// 实现 kubectl api-versions, 详见2.3
	cmds.AddCommand(apiresources.NewCmdAPIVersions(f, ioStreams))
	
	// 实现 kubectl api-resources, 详见2.3
	cmds.AddCommand(apiresources.NewCmdAPIResources(f, ioStreams))
	
	// 实现kubectl options， 可以查看子命令可以带哪些options，例如所有命令都会带-v 查看完整日志
	cmds.AddCommand(options.NewCmdOptions(ioStreams.Out))

	return cmds
}

func runHelp(cmd *cobra.Command, args []string) {
	cmd.Help()
}
```

#### 2.1 如何设置kubectl自动补全

```
# linux
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
#或者
echo "source <(kubectl completion bash)" >> /etc/profile

# mac 
source <(kubectl completion bash)
# 测试
[root@node02 ~]# kubectl tab键
annotate       apply          autoscale      completion     cordon         delete         drain          explain        kustomize      options        port-forward   rollout        set            uncordon
api-resources  attach         certificate    config         cp             describe       edit           expose         label          patch          proxy          run            taint          version
api-versions   auth           cluster-info   convert        create         diff           exec           get            logs           plugin         replace        scale          top            wait
```

#### 2.2 kubectl config

从kubectl config命令可以看出来，config优先级为： 

（1）--kubeconfig 指定的config

（2）KUBECONFIG环境变量

（3）默认的 /HOME/.root/config文件

```
root@k8s-master:~# kubectl config -h
Modify kubeconfig files using subcommands like "kubectl config set current-context my-context"

 The loading order follows these rules:

  1.  If the --kubeconfig flag is set, then only that file is loaded. The flag may only be set once and no merging takes
place.
  2.  If $KUBECONFIG environment variable is set, then it is used as a list of paths (normal path delimiting rules for
your system). These paths are merged. When a value is modified, it is modified in the file that defines the stanza. When
a value is created, it is created in the first file that exists. If no files in the chain exist, then it creates the
last file in the list.
  3.  Otherwise, ${HOME}/.kube/config is used and no merging takes place.

Available Commands:
  current-context Displays the current-context
  delete-cluster  Delete the specified cluster from the kubeconfig
  delete-context  Delete the specified context from the kubeconfig
  get-clusters    Display clusters defined in the kubeconfig
  get-contexts    Describe one or many contexts
  rename-context  Renames a context from the kubeconfig file.
  set             Sets an individual value in a kubeconfig file
  set-cluster     Sets a cluster entry in kubeconfig
  set-context     Sets a context entry in kubeconfig
  set-credentials Sets a user entry in kubeconfig
  unset           Unsets an individual value in a kubeconfig file
  use-context     Sets the current-context in a kubeconfig file
  view            Display merged kubeconfig settings or a specified kubeconfig file

Usage:
  kubectl config SUBCOMMAND [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

#### 2.3 kubectl  api-versions

```
root@k8s-master:~# ^C
root@k8s-master:~# kubectl api-versions  -h
Print the supported API versions on the server, in the form of "group/version"

Examples:
  # Print the supported API versions
  kubectl api-versions

Usage:
  kubectl api-versions [flags] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

#### 2.4 kubectl api-resources

```
root@k8s-master:~# kubectl api-resources  -h
Print the supported API resources on the server

Examples:
  # Print the supported API Resources
  kubectl api-resources
  
  # Print the supported API Resources with more information
  kubectl api-resources -o wide
  
  # Print the supported API Resources sorted by a column
  kubectl api-resources --sort-by=name
  
  # Print the supported namespaced resources
  kubectl api-resources --namespaced=true
  
  # Print the supported non-namespaced resources
  kubectl api-resources --namespaced=false
  
  # Print the supported API Resources with specific APIGroup
  kubectl api-resources --api-group=extensions

Options:
      --api-group='': Limit to resources in the specified API group.
      --cached=false: Use the cached list of resources if available.
      --namespaced=true: If false, non-namespaced resources will be returned, otherwise returning namespaced resources
by default.
      --no-headers=false: When using the default or custom-column output format, don't print headers (default print
headers).
  -o, --output='': Output format. One of: wide|name.
      --sort-by='': If non-empty, sort nodes list using specified field. The field can be either 'name' or 'kind'.
      --verbs=[]: Limit to resources that support the specified verbs.

Usage:
  kubectl api-resources [flags] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

#### 2.5 kubectl options

```
root@k8s-master:~# kubectl options
The following options can be passed to any command:

      --add-dir-header=false: If true, adds the file directory to the header
      --alsologtostderr=false: log to standard error as well as files
      --as='': Username to impersonate for the operation
      --as-group=[]: Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --cache-dir='/root/.kube/http-cache': Default HTTP cache directory
      --certificate-authority='': Path to a cert file for the certificate authority
      --client-certificate='': Path to a client certificate file for TLS
      --client-key='': Path to a client key file for TLS
      --cluster='': The name of the kubeconfig cluster to use
      --context='': The name of the kubeconfig context to use
      --insecure-skip-tls-verify=false: If true, the server's certificate will not be checked for validity. This will
make your HTTPS connections insecure
      --kubeconfig='': Path to the kubeconfig file to use for CLI requests.
      --log-backtrace-at=:0: when logging hits line file:N, emit a stack trace
      --log-dir='': If non-empty, write log files in this directory
      --log-file='': If non-empty, use this log file
      --log-file-max-size=1800: Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0,
the maximum file size is unlimited.
      --log-flush-frequency=5s: Maximum number of seconds between log flushes
      --logtostderr=true: log to standard error instead of files
      --match-server-version=false: Require server version to match client version
  -n, --namespace='': If present, the namespace scope for this CLI request
      --password='': Password for basic authentication to the API server
      --profile='none': Name of profile to capture. One of (none|cpu|heap|goroutine|threadcreate|block|mutex)
      --profile-output='profile.pprof': Name of the file to write the profile to
      --request-timeout='0': The length of time to wait before giving up on a single server request. Non-zero values
should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests.
  -s, --server='': The address and port of the Kubernetes API server
      --skip-headers=false: If true, avoid header prefixes in the log messages
      --skip-log-headers=false: If true, avoid headers when opening log files
      --stderrthreshold=2: logs at or above this threshold go to stderr
      --token='': Bearer token for authentication to the API server
      --user='': The name of the kubeconfig user to use
      --username='': Username for basic authentication to the API server
  -v, --v=0: number for the log level verbosity
      --vmodule=: comma-separated list of pattern=N settings for file-filtered logging
```

<br>

### 3. 总结

kubectl 代码结构非常清楚。主要就是：

（1）定义kubectl 命令

（2）配置kubeconfig, 并且生成一个Factory f。这一步很重要，接下来单独分析这个过程

（3）定义kubectl 各种子命令