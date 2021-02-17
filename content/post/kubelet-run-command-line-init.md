---
title: "kubelet启动--命令行初始化"
date: 2021-02-17T12:07:53+08:00
draft: false
description: "kubernetes kubelet 源码解析之命令行初始化"
keywords:  [ "kubernetes", "kubelet","源码解析"]
tags:  ["kubernetes", "源码解析", "kubelet"]
categories:  ["kubernetes"]
---

# 概述

kubelet命令行参数有177个，是kubernetes组件中配置项最多的。

kubelet的选项分为三类，一类是kubletFlags--（启动之后不能随时改变的，只能通过命令行配置，比如ip地址、路径等）；另一类是kubeletConfiguration--（可以在启动之后进行修改，可以通过命令行设置或配置文件或者dynamic Kubelet config--configmap，通过命令行配置是DEPRECATED,一些选项只能通过配置文件设置--比如NodeStatusReportFrequency）；最后是globalFlags--（只能通过命令行配置，比如日志级别、日志路径、版本）。

kubelet是如何启动、初始化并识别这么多选项的？本文就是讲kubelet启动过程中的命令初始化，至于各个组件的运行过程，后续文章再来分析。

本文基于kubernetes 1.18.6版本，请访问[源代码阅读仓库](https://github.com/wu0407/kubernetes-code-analysis)。

## 启动过程

kubelet入口文件在`cmd\kubelet\kubelet.go`，主要流程

1. 初始化随机因子
2. 创建一个cobra Command
3. 初始化日志
4. 执行cobra Command



## 创建cobra Command

这个阶段进行命令行选项的定义、默认值的设置、command的各个字段定义、命令帮助函数和命令使用函数。

主要流程

- 这里新建一个name为kubelet的flagset，为了拥有干净的flagset（不会被CommandLine的flagset污染），而不用cobra的命令行解析。

- 同时设置flagset的NormalizeFunc--将flag名包含下划线"_"转为横杆”-“，比如--iptables-drop-bit和--iptables_drop_bit是一样的。

- 新建一个kubeletFlags里面包含了一些初始选项的默认值：

  具体的field对应的命令行选项见下面[kubeletFlags选项](#kubeletflags选项)，NewKubeletFlags()定义在cmd\kubelet\app\options\options.go

    ```go
    // NewKubeletFlags will create a new KubeletFlags with default values
    func NewKubeletFlags() *KubeletFlags {
        remoteRuntimeEndpoint := ""
        if runtime.GOOS == "linux" {
            remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock"
        } else if runtime.GOOS == "windows" {
            remoteRuntimeEndpoint = "npipe:////./pipe/dockershim"
        }

        return &KubeletFlags{
            EnableServer:                        true,
            ContainerRuntimeOptions:             *NewContainerRuntimeOptions(),
            CertDirectory:                       "/var/lib/kubelet/pki",
            RootDirectory:                       defaultRootDir, // defaultRootDir为/var/lib/kubelet
            MasterServiceNamespace:              metav1.NamespaceDefault, // NamespaceDefault为default
            MaxContainerCount:                   -1,
            MaxPerPodContainerCount:             1,
            MinimumGCAge:                        metav1.Duration{Duration: 0},
            NonMasqueradeCIDR:                   "10.0.0.0/8",
            RegisterSchedulable:                 true,
            ExperimentalKernelMemcgNotification: false,
            RemoteRuntimeEndpoint:               remoteRuntimeEndpoint,
            NodeLabels:                          make(map[string]string),
            VolumePluginDir:                     "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/",
            RegisterNode:                        true,
            SeccompProfileRoot:                  filepath.Join(defaultRootDir, "seccomp"),
            // prior to the introduction of this flag, there was a hardcoded cap of 50 images
            NodeStatusMaxImages:         50,
            EnableCAdvisorJSONEndpoints: false,
        }
    }
    ```

  其中NewContainerRuntimeOptions()定义在cmd\kubelet\app\options\container_runtime.go

    ```go
    const (
        // When these values are updated, also update test/utils/image/manifest.go
        defaultPodSandboxImageName    = "k8s.gcr.io/pause"
        defaultPodSandboxImageVersion = "3.2"
    )

    var (
        defaultPodSandboxImage = defaultPodSandboxImageName +
            ":" + defaultPodSandboxImageVersion
    )

    // NewContainerRuntimeOptions will create a new ContainerRuntimeOptions with
    // default values.
    func NewContainerRuntimeOptions() *config.ContainerRuntimeOptions {
        dockerEndpoint := ""
        if runtime.GOOS != "windows" {
            dockerEndpoint = "unix:///var/run/docker.sock"
        }

        return &config.ContainerRuntimeOptions{
            ContainerRuntime:           kubetypes.DockerContainerRuntime, //DockerContainerRuntime为docker
            RedirectContainerStreaming: false,
            DockerEndpoint:             dockerEndpoint,
            DockershimRootDirectory:    "/var/lib/dockershim",
            PodSandboxImage:            defaultPodSandboxImage,
            ImagePullProgressDeadline:  metav1.Duration{Duration: 1 * time.Minute},
            ExperimentalDockershim:     false,

            //Alpha feature
            CNIBinDir:   "/opt/cni/bin",
            CNIConfDir:  "/etc/cni/net.d",
            CNICacheDir: "/var/lib/cni/cache",
        }
    }
    ```
  
- 创建一个kubeletConfiguration，用于设置kubeletConfiguration的flag选项的默认值，它通过创建v1beta1的KubeletConfiguration 然后通过scheme.Default()生成默认的配置，再通过scheme.Convert转换成内部的KubeletConfiguration。

  具体的field对应的命令行选项见下面[kubeletConfigurationFlag选项](#kubeletconfigurationflag选项)

  ```go
   KubeletConfiguration{
  		SyncFrequency: metav1.Duration{Duration: 1 * time.Minute},
  		FileCheckFrequency: metav1.Duration{Duration: 20 * time.Second},
  		HTTPCheckFrequency: metav1.Duration{Duration: 20 * time.Second},
  		Address: "0.0.0.0",
  		Port: 10250,
  		Authentication.Anonymous.Enabled: true, // by applyLegacyDefaults, by default false,
  		Authentication.Webhook.Enabled: false, // by applyLegacyDefaults, by default true,
  		Authentication.Webhook.CacheTTL: metav1.Duration{Duration: 2 * time.Minute},
  		Authorization.Mode: "AlwaysAllow", // by applyLegacyDefaults, by default "webhook",
  		Authorization.Webhook.CacheAuthorizedTTL: metav1.Duration{Duration: 5 * time.Minute},
  		Authorization.Webhook.CacheUnauthorizedTTL: metav1.Duration{Duration: 30 * time.Second},
  		ReadOnlyPort: 10255, // by applyLegacyDefaults,
  		RegistryPullQPS: 5,
  		RegistryBurst: 10,
  		EventRecordQPS: 5,
  		EventBurst: 10,
  		EnableDebuggingHandlers: true,
  		HealthzPort: 10248,
  		HealthzBindAddress: "127.0.0.1",
  		OOMScoreAdj: -999,
  		StreamingConnectionIdleTimeout: metav1.Duration{Duration: 4 * time.Hour},
  		NodeStatusReportFrequency: metav1.Duration{Duration: 5 * time.Minute},
  		NodeStatusUpdateFrequency: metav1.Duration{Duration: 10 * time.Second},
  		NodeLeaseDurationSeconds: 40,
  		ImageMinimumGCAge: metav1.Duration{Duration: 2 * time.Minute},
  		ImageGCHighThresholdPercent: 85,
  		ImageGCLowThresholdPercent: 80,
  		VolumeStatsAggPeriod: metav1.Duration{Duration: time.Minute},
  		CgroupsPerQOS: true,
  		CgroupDriver: "cgroupfs",
  		CPUManagerPolicy: "none",
  		CPUManagerReconcilePeriod: metav1.Duration{Duration: 10 * time.Second},
  		TopologyManagerPolicy: "none",
  		RuntimeRequestTimeout: metav1.Duration{Duration: 2 * time.Minute},
  		HairpinMode: "promiscuous-bridge",
  		MaxPods: 110,
  		PodPidsLimit: -1,
  		ResolverConfig: "/etc/resolv.conf",
  		CPUCFSQuota: true,
  		CPUCFSQuotaPeriod: metav1.Duration{Duration: 100 * time.Millisecond},
  		MaxOpenFiles: 1000000,
  		ContentType: "application/vnd.kubernetes.protobuf",
  		KubeAPIQPS: 5,
  		KubeAPIBurst: 10,
  		SerializeImagePulls: true,
  		EvictionHard: map[string]string{
  			"memory.available":  "100Mi",
  			"nodefs.available":  "10%",
  			"nodefs.inodesFree": "5%",
  			"imagefs.available": "15%",
  		},
  		EvictionPressureTransitionPeriod: metav1.Duration{Duration: 5 * time.Minute},
  		EnableControllerAttachDetach: true,
  		MakeIPTablesUtilChains: true,
  		IPTablesMasqueradeBit: 14,
  		IPTablesDropBit: 15,
  		FailSwapOn: true,
  		ContainerLogMaxSize: "10Mi",
  		ContainerLogMaxFiles: 5,
  		ConfigMapAndSecretChangeDetectionStrategy: "Watch",
  		EnforceNodeAllocatable: []string{"pods"},
  	}
  ```

  

- 创建cobra.Command，定义了Use、Long、DisableFlagParsing、Run这四个字段。

- kubeletFlags选项添加到flagset

- kubeletConfigurationFlag选项添加到flagset

- globalFlag添加到flagset，globalFlag包含klog、verflag、Cadvisor、logflush相关选项

- 设置help选项

- 自定义command的helpfunc和usageFunc

看到这里会发现kubelet并没有使用cobra的命令解析和定义、helpfunc和usageFunc，这是为什么呢？

是因为globalFlag里面定义的选项flag，都是定义在pflag.Command下面--通过import模块自动注册。而cobra默认的helpfunc和usageFunc和excute会执行mergePersistentFlags()--会引入pflag.Command的flag，由于历史原因有一些不需要的选项也注册在pflag.Command， 它们也会被引用进来，所以才会新建一个flagset并自己解析命令，保持干净。

代码注释中也提到了，只是不是很详细

```go
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


// AddGlobalFlags explicitly registers flags that libraries (glog, verflag, etc.) register
// against the global flagsets from "flag" and "github.com/spf13/pflag".
// We do this in order to prevent unwanted flags from leaking into the Kubelet's flagset.
func AddGlobalFlags(fs *pflag.FlagSet) {
```



### kubeletFlags选项

kubeletFlags 包含kubelet运行中不能改变的选项、或者不能安全改变的选项、不能再不同的实例共享的。

KubeletConfiguration定义了不同实例能够共享的选项。

> // KubeletFlags contains configuration flags for the Kubelet.
>
> // A configuration field should go in KubeletFlags instead of KubeletConfiguration if any of these are true:
>
> // - its value will never, or cannot safely be changed during the lifetime of a node, or
>
> // - its value cannot be safely shared between nodes at the same time (e.g. a hostname);
>
> //  KubeletConfiguration is intended to be shared between nodes.



| categories分类                        | var变量                                            | flag                                                | default默认值                                        | description                                                  |
| ------------------------------------- | -------------------------------------------------- | --------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| ContainerRuntimeOptions               | ContainerRuntime                                   | --container-runtime                                 | docker                                               | The container runtime to use. Possible values: 'docker', 'remote'. |
| ContainerRuntimeOptions               | RuntimeCgroups                                     | --runtime-cgroups                                   | ""                                                   | Optional absolute name of cgroups to create and run the runtime in. |
| ContainerRuntimeOptions               | RedirectContainerStreaming                         | --redirect-container-streaming                      | false                                                | Enables container streaming redirect. If false, kubelet will proxy container streaming data between apiserver and container runtime; if true, kubelet will return an http redirect to apiserver, and apiserver will access container runtime directly. The proxy approach is more secure, but introduces some overhead. The redirect approach is more performant, but less secure because the connection between apiserver and container runtime may not be authenticated. |
| ContainerRuntimeOptions  docker       | ExperimentalDockershim                             | --experimental-dockershim                           | false                                                | Enable dockershim only mode. In this mode, kubelet will only start dockershim without any other functionalities. This flag only serves test purpose, please do not use it unless you are conscious of what you are doing. [default=false] |
| ContainerRuntimeOptions     docker    | DockershimRootDirectory                            | --experimental-dockershim-root-directory            | /var/lib/dockershim                                  | Path to the dockershim root directory.                       |
| ContainerRuntimeOptions        docker | PodSandboxImage                                    | --pod-infra-container-image                         | k8s.gcr.io/pause:3.2                                 | The image whose network/ipc namespaces containers in each pod will use. |
| ContainerRuntimeOptions        docker | DockerEndpoint                                     | --docker-endpoint                                   | unix:///var/run/docker.sock                          | Use this for the docker endpoint to communicate with.        |
| ContainerRuntimeOptions        docker | ImagePullProgressDeadline.Duration                 | --image-pull-progress-deadline                      | 1 * time.Minute                                      | If no pulling progress is made before this deadline, the image pulling will be cancelled. |
| ContainerRuntimeOptions        docker | NetworkPluginName                                  | --network-plugin                                    | ""                                                   | <Warning: Alpha feature> The name of the network plugin to be invoked for various events in kubelet/pod lifecycle. |
| ContainerRuntimeOptions        docker | CNIConfDir                                         | --cni-conf-dir                                      | /opt/cni/conf.d                                      | "<Warning: Alpha feature> The full path of the directory in which to search for CNI config files. |
| ContainerRuntimeOptions        docker | CNIBinDir                                          | --cni-bin-dir                                       | /opt/cni/bin                                         | <Warning: Alpha feature> A comma-separated list of full paths of directories in which to search for CNI plugin binaries. |
| ContainerRuntimeOptions        docker | CNICacheDir                                        | --cni-cache-dir                                     | /var/lib/cni/cache                                   | <Warning: Alpha feature> The full path of the directory in which CNI should store cache files. |
| ContainerRuntimeOptions        docker | NetworkPluginMTU                                   | --network-plugin-mtu                                | 0                                                    | <Warning: Alpha feature> The MTU to be passed to the network plugin, to override the default. Set to 0 to use the default 1460 MTU. |
| main                                  | KubeletConfigFile                                  | --config                                            | ""                                                   | The Kubelet will load its initial configuration from this file. The path may be absolute or relative; relative paths start at the Kubelet's current working directory. Omit this flag to use the built-in default configuration values. Command-line flags override configuration from this file. |
| main                                  | KubeConfig                                         | --kubeconfig                                        | ""                                                   | Path to a kubeconfig file, specifying how to connect to the API server. Providing --kubeconfig enables API server mode, omitting --kubeconfig enables standalone mode. |
| main                                  | BootstrapKubeconfig                                | --bootstrap-kubeconfig                              | ""                                                   | Path to a kubeconfig file that will be used to get client certificate for kubelet. If the file specified by --kubeconfig does not exist, the bootstrap kubeconfig is used to request a client certificate from the API server. On success, a kubeconfig file referencing the generated client certificate and key is written to the path specified by --kubeconfig. The client certificate and key file will be stored in the directory pointed by --cert-dir. |
| main                                  | ReallyCrashForTesting                              | --really-crash-for-testing                          | false                                                | If true, when panics occur crash. Intended for testing.      |
| main                                  | ChaosChance                                        | --chaos-chance                                      | 0                                                    | If > 0.0, introduce random client errors and latency. Intended for testing. |
| main                                  | RunOnce                                            | --runonce                                           | false                                                | If true, exit after spawning pods from static pod files or remote urls. Exclusive with --enable-server |
| main                                  | EnableServer                                       | --enable-server                                     | true                                                 | Enable the Kubelet's server                                  |
| main                                  | HostnameOverride                                   | --hostname-override                                 | ""                                                   | If non-empty, will use this string as identification instead of the actual hostname. If --cloud-provider is set, the cloud provider determines the name of the node (consult cloud provider documentation to determine if and how the hostname is used). |
| main                                  | NodeIP                                             | --node-ip                                           | ""                                                   | IP address of the node. If set, kubelet will use this IP address for the node. If unset, kubelet will use the node's default IPv4 address, if any, or its default IPv6 address if it has no IPv4 addresses. You can pass `::` to make it prefer the default IPv6 address rather than the default IPv4 address. |
| main                                  | ProviderID                                         | --provider-id                                       | ""                                                   | Unique identifier for identifying the node in a machine database, i.e cloudprovider |
| main                                  | CertDirectory                                      | --cert-dir                                          | /var/lib/kubelet/pki                                 | The directory where the TLS certs are located. If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. |
| main                                  | CloudProvider                                      | --cloud-provider                                    | ""                                                   | The path to the cloud provider configuration file. Empty string for no configuration file. |
| main                                  | CloudConfigFile                                    | --cloud-config                                      | ""                                                   | The path to the cloud provider configuration file. Empty string for no configuration file. |
| main                                  | RootDirectory                                      | --root-dir                                          | /var/lib/kubelet                                     | Directory path for managing kubelet files (volume mounts,etc). |
| main                                  | DynamicConfigDir                                   | --dynamic-config-dir                                | ""                                                   | The Kubelet will use this directory for checkpointing downloaded configurations and tracking configuration health. The Kubelet will create this directory if it does not already exist. The path may be absolute or relative; relative paths start at the Kubelet's current working directory. Providing this flag enables dynamic Kubelet configuration. The DynamicKubeletConfig feature gate must be enabled to pass this flag; this gate currently defaults to true because the feature is beta. |
| main                                  | RegisterNode                                       | --register-node                                     | true                                                 | Register the node with the apiserver. If --kubeconfig is not provided, this flag is irrelevant, as the Kubelet won't have an apiserver to register with. |
| experimental                          | ExperimentalMounterPath                            | --experimental-mounter-path                         | ""                                                   | [Experimental] Path of mounter binary. Leave empty to use the default mount. |
| experimental                          | ExperimentalKernelMemcgNotification                | --experimental-kernel-memcg-notification            | false                                                | If enabled, the kubelet will integrate with the kernel memcg notification to determine if memory eviction thresholds are crossed rather than polling. |
| experimental                          | RemoteRuntimeEndpoint                              | --container-runtime-endpoint                        | unix:///var/run/dockershim.sock                      | "[Experimental] The endpoint of remote runtime service. Currently unix socket endpoint is supported on Linux, while npipe and tcp endpoints are supported on windows. Examples:'unix:///var/run/dockershim.sock', 'npipe:////./pipe/dockershim' |
| experimental                          | RemoteImageEndpoint                                | --image-service-endpoint                            | ""                                                   | [Experimental] The endpoint of remote image service. If not specified, it will be the same with container-runtime-endpoint by default. Currently unix socket endpoint is supported on Linux, while npipe and tcp endpoints are supported on windows. Examples:'unix:///var/run/dockershim.sock', 'npipe:////./pipe/dockershim' |
| experimental                          | ExperimentalCheckNodeCapabilitiesBeforeMount       | --experimental-check-node-capabilities-before-mount | false                                                | [Experimental] if set true, the kubelet will check the underlying node for required components (binaries, etc.) before performing the mount |
| experimental                          | ExperimentalNodeAllocatableIgnoreEvictionThreshold | --experimental-allocatable-ignore-eviction          | false                                                | When set to 'true', Hard Eviction Thresholds will be ignored while calculating Node Allocatable. See https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/ for more details. [default=false] |
| experimental                          | NodeLabels                                         | --node-labels                                       | ""                                                   | <Warning: Alpha feature> Labels to add when registering the node in the cluster.  Labels must be key=value pairs separated by ','. Labels in the 'kubernetes.io' namespace must begin with an allowed prefix (kubelet.kubernetes.io, node.kubernetes.io) or be in the specifically allowed set (beta.kubernetes.io/arch, beta.kubernetes.io/instance-type, beta.kubernetes.io/os, failure-domain.beta.kubernetes.io/region, failure-domain.beta.kubernetes.io/zone, kubernetes.io/arch, kubernetes.io/hostname, kubernetes.io/os, node.kubernetes.io/instance-type, topology.kubernetes.io/region, topology.kubernetes.io/zone) |
| experimental                          | VolumePluginDir                                    | --volume-plugin-dir                                 | /usr/libexec/kubernetes/kubelet-plugins/volume/exec/ | The full path of the directory in which to search for additional third party volume plugins |
| experimental                          | LockFilePath                                       | --lock-file                                         | ""                                                   | <Warning: Alpha feature> The path to file for kubelet to use as a lock file. |
| experimental                          | ExitOnLockContention                               | --exit-on-lock-contention                           | false                                                | Whether kubelet should exit upon lock-file contention.       |
| experimental                          | SeccompProfileRoot                                 | --seccomp-profile-root                              | /var/lib/kubelet/seccomp                             | <Warning: Alpha feature> Directory path for seccomp profiles. |
| experimental                          | BootstrapCheckpointPath                            | --bootstrap-checkpoint-path                         | ""                                                   | <Warning: Alpha feature> Path to the directory where the checkpoints are stored |
| experimental                          | NodeStatusMaxImages                                | --node-status-max-images                            | 50                                                   | <Warning: Alpha feature> The maximum number of images to report in Node.Status.Images. If -1 is specified, no cap will be applied. |
| deprecated                            | BootstrapKubeconfig                                | --experimental-bootstrap-kubeconfig                 | ""                                                   | Use --bootstrap-kubeconfig                                   |
| deprecated                            | MinimumGCAge.Duration                              | --minimum-container-ttl-duration                    | 0                                                    | Minimum age for a finished container before it is garbage collected. Examples: '300ms', '10s' or '2h45m'. Use --eviction-hard or --eviction-soft instead. Will be removed in a future version. |
| deprecated                            | MaxPerPodContainerCount                            | --maximum-dead-containers-per-container             | 1                                                    | Maximum number of old instances to retain per container. Each container takes up some disk space. Use --eviction-hard or --eviction-soft instead. Will be removed in a future version. |
| deprecated                            | MaxContainerCount                                  | --maximum-dead-containers                           | -1                                                   | Maximum number of old instances of containers to retain globally. Each container takes up some disk space. To disable, set to a negative number. Use --eviction-hard or --eviction-soft instead. Will be removed in a future version. |
| deprecated                            | MasterServiceNamespace                             | --master-service-namespace                          | default                                              | The namespace from which the kubernetes master services should be injected into pods. This flag will be removed in a future version. |
| deprecated                            | RegisterSchedulable                                | --register-schedulable                              | true                                                 | Register the node as schedulable. Won't have any effect if register-node is false. This flag will be removed in a future version. |
| deprecated                            | NonMasqueradeCIDR                                  | --non-masquerade-cidr                               | 10.0.0.0/8                                           | Traffic to IPs outside this range will use IP masquerade. Set to '0.0.0.0/0' to never masquerade. This flag will be removed in a future version. |
| deprecated                            | KeepTerminatedPodVolumes                           | --keep-terminated-pod-volumes                       | false                                                | Keep terminated pod volumes mounted to the node after the pod terminates. Can be useful for debugging volume related issues.   This flag will be removed in a future version. |
| deprecated                            | EnableCAdvisorJSONEndpoints                        | --enable-cadvisor-json-endpoints                    | false                                                | Enable cAdvisor json /spec and /stats/* endpoints. This flag will be removed in a future version. |



### kubeletConfigurationFlag选项

kubeletConfigurationFlag分为两类，  一类`deprecated command line`是可以在命令行（即将被废弃）和配置文件和dynamic kubelet config中配置的，另一类`configuration`是不能配置在命令行的，只能在配置文件中和dynamic kubelet config中。

| categories分类          | var变量                                             | flag                                           | default默认值                                                | description                                                  |
| ----------------------- | --------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| deprecated command line | FailSwapOn                                          | --fail-swap-on                                 | true                                                         | Makes the Kubelet fail to start if swap is enabled on the node. |
| deprecated command line | StaticPodPath                                       | --pod-manifest-path                            | ""                                                           | Path to the directory containing static pod files to run, or the path to a single static pod file. Files starting with dots will be ignored. |
| deprecated command line | SyncFrequency.Duration                              | --sync-frequency                               | 1 * time.Minute                                              | Max period between synchronizing running containers and config |
| deprecated command line | FileCheckFrequency.Duration                         | --file-check-frequency                         | 20 * time.Second                                             | Duration between checking config files for new data          |
| deprecated command line | HTTPCheckFrequency.Duration                         | --http-check-frequency                         | 20 * time.Second                                             | Duration between checking http for new data                  |
| deprecated command line | StaticPodURL                                        | --manifest-url                                 | ""                                                           | URL for accessing additional Pod specifications to run       |
| deprecated command line | StaticPodURLHeader                                  | --manifest-url-header                          | ""                                                           | Comma-separated list of HTTP headers to use when accessing the url provided to --manifest-url. Multiple headers with the same name will be added in the same order provided. This flag can be repeatedly invoked. For example: `--manifest-url-header 'a:hello,b:again,c:world' --manifest-url-header 'b:beautiful'` |
| deprecated command line | Address                                             | --address                                      | 0.0.0.0                                                      | The IP address for the Kubelet to serve on (set to `0.0.0.0` for all IPv4 interfaces and `::` for all IPv6 interfaces) |
| deprecated command line | Port                                                | --port                                         | 10250                                                        | The port for the Kubelet to serve on.                        |
| deprecated command line | ReadOnlyPort                                        | --read-only-port                               | 0                                                            | The read-only port for the Kubelet to serve on with no authentication/authorization (set to 0 to disable) |
| deprecated command line | Authentication.Anonymous.Enabled                    | --anonymous-auth                               | true                                                         | Enables anonymous requests to the Kubelet server. Requests that are not rejected by another authentication method are treated as anonymous requests. Anonymous requests have a username of system:anonymous, and a group name of system:unauthenticated. |
| deprecated command line | Authentication.Webhook.Enabled                      | --authentication-token-webhook                 | false                                                        | Use the TokenReview API to determine authentication for bearer tokens. |
| deprecated command line | Authentication.Webhook.CacheTTL.Duration            | --authentication-token-webhook-cache-ttl       | 2 * time.Minute                                              | The duration to cache responses from the webhook token authenticator. |
| deprecated command line | Authentication.X509.ClientCAFile                    | --client-ca-file                               | ""                                                           | If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate. |
| deprecated command line | Authorization.Mode                                  | --authorization-mode                           | AlwaysAllow                                                  | Authorization mode for Kubelet server. Valid options are AlwaysAllow or Webhook. Webhook mode uses the SubjectAccessReview API to determine authorization. |
| deprecated command line | Authorization.Webhook.CacheAuthorizedTTL.Duration   | --authorization-webhook-cache-authorized-ttl   | 5 * time.Minute                                              | The duration to cache 'authorized' responses from the webhook authorizer. |
| deprecated command line | Authorization.Webhook.CacheUnauthorizedTTL.Duration | --authorization-webhook-cache-unauthorized-ttl | 30 * time.Second                                             | The duration to cache 'unauthorized' responses from the webhook authorizer. |
| deprecated command line | TLSCertFile                                         | --tls-cert-file                                | ""                                                           | File containing x509 Certificate used for serving HTTPS (with intermediate certs, if any, concatenated after server cert). If --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory passed to --cert-dir. |
| deprecated command line | TLSPrivateKeyFile                                   | --tls-private-key-file                         | ""                                                           | File containing x509 private key matching --tls-cert-file.   |
| deprecated command line | ServerTLSBootstrap                                  | --rotate-server-certificates                   | false                                                        | Auto-request and rotate the kubelet serving certificates by requesting new certificates from the kube-apiserver when the certificate expiration approaches. Requires the RotateKubeletServerCertificate feature gate to be enabled, and approval of the submitted CertificateSigningRequest objects. |
| deprecated command line | TLSCipherSuites                                     | --tls-cipher-suites                            | ""                                                           | Comma-separated list of cipher suites for the server. If omitted, the default Go cipher suites will be used. <br/>                Preferred values: TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256, TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305, TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256, TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256, TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA, TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384, TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305, TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256, TLS_RSA_WITH_3DES_EDE_CBC_SHA, TLS_RSA_WITH_AES_128_CBC_SHA, TLS_RSA_WITH_AES_128_GCM_SHA256, TLS_RSA_WITH_AES_256_CBC_SHA, TLS_RSA_WITH_AES_256_GCM_SHA384. <br/>                Insecure values: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256, TLS_ECDHE_ECDSA_WITH_RC4_128_SHA, TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256, TLS_ECDHE_RSA_WITH_RC4_128_SHA, TLS_RSA_WITH_AES_128_CBC_SHA256, TLS_RSA_WITH_RC4_128_SHA. |
| deprecated command line | TLSMinVersion                                       | --tls-min-version                              | ""                                                           | Minimum TLS version supported. Possible values: VersionTLS10, VersionTLS11, VersionTLS12, VersionTLS13 |
| deprecated command line | RotateCertificates                                  | --rotate-certificates                          | false                                                        | <Warning: Beta feature> Auto rotate the kubelet client certificates by requesting new certificates from the kube-apiserver when the certificate expiration approaches. |
| deprecated command line | RegistryPullQPS                                     | --registry-qps                                 | 5                                                            | If > 0, limit registry pull QPS to this value. If 0, unlimited. |
| deprecated command line | RegistryBurst                                       | --registry-burst                               | 10                                                           | Maximum size of a bursty pulls, temporarily allows pulls to burst to this number, while still not exceeding registry-qps. Only used if --registry-qps > 0 |
| deprecated command line | EventRecordQPS                                      | --event-qps                                    | 5                                                            | If > 0, limit event creations per second to this value. If 0, unlimited. |
| deprecated command line | EventBurst                                          | --event-burst                                  | 10                                                           | Maximum size of a bursty event records, temporarily allows event records to burst to this number, while still not exceeding event-qps. Only used if --event-qps > 0 |
| deprecated command line | EnableDebuggingHandlers                             | --enable-debugging-handlers                    | true                                                         | Enables server endpoints for log collection and local running of containers and commands |
| deprecated command line | EnableContentionProfiling                           | --contention-profiling                         | false                                                        | Enable lock contention profiling, if profiling is enabled    |
| deprecated command line | HealthzPort                                         | --healthz-port                                 | 10248                                                        | The port of the localhost healthz endpoint (set to 0 to disable) |
| deprecated command line | HealthzBindAddress                                  | --healthz-bind-address                         | 127.0.0.1                                                    | The IP address for the healthz server to serve on (set to `0.0.0.0` for all IPv4 interfaces and `::` for all IPv6 interfaces) |
| deprecated command line | OOMScoreAdj                                         | --oom-score-adj                                | -999                                                         | The oom-score-adj value for kubelet process. Values must be within the range [-1000, 1000] |
| deprecated command line | ClusterDomain                                       | --cluster-domain                               | ""                                                           | Domain for this cluster. If set, kubelet will configure all containers to search this domain in addition to the host's search domains |
| deprecated command line | ClusterDNS                                          | --cluster-dns                                  | ""                                                           | Comma-separated list of DNS server IP address. This value is used for containers DNS server in case of Pods with \"dnsPolicy=ClusterFirst\". Note: all DNS servers appearing in the list MUST serve the same set of records otherwise name resolution within the cluster may not work correctly. There is no guarantee as to which DNS server may be contacted for name resolution. |
| deprecated command line | StreamingConnectionIdleTimeout.Duration             | --streaming-connection-idle-timeout            | 4 * time.Hour                                                | Maximum time a streaming connection can be idle before the connection is automatically closed. 0 indicates no timeout. Example: '5m' |
| deprecated command line | NodeStatusUpdateFrequency.Duration                  | --node-status-update-frequency                 | 10 * time.Second                                             | Specifies how often kubelet posts node status to master. Note: be cautious when changing the constant, it must work with nodeMonitorGracePeriod in nodecontroller. |
| deprecated command line | ImageMinimumGCAge.Duration                          | --minimum-image-ttl-duration                   | 2 * time.Minute                                              | Minimum age for an unused image before it is garbage collected. Examples: '300ms', '10s' or '2h45m'. |
| deprecated command line | ImageGCHighThresholdPercent                         | --image-gc-high-threshold                      | 85                                                           | The percent of disk usage after which image garbage collection is always run. Values must be within the range [0, 100], To disable image garbage collection, set to 100. |
| deprecated command line | ImageGCLowThresholdPercent                          | --image-gc-low-threshold                       | 80                                                           | The percent of disk usage before which image garbage collection is never run. Lowest disk usage to garbage collect to. Values must be within the range [0, 100] and should not be larger than that of --image-gc-high-threshold. |
| deprecated command line | VolumeStatsAggPeriod.Duration                       | --volume-stats-agg-period                      | 1 * time.Minute                                              | pecifies interval for kubelet to calculate and cache the volume disk usage for all pods and volumes. To disable volume calculations, set to 0. |
| deprecated command line | FeatureGates                                        | --feature-gates                                | ""                                                           | A set of key=value pairs that describe feature gates for alpha/experimental features. |
| deprecated command line | KubeletCgroups                                      | --kubelet-cgroups                              | ""                                                           | Optional absolute name of cgroups to create and run the Kubelet in. |
| deprecated command line | SystemCgroups                                       | --system-cgroups                               | ""                                                           | Optional absolute name of cgroups in which to place all non-kernel processes that are not already inside a cgroup under `/`. Empty for no container. Rolling back the flag requires a reboot. |
| deprecated command line | CgroupsPerQOS                                       | --cgroups-per-qos                              | true                                                         | Enable creation of QoS cgroup hierarchy, if true top level QoS and pod cgroups are created. |
| deprecated command line | CgroupDriver                                        | --cgroup-driver                                | cgroupfs                                                     | Driver that the kubelet uses to manipulate cgroups on the host. Possible values: 'cgroupfs', 'systemd' |
| deprecated command line | CgroupRoot                                          | --cgroup-root                                  | ""                                                           | Optional root cgroup to use for pods. This is handled by the container runtime on a best effort basis. Default: '', which means use the container runtime default. |
| deprecated command line | CPUManagerPolicy                                    | --cpu-manager-policy                           | none                                                         | CPU Manager policy to use. Possible values: 'none', 'static'. Default: 'none' |
| deprecated command line | CPUManagerReconcilePeriod.Duration                  | --cpu-manager-reconcile-period                 | 10 * time.Second                                             | <Warning: Alpha feature> CPU Manager reconciliation period. Examples: '10s', or '1m'. If not supplied, defaults to `NodeStatusUpdateFrequency` |
| deprecated command line | QOSReserved                                         | --qos-reserved                                 | ""                                                           | <Warning: Alpha feature> A set of ResourceName=Percentage (e.g. memory=50%) pairs that describe how pod resource requests are reserved at the QoS level. Currently only memory is supported. Requires the QOSReserved feature gate to be enabled. |
| deprecated command line | TopologyManagerPolicy                               | --topology-manager-policy                      | "none"                                                       | Topology Manager policy to use. Possible values: 'none', 'best-effort', 'restricted', 'single-numa-node'. |
| deprecated command line | RuntimeRequestTimeout.Duration                      | --runtime-request-timeout                      | 2 * time.Minute                                              | Timeout of all runtime requests except long running request - pull, logs, exec and attach. When timeout exceeded, kubelet will cancel the request, throw out an error and retry later. |
| deprecated command line | HairpinMode                                         | --hairpin-mode                                 | promiscuous-bridge                                           | How should the kubelet setup hairpin NAT. This allows endpoints of a Service to loadbalance back to themselves if they should try to access their own Service. Valid values are \"promiscuous-bridge\", \"hairpin-veth\" and \"none\". |
| deprecated command line | MaxPods                                             | --max-pods                                     | 110                                                          | Number of Pods that can run on this Kubelet.                 |
| deprecated command line | PodCIDR                                             | --pod-cidr                                     | ""                                                           | The CIDR to use for pod IP addresses, only used in standalone mode. In cluster mode, this is obtained from the master. For IPv6, the maximum number of IP's allocated is 65536 |
| deprecated command line | PodPidsLimit                                        | --pod-max-pids                                 | -1                                                           | Set the maximum number of processes per pod. If -1, the kubelet defaults to the node allocatable pid capacity. |
| deprecated command line | ResolverConfig                                      | --resolv-conf                                  | /etc/resolv.conf                                             | Resolver configuration file used as the basis for the container DNS resolution configuration. |
| deprecated command line | CPUCFSQuota                                         | --cpu-cfs-quota                                | true                                                         | Enable CPU CFS quota enforcement for containers that specify CPU limits |
| deprecated command line | CPUCFSQuotaPeriod                                   | --cpu-cfs-quota-period                         | 100 * time.Millisecond                                       | Sets CPU CFS quota period value, cpu.cfs_period_us, defaults to Linux Kernel default |
| deprecated command line | EnableControllerAttachDetach                        | --enable-controller-attach-detach              | true                                                         | Enables the Attach/Detach controller to manage attachment/detachment of volumes scheduled to this node, and disables kubelet from executing any attach/detach operations |
| deprecated command line | MakeIPTablesUtilChains                              | --make-iptables-util-chains                    | true                                                         | If true, kubelet will ensure iptables utility rules are present on host. |
| deprecated command line | IPTablesMasqueradeBit                               | --iptables-masquerade-bit                      | 14                                                           | The bit of the fwmark space to mark packets for SNAT. Must be within the range [0, 31]. Please match this parameter with corresponding parameter in kube-proxy. |
| deprecated command line | IPTablesDropBit                                     | --iptables-drop-bit                            | 15                                                           | The bit of the fwmark space to mark packets for dropping. Must be within the range [0, 31]. |
| deprecated command line | ContainerLogMaxSize                                 | --container-log-max-size                       | 10Mi                                                         | <Warning: Beta feature> Set the maximum size (e.g. 10Mi) of container log file before it is rotated. This flag can only be used with --container-runtime=remote. |
| deprecated command line | ContainerLogMaxFiles                                | --container-log-max-files                      | 5                                                            | <Warning: Beta feature> Set the maximum number of container log files that can be present for a container. The number must be >= 2. This flag can only be used with --container-runtime=remote. |
| deprecated command line | AllowedUnsafeSysctls                                | --allowed-unsafe-sysctls                       | ""                                                           | Comma-separated whitelist of unsafe sysctls or unsafe sysctl patterns (ending in *). Use these at your own risk. |
| deprecated command line | MaxOpenFiles                                        | --max-open-files                               | 1000000                                                      | Number of files that can be opened by Kubelet process.       |
| deprecated command line | ContentType                                         | --kube-api-content-type                        | application/vnd.kubernetes.protobuf                          | Content type of requests sent to apiserver.                  |
| deprecated command line | KubeAPIQPS                                          | --kube-api-qps                                 | 5                                                            | QPS to use while talking with kubernetes apiserver           |
| deprecated command line | KubeAPIBurst                                        | --kube-api-burst                               | 10                                                           | Burst to use while talking with kubernetes apiserver         |
| deprecated command line | SerializeImagePulls                                 | --serialize-image-pulls                        | true                                                         | Pull images one at a time. We recommend *not* changing the default value on nodes that run docker daemon with version < 1.9 or an Aufs storage backend. Issue #10959 has more details. |
| deprecated command line | EvictionHard                                        | --eviction-hard                                | memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<15% | A set of eviction thresholds (e.g. memory.available<1Gi) that if met would trigger a pod eviction. |
| deprecated command line | EvictionSoft                                        | --eviction-soft                                | ""                                                           | A set of eviction thresholds (e.g. memory.available<1.5Gi) that if met over a corresponding grace period would trigger a pod eviction. |
| deprecated command line | EvictionSoftGracePeriod                             | --eviction-soft-grace-period                   | ""                                                           | A set of eviction grace periods (e.g. memory.available=1m30s) that correspond to how long a soft eviction threshold must hold before triggering a pod eviction. |
| deprecated command line | EvictionPressureTransitionPeriod.Duration           | --eviction-pressure-transition-period          | 5 * time.Minute                                              | Duration for which the kubelet has to wait before transitioning out of an eviction pressure condition. |
| deprecated command line | EvictionMaxPodGracePeriod                           | --eviction-max-pod-grace-period                | 0                                                            | Maximum allowed grace period (in seconds) to use when terminating pods in response to a soft eviction threshold being met. If negative, defer to pod specified value. |
| deprecated command line | EvictionMinimumReclaim                              | --eviction-minimum-reclaim                     | 0                                                            | A set of minimum reclaims (e.g. imagefs.available=2Gi) that describes the minimum amount of resource the kubelet will reclaim when performing a pod eviction if that resource is under pressure. |
| deprecated command line | PodsPerCore                                         | --pods-per-core                                | 0                                                            | Number of Pods per core that can run on this Kubelet. The total number of Pods on this Kubelet cannot exceed max-pods, so max-pods will be used if this calculation results in a larger number of Pods allowed on the Kubelet. A value of 0 disables this limit. |
| deprecated command line | ProtectKernelDefaults                               | --protect-kernel-defaults                      | false                                                        | Default kubelet behaviour for kernel tuning. If set, kubelet errors if any of kernel tunables is different than kubelet defaults. |
| deprecated command line | ReservedSystemCPUs                                  | --reserved-cpus                                | ""                                                           | A comma-separated list of CPUs or CPU ranges that are reserved for system and kubernetes usage. This specific list will supersede cpu counts in --system-reserved and --kube-reserved. |
| deprecated command line | SystemReserved                                      | --system-reserved                              | ""                                                           | A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=500Mi,ephemeral-storage=1Gi) pairs that describe resources reserved for non-kubernetes components. Currently only cpu and memory are supported. See http://kubernetes.io/docs/user-guide/compute-resources for more detail. [default=none] |
| deprecated command line | KubeReserved                                        | --kube-reserved                                | ""                                                           | A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=500Mi,ephemeral-storage=1Gi) pairs that describe resources reserved for kubernetes system components. Currently cpu, memory and local ephemeral storage for root file system are supported. See http://kubernetes.io/docs/user-guide/compute-resources for more detail. [default=none] |
| deprecated command line | EnforceNodeAllocatable                              | --enforce-node-allocatable                     | pods                                                         | A comma separated list of levels of node allocatable enforcement to be enforced by kubelet. Acceptable options are 'none', 'pods', 'system-reserved', and 'kube-reserved'. If the latter two options are specified, '--system-reserved-cgroup' and '--kube-reserved-cgroup' must also be set, respectively. If 'none' is specified, no additional options should be set. See https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/ for more details. |
| deprecated command line | SystemReservedCgroup                                | --system-reserved-cgroup                       | ""                                                           | Absolute name of the top level cgroup that is used to manage non-kubernetes components for which compute resources were reserved via '--system-reserved' flag. Ex. '/system-reserved'. [default=''] |
| deprecated command line | KubeReservedCgroup                                  | --kube-reserved-cgroup                         | ""                                                           | Absolute name of the top level cgroup that is used to manage kubernetes components for which compute resources were reserved via '--kube-reserved' flag. Ex. '/kube-reserved'. [default=''] |
| configuration           | NodeStatusReportFrequency                           |                                                | 5 * time.Minute                                              | The frequency that kubelet posts node status to master if node status does not change. Kubelet will ignore this frequency and post node status immediately if any change is detected. It is only used when node lease feature is enabled. nodeStatusReportFrequency's default value is 1m. But if nodeStatusUpdateFrequency is set explicitly, nodeStatusReportFrequency's default value will be set to nodeStatusUpdateFrequency for backward compatibility. |
| configuration           | NodeLeaseDurationSeconds                            |                                                | 40 * time.Second                                             | The duration the Kubelet will set on its corresponding Lease, when the NodeLease feature is enabled. This feature provides an indicator of node health by having the Kubelet create and periodically renew a lease, named after the node, in the kube-node-lease namespace. If the lease expires, the node can be considered unhealthy. The lease is currently renewed every 10s, per KEP-0009. In the future, the lease renewal interval may be set based on the lease duration. Requires the NodeLease feature gate to be enabled. |
| configuration           | ConfigMapAndSecretChangeDetectionStrategy           |                                                | Watch                                                        | A mode in which config map and secret managers are running.  |
| configuration           | ShowHiddenMetricsForVersion                         |                                                | ""                                                           | The previous version for which you want to show hidden metrics. Only the previous minor version is meaningful, other values will not be allowed. The format is <major>.<minor>, e.g.: '1.16'. The purpose of this format is make sure you have the opportunity to notice if the next release hides additional metrics, rather than being surprised when they are permanently removed in the release after that. |



### GlobalFlags

globalFlags包含了klog选项、cadvisor选项、CredentialProvider选项、logflush选项。

klog选项定义通过klog.InitFlags

```go
// InitFlags is for explicitly initializing the flags.
func InitFlags(flagset *flag.FlagSet) {
	if flagset == nil {
		flagset = flag.CommandLine
	}

	flagset.StringVar(&logging.logDir, "log_dir", logging.logDir, "If non-empty, write log files in this directory")
	flagset.StringVar(&logging.logFile, "log_file", logging.logFile, "If non-empty, use this log file")
	flagset.Uint64Var(&logging.logFileMaxSizeMB, "log_file_max_size", logging.logFileMaxSizeMB,
		"Defines the maximum size a log file can grow to. Unit is megabytes. "+
			"If the value is 0, the maximum file size is unlimited.")
	flagset.BoolVar(&logging.toStderr, "logtostderr", logging.toStderr, "log to standard error instead of files")
	flagset.BoolVar(&logging.alsoToStderr, "alsologtostderr", logging.alsoToStderr, "log to standard error as well as files")
	flagset.Var(&logging.verbosity, "v", "number for the log level verbosity")
	flagset.BoolVar(&logging.skipHeaders, "add_dir_header", logging.addDirHeader, "If true, adds the file directory to the header") //这里应该是&logging.addDirHeader,在下一个klog版本v2.0.0修复了,所有的1.18版本都会有这个问题
	flagset.BoolVar(&logging.skipHeaders, "skip_headers", logging.skipHeaders, "If true, avoid header prefixes in the log messages")
	flagset.BoolVar(&logging.skipLogHeaders, "skip_log_headers", logging.skipLogHeaders, "If true, avoid headers when opening log files")
	flagset.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")
	flagset.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
	flagset.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")
}
```

**cadvisor定义过程**

cadvisor flag通过匿名import来导入的，代码保存在vendor/github.com/google/cadvisor下

```go
// ensure libs have a chance to globally register their flags
	_ "github.com/google/cadvisor/container/common"
	_ "github.com/google/cadvisor/container/containerd"
	_ "github.com/google/cadvisor/container/docker"
	_ "github.com/google/cadvisor/container/raw"
	_ "github.com/google/cadvisor/machine"
	_ "github.com/google/cadvisor/manager"
	_ "github.com/google/cadvisor/storage"
```

以`github.com/google/cadvisor/container/docker`为例，它定义了下面这些选项

```go
var ArgDockerEndpoint = flag.String("docker", "unix:///var/run/docker.sock", "docker endpoint")
var ArgDockerTLS = flag.Bool("docker-tls", false, "use TLS to connect to docker")
var ArgDockerCert = flag.String("docker-tls-cert", "cert.pem", "path to client certificate")
var ArgDockerKey = flag.String("docker-tls-key", "key.pem", "path to private key")
var ArgDockerCA = flag.String("docker-tls-ca", "ca.pem", "path to trusted CA")

// The namespace under which Docker aliases are unique.
const DockerNamespace = "docker"

// Regexp that identifies docker cgroups, containers started with
// --cgroup-parent have another prefix than 'docker'
var dockerCgroupRegexp = regexp.MustCompile(`([a-z0-9]{64})`)

var dockerEnvWhitelist = flag.String("docker_env_metadata_whitelist", "", "a comma-separated list of environment variable keys that needs to be collected for docker containers")

var (
	// Basepath to all container specific information that libcontainer stores.
	dockerRootDir string

	dockerRootDirFlag = flag.String("docker_root", "/var/lib/docker", "DEPRECATED: docker root is read from docker info (this is a fallback, default: /var/lib/docker)")

	dockerRootDirOnce sync.Once

	// flag that controls globally disabling thin_ls pending future enhancements.
	// in production, it has been found that thin_ls makes excessive use of iops.
	// in an iops restricted environment, usage of thin_ls must be controlled via blkio.
	// pending that enhancement, disable its usage.
	disableThinLs = true
)
```

cadvisor选项中除了`--docker-root`和`housekeeping-interval`不是deprecated，其他选项都是deprecated。其中`update_machine_info_interval`这个flag未注册到kubelet中，根据commit记录，`update_machine_info_interval`是在升级cadvisor版本到0.33.0时候新添加进来的，可能因为这些都是deprecated，所以没有注册到kubelet中。

```go
// These flags were also implicit from cadvisor, but are actually used by something in the core repo:
	// TODO(mtaufen): This one is stil used by our salt, but for heaven's sake it's even deprecated in cadvisor
	register(global, local, "docker_root")
	// e2e node tests rely on this
	register(global, local, "housekeeping_interval")

	// These flags were implicit from cadvisor, and are mistakes that should be registered deprecated:
	const deprecated = "This is a cadvisor flag that was mistakenly registered with the Kubelet. Due to legacy concerns, it will follow the standard CLI deprecation timeline before being removed."

	registerDeprecated(global, local, "application_metrics_count_limit", deprecated)
	registerDeprecated(global, local, "boot_id_file", deprecated)
	registerDeprecated(global, local, "container_hints", deprecated)
	registerDeprecated(global, local, "containerd", deprecated)
	registerDeprecated(global, local, "docker", deprecated)
	registerDeprecated(global, local, "docker_env_metadata_whitelist", deprecated)
	registerDeprecated(global, local, "docker_only", deprecated)
	registerDeprecated(global, local, "docker-tls", deprecated)
	registerDeprecated(global, local, "docker-tls-ca", deprecated)
	registerDeprecated(global, local, "docker-tls-cert", deprecated)
	registerDeprecated(global, local, "docker-tls-key", deprecated)
	registerDeprecated(global, local, "enable_load_reader", deprecated)
	registerDeprecated(global, local, "event_storage_age_limit", deprecated)
	registerDeprecated(global, local, "event_storage_event_limit", deprecated)
	registerDeprecated(global, local, "global_housekeeping_interval", deprecated)
	registerDeprecated(global, local, "log_cadvisor_usage", deprecated)
	registerDeprecated(global, local, "machine_id_file", deprecated)
	registerDeprecated(global, local, "storage_driver_user", deprecated)
	registerDeprecated(global, local, "storage_driver_password", deprecated)
	registerDeprecated(global, local, "storage_driver_host", deprecated)
	registerDeprecated(global, local, "storage_driver_db", deprecated)
	registerDeprecated(global, local, "storage_driver_table", deprecated)
	registerDeprecated(global, local, "storage_driver_secure", deprecated)
	registerDeprecated(global, local, "storage_driver_buffer_duration", deprecated)

```

**CredentialProviderFlags**

在cmd\kubelet\app\options\globalflags.go里匿名import

```go
	// ensure libs have a chance to globally register their flags
	_ "k8s.io/kubernetes/pkg/credentialprovider/azure"
```

在pkg\credentialprovider\azure\azure_credentials.go定义

```go
var flagConfigFile = pflag.String("azure-container-registry-config", "",
	"Path to the file containing Azure container registry configuration information.")
```

**version**

`--version`选项定义在staging\src\k8s.io\component-base\version\verflag\verflag.go，它接受三个值false (默认值)、true(不加参数默认为true)、raw(显示json格式的详细版本信息)。

后面文章专门介绍kubernetes的版本相关的发布规则、version的相关hack技术

```go
func VersionVar(p *versionValue, name string, value versionValue, usage string) {
	*p = value
	flag.Var(p, name, usage)
	// "--version" will be treated as "--version=true"
	flag.Lookup(name).NoOptDefVal = "true"
}

func Version(name string, value versionValue, usage string) *versionValue {
	p := new(versionValue)
	VersionVar(p, name, value, usage)
	return p
}

const versionFlagName = "version"

var (
	versionFlag = Version(versionFlagName, VersionFalse, "Print version information and quit")
)
```

**log选项**

定义在staging\src\k8s.io\component-base\logs\logs.go

```go
const logFlushFreqFlagName = "log-flush-frequency"

var logFlushFreq = pflag.Duration(logFlushFreqFlagName, 5*time.Second, "Maximum number of seconds between log flushes")


```



| categories分类     | var变量                      | flag                              | default默认值                            | description                                                  |
| ------------------ | ---------------------------- | --------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| klog               | logDir                       | --log-dir                         | ""                                       | If non-empty, write log files in this directory              |
| klog               | logFile                      | --log-file                        | ""                                       | If non-empty, use this log file                              |
| klog               | logFileMaxSizeMB             | --log-file-max-size               | 1800                                     | Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0, the maximum file size is unlimited. |
| klog               | toStderr                     | --logtostderr                     | true                                     | log to standard error instead of files                       |
| klog               | alsoToStderr                 | --alsologtostderr                 | false                                    | log to standard error as well as files                       |
| klog               | verbosity                    | -v, --v                           | 0                                        | number for the log level verbosity                           |
| klog               | addDirHeader                 | --add-dir-header                  | false                                    | If true, adds the file directory to the header               |
| klog               | skipHeaders                  | --skip-headers                    | false                                    | If true, avoid header prefixes in the log messages           |
| klog               | skipLogHeaders               | --skip-log-headers                | false                                    | If true, avoid headers when opening log files                |
| klog               | stderrThreshold              | --stderrthreshold                 | 2                                        | logs at or above this threshold go to stderr                 |
| klog               | vmodule                      | --vmodule                         | ""                                       | comma-separated list of pattern=N settings for file-filtered logging |
| klog               | traceLocation                | --log-backtrace-at                | ""                                       | when logging hits line file:N, emit a stack trace            |
| cadvisor           | dockerRootDirFlag            | --docker-root                     | /var/lib/docker                          | DEPRECATED: docker root is read from docker info (this is a fallback, default: /var/lib/docker) |
| cadvisor           | ArgDockerEndpoint            | --docker                          | unix:///var/run/docker.sock              | docker endpoint                                              |
| cadvisor           | ArgDockerTLS                 | --docker-tls                      | false                                    | use TLS to connect to docker                                 |
| cadvisor           | ArgDockerCert                | --docker-tls-cert                 | cert.pem                                 | path to client certificate                                   |
| cadvisor           | ArgDockerKey                 | --docker-tls-key                  | key.pem                                  | path to private key                                          |
| cadvisor           | ArgDockerCA                  | --docker-tls-ca                   | ca.pem                                   | path to trusted CA                                           |
| cadvisor           | dockerEnvWhitelist           | --docker-env-metadata-whitelist   | ""                                       | a comma-separated list of environment variable keys that needs to be collected for docker containers |
| cadvisor           | ArgContainerHints            | --container-hints                 | /etc/cadvisor/container_hints.json       | location of the container hints file                         |
| cadvisor           | ArgContainerdEndpoint        | --containerd                      | /run/containerd/containerd.sock          | containerd endpoint                                          |
| cadvisor           | ArgContainerdNamespace       | --containerd-namespace            | k8s.io                                   | containerd namespace                                         |
| cadvisor           | dockerOnly                   | --docker-only                     | false                                    | Only report docker containers in addition to root stats      |
| cadvisor           | disableRootCgroupStats       | --disable-root-cgroup-stats       | false                                    | Disable collecting root Cgroup stats                         |
| cadvisor           | machineIdFilePath            | --machine-id-file                 | /etc/machine-id,/var/lib/dbus/machine-id | Comma-separated list of files to check for machine-id. Use the first one that exists. |
| cadvisor           | bootIdFilePath               | --boot-id-file                    | /proc/sys/kernel/random/boot_id          | Comma-separated list of files to check for boot-id. Use the first one that exists. |
| cadvisor           | enableLoadReader             | --enable-load-reader              | false                                    | Whether to enable cpu load reader                            |
| cadvisor           | HousekeepingInterval         | --housekeeping-interval           | 1 * time.Second                          | Interval between container housekeepings                     |
| cadvisor           | globalHousekeepingInterval   | --global-housekeeping-interval    | 1 * time.Minute                          | Interval between global housekeepings                        |
| cadvisor           | logCadvisorUsage             | --log-cadvisor-usage              | false                                    | Whether to log the usage of the cAdvisor container           |
| cadvisor           | eventStorageAgeLimit         | --event-storage-age-limit         | default=24h                              | Max length of time for which to store events (per type). Value is a comma separated list of key values, where the keys are event types (e.g.: creation, oom) or \"default\" and the value is a duration. Default is applied to all non-specified event types |
| cadvisor           | eventStorageEventLimit       | --event-storage-event-limit       | default=100000                           | Max number of events to store (per type). Value is a comma separated list of key values, where the keys are event types (e.g.: creation, oom) or \"default\" and the value is an integer. Default is applied to all non-specified event types |
| cadvisor           | applicationMetricsCountLimit | --application-metrics-count-limit | 100                                      | Max number of application metrics to store (per container)   |
| cadvisor           | ArgDbUsername                | --storage-driver-user             | root                                     | database username                                            |
| cadvisor           | ArgDbPassword                | --storage-driver-password         | root                                     | database password                                            |
| cadvisor           | ArgDbHost                    | --storage-driver-host             | localhost:8086                           | database host:port                                           |
| cadvisor           | ArgDbName                    | --storage-driver-db               | cadvisor                                 | database name                                                |
| cadvisor           | ArgDbTable                   | --storage-driver-table            | stats                                    | table name                                                   |
| cadvisor           | ArgDbIsSecure                | --storage-driver-secure           | false                                    | use secure connection with database                          |
| cadvisor           | ArgDbBufferDuration          | --storage-driver-buffer-duration  | 60 * time.Second                         | Writes in the storage driver will be buffered for this duration, and committed to the non memory backends as a single transaction |
| CredentialProvider | flagConfigFile               | --azure-container-registry-config | ""                                       | Path to the file containing Azure container registry configuration information. |
| version            | versionFlag                  | --version                         | false                                    | Print version information and quit                           |
| log                | logFlushFreq                 | --log-flush-frequency             | 5 * time.Second                          | Maximum number of seconds between log flushes                |

### help选项

特殊的选项，就是显示命令的帮助信息，使用-h或--help。

```go
cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))
```

