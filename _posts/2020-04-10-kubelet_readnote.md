---
layout: post
title: kubelet源码阅读笔记
subtitle: ""
catalog: true
tags:
     - k8s
---

kubelet是k8s核心组件中最复杂的一个组件，它跟网络、存储、docker都得打交道. 
它是每个节点的主要节点agent，向apiserver注册节点.

### 环境

- Kubernetes v1.14.6
- Etcd 3.3.12
- Docker 18.09.9

### kubelet代码

启动函数
```
k8s.io/kubernetes/cmd/kubelet/kubelet.go -> NewKubeletCommand(初始化命令行对象) -> Run
```

NewKubeletCommand中包括默认参数、默认flag
```
kubeletFlags := options.NewKubeletFlags() // 默认值定义在k8s.io/kubernetes/cmd/kubelet/app/options/options.go
kubeletConfig, err := options.NewKubeletConfiguration() // 默认值定义在k8s.io/kubernetes/pkg/kubelet/apis/config/v1beta1/defaults.go
```

Run函数：

1. 设置featrueGate；
2. 校验命令行参数；
3. 加载kubelet配置文件；
4. 校验kubelet配置；
5. 是否启用动态kubelet配置
6. 构建默认kubelet运行依赖
   - 有dockerClientConfig，跟docker通信；
   - mounter/subpather用于volume挂载；
   - 初始化oomAdjuster对象；
   - 初始化RealOS对象，用于系统级别操作；
   - 检测要被使用的一系列volume插件 // ProbeVolumePlugins()
   - flexvolume方式动态检测volume插件
7. 是否运行dockershim（为kubelet调用提供grpc接口，转化grpc请求为docker的方式）
8. 运行kubelet

   - 操作系统初始化(针对windows平台)
   - 尝试获取kubelet本地文件锁(需要启动时加--exit-on-lock-contention、--lock-file参数)
   - 注册当前kubelet配置到/configz，可通过如下命令获取
     ```
     curl -k  -X GET -H 'Authorization: Bearer 1aYiwwOnSN4Yk5oNm7fov0ohBbk' \
            https://127.0.0.1:10250/configz
     ```
   - 初始化cloudProvider
   - 初始化eventClientConfig、heartbeatClientConfig
   - 初始化Auth，初始化一个authn、authz对象
   - 初始化cadvisor
   - 初始化ContainerManager
   - 检查运行的uid是否为0
   - 设置0号进程的oom_score_adj，默认为-999；范围是[-1000,1000]; 值越大越容易被OOM kill
   - 进入RunKubelet函数(下面重点)
   - 设置/healthz路由，可通过下面命令验证

     ```
     curl -k  -X GET -H 'Authorization: Bearer 1aYiwwOnSN4Yk5oNm7fov0ohBbk' \
         https://127.0.0.1:10250/healthz   
     ```
   - 如果是sytemd方式启动，通知systemd

RunKubelet函数

    1. 获取nodeName，可以通过kubelet配置文件进行override
    2. 获取特权模式的资源HostNetworkSources、HostPIDSources、HostIPCSources，默认都是*
    3. 初始化一个Capabilities对象(单例)，AllowPrivileged为true
    4. 设置RootDirectory(默认为/var/lib/kubelet)为.docker/config.json的可能存放路径目录
    5. 初始化kubelet对象, createAndInitKubelet（下面重点）
    6. 设置能打开的最大文件数，MaxOpenFiles默认为1000000
    7. startKubelet

createAndInitKubelet函数
```
func createAndInitKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
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

    // 进入NewMainKubelet，返回一个带有所有内部必需内部模块的kubelet对象
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
    // 发送一个事件"Starting kubelet"
    k.BirthCry()
    // container gc，每隔一分钟执行一次
    // image gc, 每隔5分钟执行一次
    // image gc有上限阀值/下限阀值配置，上限阀值默认85，大等于上限阀值就触发；下限阀值默认80，用于计算要释放的空间
    // 要释放的空间 = 总容量*（100-下限阀值）/100 - 剩余可用
    k.StartGarbageCollection()

    return k, nil
}
```

NewMainKubelet

