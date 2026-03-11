---
title: "Kubernetes CNI 插件调用流程"
date: 2026-03-10T21:40:37+08:00
tags:
- kubernetes
- cni
- calico
- containerd
- network
showToc: true
categories:
- cloud
---

## 前言

在之前的文章 [Kubernetes CNI（Container Network Interface）](https://daemon365.dev/post/cloud/kubernetes_cni_container_network_interface/) 中，我们介绍了 CNI 的基本概念与设计思路。那么，CNI 插件究竟是如何被调用的？本文将以 kubelet、containerd 和 Calico 的协作为例，梳理 CNI 插件从创建 Pod 到网络就绪的完整调用流程。

---

## 涉及项目

| 项目 | 地址 |
|------|------|
| CNI 规范与库 | https://github.com/containernetworking/cni |
| containerd | https://github.com/containerd/containerd |
| go-cni | https://github.com/containerd/go-cni |
| Kubernetes | https://github.com/kubernetes/kubernetes |

---

## CNI 插件调用流程

![cni-call](/images/cni-call.png)

### 1. kubelet 触发 Pod 创建

kubelet 检测到有新的 Pod 需要调度，随即通过 CRI（Container Runtime Interface）调用 containerd 的 API，请求创建对应的 Sandbox 容器（即 pause 容器）。

### 2. containerd 创建 Network Namespace

containerd 收到创建 Sandbox 的请求后，会先为该 Pod 创建一个独立的 **network namespace**，作为后续网络配置的基础隔离环境。

### 3. 调用 go-cni 处理网络配置

containerd 使用 **go-cni** 库来驱动网络初始化流程。go-cni 是对 `containernetworking/cni` 库的二次封装，屏蔽了底层细节，最终会调用 CNI 规范中的 `AddNetworkList` 函数。

### 4. 解析 CNI 配置与二进制目录

CNI 接口在初始化时会自动扫描并解析配置文件目录和二进制目录，默认路径如下：

- **配置文件**：`/etc/cni/net.d/`
- **插件二进制**：`/opt/cni/bin/`

配置文件中定义了使用哪个 CNI 插件（如 Calico），以及 IPAM 等参数。

### 5. 执行 Calico 二进制文件

`AddNetworkList` 根据传入的 `netns`、`IfName` 等参数，找到对应的插件二进制（由配置文件指定），以子进程方式执行 Calico 的 CNI 二进制文件，并通过环境变量传递上下文信息，例如：
```
CNI_COMMAND=ADD
CNI_CONTAINERID=<container-id>
CNI_NETNS=/proc/<pid>/ns/net
CNI_IFNAME=eth0
CNI_PATH=/opt/cni/bin
```

### 6. Calico 执行网络配置

Calico 的 CNI 二进制读取上述环境变量后，执行以下操作：

1. **IP 地址分配**：调用配置中指定的 IPAM 插件（如 `calico-ipam`），通过访问 kube-apiserver 写入 `IPAMBlock` 资源，完成 IP 地址的分配与记录。
2. **网络配置**：创建 veth pair，将一端移入 Pod 的 network namespace，另一端留在宿主机上，并设置相应的路由规则。
3. **写入网络信息**：将分配好的网卡信息（IP、网关、路由等）写入 Pod 的 network namespace，完成容器侧的网络初始化。

### 7. 结果返回给 containerd

Calico 二进制执行完成后，将网络配置结果（符合 CNI 规范的 JSON 格式）通过 **stdout** 返回给 containerd，containerd 再将结果上报给 kubelet。

### 8. Calico DaemonSet 同步路由信息

运行在每个节点上的 Calico DaemonSet 容器（`calico-node`）会通过 **watch Typha** 或直接 watch kube-apiserver 来监听 `IPAMBlock` 等资源的变化。一旦检测到新的 IP 分配，便将对应的路由信息写入：

以下会写入其中一个，看你开启的哪种模式：
- **ip route**：通过 Linux 内核路由表实现跨节点转发
- **eBPF**：通过 eBPF dataplane 实现更高效的数据面转发

Typha 的作用做是为 Calico DaemonSet 提供一个集中式的消息总线，减少每个节点直接 watch kube-apiserver 的压力，提高系统的可伸缩性。

---

## 源码分析

### 1. containerd — `RunPodSandbox`

源码位置：`internal/cri/server/sandbox_run.go`

kubelet 通过 CRI 调用 containerd 的 `RunPodSandbox`，其中网络初始化由 `setupPodNetwork` 负责：
```go
func (c *criService) RunPodSandbox(ctx context.Context, r *runtime.RunPodSandboxRequest) (_ *runtime.RunPodSandboxResponse, retErr error) {
    // ...
    if err := c.setupPodNetwork(ctx, &sandbox); err != nil {
        return nil, fmt.Errorf("failed to setup network for sandbox %q: %w", id, err)
    }
    // ...
}

func (c *criService) setupPodNetwork(ctx context.Context, sandbox *sandboxstore.Sandbox) (retErr error) {
    // ...
    // 准备 CNI 所需的命名空间配置选项
    opts, err := cniNamespaceOpts(id, config)
    if err != nil {
        return fmt.Errorf("get cni namespace options: %w", err)
    }
    // ...
    // 支持串行和并发两种模式，多网卡场景下可并发调用
    if c.config.CniConfig.NetworkPluginSetupSerially {
        result, err = netPlugin.SetupSerially(ctx, id, path, opts...)
    } else {
        result, err = netPlugin.Setup(ctx, id, path, opts...)
    }
    // ...
}
```

> `path` 这里传入的是之前创建好的 network namespace 路径，如 `/proc/<pid>/ns/net`，而不是二进制目录。

---

### 2. go-cni — `SetupSerially`

源码位置：`cni/cni.go`

go-cni 是对 `containernetworking/cni` 的上层封装，`SetupSerially` 会依次遍历所有配置的网络插件（如 calico、portmap）并逐一挂载：

```go
func (c *libcni) SetupSerially(ctx context.Context, id string, path string, opts ...NamespaceOpts) (*Result, error) {
    c.RLock()
    defer c.RUnlock()
    if err := c.ready(); err != nil {
        return nil, err
    }
    // 根据 id、netns path 以及配置选项，构建 Namespace 对象
    ns, err := newNamespace(id, path, opts...)
    if err != nil {
        return nil, err
    }
    result, err := c.attachNetworksSerially(ctx, ns)
    if err != nil {
        return nil, err
    }
    return c.createResult(result)
}

func (c *libcni) attachNetworksSerially(ctx context.Context, ns *Namespace) ([]*types100.Result, error) {
    var results []*types100.Result
    // networks 是扫面 /etc/cni/net.d/ 下的文件创建出来的
    for _, network := range c.networks {
        r, err := network.Attach(ctx, ns)
        if err != nil {
            return nil, err
        }
        results = append(results, r)
    }
    return results, nil
}

func (n *Network) Attach(ctx context.Context, ns *Namespace) (*types100.Result, error) {
    // 最终调用 CNI 规范库的 AddNetworkList，触发实际的插件二进制执行
    r, err := n.cni.AddNetworkList(ctx, n.config, ns.config(n.ifName))
    if err != nil {
        return nil, err
    }
    return types100.NewResultFromResult(r)
}
```

---

### 3. CNI 规范库 — `AddNetworkList`

源码位置：`containernetworking/cni/libcni/api.go`

`AddNetworkList` 会遍历配置列表中的所有插件（例如一个典型的 Calico 配置可能包含 `calico` 和 `bandwidth` 或 `portmap` 两个插件），依次调用 `addNetwork`：
```go
func (c *CNIConfig) AddNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
    for _, net := range list.Plugins {
        result, err = c.addNetwork(ctx, list.Name, list.CNIVersion, net, result, rt)
        if err != nil {
            return nil, fmt.Errorf("plugin %s failed (add): %w", pluginDescription(net.Network), err)
        }
    }
    return result, nil
}

func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *PluginConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
    // 在 c.Path（即 /opt/cni/bin/）中查找对应名称的二进制文件
    pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)
    if err != nil {
        return nil, err
    }
    // ...
    return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)
}

// 将运行时信息封装为 Args 结构体，供后续转换为环境变量
func (c *CNIConfig) args(action string, rt *RuntimeConf) *invoke.Args {
    return &invoke.Args{
        Command:     action,
        ContainerID: rt.ContainerID,
        NetNS:       rt.NetNS,
        PluginArgs:  rt.Args,
        IfName:      rt.IfName,
        Path:        strings.Join(c.Path, string(os.PathListSeparator)),
    }
}
```

---

### 4. 最终执行插件二进制

CNI 规范库通过 `exec.CommandContext` 直接启动插件二进制，并通过环境变量将上下文信息传入：

```go
func ExecPluginWithResult(ctx context.Context, pluginPath string, netconf []byte, args CNIArgs, exec Exec) (types.Result, error) {
    // netconf（即配置文件内容）通过 stdin 传给插件，结果从 stdout 读回
    stdoutBytes, err := exec.ExecPlugin(ctx, pluginPath, netconf, args.AsEnv())
    if err != nil {
        return nil, err
    }
    // ...
}

func (e *RawExec) ExecPlugin(ctx context.Context, pluginPath string, stdinData []byte, environ []string) ([]byte, error) {
    // ...
    c := exec.CommandContext(ctx, pluginPath)
    c.Env = environ
    // ...
    // 最多重试 5 次（主要应对插件短暂不可用的情况）
    for i := 0; i <= 5; i++ {
        err := c.Run()
        // ...
    }
    // ...
}
```

`AsEnv()` 负责将 `Args` 结构体展开为标准的 CNI 环境变量：
```go
func (args *Args) AsEnv() []string {
    env := os.Environ()
    pluginArgsStr := args.PluginArgsStr
    if pluginArgsStr == "" {
        pluginArgsStr = stringify(args.PluginArgs)
    }
    env = append(env,
        "CNI_COMMAND="+args.Command,       // 操作类型：ADD / DEL / CHECK
        "CNI_CONTAINERID="+args.ContainerID, // Pod 的容器 ID
        "CNI_NETNS="+args.NetNS,            // network namespace 路径
        "CNI_ARGS="+pluginArgsStr,          // 额外的键值对参数
        "CNI_IFNAME="+args.IfName,          // 容器内的网卡名，默认 eth0
        "CNI_PATH="+args.Path,              // CNI 二进制所在目录
    )
    return dedupEnv(env)
}
```

---

### 调用链路总结

![cni-call-code](/images/cni-call-code.png)

