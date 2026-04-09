---
title: "从源码看 Calico 如何为 Pod 配网"
date: 2026-04-08T18:41:01+08:00
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

## 简介

在 [Kubernetes CNI 插件调用流程](https://daemon365.dev/post/cloud/cni-call/) 中，介绍了 CNI 是怎么通过 config 文件调用二进制插件配置网络的。本文就顺着这条线，聚焦 Calico CNI 插件本身，看看一个 Pod 的网络是怎么一步步配起来的。

本文基于 calico v3.31.4 版本进行分析，其他版本可能会有一些差异。

---

## calico 添加网络的流程

![calico-bin](/images/calico-bin.png)

---

## 代码流程

### 入口：main 函数

calico 把多个 CNI 插件打进了同一个二进制，通过判断 `os.Args[0]`（也就是二进制文件本身的文件名）来决定走哪个插件逻辑。这里用了典型的多路复用二进制模式：同一个可执行文件，通过不同的调用名分流到不同插件逻辑，busybox 也采用过类似思路。

```go
func main() {
    _, filename := filepath.Split(os.Args[0])
    switch filename {
    case "calico", "calico.exe":
        plugin.Main(buildinfo.Version)
    case "calico-ipam", "calico-ipam.exe":
        ipamplugin.Main(buildinfo.Version)
    default:
        panic("Unknown binary name: " + filename)
    }
}
```

`plugin.Main` 里注册了 CNI 插件处理的几个标准入口，这里 Calico 注册了 Add、Del 和 Check。对 Kubernetes 场景来说，最核心的是 Add/Del，也就是创建和清理 Pod 网络时会走到的入口。

```go
func Main(version string) {
    // ......
    funcs := skel.CNIFuncs{
        Add:   cmdAdd, // 添加网络的函数
        Del:   cmdDel,
        Check: cmdDummyCheck,
    }
    // ......
}
```

---

### cmdAdd：添加网络

`cmdAdd` 是整个流程的起点。calico 支持多种 orchestrator（编排器），比如 Kubernetes、Mesos 等，这里做了一个分支判断。在 k8s 场景下，绝大多数情况都会走 `CmdAddK8s`。

```go
func cmdAdd(args *skel.CmdArgs) (err error) {
    // ......
    // 配置一些参数，比如 MTU 等

    // 如果是 k8s 调用，则调用 k8s 的 cmdAddK8s 函数
    if wepIDs.Orchestrator == api.OrchestratorKubernetes {
        if result, err = k8s.CmdAddK8s(ctx, args, conf, *wepIDs, calicoClient, endpoint); err != nil {
            return
        }
    } else {
        // ......
    }

    // ......
    return
}
```

---

### CmdAddK8s：K8s 场景下的网络配置

这个函数负责处理 K8s Pod 的网络配置。把代码流程压缩一下，基本可以分成两步：

1. **先决定 Pod 用哪个 IP，以及这个 IP 的生命周期由谁管理**
2. **再把这个结果交给 dataplane 层，完成 veth、地址、路由等实际配置**

第一步里，Calico 常见的分支主要有三种：

- **默认路径**：没有相关 annotation，调用 Calico IPAM 分配地址
- **ipAddrsNoIpam**：直接使用 annotation 里给定的地址，不经过当前配置的 IPAM
- **ipAddrs**：地址由 annotation 指定，但仍纳入 Calico IPAM 的生命周期管理

```go
func CmdAddK8s(ctx context.Context, args *skel.CmdArgs, conf types.NetConf, epIDs utils.WEPIdentifiers, calicoClient calicoclient.Interface, endpoint *libapi.WorkloadEndpoint) (*cniv1.Result, error) {
    // ......

    if conf.Policy.PolicyType == "k8s" {
        // 配置一些标签和注解
    }

    ipAddrsNoIpam := annot["cni.projectcalico.org/ipAddrsNoIpam"]
    ipAddrs := annot["cni.projectcalico.org/ipAddrs"]

    switch {
    case ipAddrs == "" && ipAddrsNoIpam == "":
        // 调用 IPAM 插件进行 IP 地址分配
        result, err = utils.AddIPAM(conf, args, logger)
        if err != nil {
            return nil, err
        }

    case ipAddrs != "" && ipAddrsNoIpam != "":
        // 报错，不能同时设置两个 annotation

    case ipAddrsNoIpam != "":
        // 不调用 IPAM，直接使用 annotation 里的 IP

    case ipAddrs != "":
        // 调用 IPAM 管理生命周期，但 IP 从 annotation 里取
    }

    // 做网络配置
    hostVethName, contVethMac, err := d.DoNetworking(
        ctx, calicoClient, args, result, desiredVethName, routes, endpoint, annot)
    if err != nil {
        logger.WithError(err).Error("Error setting up networking")
        releaseIPAM()
        return nil, err
    }

    // 配置 result
    // ......

    return result, nil
}
```

---

### DoNetworking & DoWorkloadNetnsSetUp：配置 veth 和路由

到这一层，流程就从“决定配置什么”进入“实际修改 netns、链路、地址和路由”的阶段了。核心动作是创建 veth pair，并在容器 netns 内把地址和路由配好。

veth pair 是 Linux 内核提供的一种虚拟网络设备，两端成对出现，数据从一端进，从另一端出。calico 用它来打通容器内部和宿主机之间的网络通路：一端在容器 netns 里（通常叫 `eth0`），另一端在宿主机的 root netns 里（名字类似 `cali1a2b3c4d`）。

```go
func (d *linuxDataplane) DoNetworking(
    ctx context.Context,
    calicoClient calicoclient.Interface,
    args *skel.CmdArgs,
    result *cniv1.Result,
    desiredVethName string,
    routes []*net.IPNet,
    endpoint *api.WorkloadEndpoint,
    annotations map[string]string,
) (hostVethName, contVethMAC string, err error) {
    // ......
    contVethMAC, err = d.DoWorkloadNetnsSetUp(
        hostNlHandle,
        args.Netns,
        result.IPs,
        args.IfName,
        hostVethName,
        routes,
        annotations,
    )
    // ......
    return hostVethName, contVethMAC, err
}
```

`DoWorkloadNetnsSetUp` 会进入容器的 network namespace 来操作，这里用的是 `ns.WithNetNSPath`，它内部通过 `setns` 系统调用切换 netns。

```go
func (d *linuxDataplane) DoWorkloadNetnsSetUp(
    hostNlHandle *netlink.Handle,
    netnsPath string,
    ipAddrs []*cniv1.IPConfig,
    contVethName string,
    hostVethName string,
    routes []*net.IPNet,
    annotations map[string]string,
) (contVethMAC string, err error) {
    // 如果宿主机上同名的 hostVeth 已经存在（比如 Pod 重建），先清掉
    if oldHostVeth, err := hostNlHandle.LinkByName(hostVethName); err == nil {
        if err = hostNlHandle.LinkDel(oldHostVeth); err != nil {
            return "", fmt.Errorf("failed to delete old hostVeth %v: %v", hostVethName, err)
        }
        d.logger.Infof("Cleaning old hostVeth: %v", hostVethName)
    }

    err = ns.WithNetNSPath(netnsPath, func(hostNS ns.NetNS) error {
        la := netlink.NewLinkAttrs()
        la.Name = contVethName
        la.MTU = d.mtu
        la.NumTxQueues = d.queues
        la.NumRxQueues = d.queues
        // 创建 veth pair：容器端叫 contVethName，宿主机端叫 hostVethName
        veth := &netlink.Veth{
            LinkAttrs:     la,
            PeerName:      hostVethName,
            PeerNamespace: netlink.NsFd(int(hostNS.Fd())),
        }

        if err := netlink.LinkAdd(veth); err != nil {
            d.logger.Errorf("Error adding veth %+v: %s", veth, err)
            return err
        }

        hostVeth, err := hostNlHandle.LinkByName(hostVethName)
        if err != nil {
            err = fmt.Errorf("failed to lookup %q: %v", hostVethName, err)
            return err
        }
        // ......

        // 根据 IP 版本设置掩码（容器 IP 用 /32 或 /128，而不是子网掩码）
        var hasIPv4, hasIPv6 bool
        for _, addr := range ipAddrs {
            if addr.Address.IP.To4() != nil {
                hasIPv4 = true
                addr.Address.Mask = net.CIDRMask(32, 32)
            } else if addr.Address.IP.To16() != nil {
                hasIPv6 = true
                addr.Address.Mask = net.CIDRMask(128, 128)
            }
        }

        if hasIPv4 {
            // 添加一条指向 169.254.1.1 的链路路由
            gw := net.IPv4(169, 254, 1, 1)
            gwNet := &net.IPNet{IP: gw, Mask: net.CIDRMask(32, 32)}
            err := netlink.RouteAdd(
                &netlink.Route{
                    LinkIndex: contVeth.Attrs().Index,
                    Scope:     netlink.SCOPE_LINK,
                    Dst:       gwNet,
                },
            )
            if err != nil {
                return fmt.Errorf("failed to add route inside the container: %v", err)
            }

            for _, r := range routes {
                if r.IP.To4() == nil {
                    continue
                }
                if err = ip.AddRoute(r, gw, contVeth); err != nil {
                    return fmt.Errorf("failed to add IPv4 route for %v via %v: %v", r, gw, err)
                }
            }
        }

        // ......
        return nil
    })

    if err != nil {
        d.logger.Errorf("Error creating veth: %s", err)
        return "", err
    }

    return
}
```

这里有一个很容易让人第一次看到时困惑的点：容器内地址会被改成 `/32`（IPv6 时是 `/128`）。这意味着它不是传统二层子网里“和网关同网段”的配置，而更像一个点到点、显式路由模型。容器真正可达的下一跳不是“同网段网关地址”，而是通过一条 `scope link` 路由，把 `169.254.1.1` 指到当前接口。

---

### 为什么路由网关是 169.254.1.1？

这是 calico 的一个经典设计，经常让初次接触的人感到困惑。在容器里看到的路由表大概是这样的：

```bash
ip r
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

这里有几个关键点：

1. **169.254.1.1 不是宿主机真实配置在接口上的网关地址**。它更准确地说，是 Calico 在容器侧使用的一个链路本地网关地址。
2. **容器里的默认路由和业务路由都会指向它**。同时 `169.254.1.1` 自身又通过 `scope link` 路由绑定在 `eth0` 上，所以从容器视角看，它就是当前接口可达的下一跳。
3. **宿主机侧不会真的把这个 IP 配到 veth 上**。相反，host veth 会开启 `proxy_arp`，当容器对 `169.254.1.1` 发起 ARP 查询时，由宿主机侧接口代为响应。
4. **proxy ARP 的使用范围是收敛的**。Calico 不是让容器把所有目标地址都依赖 proxy ARP 解析；这里主要是为了让容器把 `169.254.1.1` 视作可达网关。
5. **后续真正的转发发生在宿主机路由表里**。数据包一旦进入 host 侧 veth，就由宿主机根据路由、策略和后续 dataplane 机制继续处理。

这种设计的好处是：容器侧不需要感知宿主机上任何“真实网关 IP”，接口配置可以保持统一，而转发决策则统一收敛到宿主机。

---

## 如何在节点上验证这条流程

前面的内容如果只停留在源码层面，读起来还是容易飘。比较好的方式，是到节点上把几个关键现象对一下。

### 1. 看容器里的路由

```bash
kubectl exec -it <pod> -- ip r
```

预期会看到类似结果：

```bash
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

这说明容器里的默认流量确实是先交给 `169.254.1.1`，而不是交给一个和 Pod IP 同网段的“传统网关”。

### 2. 看 host 侧 veth 是否开启了 proxy_arp

先在节点上找到和这个 Pod 对应的 host veth，然后查看：

```bash
sysctl net.ipv4.conf.<host-veth>.proxy_arp
```

或者：

```bash
cat /proc/sys/net/ipv4/conf/<host-veth>/proxy_arp
```

如果值是 `1`，就说明这个接口会替 `169.254.1.1` 响应 ARP。

### 3. 看容器是否把 169.254.1.1 当成邻居

```bash
kubectl exec -it <pod> -- ip neigh
```

必要时也可以在节点上抓包，看容器是否真的在查询 `169.254.1.1` 的 ARP，以及最终拿到的是 host veth 的 MAC。

### 4. 看当前节点拿到了哪些 IPAM block

```bash
calicoctl get ipamblocks -o wide
```

或者结合节点名过滤相关资源，确认这个节点当前关联了哪些 block、这些 block 是否已经接近分满。

---

## calico-ipam

IPAM 部分是另一个二进制（或者同一个二进制但文件名是 `calico-ipam`），负责 IP 地址的分配和回收。

### 整体结构：Block 是核心

calico IPAM 的设计里有一个很重要的概念叫 **Block**。calico 不是把整个 IP Pool 当成一个平面地址池逐个分配，而是先把大的 IP Pool（比如 `192.168.0.0/16`）切分成一块一块的 Block，再把每个 Block 关联给某个节点，节点随后从自己的 Block 里给 Pod 分 IP。

默认情况下，IPv4 block 大小是 `/26`，也就是每个 block 有 64 个地址。这个默认值并不是随便定的，它本质上是在“每节点可承载的 Pod 数量”和“路由聚合、状态同步的规模”之间做折中。

这样设计的好处是：
- **减少 API Server 压力**：一个 Block 里 64 个 IP，只需要一次 CRD 操作就能表示
- **配合 BGP 路由聚合**：节点可以把自己拥有的 Block 作为一条路由在 BGP 里广播，减少路由条目数量
- **分配效率高**：同一个节点的 IP 通常在同一个 Block 里，不需要全局锁

如果一个节点把当前关联 block 里的地址分完了，Calico 会继续给它申领新的 block；如果受 `MaxBlocksPerHost` 等限制拿不到新的 block，某些情况下就只能从其他节点已关联的 block 中借地址来分配。

### Block 对应的 CRD

在 k8s 里，每个 Block 是一个 `IPAMBlock` CRD 资源，可以用 calicoctl 或者 kubectl 查看：

```bash
# 查看所有 IPAMBlock
calicoctl get ipamblock
# 或者
kubectl get ipamblocks.crd.projectcalico.org -o yaml

# 查看某个具体的 block
calicoctl get ipamblock 192.168.1.0-26 -o yaml
```

一个典型的 IPAMBlock 长这样：

```yaml
apiVersion: crd.projectcalico.org/v1
kind: IPAMBlock
metadata:
  name: 192-168-1-0-26
spec:
  affinity: host:node-1          # 这个 block 归属于 node-1
  cidr: 192.168.1.0/26
  allocations:                   # 64 个槽位，nil 表示未分配
    - null
    - 0                          # 索引 0 的属性
    - null
    - 1
    # ...
  attributes:                    # 每个属性记录是哪个 Pod 在用
    - handle_id: k8s-pod-namespace.podname
      secondary:
        node: node-1
        pod: podname
        namespace: namespace
  unallocated:                   # 还没分配出去的偏移量列表
    - 0
    - 2
    - 5
    # ...
```

除了 IPAMBlock，calico 还维护了 `IPAMHandle` 资源，用来记录某个 handle（通常对应一个 Pod）持有哪些 IP，方便回收时做反向查找：

```bash
calicoctl get ipamhandle
kubectl get ipamhandles.crd.projectcalico.org
```

### cmdAdd：IPAM 的入口

```go
func cmdAdd(args *skel.CmdArgs) error {
    // ......
    autoAssignWithLock := func(calicoClient client.Interface, assignArgs ipam.AutoAssignArgs) (*ipam.IPAMAssignments, *ipam.IPAMAssignments, error) {
        // 先获取一个全局的文件锁，防止同一台机器上多个 CNI 调用并发修改 IPAM 状态
        unlock := acquireIPAMLockBestEffort(conf.IPAMLockFile)
        defer unlock()
        ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
        defer cancel()
        // 正式分配 IP
        return calicoClient.IPAM().AutoAssign(ctx, assignArgs)
    }
    v4Assignments, v6Assignments, err := autoAssignWithLock(calicoClient, assignArgs)
    // ......
}
```

这里有个细节值得注意：calico 在调用 `AutoAssign` 之前，会先拿一个本地的文件锁（`IPAMLockFile`）。这是因为同一台节点上可能同时有多个容器在启动（比如 DaemonSet 滚动更新），如果并发请求打到 datastore 上，很容易出现竞态条件。文件锁是一个粗粒度但有效的保护手段。

### AutoAssign：分配 IP 的核心逻辑

```go
func (c ipamClient) AutoAssign(ctx context.Context, args AutoAssignArgs) (*IPAMAssignments, *IPAMAssignments, error) {
    // ......
    if args.Num4 != 0 {
        v4ia, err = c.autoAssign(ctx, args.Num4, args.HandleID, args.Attrs, args.IPv4Pools, 4, hostname, args.MaxBlocksPerHost, args.HostReservedAttrIPv4s, args.IntendedUse, args.Namespace)
    }
    // ......
    return v4ia, v6ia, nil
}
```

### findOrClaimBlock：找到或申领一个 Block

这是 IPAM 里最复杂也最关键的部分。每次分配 IP 时，calico 需要先确定从哪个 Block 里拿。大致流程是：

1. **优先找当前节点已有亲和性的 Block**：如果这个节点之前已经被分配过 Block（即 `spec.affinity: host:当前节点`），优先从这些 Block 里找空余的槽位
2. **如果没有可用 Block 或已有 Block 都满了**：尝试从 IP Pool 里划出一个新的 `/26` Block，写入 datastore，并把 affinity 设置为当前节点
3. **如果超过了节点最大 Block 数限制（`MaxBlocksPerHost`）**：就只能尝试从其他节点"借"Block 里的空余 IP（跨节点分配，路由会更复杂）

这里用到了乐观锁机制——分配 Block 时会做 CAS（Compare-And-Swap）操作，如果有并发冲突，会重试，重试次数由 `datastoreRetries` 控制。

```go
func (c ipamClient) autoAssign(ctx context.Context, num int, handleID *string, ...) (*IPAMAssignments, error) {
    for len(ia.IPs) < num {
        rem := num - len(ia.IPs)
        if maxNumBlocks > 0 && numBlocksOwned >= maxNumBlocks {
            s.allowNewClaim = false
        }
        // 找到一个可用的 Block，或者申领一个新的
        b, newlyClaimed, err := s.findOrClaimBlock(ctx, 1)

        for i := 0; i < datastoreRetries; i++ {
            newIPs, err := c.assignFromExistingBlock(ctx, b, rem, handleID, attrs, affinityCfg, config.StrictAffinity, reservations)
            if err != nil {
                // 出现竞争冲突，重试
                // ......
            }
            rem = num - len(ia.IPs)
            break
        }
    }
}
```

### 从 Block 里分配具体 IP

找到 Block 之后，具体的 IP 分配在 `autoAssign`（Block 级别）里完成。逻辑很直接：遍历 `Unallocated` 列表，挨个检查每个槽位，跳过保留地址，找到能用的就分配：

```go
func (b *allocationBlock) autoAssign(num int, handleID *string, affinityCfg AffinityConfig, attrs map[string]string, affinityCheck bool, reservations addrFilter) ([]cnet.IPNet, error) {
    _, mask, _ := cnet.ParseCIDR(b.CIDR.String())
    var ips []cnet.IPNet
    updatedUnallocated := b.Unallocated[:0]
    var attrIndexPtr *int
    for idx, ordinal := range b.Unallocated {
        // 已经够了，停止分配
        if len(ips) >= num {
            updatedUnallocated = append(updatedUnallocated, b.Unallocated[idx:]...)
            break
        }
        // 检查是不是保留地址（比如网络地址、广播地址、节点保留地址等）
        addr := b.OrdinalToIP(ordinal)
        if reservations.MatchesIP(addr) {
            log.WithField("addr", addr).Debug("Skipping reserved IP.")
            updatedUnallocated = append(updatedUnallocated, ordinal)
            continue
        }
        // 分配这个 IP
        if attrIndexPtr == nil {
            attrIndex := b.findOrAddAttribute(handleID, attrs)
            attrIndexPtr = &attrIndex
        }
        b.Allocations[ordinal] = attrIndexPtr
        ipNet := *mask
        ipNet.IP = addr.IP
        ips = append(ips, ipNet)

        b.SetSequenceNumberForOrdinal(ordinal)
        continue
    }
    b.Unallocated = updatedUnallocated
    return ips, nil
}
```

分配完之后，calico 会把修改后的 IPAMBlock 写回 datastore（etcd 或者 k8s API），同时创建/更新对应的 IPAMHandle。这两个写操作也是用乐观锁保护的，如果 resourceVersion 对不上，说明有并发修改，重头来过。

---

## 常用的 IPAM 排查命令

实际排查问题时，以下这些命令比较常用：

```bash
# 查看 IP 池
calicoctl get ippool -o wide

# 查看所有 Block 的分配情况
calicoctl get ipamblock -o wide

# 查看某个节点拥有哪些 Block
calicoctl ipam show --show-blocks

# 查看具体的 IP 分配情况（需要指定 IP 池）
calicoctl ipam show --ip=192.168.1.5

# 检查 IPAM 的整体使用率
calicoctl ipam show

# 释放一个泄漏的 IP（谨慎操作）
calicoctl ipam release --ip=192.168.1.5
```

如果遇到 IP 分配失败的问题，通常先看一下 `ipamblock` 里有没有满的 Block 堆积在某个节点，再看看 `ipamhandle` 里有没有孤儿 handle（对应的 Pod 已经删了但 handle 还在）。后者是比较常见的 IP 泄漏场景，可以通过 `calicoctl ipam check` 来检测。