1. 判断rootDirectory、SyncFrequency.Duration、MakeIPTablesUtilChains、IPTablesMasqueradeBit、IPTablesDropBit值的合法性
2. 获取nodeName
3. 初始化PodConfig
    ```
    // 如何加载static pod的呢？
    if kubeDeps.PodConfig == nil {
            var err error
            kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, bootstrapCheckpointPath)
            if err != nil {
                return nil, err
            }
        }
    
    // makePodSourceConfig creates a config.PodConfig from the given
    // KubeletConfiguration or returns an error.
    func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, bootstrapCheckpointPath string) (*config.PodConfig, error) {
        // 是否有设置StaticPodURLHeader
        manifestURLHeader := make(http.Header)
        if len(kubeCfg.StaticPodURLHeader) > 0 {
            for k, v := range kubeCfg.StaticPodURLHeader {
                for i := range v {
                    manifestURLHeader.Add(k, v[i])
                }
            }
        }
        // 初始化一个PodConfig对象
        // source of all configuration
        cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder)
    
        // define file config source
        if kubeCfg.StaticPodPath != "" {
            klog.Infof("Adding pod path: %v", kubeCfg.StaticPodPath)
            // 默认间隔20s, 读取static_path下的所有内容
            config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(kubetypes.FileSource))
        }
    
        // define url config source
        if kubeCfg.StaticPodURL != "" {
            klog.Infof("Adding pod url %q with HTTP header %v", kubeCfg.StaticPodURL, manifestURLHeader)
            config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(kubetypes.HTTPSource))
        }
    
        // Restore from the checkpoint path
        // NOTE: This MUST happen before creating the apiserver source
        // below, or the checkpoint would override the source of truth.
    
        var updatechannel chan<- interface{}
        if bootstrapCheckpointPath != "" {
            klog.Infof("Adding checkpoint path: %v", bootstrapCheckpointPath)
            updatechannel = cfg.Channel(kubetypes.ApiserverSource)
            err := cfg.Restore(bootstrapCheckpointPath, updatechannel)
            if err != nil {
                return nil, err
            }
        }
    
        if kubeDeps.KubeClient != nil {
            klog.Infof("Watching apiserver")
            if updatechannel == nil {
                updatechannel = cfg.Channel(kubetypes.ApiserverSource)
            }
            // watch apiserver获取该节点下所有的pod信息，传入cfg.Channel(kubetypes.ApiserverSource)
            config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, updatechannel)
        }
        return cfg, nil
    }
    // 后面交给podManager处理
    ```
4.  初始化containerGCPolicy, 容器垃圾回收的策略
5.  初始化daemonEndpoints,  kubelet运行监听的端口
6.  初始化imageGCPolicy，镜像垃圾回收的策略
7.  初始化evictionConfig，包含驱散操作的相关参数
8.  初始化一个Kubelet对象
9.  初始化一个cloudResourceSyncManager对象，用于处理跟cloud provider交互的请求
10. 初始化secretManager、configMapManager对象，默认ConfigMapAndSecretChangeDetectionStrategy是Watch
11. 初始化machineInfo，内置cadvisor收集的节点信息
12. 初始化一个流控对象imageBackOff，用于镜像失败重试(默认间隔10s，总时长为5分钟)
13. 初始化livenessManager、checkpointManager、podManager、statusManager对象
14. 初始化NetworkPluginSettings对象，看着是把cni相关信息传给container runtime shim
15. 初始化resourceAnalyzer对象，收集pod磁盘使用量
16. 启动cri shim，作为grpc server, 判断cri是否支持logging driver
17. 初始化kubeGenericRuntimeManager对象
18. 初始化StatsProvider对象，根据runtime为docker且os为linux的话来判断选择不同的statsProvider
19. 初始化PodLifecycleEventGenerator对象
20. 初始化containerGC、imageManager、containerLogManager、serverCertificateManager对象
21. 初始化probeManager、tokenManager、volumePluginMgr、volumeManager、evictionManager、evictionAdmitHandler对象
22. 初始化admitHandlers、softAdmitHandlers

看到很多地方都用了Backoff对象用于失败重试
```
        backoff := wait.Backoff{
            Duration: 2 * time.Second,
            Factor:   2,
            Jitter:   0.1,
            Steps:    5,
        }
        if err := wait.ExponentialBackoff(backoff, m.rotateCerts); err != nil {
            utilruntime.HandleError(fmt.Errorf("Reached backoff limit, still unable to rotate certs: %v", err))
            wait.PollInfinite(32*time.Second, m.rotateCerts)
        }
```
每次backOff近似等于Duration*Factor，实际上比这个大，steps控制次数

执行完createAndInitKubelet后，接下来到startKubelet
1. 启动kubelet中的各种xxxManager
    - 启动cloudResourceSyncManager, 根据cloudProvider的后端，获得节点的地址
    - 启动imageGCManager, 
    - 启动serverCertificateManager
    - 启动oomWatcher, 记录cadvisor监测到的oom
    - 启动resourceAnalyzer, 实际上是调用fsResourceAnalyzer
    - 启动volumeManager
    - syncNodeStatus, 往apiserver注册node(默认间隔10s)
    - updateRuntimeUp(间隔5s), 设置node的ready状态，启动containerManager、evictionManager、containerLogManager、pluginWatcher(CSIPlugin/DevicePlugin Registration)
    - syncNetworkUtil(间隔1m), 同步nat表的KUBE-MARK-DROP和KUBE-MARK-MASQ链的规则
    - podKiller(间隔1s), 监听kill pod的channel
    - 启动statusManager、probeManager、runtimeClassManager
    - 启动pleg(pod lifecycle event generator)
    - 进入syncLoop(监听来自三种方式pod的变化file、apiserver、http)
2. 启动kubelet server

### 参考链接

- [https://blog.tianfeiyu.com/source-code-reading-notes/kubernetes/kubelet_init.html](https://blog.tianfeiyu.com/source-code-reading-notes/kubernetes/kubelet_init.html)