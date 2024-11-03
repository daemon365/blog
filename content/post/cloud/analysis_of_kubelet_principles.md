---
title: "kubelet 原理分析"
date: "2024-05-01T12:40:00+08:00"
tags: 
- kubelet
- kubernetes
- 源码分析
- golang
showToc: true
---

## Reference
- https://atbug.com/kubelet-source-code-analysis/

## kubelet 简介

kubernetes 分为控制面和数据面，kubelet 就是数据面最主要的组件，在每个节点上启动，主要负责容器的创建、启停、监控、日志收集等工作。它是一个在每个集群节点上运行的代理，负责确保节点上的容器根据PodSpec（Pod定义文件）正确运行。

Kubelet执行以下几项重要功能：
- Pod生命周期管理：Kubelet根据从API服务器接收到的PodSpecs创建、启动、终止容器。它负责启动Pod中的容器，并确保它们按预期运行。
- 节点状态监控：Kubelet定期监控节点和容器的状态，并将状态报告回集群的控制平面。这使得集群中的其他组件能够做出相应的调度决策。
- 资源管理：Kubelet负责管理分配给每个Pod的资源。这包括CPU、内存和磁盘存储资源。
- 健康检查：Kubelet可以执行容器健康检查，并根据检查结果决定是否需要重启容器。
- 与容器运行时的通信：Kubelet与容器运行时（如Docker、containerd等）通信，以管理容器的生命周期。
- 秘密和配置管理：Kubelet负责将秘密、配置映射等挂载到Pod的容器中，以便应用程序可以访问这些配置。
- 服务发现和负载均衡：尽管Kubelet本身不直接处理服务发现，但它通过设置网络规则和环境变量来支持容器内的服务发现机制。

## kubelet 架构

![](/images/52795638-fd1a-4286-bfc7-170869122330.png)

kubelet 的架构由 N 多的组件组成，下面简单介绍下比较重要的几个：

- Sync Loop: 这是Kubelet活动的核心，负责同步Pod的状态。同步循环会定期从API服务器获取PodSpecs，并确保容器的当前状态与这些规格相匹配。
- PodConfig: 负责将各个配置源转换成 PodSpecs，可以选择的配置源包括：Kube-apiserver、本地文件、HTTP。
- PLEG(Pod Lifecycle Event Generator)： 负责监测和缓存Pod生命周期事件，如创建、启动或停止容器，然后将这些事件通知 Sync Loop。
- PodWorkers: 负责管理 Pod 的生命周期事件处理。当 Pod 生命周期事件 PLEG 检测到新的事件时，PodWorkers 会被调用来处理这些事件，包括启动新的 Pod、更新现有的 Pod、或者停止和清理不再需要的 Pod。
- PodManager: 存储 Pod 的期望状态，kubelet 服务的不同渠道的 Pod。
- ContainerRuntime: 顾名思义，容器运行时。与遵循 CRI 规范的高级容器运行时进行交互。
- StatsProvider: 提供节点和容器的统计信息，有 cAdvisor 和 CRI 两种实现。
- ProbeManager: 负责执行容器的健康检查，包括 Liveness，Startup 和 Readiness 检查。
- VolumeManager: 负责管理 Pod 的卷，包括挂载和卸载卷。
- ImageManager: 负责管理镜像，包括拉取、删除、镜像 GC 等。
- DeviceManager: 负责管理设备，包括 GPU、RDMA 等。
- PluginManager: PluginManager 运行一组异步循环，根据此节点确定哪些插件需要注册/取消注册并执行。如 CSI 驱动和设备管理器插件（Device Plugin）。
- CertificateManager: 处理证书轮换。
- OOMWatcher: 从系统日志中获取容器的 OOM 日志，将其封装成事件并记录。

## 流程

首先在 `cmd/kubelet` 中使用传入命令行参数的方式初始化配置，然后创建 `pkg/kubelet` 中的 Bootstrap inferface, kubelet struct 实现了这个接口， 然后调用 `Run` 方法启动 kubelet。

```GO
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go k.Run(podCfg.Updates())

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(kubeCfg, kubeDeps.TLSOptions, kubeDeps.Auth, kubeDeps.TracerProvider)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(netutils.ParseIPSloppy(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
	go k.ListenAndServePodResources()
}
```

### Bootstrap

```go
// Bootstrap is a bootstrapping interface for kubelet, targets the initialization protocol
type Bootstrap interface {
	GetConfiguration() kubeletconfiginternal.KubeletConfiguration
	BirthCry()
	StartGarbageCollection()
	ListenAndServe(kubeCfg *kubeletconfiginternal.KubeletConfiguration, tlsOptions *server.TLSOptions, auth server.AuthInterface, tp trace.TracerProvider)
	ListenAndServeReadOnly(address net.IP, port uint)
	ListenAndServePodResources()
	Run(<-chan kubetypes.PodUpdate)
	RunOnce(<-chan kubetypes.PodUpdate) ([]RunPodResult, error)
}

type Kubelet struct {
    // ...
}
```

方法：
- GetConfiguration: 获取 kubelet 的配置
- BirthCry: 打印 kubelet 启动信息
- StartGarbageCollection: 启动垃圾回收
- ListenAndServe: 启动 kubelet 服务
- ListenAndServeReadOnly: 启动只读服务
- ListenAndServePodResources: 启动 pod 资源服务
- Run: 启动 kubelet 的同步循环
- RunOnce: 启动一次同步循环

