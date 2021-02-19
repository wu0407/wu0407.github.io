---
title: "kubelet启动--FeatureGate初始化"
date: 2021-02-19T11:21:49+08:00
draft: false
description: "kubelet的FeatureGate初始化和修改和使用"
keywords:  [ "kubernetes", "kubelet", "FeatureGate","源码解析"]
tags:  ["kubernetes", "源码解析", "kubelet"]
categories:  ["kubernetes"]
---

kubernetes有很多的功能特性，这些特性一般都有关联的KEP。特性的成熟度有alpha、beta、GA、Deprecated，alpha代表不稳定，beta代表相对稳定有可能有bug，GA代表已经可用。 特性的生命阶段有KEP提出、alpha阶段、beta阶段、GA阶段、废弃。alpha阶段默认不启用，beta阶段默认启用 。更多feature gate相关知识请访问--[Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)和[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md)。

feature gate用来控制某项特性打开或关闭，当然GA的特性不能关闭。本文以kubelet源码为例，分析feature gate如何工作的。<!--more--> 

本文基于kubernetes 1.18.6版本，请访问[源代码阅读仓库](https://github.com/wu0407/kubernetes-code-analysis)。

## 模块初始化

在cmd\kubelet\app\server.go会加载`k8s.io/apiserver/pkg/util/feature`和`k8s.io/component-base/featuregate`和`k8s.io/kubernetes/pkg/features`。其中`k8s.io/apiserver/pkg/util/feature`定义了两个全局变量`DefaultMutableFeatureGate`和`DefaultFeatureGate`。`k8s.io/component-base/featuregate`定义了`featureGate`类型和类型的方法。`k8s.io/kubernetes/pkg/features`定义了默认的featuregate设置。



### 加载k8s.io/apiserver/pkg/util/feature

DefaultMutableFeatureGate 和DefaultFeatureGate是相同的，只是DefaultMutableFeatureGate对外提供读写能力，而DefaultFeatureGate提供只读。

```go
package feature

import (
	"k8s.io/component-base/featuregate"
)

var (
	// DefaultMutableFeatureGate is a mutable version of DefaultFeatureGate.
	// Only top-level commands/options setup and the k8s.io/component-base/featuregate/testing package should make use of this.
	// Tests that need to modify feature gates for the duration of their test should use:
	//   defer featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.<FeatureName>, <value>)()
	DefaultMutableFeatureGate featuregate.MutableFeatureGate = featuregate.NewFeatureGate()

	// DefaultFeatureGate is a shared global FeatureGate.
	// Top-level commands/options setup that needs to modify this feature gate should use DefaultMutableFeatureGate.
	DefaultFeatureGate featuregate.FeatureGate = DefaultMutableFeatureGate
)
```

featuregate.NewFeatureGate() 返回一个known包含AllAlpha和AllBeta的featureSpec，AllAlpha代表所有alpha阶段的featureGate，它默认为不启用；AllBeta代表所有beta阶段的featureGate，默认为不启用。

```go
&featureGate{
		featureGateName: "k8s.io/apiserver/pkg/util/feature/feature_gate.go",
		known:           knownValue,  //包含AllAlpha和AllBeta的FeatureSpec的atomic.Value
		special:         specialFeatures,
		enabled:         enabledValue, //空的atomic.Value
	}
```

staging\src\k8s.io\component-base\featuregate\feature_gate.go

```go
const (
	flagName = "feature-gates"

	// allAlphaGate is a global toggle for alpha features. Per-feature key
	// values override the default set by allAlphaGate. Examples:
	//   AllAlpha=false,NewFeature=true  will result in newFeature=true
	//   AllAlpha=true,NewFeature=false  will result in newFeature=false
	allAlphaGate Feature = "AllAlpha"

	// allBetaGate is a global toggle for beta features. Per-feature key
	// values override the default set by allBetaGate. Examples:
	//   AllBeta=false,NewFeature=true  will result in NewFeature=true
	//   AllBeta=true,NewFeature=false  will result in NewFeature=false
	allBetaGate Feature = "AllBeta"
)

var (
	// The generic features.
	defaultFeatures = map[Feature]FeatureSpec{
		allAlphaGate: {Default: false, PreRelease: Alpha},
		allBetaGate:  {Default: false, PreRelease: Beta},
	}

    	// Special handling for a few gates.
	specialFeatures = map[Feature]func(known map[Feature]FeatureSpec, enabled map[Feature]bool, val bool){
		allAlphaGate: setUnsetAlphaGates,
		allBetaGate:  setUnsetBetaGates,
	}
)

type FeatureSpec struct {
	// Default is the default enablement state for the feature
	Default bool
	// LockToDefault indicates that the feature is locked to its default and cannot be changed
	LockToDefault bool
	// PreRelease indicates the maturity level of the feature
	PreRelease prerelease
}

// FeatureGate indicates whether a given feature is enabled or not
type FeatureGate interface {
	// Enabled returns true if the key is enabled.
	Enabled(key Feature) bool
	// KnownFeatures returns a slice of strings describing the FeatureGate's known features.
	KnownFeatures() []string
	// DeepCopy returns a deep copy of the FeatureGate object, such that gates can be
	// set on the copy without mutating the original. This is useful for validating
	// config against potential feature gate changes before committing those changes.
	DeepCopy() MutableFeatureGate
}

// MutableFeatureGate parses and stores flag gates for known features from
// a string like feature1=true,feature2=false,...
type MutableFeatureGate interface {
	FeatureGate

	// AddFlag adds a flag for setting global feature gates to the specified FlagSet.
	AddFlag(fs *pflag.FlagSet)
	// Set parses and stores flag gates for known features
	// from a string like feature1=true,feature2=false,...
	Set(value string) error
	// SetFromMap stores flag gates for known features from a map[string]bool or returns an error
	SetFromMap(m map[string]bool) error
	// Add adds features to the featureGate.
	Add(features map[Feature]FeatureSpec) error
}

// featureGate implements FeatureGate as well as pflag.Value for flag parsing.
type featureGate struct {
	featureGateName string

	special map[Feature]func(map[Feature]FeatureSpec, map[Feature]bool, bool)

	// lock guards writes to known, enabled, and reads/writes of closed
	lock sync.Mutex
	// known holds a map[Feature]FeatureSpec
	known *atomic.Value
	// enabled holds a map[Feature]bool
	enabled *atomic.Value
	// closed is set to true when AddFlag is called, and prevents subsequent calls to Add
	closed bool
}

type prerelease string

const (
	// Values for PreRelease.
	Alpha = prerelease("ALPHA")
	Beta  = prerelease("BETA")
	GA    = prerelease("")

	// Deprecated
	Deprecated = prerelease("DEPRECATED")
)

func NewFeatureGate() *featureGate {
	known := map[Feature]FeatureSpec{}
	for k, v := range defaultFeatures {
		known[k] = v
	}

	knownValue := &atomic.Value{}
	knownValue.Store(known)

	enabled := map[Feature]bool{}
	enabledValue := &atomic.Value{}
	enabledValue.Store(enabled)

	f := &featureGate{
		featureGateName: naming.GetNameFromCallsite(internalPackages...),
		known:           knownValue,
		special:         specialFeatures,
		enabled:         enabledValue,
	}
	return f
}

```

### 加载k8s.io/kubernetes/pkg/features

定义了所有已知的Feature和它的默认启用设置和成熟度，其中init()将默认的配置导入到utilfeature.DefaultMutableFeatureGate，所以DefaultMutableFeatureGate或DefaultFeatureGate的known保存了所有的默认的FeatureSpec和AllAlpha和AllBeta，其中DefaultFeatureGate.enabled是空的。

```go

const (
	// Every feature gate should add method here following this template:
	//
	// // owner: @username
	// // alpha: v1.X
	// MyFeature featuregate.Feature = "MyFeature"

	// owner: @tallclair
	// beta: v1.4
	AppArmor featuregate.Feature = "AppArmor"

	// owner: @mtaufen
	// alpha: v1.4
	// beta: v1.11
	DynamicKubeletConfig featuregate.Feature = "DynamicKubeletConfig"
    .......
)

func init() {
	runtime.Must(utilfeature.DefaultMutableFeatureGate.Add(defaultKubernetesFeatureGates))
}

// defaultKubernetesFeatureGates consists of all known Kubernetes-specific feature keys.
// To add a new feature, define a key for it above and add it here. The features will be
// available throughout Kubernetes binaries.
var defaultKubernetesFeatureGates = map[featuregate.Feature]featuregate.FeatureSpec{
	AppArmor:             {Default: true, PreRelease: featuregate.Beta},
	DynamicKubeletConfig: {Default: true, PreRelease: featuregate.Beta},
	ExperimentalHostUserNamespaceDefaultingGate: {Default: false, PreRelease: featuregate.Beta},
	DevicePlugins:                  {Default: true, PreRelease: featuregate.Beta},
    .........
}

```

FeatureGate.Add定义在staging\src\k8s.io\component-base\featuregate\feature_gate.go

遍历defaultKubernetesFeatureGates，如果Feature不在DefaultMutableFeatureGate的known中，则保存到known；已经存在且FeatureSpec一样，则忽略；已经存在且FeatureSpec不一样则报错。

```go
// Add adds features to the featureGate.
func (f *featureGate) Add(features map[Feature]FeatureSpec) error {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.closed {
		return fmt.Errorf("cannot add a feature gate after adding it to the flag set")
	}

	// Copy existing state
	known := map[Feature]FeatureSpec{}
	for k, v := range f.known.Load().(map[Feature]FeatureSpec) {
		known[k] = v
	}

	for name, spec := range features {
		if existingSpec, found := known[name]; found {
			if existingSpec == spec {
				continue
			}
			return fmt.Errorf("feature gate %q with different spec already exists: %v", name, existingSpec)
		}

		known[name] = spec
	}

	// Persist updated state
	f.known.Store(known)

	return nil
}
```

## 设置feature gate

有两种方式设置feature gate，一种命令行方式--比如`--feature-gates=HPAScaleToZero=true,EphemeralContainers=true`，另一种方式在配置文件配置（或者dynamic kubelet config方式），比如：

```
featureGates:
  SupportNodePidsLimit: true
```

`--feature-gates`命令行选项定义配置

```go
fs.Var(cliflag.NewMapStringBool(&c.FeatureGates), "feature-gates", "A set of key=value pairs that describe feature gates for alpha/experimental features. "+
		"Options are:\n"+strings.Join(utilfeature.DefaultFeatureGate.KnownFeatures(), "\n"))
```



在kubelet启动时候，将指定feature设置保存到DefaultFeatureGate.enabled。

- 执行cleanFlagSet.Parse(args)命令行解析--最终会将命令行参数保存到KubeletConfigure.FeatureGates里。
- 执行utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates)将kubeletConfig.FeatureGates保存到DefaultFeatureGate。
- 如果配置文件设置了FeatureGates，则与命令行的FeatureGates进行合并，出现冲突的以命令行设置为准。
- 如果使用dynamic kubeletconfig，且dynamic kubeletconfig 磁盘中存在assigned或last-known-good文件，则与命令行的FeatureGates进行合并，出现冲突的以命令行设置为准。详细的dynamic kubeletconfig分析，后续文章会分析。

