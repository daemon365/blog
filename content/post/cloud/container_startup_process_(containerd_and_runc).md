---
title: "容器启动流程（containerd 和 runc）"
date: "2023-12-07T00:00:00+08:00"
tags: 
- docker
- containerd
- runc
- containerd-shim
- kubernetes
- 源码分析
showToc: true
---


## 启动流程

containerd 作为一个 api 服务，提供了一系列的接口供外部调用，比如创建容器、删除容器、创建镜像、删除镜像等等。使用 docker 和 ctr 等工具，都是通过调用 containerd 的 api 来实现的。
kubelet 通过 cri 调用 containerd 和这些不一样，后续我会介绍到。

containerd 创建容器流程如下：

1. 接收到 api 请求，通过调用 containerd-shim-runc-v2 调用 runc 创建容器，主要是做解压文件和准备环境的工作。
2. 接收到 api 请求，创建一个 task，task 是一个容器的抽象，包含了容器的所有信息，比如容器的 id、容器的状态、容器的配置等等。
3. containerd 启动一个 containerd-shim-runc-v2 进程。
4. containerd-shim-runc-v2 进程 在启动一个 containerd-shim-runc-v2 进程，然后第一个 containerd-shim-runc-v2 进程退出。
5. containerd 通过 IPC 通信，让第二个 containerd-shim-runc-v2 启动容器。
6. containerd-shim-runc-v2 进程通过调用 runc start 启动容器。
7. runc 会调用 runc init 启动容器的 init 进程。
8. runc init 进程会调用 `unix.Exec` 的方式，替换自己的进程，启动容器的第一个进程。这个进程既是容器的启动命令，也是容器的 pid 1 进程。完成之后，runc create 进程退出。

这样 containerd-shim-runc-v2 的父进程就是 init 进程（1），而 init 进程的父进程是 containerd-shim-runc-v2 进程，这样就形成了一个进程树。

我通过 docker 启动一个容器，示例一下：

```bash
❯ docker run -d --rm -it docker.m.daocloud.io/ubuntu:22.10 sleep 3000
❯ ps -ef|grep "sleep 3000"
root       15042   15021  0 22:02 pts/0    00:00:00 sleep 3000
❯ ps -ef|grep "15021"
root       15021       1  0 22:02 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 4346ca602cd85d35b0a4a81762be6142bc6a2222f859f4af47563992efc3c59c -address /run/containerd/containerd.sock
root       15042   15021  0 22:02 pts/0    00:00:00 sleep 3000
```

可以看到我们的结论是正确的。


### 疑问解答

**1.为什么要创建两个 containerd-shim 不嫌麻烦吗？**

因为 第一个 containerd-shim 会在创建完第二个 containerd-shim 后退出，而作为第一个进程子进程的第二个 containerd-shim 会成为孤儿进程，这样就会被 init 进程接管，而和 containerd 本身脱离了关系。

**2.为什么要想法设法把 containerd-shim 挂在 init 进程下面，而不是 containerd？**

为了保证稳定性和独立性。这样做可以确保即使 containerd 崩溃或重启，由 containerd-shim 管理的容器进程仍然可以继续运行，不受影响。此外，这种设计还有助于更好地管理资源和防止资源泄露。

**3.为什么 runc start 进程退出了 runc init 进程（用户进程）没有变成 init 的子进程 而是containerd-shim的子进程？**

因为 containerd-shim 做了 unix 的 `PR_SET_CHILD_SUBREAPER` 调用, 这个系统调用大概作用为 当这个进程的子子孙孙进程变成孤儿进程的时候，这个进程会接管这个孤儿进程，而不是 init 进程接管。

## 架构图

![](/images/1a309094-015a-4bcf-b371-134a5a275089.png)

## 代码分析

### containerd api 注册 代码分析


```GO
var register = struct {
	sync.RWMutex
	r plugin.Registry
}{}

type Registry []*Registration

type Registration struct {
	// Type of the plugin
	Type Type
	// ID of the plugin
	ID string
	// Config specific to the plugin
	Config interface{}
	// Requires is a list of plugins that the registered plugin requires to be available
	Requires []Type

	// InitFn is called when initializing a plugin. The registration and
	// context are passed in. The init function may modify the registration to
	// add exports, capabilities and platform support declarations.
	InitFn func(*InitContext) (interface{}, error)

	// ConfigMigration allows a plugin to migrate configurations from an older
	// version to handle plugin renames or moving of features from one plugin
	// to another in a later version.
	// The configuration map is keyed off the plugin name and the value
	// is the configuration for that objects, with the structure defined
	// for the plugin. No validation is done on the value before performing
	// the migration.
	ConfigMigration func(context.Context, int, map[string]interface{}) error
}
```