```GO
func (kl *Kubelet) StartGarbageCollection() {
	loggedContainerGCFailure := false
	go wait.Until(func() {
		ctx := context.Background()
		if err := kl.containerGC.GarbageCollect(ctx); err != nil {
			klog.ErrorS(err, "Container garbage collection failed")
			kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ContainerGCFailed, err.Error())
			loggedContainerGCFailure = true
		} else {
			var vLevel klog.Level = 4
			if loggedContainerGCFailure {
				vLevel = 1
				loggedContainerGCFailure = false
			}

			klog.V(vLevel).InfoS("Container garbage collection succeeded")
		}
	}, ContainerGCPeriod, wait.NeverStop)

	// when the high threshold is set to 100, stub the image GC manager
	if kl.kubeletConfiguration.ImageGCHighThresholdPercent == 100 {
		klog.V(2).InfoS("ImageGCHighThresholdPercent is set 100, Disable image GC")
		return
	}

	prevImageGCFailed := false
	go wait.Until(func() {
		ctx := context.Background()
		if err := kl.imageManager.GarbageCollect(ctx); err != nil {
			if prevImageGCFailed {
				klog.ErrorS(err, "Image garbage collection failed multiple times in a row")
				// Only create an event for repeated failures
				kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ImageGCFailed, err.Error())
			} else {
				klog.ErrorS(err, "Image garbage collection failed once. Stats initialization may not have completed yet")
			}
			prevImageGCFailed = true
		} else {
			var vLevel klog.Level = 4
			if prevImageGCFailed {
				vLevel = 1
				prevImageGCFailed = false
			}

			klog.V(vLevel).InfoS("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)
}

```

大致的流程为使用 `containerGC` 启动容器垃圾回收，当ImageGCHighThresholdPercent 为100时，不启动镜像垃圾回收，否则使用 `imageManager` 启动镜像垃圾回收。

```GO
// RunOnce polls from one configuration update and run the associated pods.
func (kl *Kubelet) RunOnce(updates <-chan kubetypes.PodUpdate) ([]RunPodResult, error) {
	ctx := context.Background()
	// Setup filesystem directories.
	if err := kl.setupDataDirs(); err != nil {
		return nil, err
	}

	// If the container logs directory does not exist, create it.
	if _, err := os.Stat(ContainerLogsDir); err != nil {
		if err := kl.os.MkdirAll(ContainerLogsDir, 0755); err != nil {
			klog.ErrorS(err, "Failed to create directory", "path", ContainerLogsDir)
		}
	}

	select {
	case u := <-updates:
		klog.InfoS("Processing manifest with pods", "numPods", len(u.Pods))
		result, err := kl.runOnce(ctx, u.Pods, runOnceRetryDelay)
		klog.InfoS("Finished processing pods", "numPods", len(u.Pods))
		return result, err
	case <-time.After(runOnceManifestDelay):
		return nil, fmt.Errorf("no pod manifest update after %v", runOnceManifestDelay)
	}
}

// runOnce runs a given set of pods and returns their status.
func (kl *Kubelet) runOnce(ctx context.Context, pods []*v1.Pod, retryDelay time.Duration) (results []RunPodResult, err error) {
	ch := make(chan RunPodResult)
	admitted := []*v1.Pod{}
	for _, pod := range pods {
		// Check if we can admit the pod.
		if ok, reason, message := kl.canAdmitPod(admitted, pod); !ok {
			kl.rejectPod(pod, reason, message)
			results = append(results, RunPodResult{pod, nil})
			continue
		}

		admitted = append(admitted, pod)
		go func(pod *v1.Pod) {
			err := kl.runPod(ctx, pod, retryDelay)
			ch <- RunPodResult{pod, err}
		}(pod)
	}

	klog.InfoS("Waiting for pods", "numPods", len(admitted))
	failedPods := []string{}
	for i := 0; i < len(admitted); i++ {
		res := <-ch
		results = append(results, res)
		if res.Err != nil {
			failedContainerName, err := kl.getFailedContainers(ctx, res.Pod)
			if err != nil {
				klog.InfoS("Unable to get failed containers' names for pod", "pod", klog.KObj(res.Pod), "err", err)
			} else {
				klog.InfoS("Unable to start pod because container failed", "pod", klog.KObj(res.Pod), "containerName", failedContainerName)
			}
			failedPods = append(failedPods, format.Pod(res.Pod))
		} else {
			klog.InfoS("Started pod", "pod", klog.KObj(res.Pod))
		}
	}
	if len(failedPods) > 0 {
		return results, fmt.Errorf("error running pods: %v", failedPods)
	}
	klog.InfoS("Pods started", "numPods", len(pods))
	return results, err
}
```

大致作用为从 `updates` 中获取 PodUpdate，然后调用 `runOnce` 方法，该方法会调用 `runPod` 方法启动 Pod。

### Run

```GO
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	ctx := context.Background()
	if kl.logServer == nil {
		file := http.FileServer(http.Dir(nodeLogDir))
		if utilfeature.DefaultFeatureGate.Enabled(features.NodeLogQuery) && kl.kubeletConfiguration.EnableSystemLogQuery {
			kl.logServer = http.StripPrefix("/logs/", http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
				if nlq, errs := newNodeLogQuery(req.URL.Query()); len(errs) > 0 {
					http.Error(w, errs.ToAggregate().Error(), http.StatusBadRequest)
					return
				} else if nlq != nil {
					if req.URL.Path != "/" && req.URL.Path != "" {
						http.Error(w, "path not allowed in query mode", http.StatusNotAcceptable)
						return
					}
					if errs := nlq.validate(); len(errs) > 0 {
						http.Error(w, errs.ToAggregate().Error(), http.StatusNotAcceptable)
						return
					}
					// Validation ensures that the request does not query services and files at the same time
					if len(nlq.Services) > 0 {
						journal.ServeHTTP(w, req)
						return
					}
					// Validation ensures that the request does not explicitly query multiple files at the same time
					if len(nlq.Files) == 1 {
						// Account for the \ being used on Windows clients
						req.URL.Path = filepath.ToSlash(nlq.Files[0])
					}
				}
				// Fall back in case the caller is directly trying to query a file
				// Example: kubectl get --raw /api/v1/nodes/$name/proxy/logs/foo.log
				file.ServeHTTP(w, req)
			}))
		} else {
			kl.logServer = http.StripPrefix("/logs/", file)
		}
	}
	if kl.kubeClient == nil {
		klog.InfoS("No API server defined - no node status update will be sent")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.ErrorS(err, "Failed to initialize internal modules")
		os.Exit(1)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start two go-routines to update the status.
		//
		// The first will report to the apiserver every nodeStatusUpdateFrequency and is aimed to provide regular status intervals,
		// while the second is used to provide a more timely status update during initialization and runs an one-shot update to the apiserver
		// once the node becomes ready, then exits afterwards.
		//
		// Introduce some small jittering to ensure that over time the requests won't start
		// accumulating at approximately the same time from the set of nodes due to priority and
		// fairness effect.
		go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(context.Background())
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start component sync loops.
	kl.statusManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.
	kl.pleg.Start()

	// Start eventedPLEG only if EventedPLEG feature gate is enabled.
	if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
		kl.eventedPleg.Start()
	}

	kl.syncLoop(ctx, updates, kl)
}
```