```go
		Run: func(cmd *cobra.Command, args []string) {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				cmd.Usage()
				klog.Fatal(err)
			}	
			.......
			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				klog.Fatal(err)
			}

			// validate the initial KubeletFlags
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				klog.Fatal(err)
			}

			if kubeletFlags.ContainerRuntime == "remote" && cleanFlagSet.Changed("pod-infra-container-image") {
				klog.Warning("Warning: For remote container runtime, --pod-infra-container-image is ignored in kubelet, which should be set in that remote runtime instead")
			}

			// load kubelet config file, if provided
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					klog.Fatal(err)
				}
				// We must enforce flag precedence by re-parsing the command line into the new object.
				// This is necessary to preserve backwards-compatibility across binary upgrades.
				// See issue #56171 for more details.
				// 重新解析args，命令行选项覆盖kubeletConfig里的选项
				// 其中FeatureGates为合并
				// 忽略Credential、klog、cadvisor选项--这些选项不在kubeletConfig里
				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					klog.Fatal(err)
				}
				// update feature gates based on new config
				if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					klog.Fatal(err)
				}
			}

			// We always validate the local configuration (command line + config file).
			// This is the default "last-known-good" config for dynamic config, and must always remain valid.
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig); err != nil {
				klog.Fatal(err)
			}

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
					klog.Fatal(err)
				}
				// If we should just use our existing, local config, the controller will return a nil config
				if dynamicKubeletConfig != nil {
					kubeletConfig = dynamicKubeletConfig
					// Note: flag precedence was already enforced in the controller, prior to validation,
					// by our above transform function. Now we simply update feature gates from the new config.
					if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
						klog.Fatal(err)
					}
				}
			}
```



