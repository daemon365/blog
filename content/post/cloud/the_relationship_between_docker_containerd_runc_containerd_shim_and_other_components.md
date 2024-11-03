---
title: "docker containerd runc containerd-shim等组件的关系"
date: "2024-05-09T20:56:00+08:00"
tags: 
- docker
- containerd
- runc
- containerd-shim
- kubernetes
- cri
showToc: true
---


## 早期 kubelet 创建容器工作原理

因为 docker 出生的比 k8s 早，所以 k8s 早期的容器运行时都是基于 docker 的，kubelet 通过 docker 的 api 创建容器。后来，k8s 官方不想绑死在 docker 这架马车上，就把容器运行时抽象出来，定义了一个接口，叫 CRI ([container runtime interface](https://github.com/kubernetes/cri-api))，容器运行时接口, 通过这个接口，kubelet 可以和任何容器运行时交互。但是，docker 并没有实现这个接口，k8s 也不想直接失去 docker 的用户，所以 k8s 官方在 kubelet 中实现了一个叫 docker-shim 的组件，这个组件简单来说就是把 cri 接口转换成 docker 的 api，这样 kubelet 就可以和 docker 交互了, 这个组件在 kuberbetes 1.24 版本中已经被移除了。至于实现了 cri 接口的容器运行时，比如 containerd，cri-o 等，kubelet 可以直接和它们交互。

调用架构图如下：

![](/images/481a4d67-401a-4e8e-af61-3534d23a7d69.png)

目前 dockershim 组件已经删除，不能使用了，所以 k8s 1.24 版本之后，kubelet 只能和实现了 cri 接口的容器运行时交互，比如 containerd，cri-o 等。

这里建议使用 containerd 因为 containerd 是 docker 官方出品的，而且 containerd 也是 docker 的核心组件，docker 的容器运行时就是基于 containerd 的，所以 containerd 的稳定性和可靠性都是有保障的。

## docker containerd runc 的关系

因为 podman 等新兴 container runtime 的崛起，docker 不想失去定义标准的机会，所以 docker 官方把 containerd 从 docker 中分离出来，独立成一个项目，实现了 cri 接口，这种 kubelet 就可以通过 cri 直接调用 containerd 了。然后，docker 官方又把 runc 从 containerd 中分离出来，独立成一个项目，定义了一个叫 OCI ([Open Container Initiative](https://opencontainers.org/)) 的标准，这个标准定义了容器的格式和运行时，runc 就是这个标准的实现，目前实现  oci 的还有 crun youki keta 等。

因为 containerd 和 runc 脱胎于 docker，docker 又不能维护两份代码，所以 docker 就通过调用 containerd ，containerd 再 通过配置实现 oci 标准的 runc 来创建容器。 当然，你也可以手动配置其他实现了 oci 标准的容器运行时。

调用架构图如下：

![](/images/d7b1939a-5608-43da-a883-42ad662e94c7.png)

在上图中可以看到 containerd 不是直接调用 runc 的，而是通过 containerd-shim 来调用 runc 的，这个是为什么？

## runc

runc 是一款设计精巧的命令行工具，专注于创建和运行符合 Open Container Initiative（OCI）规范的容器。执行 runc start 时，它首先通过 fork 创建一个子进程，在这个新进程中进行一系列容器运行的准备工作，包括准备文件系统、配置 namespaces 和 cgroups 。接着，通过 execve 系统调用，这个子进程变身为容器的首个进程——通常被称作“init”进程——并执行用户指定的首个命令（例如，bash）。

如果首个命令是一个shell（比如 bash），当执行一个shell命令（例如 ls）时，bash 会 fork 并执行相应的子进程。这个新的子进程执行 ls 命令并在完成任务后退出。此后，bash 可能继续接受新的命令，或在结束会话后终止。

当容器的“init”进程终止时，整个容器也会按照规定的生命周期走向结束。不同的命令和应用会在这个基本框架下有不同的具体行为，但总体流程大致一致。

如果这些容器的进的父进程是 containerd ，那么当 containerd 进程挂掉或者重启时，容器的进程也会挂掉，这样就不符合容器的定义了，所以 containerd 通过 containerd-shim 来调用 runc，这样当 containerd 挂掉时，容器的进程还是会继续运行的。

## containerd-shim

containerd-shim 是一个轻量级的代理进程，它的主要作用是：
1. 通过runC命令可以启动、执行容器、进程；
2. 监控容器进程状态，当容器执行完成后，通过exit fifo文件报告容器进程结束状态；
3. 当此容器SHIM的第一个实例进程被杀死后，reaper掉所有其子进程；

当 containerd 通过 containerd-shim 来调用 runc 后, 会把 containerd-shim 的挂到 system （pid=1）的进程下，这样当 containerd 挂掉或者重启时，containerd-shim 还是会继续运行的，这样就保证了容器的进程不会挂掉。

验证，这里我随便启动了一下 docker 容器看下效果：

```BASH
# 启动的nginx 容器
root       19455   19435  0 22:20 ?        00:00:00 nginx: master process nginx -g daemon off;
# nginx 进程的父进程是 containerd-shim
root       19435       1  0 22:20 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 0af95b326dfc8fee31bd28abb61e5d23a9cee98fada2b32c5ade852a0782f559 -address /run/containerd/containerd.sock
# containerd-shim 的父进程是 systemd
```