代码的流程为：
1. 检查是否需要创建日志服务器 如果需要则创建
2. 启动云提供商同步管理器
3. 初始化模块，如果出错则打印日志并退出
4. 启动卷管理器
5. 启动两个 goroutine 来更新状态，一个是定时更新，一个是在初始化时更新
6. 启动同步租约的goroutine
7. 定期更新RuntimeUp状态的goroutine
8. 设置iptables规则
9. 启动组件同步循环
10. 如果启用了RuntimeClasses，则启动RuntimeClasses同步循环
11. 启动Pod Lifecycle Event Generator
12. 如果启用了EventedPLEG特性，则启动EventedPLEG
13. 启动 syncloop


### syncLoop

```GO
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.InfoS("Starting kubelet main sync loop")
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	// Responsible for checking limits in resolv.conf
	// The limits do not have anything to do with individual pods
	// Since this is called in syncLoop, we don't need to call it anywhere else
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}

	for {
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.ErrorS(err, "Skipping pod synchronization")
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```
代码流程为：
1. 从 pleg 获取 update 事件 他是一个 channel
2. 进入循环 如果 runtimeState 有错误 sleep 一会儿然后跳过
3. 执行  syncLoopIteration

#### syncLoopIteration

```go
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.ErrorS(nil, "Kubelet does not support snapshot update")
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		kl.sourcesReady.AddSource(u.Source)

	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjSlice(podsToSync))
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
		} else {
			start := time.Now()
			klog.V(4).InfoS("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(ctx); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
			duration := time.Since(start)
			if duration > housekeepingWarningDuration {
				klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than expected", "expected", housekeepingWarningDuration, "actual", duration.Round(time.Millisecond))
			}
			klog.V(4).InfoS("SyncLoop (housekeeping) end", "duration", duration.Round(time.Millisecond))
		}
	}
	return true
}
```

首先解释一下这个函数的参数：
- configCh: 将配置更改的 Pod 分派给适当的处理程序回调函数
- plegCh: 更新运行时缓存；同步 Pod
- syncCh: 同步所有等待同步的 Pod
- housekeepingCh: 触发 Pod 的清理
- health manager: 同步失败的 Pod 或其中一个或多个容器的健康检查失败的 Pod

代码流程为：
- 如果 updates channel 有消息 则使用 handler 调用对应方法做处理
- 如果 plegCh 有消息 则使用 handler 的 HandlePodSyncs 做同步
- 如果 syncCh 有消息 代表到了同步时间 做同步操作
- 如果是 三种 probe 的更新 则使用 handleProbeSync 做同步
- 如果 housekeepingCh 有消息 则使用 handler 的 HandlePodCleanups 做清理


```go
func handleProbeSync(kl *Kubelet, update proberesults.Update, handler SyncHandler, probe, status string) {
	// We should not use the pod from manager, because it is never updated after initialization.
	pod, ok := kl.podManager.GetPodByUID(update.PodUID)
	if !ok {
		// If the pod no longer exists, ignore the update.
		klog.V(4).InfoS("SyncLoop (probe): ignore irrelevant update", "probe", probe, "status", status, "update", update)
		return
	}
	klog.V(1).InfoS("SyncLoop (probe)", "probe", probe, "status", status, "pod", klog.KObj(pod))
	handler.HandlePodSyncs([]*v1.Pod{pod})
}
```

handleProbeSync也是使用 handler 的 HandlePodSyncs 做同步


### handle (SyncHandler)

```GO
// SyncHandler is an interface implemented by Kubelet, for testability
type SyncHandler interface {
	HandlePodAdditions(pods []*v1.Pod)
	HandlePodUpdates(pods []*v1.Pod)
	HandlePodRemoves(pods []*v1.Pod)
	HandlePodReconcile(pods []*v1.Pod)
	HandlePodSyncs(pods []*v1.Pod)
	HandlePodCleanups(ctx context.Context) error
}
```


也是 kubelet struct 实现了这个接口