**SetFromMap**

将kubeletConfig.FeatureGates保存到utilfeature.DefaultFeatureGate中。

 保存的Feature必须保证下面几个条件:

- 已知的featureSpec--在所有已知的FeatureSpec里--defaultKubernetesFeatureGates。
- 不允许修改锁定默认值的FeatureSpec。

特殊的FeatureSpec:

- AllAlpha将所有已知的FeatureSpec里的alpha阶段且未被明确设置(不在enabled中)FeatureSpec，enabled设置为指定值。
- AllBeta将所有已知的FeatureSpec里的beta阶段且未被明确设置(不在enabled中)FeatureSpec，enabled设置为指定值。

定义在staging\src\k8s.io\component-base\featuregate\feature_gate.go

```go
// SetFromMap stores flag gates for known features from a map[string]bool or returns an error
func (f *featureGate) SetFromMap(m map[string]bool) error {
	f.lock.Lock()
	defer f.lock.Unlock()

	// Copy existing state
	known := map[Feature]FeatureSpec{}
	for k, v := range f.known.Load().(map[Feature]FeatureSpec) {
		known[k] = v
	}
	enabled := map[Feature]bool{}
	for k, v := range f.enabled.Load().(map[Feature]bool) {
		enabled[k] = v
	}

	for k, v := range m {
		k := Feature(k)
        // 判断是否是已知的featureSpec,不是已知的,直接报错
		featureSpec, ok := known[k]
		if !ok {
			return fmt.Errorf("unrecognized feature gate: %s", k)
		}
		if featureSpec.LockToDefault && featureSpec.Default != v {
			return fmt.Errorf("cannot set feature gate %v to %v, feature is locked to %v", k, v, featureSpec.Default)
		}
		enabled[k] = v
		// Handle "special" features like "all alpha gates"
		// 遇到AllAlpha或AllBeta将enable值改为v
		if fn, found := f.special[k]; found {
			fn(known, enabled, v)
		}

		if featureSpec.PreRelease == Deprecated {
			klog.Warningf("Setting deprecated feature gate %s=%t. It will be removed in a future release.", k, v)
		} else if featureSpec.PreRelease == GA {
			klog.Warningf("Setting GA feature gate %s=%t. It will be removed in a future release.", k, v)
		}
	}

	// Persist changes
	f.known.Store(known)
	f.enabled.Store(enabled)

	klog.V(1).Infof("feature gates: %v", f.enabled)
	return nil
}

```