通过 init 把接口注册进去 比如 task api 注册

代码位置 ： `services/tasks/local.go`
```GO
func init() {
	registry.Register(&plugin.Registration{
		Type:     plugins.ServicePlugin,
		ID:       services.TasksService,
		Requires: tasksServiceRequires,
		Config:   &Config{},
		InitFn:   initFunc,
	})

	timeout.Set(stateTimeout, 2*time.Second)
}

func initFunc(ic *plugin.InitContext) (interface{}, error) {
	config := ic.Config.(*Config)

	v2r, err := ic.GetByID(plugins.RuntimePluginV2, "task")
	if err != nil {
		return nil, err
	}

	m, err := ic.GetSingle(plugins.MetadataPlugin)
	if err != nil {
		return nil, err
	}

	ep, err := ic.GetSingle(plugins.EventPlugin)
	if err != nil {
		return nil, err
	}

	monitor, err := ic.GetSingle(plugins.TaskMonitorPlugin)
	if err != nil {
		if !errors.Is(err, plugin.ErrPluginNotFound) {
			return nil, err
		}
		monitor = runtime.NewNoopMonitor()
	}

	db := m.(*metadata.DB)
	l := &local{
		containers: metadata.NewContainerStore(db),
		store:      db.ContentStore(),
		publisher:  ep.(events.Publisher),
		monitor:    monitor.(runtime.TaskMonitor),
		v2Runtime:  v2r.(runtime.PlatformRuntime),
	}

	v2Tasks, err := l.v2Runtime.Tasks(ic.Context, true)
	if err != nil {
		return nil, err
	}
	for _, t := range v2Tasks {
		l.monitor.Monitor(t, nil)
	}

	if err := blockio.SetConfig(config.BlockIOConfigFile); err != nil {
		log.G(ic.Context).WithError(err).Errorf("blockio initialization failed")
	}
	if err := rdt.SetConfig(config.RdtConfigFile); err != nil {
		log.G(ic.Context).WithError(err).Errorf("RDT initialization failed")
	}

	return l, nil
}
```

然后在 containerd 启动的时候 注册api

```GO
loaded := registry.Graph(filter(config.DisabledPlugins))

for _, p := range loaded {
		result := p.Init(initContext)
		if err := initialized.Add(result); err != nil {
			return nil, fmt.Errorf("could not add plugin result to plugin set: %w", err)
		}

		instance, err := result.Instance()

		delete(required, id)
		// check for grpc services that should be registered with the server
		if src, ok := instance.(grpcService); ok {
			grpcServices = append(grpcServices, src)
		}
		if src, ok := instance.(ttrpcService); ok {
			ttrpcServices = append(ttrpcServices, src)
		}
		if service, ok := instance.(tcpService); ok {
			tcpServices = append(tcpServices, service)
		}

		s.plugins = append(s.plugins, result)
	}

	// register services after all plugins have been initialized
	for _, service := range grpcServices {
		if err := service.Register(grpcServer); err != nil {
			return nil, err
		}
	}
	for _, service := range ttrpcServices {
		if err := service.RegisterTTRPC(ttrpcServer); err != nil {
			return nil, err
		}
	}
	for _, service := range tcpServices {
		if err := service.RegisterTCP(tcpServer); err != nil {
			return nil, err
		}
	}
```

### create task


```go
func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {

	rtime := l.v2Runtime

	_, err = rtime.Get(ctx, r.ContainerID)
	if err != nil && !errdefs.IsNotFound(err) {
		return nil, errdefs.ToGRPC(err)
	}
	if err == nil {
		return nil, errdefs.ToGRPC(fmt.Errorf("task %s: %w", r.ContainerID, errdefs.ErrAlreadyExists))
	}
	c, err := rtime.Create(ctx, r.ContainerID, opts)

}

func (m *TaskManager) Create(ctx context.Context, taskID string, opts runtime.CreateOpts) (runtime.Task, error) {
	

	// 启动第一个 containerd-shim-runc-v2 进程
	shimTask, err := newShimTask(shim)
	if err != nil {
		return nil, err
	}
    // 给第二个 containerd-shim-runc-v2 进程传递参数
	t, err := shimTask.Create(ctx, opts)
	

	return t, nil
}

```