```GO
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		kl.podResizeMutex.Lock()
		defer kl.podResizeMutex.Unlock()
	}
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		kl.podManager.AddPod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			activePods := kl.filterOutInactivePods(existingPods)

			if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
				// To handle kubelet restarts, test pod admissibility using AllocatedResources values
				// (for cpu & memory) from checkpoint store. If found, that is the source of truth.
				podCopy := pod.DeepCopy()
				kl.updateContainerResourceAllocation(podCopy)

				// Check if we can admit the pod; if not, reject it.
				if ok, reason, message := kl.canAdmitPod(activePods, podCopy); !ok {
					kl.rejectPod(pod, reason, message)
					continue
				}
				// For new pod, checkpoint the resource values at which the Pod has been admitted
				if err := kl.statusManager.SetPodAllocation(podCopy); err != nil {
					//TODO(vinaykul,InPlacePodVerticalScaling): Can we recover from this in some way? Investigate
					klog.ErrorS(err, "SetPodAllocation failed", "pod", klog.KObj(pod))
				}
			} else {
				// Check if we can admit the pod; if not, reject it.
				if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
					kl.rejectPod(pod, reason, message)
					continue
				}
			}
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodCreate,
			StartTime:  start,
		})
	}
}


func (kl *Kubelet) HandlePodUpdates(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.UpdatePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
		}

		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodUpdate,
			StartTime:  start,
		})
	}
}


func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.RemovePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		// Deletion is allowed to fail because the periodic cleanup routine
		// will trigger deletion again.
		if err := kl.deletePod(pod); err != nil {
			klog.V(2).InfoS("Failed to delete pod", "pod", klog.KObj(pod), "err", err)
		}
	}
}

func (kl *Kubelet) HandlePodReconcile(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		// Update the pod in pod manager, status manager will do periodically reconcile according
		// to the pod manager.
		kl.podManager.UpdatePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			// Static pods should be reconciled the same way as regular pods
		}

		// TODO: reconcile being calculated in the config manager is questionable, and avoiding
		// extra syncs may no longer be necessary. Reevaluate whether Reconcile and Sync can be
		// merged (after resolving the next two TODOs).

		// Reconcile Pod "Ready" condition if necessary. Trigger sync pod for reconciliation.
		// TODO: this should be unnecessary today - determine what is the cause for this to
		// be different than Sync, or if there is a better place for it. For instance, we have
		// needsReconcile in kubelet/config, here, and in status_manager.
		if status.NeedToReconcilePodReadiness(pod) {
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodSync,
				StartTime:  start,
			})
		}

		// After an evicted pod is synced, all dead containers in the pod can be removed.
		// TODO: this is questionable - status read is async and during eviction we already
		// expect to not have some container info. The pod worker knows whether a pod has
		// been evicted, so if this is about minimizing the time to react to an eviction we
		// can do better. If it's about preserving pod status info we can also do better.
		if eviction.PodIsEvicted(pod.Status) {
			if podStatus, err := kl.podCache.Get(pod.UID); err == nil {
				kl.containerDeletor.deleteContainersInPod("", podStatus, true)
			}
		}
	}
}

func (kl *Kubelet) HandlePodSyncs(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			// Syncing a mirror pod is a programmer error since the intent of sync is to
			// batch notify all pending work. We should make it impossible to double sync,
			// but for now log a programmer error to prevent accidental introduction.
			klog.V(3).InfoS("Programmer error, HandlePodSyncs does not expect to receive mirror pods", "podUID", pod.UID, "mirrorPodUID", mirrorPod.UID)
			continue
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodSync,
			StartTime:  start,
		})
	}
}

func (kl *Kubelet) HandlePodCleanups(ctx context.Context) error {
	// The kubelet lacks checkpointing, so we need to introspect the set of pods
	// in the cgroup tree prior to inspecting the set of pods in our pod manager.
	// this ensures our view of the cgroup tree does not mistakenly observe pods
	// that are added after the fact...
	var (
		cgroupPods map[types.UID]cm.CgroupName
		err        error
	)
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		cgroupPods, err = pcm.GetAllPodsFromCgroups()
		if err != nil {
			return fmt.Errorf("failed to get list of pods that still exist on cgroup mounts: %v", err)
		}
	}

	allPods, mirrorPods, orphanedMirrorPodFullnames := kl.podManager.GetPodsAndMirrorPods()

	// Pod phase progresses monotonically. Once a pod has reached a final state,
	// it should never leave regardless of the restart policy. The statuses
	// of such pods should not be changed, and there is no need to sync them.
	// TODO: the logic here does not handle two cases:
	//   1. If the containers were removed immediately after they died, kubelet
	//      may fail to generate correct statuses, let alone filtering correctly.
	//   2. If kubelet restarted before writing the terminated status for a pod
	//      to the apiserver, it could still restart the terminated pod (even
	//      though the pod was not considered terminated by the apiserver).
	// These two conditions could be alleviated by checkpointing kubelet.

	// Stop the workers for terminated pods not in the config source
	klog.V(3).InfoS("Clean up pod workers for terminated pods")
	workingPods := kl.podWorkers.SyncKnownPods(allPods)

	// Reconcile: At this point the pod workers have been pruned to the set of
	// desired pods. Pods that must be restarted due to UID reuse, or leftover
	// pods from previous runs, are not known to the pod worker.

	allPodsByUID := make(map[types.UID]*v1.Pod)
	for _, pod := range allPods {
		allPodsByUID[pod.UID] = pod
	}

	// Identify the set of pods that have workers, which should be all pods
	// from config that are not terminated, as well as any terminating pods
	// that have already been removed from config. Pods that are terminating
	// will be added to possiblyRunningPods, to prevent overly aggressive
	// cleanup of pod cgroups.
	stringIfTrue := func(t bool) string {
		if t {
			return "true"
		}
		return ""
	}
	runningPods := make(map[types.UID]sets.Empty)
	possiblyRunningPods := make(map[types.UID]sets.Empty)
	for uid, sync := range workingPods {
		switch sync.State {
		case SyncPod:
			runningPods[uid] = struct{}{}
			possiblyRunningPods[uid] = struct{}{}
		case TerminatingPod:
			possiblyRunningPods[uid] = struct{}{}
		default:
		}
	}

	// Retrieve the list of running containers from the runtime to perform cleanup.
	// We need the latest state to avoid delaying restarts of static pods that reuse
	// a UID.
	if err := kl.runtimeCache.ForceUpdateIfOlder(ctx, kl.clock.Now()); err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}
	runningRuntimePods, err := kl.runtimeCache.GetPods(ctx)
	if err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}

	// Stop probing pods that are not running
	klog.V(3).InfoS("Clean up probes for terminated pods")
	kl.probeManager.CleanupPods(possiblyRunningPods)

	// Remove orphaned pod statuses not in the total list of known config pods
	klog.V(3).InfoS("Clean up orphaned pod statuses")
	kl.removeOrphanedPodStatuses(allPods, mirrorPods)

	// Remove orphaned pod user namespace allocations (if any).
	klog.V(3).InfoS("Clean up orphaned pod user namespace allocations")
	if err = kl.usernsManager.CleanupOrphanedPodUsernsAllocations(allPods, runningRuntimePods); err != nil {
		klog.ErrorS(err, "Failed cleaning up orphaned pod user namespaces allocations")
	}

	// Remove orphaned volumes from pods that are known not to have any
	// containers. Note that we pass all pods (including terminated pods) to
	// the function, so that we don't remove volumes associated with terminated
	// but not yet deleted pods.
	// TODO: this method could more aggressively cleanup terminated pods
	// in the future (volumes, mount dirs, logs, and containers could all be
	// better separated)
	klog.V(3).InfoS("Clean up orphaned pod directories")
	err = kl.cleanupOrphanedPodDirs(allPods, runningRuntimePods)
	if err != nil {
		// We want all cleanup tasks to be run even if one of them failed. So
		// we just log an error here and continue other cleanup tasks.
		// This also applies to the other clean up tasks.
		klog.ErrorS(err, "Failed cleaning up orphaned pod directories")
	}

	// Remove any orphaned mirror pods (mirror pods are tracked by name via the
	// pod worker)
	klog.V(3).InfoS("Clean up orphaned mirror pods")
	for _, podFullname := range orphanedMirrorPodFullnames {
		if !kl.podWorkers.IsPodForMirrorPodTerminatingByFullName(podFullname) {
			_, err := kl.mirrorPodClient.DeleteMirrorPod(podFullname, nil)
			if err != nil {
				klog.ErrorS(err, "Encountered error when deleting mirror pod", "podName", podFullname)
			} else {
				klog.V(3).InfoS("Deleted mirror pod", "podName", podFullname)
			}
		}
	}

	// After pruning pod workers for terminated pods get the list of active pods for
	// metrics and to determine restarts.
	activePods := kl.filterOutInactivePods(allPods)
	allRegularPods, allStaticPods := splitPodsByStatic(allPods)
	activeRegularPods, activeStaticPods := splitPodsByStatic(activePods)
	metrics.DesiredPodCount.WithLabelValues("").Set(float64(len(allRegularPods)))
	metrics.DesiredPodCount.WithLabelValues("true").Set(float64(len(allStaticPods)))
	metrics.ActivePodCount.WithLabelValues("").Set(float64(len(activeRegularPods)))
	metrics.ActivePodCount.WithLabelValues("true").Set(float64(len(activeStaticPods)))
	metrics.MirrorPodCount.Set(float64(len(mirrorPods)))

	// At this point, the pod worker is aware of which pods are not desired (SyncKnownPods).
	// We now look through the set of active pods for those that the pod worker is not aware of
	// and deliver an update. The most common reason a pod is not known is because the pod was
	// deleted and recreated with the same UID while the pod worker was driving its lifecycle (very
	// very rare for API pods, common for static pods with fixed UIDs). Containers that may still
	// be running from a previous execution must be reconciled by the pod worker's sync method.
	// We must use active pods because that is the set of admitted pods (podManager includes pods
	// that will never be run, and statusManager tracks already rejected pods).
	var restartCount, restartCountStatic int
	for _, desiredPod := range activePods {
		if _, knownPod := workingPods[desiredPod.UID]; knownPod {
			continue
		}

		klog.V(3).InfoS("Pod will be restarted because it is in the desired set and not known to the pod workers (likely due to UID reuse)", "podUID", desiredPod.UID)
		isStatic := kubetypes.IsStaticPod(desiredPod)
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(desiredPod)
		if pod == nil || wasMirror {
			klog.V(2).InfoS("Programmer error, restartable pod was a mirror pod but activePods should never contain a mirror pod", "podUID", desiredPod.UID)
			continue
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodCreate,
			Pod:        pod,
			MirrorPod:  mirrorPod,
		})

		// the desired pod is now known as well
		workingPods[desiredPod.UID] = PodWorkerSync{State: SyncPod, HasConfig: true, Static: isStatic}
		if isStatic {
			// restartable static pods are the normal case
			restartCountStatic++
		} else {
			// almost certainly means shenanigans, as API pods should never have the same UID after being deleted and recreated
			// unless there is a major API violation
			restartCount++
		}
	}
	metrics.RestartedPodTotal.WithLabelValues("true").Add(float64(restartCountStatic))
	metrics.RestartedPodTotal.WithLabelValues("").Add(float64(restartCount))

	// Complete termination of deleted pods that are not runtime pods (don't have
	// running containers), are terminal, and are not known to pod workers.
	// An example is pods rejected during kubelet admission that have never
	// started before (i.e. does not have an orphaned pod).
	// Adding the pods with SyncPodKill to pod workers allows to proceed with
	// force-deletion of such pods, yet preventing re-entry of the routine in the
	// next invocation of HandlePodCleanups.
	for _, pod := range kl.filterTerminalPodsToDelete(allPods, runningRuntimePods, workingPods) {
		klog.V(3).InfoS("Handling termination and deletion of the pod to pod workers", "pod", klog.KObj(pod), "podUID", pod.UID)
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodKill,
			Pod:        pod,
		})
	}

	// Finally, terminate any pods that are observed in the runtime but not present in the list of
	// known running pods from config. If we do terminate running runtime pods that will happen
	// asynchronously in the background and those will be processed in the next invocation of
	// HandlePodCleanups.
	var orphanCount int
	for _, runningPod := range runningRuntimePods {
		// If there are orphaned pod resources in CRI that are unknown to the pod worker, terminate them
		// now. Since housekeeping is exclusive to other pod worker updates, we know that no pods have
		// been added to the pod worker in the meantime. Note that pods that are not visible in the runtime
		// but which were previously known are terminated by SyncKnownPods().
		_, knownPod := workingPods[runningPod.ID]
		if !knownPod {
			one := int64(1)
			killPodOptions := &KillPodOptions{
				PodTerminationGracePeriodSecondsOverride: &one,
			}
			klog.V(2).InfoS("Clean up containers for orphaned pod we had not seen before", "podUID", runningPod.ID, "killPodOptions", killPodOptions)
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				UpdateType:     kubetypes.SyncPodKill,
				RunningPod:     runningPod,
				KillPodOptions: killPodOptions,
			})

			// the running pod is now known as well
			workingPods[runningPod.ID] = PodWorkerSync{State: TerminatingPod, Orphan: true}
			orphanCount++
		}
	}
	metrics.OrphanedRuntimePodTotal.Add(float64(orphanCount))

	// Now that we have recorded any terminating pods, and added new pods that should be running,
	// record a summary here. Not all possible combinations of PodWorkerSync values are valid.
	counts := make(map[PodWorkerSync]int)
	for _, sync := range workingPods {
		counts[sync]++
	}
	for validSync, configState := range map[PodWorkerSync]string{
		{HasConfig: true, Static: true}:                "desired",
		{HasConfig: true, Static: false}:               "desired",
		{Orphan: true, HasConfig: true, Static: true}:  "orphan",
		{Orphan: true, HasConfig: true, Static: false}: "orphan",
		{Orphan: true, HasConfig: false}:               "runtime_only",
	} {
		for _, state := range []PodWorkerState{SyncPod, TerminatingPod, TerminatedPod} {
			validSync.State = state
			count := counts[validSync]
			delete(counts, validSync)
			staticString := stringIfTrue(validSync.Static)
			if !validSync.HasConfig {
				staticString = "unknown"
			}
			metrics.WorkingPodCount.WithLabelValues(state.String(), configState, staticString).Set(float64(count))
		}
	}
	if len(counts) > 0 {
		// in case a combination is lost
		klog.V(3).InfoS("Programmer error, did not report a kubelet_working_pods metric for a value returned by SyncKnownPods", "counts", counts)
	}

	// Remove any cgroups in the hierarchy for pods that are definitely no longer
	// running (not in the container runtime).
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		klog.V(3).InfoS("Clean up orphaned pod cgroups")
		kl.cleanupOrphanedPodCgroups(pcm, cgroupPods, possiblyRunningPods)
	}

	// Cleanup any backoff entries.
	kl.backOff.GC()
	return nil
}
```

