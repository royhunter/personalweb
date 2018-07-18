title: runC源码解读一
date: 2016-02-03 22:03:35
tags: Container
categories: Container
banner: http://7xoxkz.com1.z0.glb.clouddn.com/runc_4.png
---
runC作为开放容器标准组织推出的开源容器项目，该工具可以及其轻量级的创建container，通过便捷的命令行工具并对其管理和操作。从整体架构来看，其对平台依赖较小，整个project都使用了Golang进行了开发。
<!--more--> 

## 整体架构
runC的架构比较简单，从技术构成角度来看，主要是一个便捷的命令行工具+libcontainer。

![](http://7xoxkz.com1.z0.glb.clouddn.com/runc_1.PNG)

## CLI

runC的CLI是使用的[github.com/codegangsta/cli](github.com/codegangsta/cli), 该cli库也是Golang的第三方库，提供了一个易于使用的命令行接口。

```bash
$greet help
NAME:
    greet - fight the loneliness!

USAGE:
    greet [global options] command [command options] [arguments...]

VERSION:
    0.0.0

COMMANDS:
    help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS
    --version   Shows version information
```

用户可自由添加CLI中所需的commands与options并提供对应的action function即可。
```bash
app.Commands = []cli.Command{
  {
    Name:      "add",
    Aliases:     []string{"a"},
    Usage:     "add a task to the list",
    Action: func(c *cli.Context) {
      println("added task: ", c.Args().First())
    },
  },
...
```
具体API可参看[github.com/codegangsta/cli](github.com/codegangsta/cli)的官方说明文档，这里主要分析libcontainer的实现。

## libcontainer
libcontainer为container的创建提供了一种golang的实现，包括container的namespace,cgroups,capability,filesystem access controls. 可以由使用者自由管理container的生命周期，包括container的checkpoint和restore。

### Container创建流程
![](http://7xoxkz.com1.z0.glb.clouddn.com/runc_2.PNG)


1 runc start命令行执行函数首先会导入配置文件，配置文件即config.json和runtime.json.都为json格式，主要描述了运行container所需的系统配置。 配置文件可以由用户自定义或者使用#runc spec自动生成默认的配置。

2 在创建container前创建了Factory，Process，LinuxContainer等对象，这些对象主要是对container的运行环境进行了抽象的封装。

```go
// LinuxFactory implements the default factory interface for linux based systems.
type LinuxFactory struct {
	// Root directory for the factory to store state.
	Root string

	// InitPath is the absolute path to the init binary.
	InitPath string

	// InitArgs are arguments for calling the init responsibilities for spawning
	// a container.
	InitArgs []string

	// CriuPath is the path to the criu binary used for checkpoint and restore of
	// containers.
	CriuPath string

	// Validator provides validation to container configurations.
	Validator validate.Validator

	// NewCgroupsManager returns an initialized cgroups manager for a single container.
	NewCgroupsManager func(config *configs.Cgroup, paths map[string]string) cgroups.Manager
}
// Process specifies the configuration and IO for a process inside a container.
type Process struct {
	// The command to be run followed by any arguments.
	Args []string

	// Env specifies the environment variables for the process.
	Env []string

	// User will set the uid and gid of the executing process running inside the container
	// local to the container's user and group configuration.
	User string

	// Cwd will change the processes current working directory inside the container's rootfs.
	Cwd string

	// Stdin is a pointer to a reader which provides the standard input stream.
	Stdin io.Reader

	// Stdout is a pointer to a writer which receives the standard output stream.
	Stdout io.Writer

	// Stderr is a pointer to a writer which receives the standard error stream.
	Stderr io.Writer

	// ExtraFiles specifies additional open files to be inherited by the container
	ExtraFiles []*os.File

	// consolePath is the path to the console allocated to the container.
	consolePath string

	// Capabilities specify the capabilities to keep when executing the process inside the container
	// All capabilities not specified will be dropped from the processes capability mask
	Capabilities []string

	ops processOperations
}
type linuxContainer struct {
	id            string
	root          string
	config        *configs.Config
	cgroupManager cgroups.Manager
	initPath      string
	initArgs      []string
	initProcess   parentProcess
	criuPath      string
	m             sync.Mutex
	criuVersion   int
	state         containerState
}
```

3 在loadFactory()中使用了InitArgs(os.Args[0],"init")(l)来初始化container的执行入口。而l.InitPath = "/proc/self/exe"，即选择了自身的二进制作为进程创建的入口。

4 newParentProcess()创建了一个PIPE作为runc start进程与container内部init进程通信管道。同时创建一个commandTemplate作为ParentProcess运行的模板，commandTemplate使用的是GO package中的os/exec库，见[https://golang.org/pkg/os/exec/](https://golang.org/pkg/os/exec/),主要的结构为Cmd,封装了运行一个新的执行体所需要的各种环境。而namespace，filesystem及其它所需配置的运行环境信息都注册在了Cmd这个结构中。

```go
type Cmd struct {
        // Path is the path of the command to run.
        Path string

        // Args holds command line arguments, including the command as Args[0].
        Args []string

        // Env specifies the environment of the process.
        Env []string

        // Dir specifies the working directory of the command.
        Dir string

        Stdin io.Reader
        Stdout io.Writer
        Stderr io.Writer

        ExtraFiles []*os.File

        SysProcAttr *syscall.SysProcAttr

        // Process is the underlying process, once started.
        Process *os.Process

        ProcessState *os.ProcessState
}

```

5 通过parent.start()调用cmd.start()创建了新的进程。而此时新的进程使用/proc/self/exec作为执行入口，参数为"init".Go语言中的init函数比较特殊,其会在main函数调用之前调用，所以在新的进程中func init()会直接调用，而不会去执行main函数。

```go
func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" {
		runtime.GOMAXPROCS(1)
		runtime.LockOSThread()
		factory, _ := libcontainer.New("")
		if err := factory.StartInitialization(); err != nil {
			fatal(err)
		}
		panic("--this line should have never been executed, congratulations--")
	}
}
```

6 init()会从之前创建的pipe里读取父进程同步过来的配置进行初始化，初始化完成后会sync一个Ready的状态到父进程，父进程收到后同步一个此时子进程已经是在容器环境中了，最后container会调用system.Exec()执行用户在json文件中配置的binary。

7 linuxStandardInit.init()是container内部进行初始化的主要函数,根据runc start进程发来的config进行初始化

```bash
setupNetwork
setupRoute
setupRlimits
setOomScoreAdj
setupRootfs
Sethostname
ApplyProfile
SetProcessLabel
remountReadonly
maskFile
```

这样一个完整的container初始化完成并创建成功。后续我们再看libcontainer是如何对各个子模块进行初始化和配置的。