**kubeletConfigFlagPrecedence**

重新解析命令行参数，覆盖kubeletConfig配置。

- 新建一个flagset fs--只包含KubeletFlags和KubeletConfigFlags和忽略Globalsflag选项。

- 保存原始的kc.FeatureGates。

- 重新解析命令行参数，这里会将命令行FeatureGates参数保存到kc.FeatureGates。

- 聚合原来的kc.FeatureGates和命令行解析后的kc.FeatureGates，当原来的kc.FeatureGates（配置文件中的FeatureGate）不在命令行FeatureGates中，才会添加到kc.FeatureGates--当命令行FeatureGates与配置文件中的FeatureGates冲突时候，以命令行FeatureGates为准。

```go
// kubeletCosnfigFlagPrecedence re-parses flags over the KubeletConfiguration object.
// We must enforce flag precedence by re-parsing the command line into the new object.
// This is necessary to preserve backwards-compatibility across binary upgrades.
// See issue #56171 for more details.
func kubeletConfigFlagPrecedence(kc *kubeletconfiginternal.KubeletConfiguration, args []string) error {
	// We use a throwaway kubeletFlags and a fake global flagset to avoid double-parses,
	// as some Set implementations accumulate values from multiple flag invocations.
	fs := newFakeFlagSet(newFlagSetWithGlobals())
	// register throwaway KubeletFlags
	options.NewKubeletFlags().AddFlags(fs)
	// register new KubeletConfiguration
	options.AddKubeletConfigFlags(fs, kc)
	// Remember original feature gates, so we can merge with flag gates later
	original := kc.FeatureGates
	// re-parse flags
	if err := fs.Parse(args); err != nil {
		return err
	}
	// Add back feature gates that were set in the original kc, but not in flags
	for k, v := range original {
		if _, ok := kc.FeatureGates[k]; !ok {
			kc.FeatureGates[k] = v
		}
	}
	return nil
}
```

## 使用配置

以kubelet中的一段代码为例，使用utilfeature.DefaultFeatureGate.Enabled(key Feature) bool来判断是否启用了某个feature。

```go
import (
	utilfeature "k8s.io/apiserver/pkg/util/feature"
    features "k8s.io/kubernetes/pkg/features"
    .....
)

.....
// If the kubelet config controller is available, and dynamic config is enabled, start the config and status sync loops
	if utilfeature.DefaultFeatureGate.Enabled(features.DynamicKubeletConfig) && len(s.DynamicConfigDir.Value()) > 0 &&
		kubeDeps.KubeletConfigController != nil && !standaloneMode && !s.RunOnce {
		if err := kubeDeps.KubeletConfigController.StartSync(kubeDeps.KubeClient, kubeDeps.EventClient, string(nodeName)); err != nil {
			return err
		}
	}
```

Enabled判断是否启用某个feature

- 首先检查enabled里是否存在这个feature，因为只有明确设置的feature才会保存在这里，剩下所有默认的不会保存在这里。
- 判断是否是已知的feature，如果是已知的feature，则返回它的Default设置；否则pannic。

定义在cmd\kubelet\app\server.go

```go
// Enabled returns true if the key is enabled.  If the key is not known, this call will panic.
func (f *featureGate) Enabled(key Feature) bool {
	if v, ok := f.enabled.Load().(map[Feature]bool)[key]; ok {
		return v
	}
	if v, ok := f.known.Load().(map[Feature]FeatureSpec)[key]; ok {
		return v.Default
	}

	panic(fmt.Errorf("feature %q is not registered in FeatureGate %q", key, f.featureGateName))
}
```