可以看到这些函数基本都是把pod交给 podWorkers 去处理


## podconfig

上文中的 updates channel 是从 podconfig 中获取的 那就来看看 podconfig 是怎么工作的

```go
type SourcesReady interface {
	// AddSource adds the specified source to the set of sources managed.
	AddSource(source string)
	// AllReady returns true if the currently configured sources have all been seen.
	AllReady() bool
}

// sourcesImpl implements SourcesReady.  It is thread-safe.
type sourcesImpl struct {
	// lock protects access to sources seen.
	lock sync.RWMutex
	// set of sources seen.
	sourcesSeen sets.String
	// sourcesReady is a function that evaluates if the sources are ready.
	sourcesReadyFn SourcesReadyFn
}
```

这里定义了接口 接口提中可以存储多个 source 那么有什么 source 呢

### kube-apiserver

```go
func NewSourceApiserver(c clientset.Interface, nodeName types.NodeName, nodeHasSynced func() bool, updates chan<- interface{}) {
	lw := cache.NewListWatchFromClient(c.CoreV1().RESTClient(), "pods", metav1.NamespaceAll, fields.OneTermEqualSelector("spec.nodeName", string(nodeName)))

	// The Reflector responsible for watching pods at the apiserver should be run only after
	// the node sync with the apiserver has completed.
	klog.InfoS("Waiting for node sync before watching apiserver pods")
	go func() {
		for {
			if nodeHasSynced() {
				klog.V(4).InfoS("node sync completed")
				break
			}
			time.Sleep(WaitForAPIServerSyncPeriod)
			klog.V(4).InfoS("node sync has not completed yet")
		}
		klog.InfoS("Watching apiserver")
		newSourceApiserverFromLW(lw, updates)
	}()
}

// newSourceApiserverFromLW holds creates a config source that watches and pulls from the apiserver.
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
	}
	r := cache.NewReflector(lw, &v1.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0)
	go r.Run(wait.NeverStop)
}
```

