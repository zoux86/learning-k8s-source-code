Table of Contents
=================

  * [0. 背景](#0-背景)
  * [1. 实现testPlugin](#1-实现testplugin)
  * [2. 注册testPlugin](#2-注册testplugin)
  * [3 结果验证](#3-结果验证)

### 0. 背景

这里就是构造了一个简单的案例。如果pod包含了test-plugin这个annotations, 就不让pod调度，一直处于pending状态。

**具体步骤如下：**

### 1. 实现testPlugin

<br>

只需要增加对应文件即可：

pkg/scheduler/framework/plugins/testplugin/test_plugin.go

```
/*
Copyright 2019 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package testplugin

import (
	"context"
	"k8s.io/klog"

	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/kubernetes/pkg/scheduler/framework/plugins/migration"
	framework "k8s.io/kubernetes/pkg/scheduler/framework/v1alpha1"
	"k8s.io/kubernetes/pkg/scheduler/nodeinfo"
)

// NodeName is a plugin that checks if a pod spec node name matches the current node.
type TestPlugin struct{}

var _ framework.FilterPlugin = &TestPlugin{}

// Name is the name of the plugin used in the plugin registry and configurations.
const Name = "TestPlugin"

// Name returns name of the plugin. It is used in logs, etc.
func (pl *TestPlugin) Name() string {
	return Name
}

// Filter invoked at the filter extension point.
func (pl *TestPlugin) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *nodeinfo.NodeInfo) *framework.Status {
	_, exist := pod.GetAnnotations()["test-plugin"]
	klog.Infof("[zoux testPlugin for pod", pod.Name)
	if !exist {
		return migration.PredicateResultToFrameworkStatus(nil, nil)
	}

	return framework.NewStatus(framework.Unschedulable, "testPlugin")

}

// New initializes a new plugin and returns it.
func New(_ *runtime.Unknown, _ framework.FrameworkHandle) (framework.Plugin, error) {
	return &TestPlugin{}, nil
}

```

### 2. 注册testPlugin

pkg/scheduler/framework/plugins/default_registry.go

(1) 在NewDefaultRegistry进行赋值

```
func NewDefaultRegistry(args *RegistryArgs) framework.Registry {
	return framework.Registry{
		defaultpodtopologyspread.Name:        defaultpodtopologyspread.New,
		imagelocality.Name:                   imagelocality.New,
		tainttoleration.Name:                 tainttoleration.New,
		nodename.Name:                        nodename.New,
		testplugin.Name:                      testplugin.New,
		nodeports.Name:                       nodeports.New,
		nodepreferavoidpods.Name:             nodepreferavoidpods.New,
		nodeaffinity.Name:                    nodeaffinity.New,
		podtopologyspread.Name:               podtopologyspread.New,
		。。。
}
```

（2）在NewDefaultConfigProducerRegistry进行注册

```
// NewDefaultConfigProducerRegistry creates a new producer registry.
func NewDefaultConfigProducerRegistry() *ConfigProducerRegistry {
	registry := &ConfigProducerRegistry{
		PredicateToConfigProducer: make(map[string]ConfigProducer),
		PriorityToConfigProducer:  make(map[string]ConfigProducer),
	}
	// Register Predicates.
	registry.RegisterPredicate(predicates.GeneralPred,
		func(_ ConfigProducerArgs) (plugins config.Plugins, pluginConfig []config.PluginConfig) {
			// GeneralPredicate is a combination of predicates.
			plugins.Filter = appendToPluginSet(plugins.Filter, noderesources.FitName, nil)
			plugins.Filter = appendToPluginSet(plugins.Filter, nodename.Name, nil)
			plugins.Filter = appendToPluginSet(plugins.Filter, nodeports.Name, nil)
			plugins.Filter = appendToPluginSet(plugins.Filter, nodeaffinity.Name, nil)
			plugins.Filter = appendToPluginSet(plugins.Filter, testplugin.Name, nil)
			return
		})

```

### 3 结果验证

编译后，验证即可。日志忘保存，不贴出来了。。