### cri 代码

cri 和 task 和上述的是一样的， 通过 register 注册 api.

```go
func init() {

	registry.Register(&plugin.Registration{
		Type: plugins.GRPCPlugin,
		ID:   "cri",
		Requires: []plugin.Type{
			plugins.CRIImagePlugin,
			plugins.InternalPlugin,
			plugins.SandboxControllerPlugin,
			plugins.NRIApiPlugin,
			plugins.EventPlugin,
			plugins.ServicePlugin,
			plugins.LeasePlugin,
			plugins.SandboxStorePlugin,
		},
		InitFn: initCRIService,
	})
}

```

startContainer 接口

```GO
func (in *instrumentedService) StartContainer(ctx context.Context, r *runtime.StartContainerRequest) (_ *runtime.StartContainerResponse, err error) {
	if err := in.checkInitialized(); err != nil {
		return nil, err
	}
	log.G(ctx).Infof("StartContainer for %q", r.GetContainerId())
	defer func() {
		if err != nil {
			log.G(ctx).WithError(err).Errorf("StartContainer for %q failed", r.GetContainerId())
		} else {
			log.G(ctx).Infof("StartContainer for %q returns successfully", r.GetContainerId())
		}
	}()
	res, err := in.c.StartContainer(ctrdutil.WithNamespace(ctx), r)
	return res, errdefs.ToGRPC(err)
}

func (c *criService) StartContainer(ctx context.Context, r *runtime.StartContainerRequest) (retRes *runtime.StartContainerResponse, retErr error) {
task, err := container.NewTask(ctx, ioCreation, taskOpts...)
}

func (c *container) NewTask(ctx context.Context, ioCreate cio.Creator, opts ...NewTaskOpts) (_ Task, err error) {
	// 通过 unix socket 的方式调用上述的 create task 接口
	response, err := c.client.TaskService().Create(ctx, request)
}

```

### containerd-shim

```go
func run(ctx context.Context, manager Manager, config Config) error {
	
	// Handle explicit actions
	switch action {
	case "delete":
	case "start":
    // 如果是 start 参数的话启动一个 containerd-shim-runc-v2 进程
		opts := StartOpts{
			Address:      addressFlag,
			TTRPCAddress: ttrpcAddress,
			Debug:        debugFlag,
		}

		params, err := manager.Start(ctx, id, opts)
		if err != nil {
			return err
		}

		data, err := json.Marshal(&params)
		if err != nil {
			return fmt.Errorf("failed to marshal bootstrap params to json: %w", err)
		}

		if _, err := os.Stdout.Write(data); err != nil {
			return err
		}

		return nil
	}


}

// manager.Start 中创建的 command 指定三个参数 Namespace 容器 id 和 containerd socket 文件的地址
func newCommand(ctx context.Context, id, containerdAddress, containerdTTRPCAddress string, debug bool) (*exec.Cmd, error) {
	ns, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return nil, err
	}
	self, err := os.Executable()
	if err != nil {
		return nil, err
	}
	cwd, err := os.Getwd()
	if err != nil {
		return nil, err
	}
	args := []string{
		"-namespace", ns,
		"-id", id,
		"-address", containerdAddress,
	}
	if debug {
		args = append(args, "-debug")
	}
	cmd := exec.Command(self, args...)
	cmd.Dir = cwd
	cmd.Env = append(os.Environ(), "GOMAXPROCS=4")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
	}
	return cmd, nil
}

```

第二个 containerd-shim 也会开启一些api服务 ，比如启动容器

```GO
func (s *service) Start(ctx context.Context, r *taskAPI.StartRequest) (*taskAPI.StartResponse, error) {
    p, err := container.Start(ctx, r)
}

func (c *Container) Start(ctx context.Context, r *task.StartRequest) (process.Process, error) {
	p, err := c.Start(r.ExecID)
}

// command 及调用 runc 启动 runc create 的进程
func (r *Runc) Start(context context.Context, id string) error {
	return r.runOrError(r.command(context, "start", id))
}
```