可以看到代码简单 就是 watrch apiserver 的 pod 然后把 pods 传给 updates channel

### file

```go
func (s *sourceFile) doWatch() error {
	_, err := os.Stat(s.path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return &retryableError{"path does not exist, ignoring"}
	}

	w, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("unable to create inotify: %v", err)
	}
	defer w.Close()

	err = w.Add(s.path)
	if err != nil {
		return fmt.Errorf("unable to create inotify for path %q: %v", s.path, err)
	}

	for {
		select {
		case event := <-w.Events:
			if err = s.produceWatchEvent(&event); err != nil {
				return fmt.Errorf("error while processing inotify event (%+v): %v", event, err)
			}
		case err = <-w.Errors:
			return fmt.Errorf("error while watching %q: %v", s.path, err)
		}
	}
}

func (s *sourceFile) produceWatchEvent(e *fsnotify.Event) error {
	// Ignore file start with dots
	if strings.HasPrefix(filepath.Base(e.Name), ".") {
		klog.V(4).InfoS("Ignored pod manifest, because it starts with dots", "eventName", e.Name)
		return nil
	}
	var eventType podEventType
	switch {
	case (e.Op & fsnotify.Create) > 0:
		eventType = podAdd
	case (e.Op & fsnotify.Write) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Chmod) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Remove) > 0:
		eventType = podDelete
	case (e.Op & fsnotify.Rename) > 0:
		eventType = podDelete
	default:
		// Ignore rest events
		return nil
	}

	s.watchEvents <- &watchEvent{e.Name, eventType}
	return nil
}

func (s *sourceFile) run() {
	listTicker := time.NewTicker(s.period)

	go func() {
		// Read path immediately to speed up startup.
		if err := s.listConfig(); err != nil {
			klog.ErrorS(err, "Unable to read config path", "path", s.path)
		}
		for {
			select {
			case <-listTicker.C:
				if err := s.listConfig(); err != nil {
					klog.ErrorS(err, "Unable to read config path", "path", s.path)
				}
			case e := <-s.watchEvents:
				if err := s.consumeWatchEvent(e); err != nil {
					klog.ErrorS(err, "Unable to process watch event")
				}
			}
		}
	}()

	s.startWatch()
}
```

上面是 file source 的代码 也是 watch 文件变化 然后把变化的 pod 传给 updates channel
这个主要的作用就是 kubeadm 等部署 k8s 集群 使用文件拉起 kube-apiserver etcd kube-controller-manager kube-scheduler 等组件

### http

```go
func (s *sourceURL) run() {
	if err := s.extractFromURL(); err != nil {
		// Don't log this multiple times per minute. The first few entries should be
		// enough to get the point across.
		if s.failureLogs < 3 {
			klog.InfoS("Failed to read pods from URL", "err", err)
		} else if s.failureLogs == 3 {
			klog.InfoS("Failed to read pods from URL. Dropping verbosity of this message to V(4)", "err", err)
		} else {
			klog.V(4).InfoS("Failed to read pods from URL", "err", err)
		}
		s.failureLogs++
	} else {
		if s.failureLogs > 0 {
			klog.InfoS("Successfully read pods from URL")
			s.failureLogs = 0
		}
	}
}

func (s *sourceURL) applyDefaults(pod *api.Pod) error {
	return applyDefaults(pod, s.url, false, s.nodeName)
}

func (s *sourceURL) extractFromURL() error {
	req, err := http.NewRequest("GET", s.url, nil)
	if err != nil {
		return err
	}
	req.Header = s.header
	resp, err := s.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	data, err := utilio.ReadAtMost(resp.Body, maxConfigLength)
	if err != nil {
		return err
	}
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("%v: %v", s.url, resp.Status)
	}
	if len(data) == 0 {
		// Emit an update with an empty PodList to allow HTTPSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return fmt.Errorf("zero-length data received from %v", s.url)
	}
	// Short circuit if the data has not changed since the last time it was read.
	if bytes.Equal(data, s.data) {
		return nil
	}
	s.data = data

	// First try as it is a single pod.
	parsed, pod, singlePodErr := tryDecodeSinglePod(data, s.applyDefaults)
	if parsed {
		if singlePodErr != nil {
			// It parsed but could not be used.
			return singlePodErr
		}
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{pod}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

	// That didn't work, so try a list of pods.
	parsed, podList, multiPodErr := tryDecodePodList(data, s.applyDefaults)
	if parsed {
		if multiPodErr != nil {
			// It parsed but could not be used.
			return multiPodErr
		}
		pods := make([]*v1.Pod, 0, len(podList.Items))
		for i := range podList.Items {
			pods = append(pods, &podList.Items[i])
		}
		s.updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

	return fmt.Errorf("%v: received '%v', but couldn't parse as "+
		"single (%v) or multiple pods (%v)",
		s.url, string(data), singlePodErr, multiPodErr)
}

```

http source 是 kubelet 本身开启 http 服务 通过调用 kubelet 的 http 接口来管理 pod 这个主要的作用是 给那些不想部署集群 只想使用 kubelet 的需求提供的

## PLEG

PLEG（Pod Lifecycle Event Generator）是 Kubernetes 中的一个关键组件，它负责监视和处理 Pod 的生命周期事件。PLEG 运行在每个节点上，并与 kubelet 组件紧密配合工作。

PLEG 的主要功能包括：

1. 监控容器状态：PLEG 监控每个节点上正在运行的容器的状态，并根据其状态变化生成相应的事件。
2. 生成事件：当容器的状态发生变化时，PLEG 会生成相应的事件，例如容器的创建、启动、停止、退出等事件。
3. 同步状态：PLEG 通过与 kubelet 进行交互，将容器的状态信息同步给 kubelet，使 kubelet 能够了解容器的当前状态。
4. 故障处理：PLEG 检测容器的状态变化，并在发现容器失败或异常时生成相应的事件，以便 kubelet 采取适当的故障处理措施。

PLEG 的设计目标是提供高效可靠的容器生命周期事件处理。它使用操作系统的文件系统事件和容器运行时的状态查询机制来监视容器的状态变化，从而及时地生成相应的事件。这些事件对于监控、日志记录、故障排除和自动恢复等方面非常重要。

```go
type PodLifecycleEventGenerator interface {
	Start()
	Watch() chan *PodLifecycleEvent
	Healthy() (bool, error)
	UpdateCache(*kubecontainer.Pod, types.UID) (error, bool)
}
```

kubelet 中实现了两种 PLEG，分别是：
- 通用 PLEG：用于处理普通容器的生命周期事件。使用轮询机制监控容器的状态变化，因此可能会有一定的延迟。而且耗费资源较多，不适合大规模部署。
- Event PLEG：用于处理事件容器的生命周期事件。使用事件机制监控容器的状态变化，因此响应速度较快。但是，他需要 container runtime 支持事件机制。

### GenericPLEG

```go
func (g *GenericPLEG) Watch() chan *PodLifecycleEvent {
	return g.eventChannel
}


func (g *GenericPLEG) Start() {
	g.runningMu.Lock()
	defer g.runningMu.Unlock()
	if !g.isRunning {
		g.isRunning = true
		g.stopCh = make(chan struct{})
		go wait.Until(g.Relist, g.relistDuration.RelistPeriod, g.stopCh)
	}
}

func (g *GenericPLEG) Relist() {
	g.relistLock.Lock()
	defer g.relistLock.Unlock()

	ctx := context.Background()
	klog.V(5).InfoS("GenericPLEG: Relisting")

	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
	}

	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
	}()

	// Get all the pods.
	podList, err := g.runtime.GetPods(ctx, true)
	if err != nil {
		klog.ErrorS(err, "GenericPLEG: Unable to retrieve pods")
		return
	}

	g.updateRelistTime(timestamp)

	pods := kubecontainer.Pods(podList)
	// update running pod and container count
	updateRunningPodAndContainerMetrics(pods)
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err, updated := g.updateCache(ctx, pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).ErrorS(err, "PLEG: Ignoring events for pod", "pod", klog.KRef(pod.Namespace, pod.Name))

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
				if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
					if !updated {
						continue
					}
				}
			}
		}
		// Update the internal storage and send out the events.
		g.podRecords.update(pid)

		// Map from containerId to exit code; used as a temporary cache for lookup
		containerExitCode := make(map[string]int)

		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.Inc()
				klog.ErrorS(nil, "Event channel is full, discard this relist() cycle event")
			}
			// Log exit code of containers when they finished in a particular event
			if events[i].Type == ContainerDied {
				// Fill up containerExitCode map for ContainerDied event when first time appeared
				if len(containerExitCode) == 0 && pod != nil && g.cache != nil {
					// Get updated podStatus
					status, err := g.cache.Get(pod.ID)
					if err == nil {
						for _, containerStatus := range status.ContainerStatuses {
							containerExitCode[containerStatus.ID.ID] = containerStatus.ExitCode
						}
					}
				}
				if containerID, ok := events[i].Data.(string); ok {
					if exitCode, ok := containerExitCode[containerID]; ok && pod != nil {
						klog.V(2).InfoS("Generic (PLEG): container finished", "podID", pod.ID, "containerID", containerID, "exitCode", exitCode)
					}
				}
			}
		}
	}

	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).InfoS("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err, _ := g.updateCache(ctx, pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).ErrorS(err, "PLEG: pod failed reinspection", "pod", klog.KRef(pod.Namespace, pod.Name))
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	g.podsToReinspect = needsReinspection
}
```