### runc 

```go
func (r *runner) run(config *specs.Process) (int, error) {
    switch r.action {
	case CT_ACT_CREATE:
		err = r.container.Start(process)
	case CT_ACT_RESTORE:
		err = r.container.Restore(process, r.criuOpts)
	case CT_ACT_RUN:
		err = r.container.Run(process)
	default:
		panic("Unknown action")
	}
}

func (c *Container) Start(process *Process) error {
	c.m.Lock()
	defer c.m.Unlock()
	if c.config.Cgroups.Resources.SkipDevices {
		return errors.New("can't start container with SkipDevices set")
	}
	if process.Init {
		if err := c.createExecFifo(); err != nil {
			return err
		}
	}
	if err := c.start(process); err != nil {
		if process.Init {
			c.deleteExecFifo()
		}
		return err
	}
	return nil
}

// 调用 runc init 进程 /proc/self/exe 是自己的二进制文件
func (c *Container) newParentProcess(p *Process) (parentProcess, error) {
	comm, err := newProcessComm()
	if err != nil {
		return nil, err
	}

	// Make sure we use a new safe copy of /proc/self/exe or the runc-dmz
	// binary each time this is called, to make sure that if a container
	// manages to overwrite the file it cannot affect other containers on the
	// system. For runc, this code will only ever be called once, but
	// libcontainer users might call this more than once.
	p.closeClonedExes()
	var (
		exePath string
		// only one of dmzExe or safeExe are used at a time
		dmzExe, safeExe *os.File
	)
	if dmz.IsSelfExeCloned() {
		// /proc/self/exe is already a cloned binary -- no need to do anything
		logrus.Debug("skipping binary cloning -- /proc/self/exe is already cloned!")
		exePath = "/proc/self/exe"
	} 

	cmd := exec.Command(exePath, "init")
	cmd.Args[0] = os.Args[0]
	cmd.Stdin = p.Stdin
	cmd.Stdout = p.Stdout
	cmd.Stderr = p.Stderr
	cmd.Dir = c.config.Rootfs
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &unix.SysProcAttr{}
	}
	
}

```

runc init


```go
func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" {
		// This is the golang entry point for runc init, executed
		// before main() but after libcontainer/nsenter's nsexec().
		libcontainer.Init()
	}
}

// libcontainer.Init() 中调用的
func startInitialization() (retErr error) {
    return containerInit(it, &config, syncPipe, consoleSocket, pidfdSocket, fifofd, logFD, dmzExe, mountFds{sourceFds: mountSrcFds, idmapFds: idmapFds})
}

// linuxSetnsInit 是 exec 的时候调用的 在启动的容器执行命令
// initStandard 是启动容器
func containerInit(t initType, config *initConfig, pipe *syncSocket, consoleSocket, pidfdSocket *os.File, fifoFd, logFd int, dmzExe *os.File, mountFds mountFds) error {
	if err := populateProcessEnvironment(config.Env); err != nil {
		return err
	}

	switch t {
	case initSetns:
		// mount and idmap fds must be nil in this case. We don't mount while doing runc exec.
		if mountFds.sourceFds != nil || mountFds.idmapFds != nil {
			return errors.New("mount and idmap fds must be nil; can't mount from exec")
		}

		i := &linuxSetnsInit{
			pipe:          pipe,
			consoleSocket: consoleSocket,
			pidfdSocket:   pidfdSocket,
			config:        config,
			logFd:         logFd,
			dmzExe:        dmzExe,
		}
		return i.Init()
	case initStandard:
		i := &linuxStandardInit{
			pipe:          pipe,
			consoleSocket: consoleSocket,
			pidfdSocket:   pidfdSocket,
			parentPid:     unix.Getppid(),
			config:        config,
			fifoFd:        fifoFd,
			logFd:         logFd,
			dmzExe:        dmzExe,
			mountFds:      mountFds,
		}
		return i.Init()
	}
	return fmt.Errorf("unknown init type %q", t)
}


func (l *linuxStandardInit) Init() error {
    return system.Exec(name, l.config.Args, os.Environ())
}

// 替换进程
func Exec(cmd string, args []string, env []string) error {
	for {
		err := unix.Exec(cmd, args, env)
		if err != unix.EINTR {
			return &os.PathError{Op: "exec", Path: cmd, Err: err}
		}
	}
}
```