可以看到 GenericPLEG 会定时从 runtime 获取 pods 然后和缓存中的旧的 pod 进行对比 然后生成事件发送给 eventChannel

### EventPLEG

```go
func (e *EventedPLEG) Start() {
	e.runningMu.Lock()
	defer e.runningMu.Unlock()
	if isEventedPLEGInUse() {
		return
	}
	setEventedPLEGUsage(true)
	e.stopCh = make(chan struct{})
	e.stopCacheUpdateCh = make(chan struct{})
	go wait.Until(e.watchEventsChannel, 0, e.stopCh)
	go wait.Until(e.updateGlobalCache, globalCacheUpdatePeriod, e.stopCacheUpdateCh)
}


func (e *EventedPLEG) watchEventsChannel() {
	containerEventsResponseCh := make(chan *runtimeapi.ContainerEventResponse, cap(e.eventChannel))
	defer close(containerEventsResponseCh)

	// Get the container events from the runtime.
	go func() {
		numAttempts := 0
		for {
			if numAttempts >= e.eventedPlegMaxStreamRetries {
				if isEventedPLEGInUse() {
					// Fall back to Generic PLEG relisting since Evented PLEG is not working.
					klog.V(4).InfoS("Fall back to Generic PLEG relisting since Evented PLEG is not working")
					e.Stop()
					e.genericPleg.Stop()       // Stop the existing Generic PLEG which runs with longer relisting period when Evented PLEG is in use.
					e.Update(e.relistDuration) // Update the relisting period to the default value for the Generic PLEG.
					e.genericPleg.Start()
					break
				}
			}

			err := e.runtimeService.GetContainerEvents(containerEventsResponseCh)
			if err != nil {
				metrics.EventedPLEGConnErr.Inc()
				numAttempts++
				e.Relist() // Force a relist to get the latest container and pods running metric.
				klog.V(4).InfoS("Evented PLEG: Failed to get container events, retrying: ", "err", err)
			}
		}
	}()

	if isEventedPLEGInUse() {
		e.processCRIEvents(containerEventsResponseCh)
	}
}

// 转换 runtimeapi.ContainerEventResponse 为 PodLifecycleEvent
func (e *EventedPLEG) processCRIEvents(containerEventsResponseCh chan *runtimeapi.ContainerEventResponse) {
	for event := range containerEventsResponseCh {
		// Ignore the event if PodSandboxStatus is nil.
		// This might happen under some race condition where the podSandbox has
		// been deleted, and therefore container runtime couldn't find the
		// podSandbox for the container when generating the event.
		// It is safe to ignore because
		// a) a event would have been received for the sandbox deletion,
		// b) in worst case, a relist will eventually sync the pod status.
		// TODO(#114371): Figure out a way to handle this case instead of ignoring.
		if event.PodSandboxStatus == nil || event.PodSandboxStatus.Metadata == nil {
			klog.ErrorS(nil, "Evented PLEG: received ContainerEventResponse with nil PodSandboxStatus or PodSandboxStatus.Metadata", "containerEventResponse", event)
			continue
		}

		podID := types.UID(event.PodSandboxStatus.Metadata.Uid)
		shouldSendPLEGEvent := false

		status, err := e.runtime.GeneratePodStatus(event)
		if err != nil {
			// nolint:logcheck // Not using the result of klog.V inside the
			// if branch is okay, we just use it to determine whether the
			// additional "podStatus" key and its value should be added.
			if klog.V(6).Enabled() {
				klog.ErrorS(err, "Evented PLEG: error generating pod status from the received event", "podUID", podID, "podStatus", status)
			} else {
				klog.ErrorS(err, "Evented PLEG: error generating pod status from the received event", "podUID", podID)
			}
		} else {
			if klogV := klog.V(6); klogV.Enabled() {
				klogV.InfoS("Evented PLEG: Generated pod status from the received event", "podUID", podID, "podStatus", status)
			} else {
				klog.V(4).InfoS("Evented PLEG: Generated pod status from the received event", "podUID", podID)
			}
			// Preserve the pod IP across cache updates if the new IP is empty.
			// When a pod is torn down, kubelet may race with PLEG and retrieve
			// a pod status after network teardown, but the kubernetes API expects
			// the completed pod's IP to be available after the pod is dead.
			status.IPs = e.getPodIPs(podID, status)
		}

		e.updateRunningPodMetric(status)
		e.updateRunningContainerMetric(status)
		e.updateLatencyMetric(event)

		if event.ContainerEventType == runtimeapi.ContainerEventType_CONTAINER_DELETED_EVENT {
			for _, sandbox := range status.SandboxStatuses {
				if sandbox.Id == event.ContainerId {
					// When the CONTAINER_DELETED_EVENT is received by the kubelet,
					// the runtime has indicated that the container has been removed
					// by the runtime and hence, it must be removed from the cache
					// of kubelet too.
					e.cache.Delete(podID)
				}
			}
			shouldSendPLEGEvent = true
		} else {
			if e.cache.Set(podID, status, err, time.Unix(event.GetCreatedAt(), 0)) {
				shouldSendPLEGEvent = true
			}
		}

		if shouldSendPLEGEvent {
			e.processCRIEvent(event)
		}
	}
}


```

从代码中可以看到 EventPLEG 可以从 runtime 获取容器事件。