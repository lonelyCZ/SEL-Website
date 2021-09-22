+++
id = "158"

title = "Docker源码分析（三）：Docker Daemon启动"
description = "【Docker源码分析（三）：Docker Daemon启动】Docker作为业界备受推崇的轻量级虚拟化容器管理引擎，其强大的后台能力全凭Docker Daemon。本文从源码出发，介绍了Docker Daemon的启动流程，并深入分析启动流程中每一步骤的实现细节。Docker的运行真可谓，载体为daemon，调度管理由engine，任务执行靠job。"
tags = ["Docker"]
date = "2014-12-02 13:02:44"
author = "孙宏亮"
banner = "img/blogs/158/docker.jpg"
categories = ["Docker"]

+++

# 【摘要】

Docker作为业界备受推崇的轻量级虚拟化容器管理引擎，其强大的后台能力全凭Docker Daemon。本文从源码出发，介绍了Docker Daemon的启动流程，并深入分析启动流程中每一步骤的实现细节。Docker的运行真可谓，载体为daemon，调度管理由engine，任务执行靠job。

<!--more-->

**1 前言**
========

Docker诞生以来，便引领了轻量级虚拟化容器领域的技术热潮。在这一潮流下，Google、IBM、Redhat等业界翘楚纷纷加入Docker阵营。虽然目前Docker仍然主要基于Linux平台，但是Microsoft却多次宣布对Docker的支持，从先前宣布的Azure支持Docker与Kubernetes，到如今宣布的下一代Windows Server原生态支持Docker。Microsoft的这一系列举措多少喻示着向Linux世界的妥协，当然这也不得不让世人对Docker的巨大影响力有重新的认识。 

Docker的影响力不言而喻，但如果需要深入学习Docker的内部实现，笔者认为最重要的是理解Docker Daemon。在Docker架构中，Docker Client通过特定的协议与Docker Daemon进行通信，而Docker Daemon主要承载了Docker运行过程中的大部分工作。本文即为《Docker源码分析》系列的第三篇­——Docker Daemon篇。

**2 Docker Daemon简介**
=====================

Docker Daemon是Docker架构中运行在后台的守护进程，大致可以分为Docker Server、Engine和Job三部分。Docker Daemon可以认为是通过Docker Server模块接受Docker Client的请求，并在Engine中处理请求，然后根据请求类型，创建出指定的Job并运行，运行过程的作用有以下几种可能：向Docker Registry获取镜像，通过graphdriver执行容器镜像的本地化操作，通过networkdriver执行容器网络环境的配置，通过execdriver执行容器内部运行的执行工作等。 

以下为Docker Daemon的架构示意图：
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605752257/sel/docker-3-1_wdck6w.jpg" alt="docker-3-1" style="zoom:50%;" />
</center>


**3 Docker Daemon源码分析内容安排**
===========================

本文从源码的角度，主要分析Docker Daemon的启动流程。由于Docker Daemon和Docker Client的启动流程有很大的相似之处，故在介绍启动流程之后，本文着重分析启动流程中最为重要的环节：创建daemon过程中mainDaemon()的实现。

**4 Docker Daemon的启动流程**
========================

由于Docker Daemon和Docker Client的启动都是通过可执行文件docker来完成的，因此两者的启动流程非常相似。Docker可执行文件运行时，运行代码通过不同的命令行flag参数，区分两者，并最终运行两者各自相应的部分。 

启动Docker Daemon时，一般可以使用以下命令：docker --daemon=true; docker –d; docker –d=true等。接着由docker的main()函数来解析以上命令的相应flag参数，并最终完成Docker Daemon的启动。 首先，附上Docker Daemon的启动流程图：
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605752754/sel/docker-3-2_l6tpa9.jpg" alt="docker-3-2" style="zoom:33%;" />
</center>

由于《Docker源码分析》系列之[Docker Client篇](http://www.sel.zju.edu.cn/?p=147)中，已经涉及了关于Docker中main()函数运行的很多前续工作（可参见[Docker Client篇](http://www.sel.zju.edu.cn/?p=147)），并且Docker Daemon的启动也会涉及这些工作，故本文略去相同部分，而主要针对后续仅和Docker Daemon相关的内容进行深入分析，即mainDaemon()的具体源码实现。

**5 mainDaemon( )的具体实现**
========================

通过Docker Daemon的流程图，可以得出一个这样的结论：有关Docker Daemon的所有的工作，都被包含在mainDaemon()方法的实现中。 宏观来讲，mainDaemon()完成创建一个daemon进程，并使其正常运行。 

从功能的角度来说，mainDaemon()实现了两部分内容：第一，创建Docker运行环境；第二，服务于Docker Client，接收并处理相应请求。 

从实现细节来讲，[mainDaemon()的实现过程](https://github.com/docker/docker/blob/v1.2.0/docker/daemon.go#L28)主要包含以下步骤：

*   daemon的配置初始化（这部分在init()函数中实现，即在mainDaemon()运行前就执行，但由于这部分内容和mainDaemon()的运行息息相关，故可认为是mainDaemon()运行的先决条件）；
*   命令行flag参数检查；
*   创建engine对象；
*   设置engine的信号捕获及处理方法；
*   加载builtins；
*   使用goroutine加载daemon对象并运行；
*   打印Docker版本及驱动信息；
*   Job之”serveapi”的创建与运行。

下文将一一深入分析以上步骤。

**5.0 配置初始化**
-------------

在mainDaemon()运行之前，关于Docker Daemon所需要的config配置信息均已经初始化完毕。具体实现如下，位于[./docker/docker/daemon.go](https://github.com/docker/docker/blob/master/docker/daemon.go#L21)：

```go
var (
    daemonCfg = &daemon.Config{}
)
func init() {
    daemonCfg.InstallFlags()
}
```

首先，声明一个为daemon包中Config类型的变量，名为daemonCfg。而Config对象，定义了Docker Daemon所需的配置信息。在Docker Daemon在启动时，daemonCfg变量被传递至Docker Daemon并被使用。 

Config对象的定义如下（含部分属性的解释），位于[./docker/daemon/config.go](https://github.com/docker/docker/blob/v1.2.0/daemon/config.go#L20)：

```go
    type Config struct {
        Pidfile                  string   //Docker Daemon所属进程的PID文件
        Root                   string   //Docker运行时所使用的root路径
        AutoRestart             bool    //已被启用，转而支持docker run时的重启
        Dns                   []string  //Docker使用的DNS Server地
        DnsSearch              []string  //Docker使用的指定的DNS查找域名
        Mirrors                 []string  //指定的优先Docker Registry镜像
        EnableIptables           bool    //启用Docker的iptables功能
        EnableIpForward         bool    //启用net.ipv4.ip_forward功能
        EnableIpMasq            bool      //启用IP伪装技术
        DefaultIp                net.IP     //绑定容器端口时使用的默认IP
        BridgeIface              string      //添加容器网络至已有的网桥
        BridgeIP                 string     //创建网桥的IP地址
        FixedCIDR               string     //指定IP的IPv4子网，必须被网桥子网包含
        InterContainerCommunication   bool  //是否允许相同host上容器间的通信
        GraphDriver             string      //Docker运行时使用的特定存储驱动
        GraphOptions            []string   //可设置的存储驱动选项
        ExecDriver               string    // Docker运行时使用的特定exec驱动
        Mtu                    int      //设置容器网络的MTU
        DisableNetwork          bool     //有定义，之后未初始化
        EnableSelinuxSupport      bool     //启用SELinux功能的支持
        Context                 map[string][]string   //有定义，之后未初始化
    }
```


已经有声明的daemonCfg之后，init()函数实现了daemonCfg变量中各属性的赋值，具体的实现为：daemonCfg.InstallFlags()，位于[./docker/daemon/config.go](https://github.com/docker/docker/blob/v1.2.0/daemon/config.go#L45)，代码如下：

```go
    func (config *Config) InstallFlags() {
        flag.StringVar(&config.Pidfile, []string{"p", "-pidfile"}, "/var/run/docker.pid", "Path to use for daemon PID file")
        flag.StringVar(&config.Root, []string{"g", "-graph"}, "/var/lib/docker", "Path to use as the root of the Docker runtime")
        ……
        opts.IPVar(&config.DefaultIp, []string{"#ip", "-ip"}, "0.0.0.0", "Default IP address to use when binding container ports")
        opts.ListVar(&config.GraphOptions, []string{"-storage-opt"}, "Set storage driver options")
        ……
    }
```


在InstallFlags()函数的实现过程中，主要是定义某种类型的flag参数，并将该参数的值绑定在config变量的指定属性上，如：

```go
    flag.StringVar(&config.Pidfile, []string{"p", "-pidfile"}, "/var/run/docker.pid", "Path to use for daemon PID file"
```


以上语句的含义为：

*   定义一个为String类型的flag参数；
*   该flag的名称为”p”或者”-pidfile”;
*   该flag的值为” /var/run/docker.pid”,并将该值绑定在变量config.Pidfile上；
*   该flag的描述信息为"Path to use for daemon PID file"。

至此，关于Docker Daemon所需要的配置信息均声明并初始化完毕。

**5.1 flag参数检查**
----------------

从这一节开始，真正进入Docker Daemon的mainDaemon()运行分析。 

第一个步骤即flag参数的检查。具体而言，即当docker命令经过flag参数解析之后，判断剩余的参数是否为0。若为0，则说明Docker Daemon的启动命令无误，正常运行；若不为0，则说明在启动Docker Daemon的时候，传入了多余的参数，此时会输出错误提示，并退出运行程序。具体代码如下：

```go
if flag.NArg() != 0 {
    flag.Usage()
    return
}
```


**5.2 创建engine对象**
------------------

在mainDaemon()运行过程中，flag参数检查完毕之后，随即创建engine对象，代码如下：

```go
eng := engine.New()
```

Engine是Docker架构中的运行引擎，同时也是Docker运行的核心模块。Engine扮演着Docker container存储仓库的角色，并且通过job的形式来管理这些容器。 

在[./docker/engine/engine.go](https://github.com/docker/docker/blob/v1.2.0/engine/engine.go#L47)中,Engine结构体的定义如下：

```go
    type Engine struct {
        handlers   map[string]Handler
        catchall   Handler
        hack       Hack // data for temporary hackery (see hack.go)
        id         string
        Stdout     io.Writer
        Stderr     io.Writer
        Stdi    n      io.Reader
        Logging    bool
        tasks      sync.WaitGroup
        l          sync.RWMutex // lock for shutdown
        shutdown   bool
        onShutdown []func() // shutdown handlers
    }
```


其中，Engine结构体中最为重要的即为handlers属性。该handlers属性为map类型，key为string类型，value为Handler类型。其中Handler类型的定义如下：

```go
 type Handler func(*Job) Status
```

可见，Handler为一个定义的函数。该函数传入的参数为Job指针，返回为Status状态。 

介绍完Engine以及Handler，现在真正进入New()函数的实现中：

```go
    func New() *Engine {
        eng := &Engine{
            handlers: make(map[string]Handler),
            id:       utils.RandomString(),
            Stdout:   os.Stdout,
            Stderr:   os.Stderr,
            Stdin:    os.Stdin,
            Logging:  true,
        }
        eng.Register("commands", func(job *Job) Status {
            for _, name := range eng.commands() {
                job.Printf("%s\n", name)
            }
            return StatusOK
        })
        // Copy existing global handlers
        for k, v := range globalHandlers {
            eng.handlers[k] = v
        }
        return eng
    }
```


分析以上代码，可以知道New()函数最终返回一个Engine对象。而在代码实现部分，第一个工作即为创建一个Engine结构体实例eng；第二个工作是向eng对象注册名为commands的Handler，其中Handler为临时定义的函数func(job \*Job) Status{ } , 该函数的作用是通过job来打印所有已经注册完毕的command名称，最终返回状态StatusOK；第三个工作是：将已定义的变量globalHandlers中的所有的Handler，都复制到eng对象的handlers属性中。最后成功返回eng对象。

**5.3 设置engine的信号捕获**
---------------------

回到mainDaemon()函数的运行中，执行后续代码：

```bash
signal.Trap(eng.Shutdown)
```

该部分代码的作用是：在Docker Daemon的运行中，设置Trap特定信号的处理方法，特定信号有SIGINT，SIGTERM以及SIGQUIT；当程序捕获到SIGINT或者SIGTERM信号时，执行相应的善后操作，最后保证Docker Daemon程序退出。 

该部分的代码的实现位于[./docker/pkg/signal/trap.go](https://github.com/docker/docker/blob/v1.2.0/pkg/signal/trap.go#L20)。实现的流程分为以下4个步骤：

*   创建并设置一个channel，用于发送信号通知；
*   定义signals数组变量，初始值为os.SIGINT, os.SIGTERM;若环境变量DEBUG为空的话，则添加os.SIGQUIT至signals数组；
*   通过gosignal.Notify(c, signals...)中Notify函数来实现将接收到的signal信号传递给c。需要注意的是只有signals中被罗列出的信号才会被传递给c，其余信号会被直接忽略；
*   创建一个goroutine来处理具体的signal信号，当信号类型为os.Interrupt或者syscall.SIGTERM时，执行传入Trap函数的具体执行方法，形参为cleanup(),实参为eng.Shutdown。

Shutdown()函数的定义位于[./docker/engine/engine.go](https://github.com/docker/docker/blob/v1.2.0/engine/engine.go#L153)，主要做的工作是为Docker Daemon的关闭做一些善后工作。 

善后工作如下：

*   Docker Daemon不再接收任何新的Job；
*   Docker Daemon等待所有存活的Job执行完毕；
*   Docker Daemon调用所有shutdown的处理方法；
*   当所有的handler执行完毕，或者15秒之后，Shutdown()函数返回。

由于在signal.Trap( eng.Shutdown )函数的具体实现中执行eng.Shutdown，在执行完eng.Shutdown之后，随即执行[os.Exit(0)](https://github.com/docker/docker/blob/v1.2.0/pkg/signal/trap.go#L41)，完成当前程序的立即退出。

**5.4 加载builtins**
------------------

为eng设置完Trap特定信号的处理方法之后，Docker Daemon实现了builtins的加载。代码实现如下：

```go
if err := builtins.Register(eng); err != nil {
    log.Fatal(err)
}
```


加载builtins的主要工作是为：为engine注册多个Handler，以便后续在执行相应任务时，运行指定的Handler。这些Handler包括：网络初始化、web API服务、事件查询、版本查看、Docker Registry验证与搜索。代码实现位于[./docker/builtins/builtins.go](https://github.com/docker/docker/blob/v1.2.0/builtins/builtins.go#L16),如下：

```go
    func Register(eng *engine.Engine) error {
        if err := daemon(eng); err != nil {
            return err
        }
        if err := remote(eng); err != nil {
            return err
        }
        if err := events.New().Install(eng); err != nil {
            return err
        }
        if err := eng.Register("version", dockerVersion); err != nil {
            return err
        }
        return registry.NewService().Install(eng)
    }
```


以下分析实现过程中最为主要的5个部分：daemon(eng)、remote(eng)、events.New().Install(eng)、eng.Register(“version”,dockerVersion)以及registry.NewService().Install(eng)。

### **5.4.1 注册初始化网络驱动的Handler**

daemon(eng)的实现过程，主要为eng对象注册了一个key为”init\_networkdriver”的Handler，该Handler的值为bridge.InitDriver函数，代码如下：

```go
    func daemon(eng *engine.Engine) error {
        return eng.Register("init_networkdriver", bridge.InitDriver)
    }    
```


需要注意的是，向eng对象注册Handler，并不代表Handler的值函数会被直接运行，如bridge.InitDriver，并不会直接运行，而是将bridge.InitDriver的函数入口，写入eng的handlers属性中。 Bridge.InitDriver的具体实现位于[./docker/daemon/networkdriver/bridge/driver.go](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L79) ，主要作用为：

*   获取为Docker服务的网络设备的地址； \*S 创建指定IP地址的网桥；
*   配置网络iptables规则；
*   另外还为eng对象注册了多个Handler,如 ”allocate\_interface”， ”release\_interface”， ”allocate\_port”，”link”。

### **5.4.2 注册API服务的Handler**

remote(eng)的实现过程，主要为eng对象注册了两个Handler，分别为”serveapi”与”acceptconnections”。代码实现如下：

```go
      func remote(eng *engine.Engine) error { if err := eng.Register("serveapi", apiserver.ServeApi); err != nil { return err } return eng.Register("acceptconnections", apiserver.AcceptConnections) }
```

注册的两个Handler名称分别为”serveapi”与”acceptconnections”,相应的执行方法分别为apiserver.ServeApi与apiserver.AcceptConnections，具体实现位于[./docker/api/server/server.go](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go)。其中，ServeApi执行时，通过循环多种协议，创建出goroutine来配置指定的http.Server，最终为不同的协议请求服务；而AcceptConnections的实现主要是为了通知init守护进程，Docker Daemon已经启动完毕，可以让Docker Daemon进程接受请求。

### **5.4.3 注册events事件的Handler**

events.New().Install(eng)的实现过程，为Docker注册了多个event事件，功能是给Docker用户提供API，使得用户可以通过这些API查看Docker内部的events信息，log信息以及subscribers\_count信息。具体的代码位于[./docker/events/events.go](https://github.com/docker/docker/blob/v1.2.0/events/events.go#L29)，如下：

```go
    func (e *Events) Install(eng *engine.Engine) error {
        jobs := map[string]engine.Handler{
            "events":            e.Get,
            "log":               e.Log,
            "subscribers_count": e.SubscribersCount,
        }
        for name, job := range jobs {
            if err := eng.Register(name, job); err != nil {
                return err
            }
        }
        return nil
    }
```


### **5.4.4 注册版本的Handler**

eng.Register(“version”,dockerVersion)的实现过程，向eng对象注册key为”version”，value为”dockerVersion”执行方法的Handler，dockerVersion的执行过程中，会向名为version的job的标准输出中写入Docker的版本，Docker API的版本，git版本，Go语言运行时版本以及操作系统等版本信息。dockerVersion的具体实现如下：

```go
    func dockerVersion(job *engine.Job) engine.Status {
        v := &engine.Env{}
        v.SetJson("Version", dockerversion.VERSION)
        v.SetJson("ApiVersion", api.APIVERSION)
        v.Set("GitCommit", dockerversion.GITCOMMIT)
        v.Set("GoVersion", runtime.Version())
        v.Set("Os", runtime.GOOS)
        v.Set("Arch", runtime.GOARCH)
        if kernelVersion, err := kernel.GetKernelVersion(); err == nil {
            v.Set("KernelVersion", kernelVersion.String())
        }
        if _, err := v.WriteTo(job.Stdout); err != nil {
            return job.Error(err)
        }
        return engine.StatusOK
    }
```


### **5.4.5 注册registry的Handler**

registry.NewService().Install(eng)的实现过程位于[./docker/registry/service.go](https://github.com/docker/docker/blob/v1.2.0/registry/service.go#L25)，在eng对象对外暴露的API信息中添加docker registry的信息。当registry.NewService()成功被Install安装完毕的话，则有两个调用能够被eng使用：”auth”，向公有registry进行认证；”search”，在公有registry上搜索指定的镜像。 Install的具体实现如下：

```go
    func (s *Service) Install(eng *engine.Engine) error {
        eng.Register("auth", s.Auth)
        eng.Register("search", s.Search)
        return nil
    }
```


至此，所有builtins的加载全部完成，实现了向eng对象注册特定的Handler。

**5.5 使用goroutine加载daemon对象并运行**
--------------------------------

执行完builtins的加载，回到mainDaemon()的执行，通过一个goroutine来加载daemon对象并开始运行。这一环节的执行，主要包含三个步骤： 

通过init函数中初始化的daemonCfg与eng对象来创建一个daemon对象d； 通过daemon对象的Install函数，向eng对象中注册众多的Handler； 在Docker Daemon启动完毕之后，运行名为”acceptconnections”的job，主要工作为向init守护进程发送”READY=1”信号，以便开始正常接受请求。 代码实现如下：

```go
go func() {
    d, err := daemon.MainDaemon(daemonCfg, eng)
    if err != nil {
        log.Fatal(err)
    }
    if err := d.Install(eng); err != nil {
        log.Fatal(err)
    }
    if err := eng.Job("acceptconnections").Run(); err != nil {
        log.Fatal(err)
    }
}()
```


以下分别分析三个步骤所做的工作。

### **5.5.1 创建daemon对象**

daemon.MainDaemon(daemonCfg, eng)是创建daemon对象d的核心部分。主要作用为初始化Docker Daemon的基本环境，如处理config参数，验证系统支持度，配置Docker工作目录，设置与加载多种driver，创建graph环境等，验证DNS配置等。 

由于daemon.MainDaemon(daemonCfg, eng)是加载Docker Daemon的核心部分，且篇幅过长，故安排《Docker源码分析》系列的第四篇专文分析这部分。

### **5.5.2 通过daemon对象为engine注册Handler**

当创建完daemon对象，goroutine执行d.Install(eng)，具体实现位于[./docker/daemon/daemon.go](https://github.com/docker/docker/blob/v1.2.0/daemon/daemon.go#L100):

```go
    func (daemon *Daemon) Install(eng *engine.Engine) error {
        for name, method := range map[string]engine.Handler{
            "attach":            daemon.ContainerAttach,
            ……
            "image_delete":      daemon.ImageDelete, 
        } {
            if err := eng.Register(name, method); err != nil {
                return err
            }
        }
        if err := daemon.Repositories().Install(eng); err != nil {
            return err
        }
        eng.Hack_SetGlobalVar("httpapi.daemon", daemon)
        return nil
    }
```


以上代码的实现分为三部分：

*   向eng对象中注册众多的Handler对象；
*   daemon.Repositories().Install(eng)实现了向eng对象注册多个与image相关的Handler，Install的实现位于[./docker/graph/service.go](https://github.com/docker/docker/blob/v1.2.0/graph/service.go#L12)；
*   eng.Hack\_SetGlobalVar("httpapi.daemon", daemon)实现向eng对象中map类型的hack对象中添加一条记录，key为”httpapi.daemon”，value为daemon。

### **5.5.3 运行acceptconnections的job**

在goroutine内部最后运行名为”acceptconnections”的job，主要作用是通知init守护进程，Docker Daemon可以开始接受请求了。 

这是源码分析系列中第一次涉及具体Job的运行，以下简单分析”acceptconnections”这个job的运行。 

可以看到首先执行eng.Job("acceptconnections")，返回一个Job，随后再执行eng.Job("acceptconnections").Run()，也就是该执行Job的run函数。 

eng.Job(“acceptconnections”)的实现位于[./docker/engine/engine.go](https://github.com/docker/docker/blob/v1.2.0/engine/engine.go#L115)，如下：

```go
    func (eng *Engine) Job(name string, args ...string) *Job {
        job := &Job{
            Eng:    eng,
            Name:   name,
            Args:   args,
            Stdin:  NewInput(),
            Stdout: NewOutput(),
            Stderr: NewOutput(),
            env:    &Env{},
        }
    if eng.Logging {
            job.Stderr.Add(utils.NopWriteCloser(eng.Stderr))
        }
        if handler, exists := eng.handlers[name]; exists {
            job.handler = handler
        } else if eng.catchall != nil && name != "" {
            job.handler = eng.catchall
        }
        return job
    }
```

由以上代码可知，首先创建一个类型为Job的job对象，该对象中Eng属性为函数的调用者eng，Name属性为”acceptconnections”，没有参数传入。另外在eng对象所有的handlers属性中寻找键为”acceptconnections”记录的值，由于在加载builtins操作中的remote(eng)中已经向eng注册过这样的一条记录，key为”acceptconnections”，value为apiserver.AcceptConnections。因此job对象的handler为apiserver.AcceptConnections。最后返回已经初始化完毕的对象job。 

创建完job对象之后，随即执行该job对象的run()函数。Run()函数的实现位于[./docker/engine/job.go](https://github.com/docker/docker/blob/v1.2.0/engine/job.go#L48)，该函数执行指定的job，并在job执行完成前一直阻塞。对于名为”acceptconnections”的job对象，运行代码为[job.status = job.handler(job)](https://github.com/docker/docker/blob/v1.2.0/engine/job.go#L79)，由于job.handler值为apiserver.AcceptConnections，故真正执行的是job.status = apiserver.AcceptConnections(job)。 

进入AcceptConnections的具体实现，位于[./docker/api/server/server.go](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1370),如下：

```go
    func AcceptConnections(job *engine.Job) engine.Status {
        // Tell the init daemon we are accepting requests
        go  systemd.SdNotify("READY=1")
        if activationLock != nil {
            close(activationLock)
        }
        return engine.StatusOK
    }
```

重点为go systemd.SdNotify("READY=1")的实现，位于[./docker/pkg/system/sd\_notify.go](https://github.com/docker/docker/blob/v1.2.0/pkg/systemd/sd_notify.go#L12)，主要作用是通知init守护进程Docker Daemon的启动已经全部完成，潜在的功能是使得Docker Daemon开始接受Docker Client发送来的API请求。 

至此，已经完成通过goroutine来加载daemon对象并运行。

**5.6 打印Docker版本及驱动信息**
-----------------------

回到mainDaemon()的运行流程中，在goroutine的执行之时，mainDaemon()函数内部其它代码也会并发执行。 

第一个执行的即为显示docker的版本信息，以及ExecDriver和GraphDriver这两个驱动的具体信息，代码如下：

```go
log.Printf("docker daemon: %s %s; execdriver: %s; graphdriver: %s",
    dockerversion.VERSION,
    dockerversion.GITCOMMIT,
    daemonCfg.ExecDriver,
    daemonCfg.GraphDriver,
)
```


**5.7 Job之serveapi的创建与运行**
--------------------------

打印部分Docker具体信息之后，Docker Daemon立即创建并运行名为”serveapi”的job，主要作用为让Docker Daemon提供API访问服务。实现代码位于[./docker/docker/daemon.go#L66](https://github.com/docker/docker/blob/v1.2.0/docker/daemon.go#L66)，如下：

```go
    job := eng.Job("serveapi", flHosts...)
    job.SetenvBool("Logging", true)
    job.SetenvBool("EnableCors", *flEnableCors)
    job.Setenv("Version", dockerversion.VERSION)
    job.Setenv("SocketGroup", *flSocketGroup)
    
    job.SetenvBool("Tls", *flTls)
    job.SetenvBool("TlsVerify", *flTlsVerify)
    job.Setenv("TlsCa", *flCa)
    job.Setenv("TlsCert", *flCert)
    job.Setenv("TlsKey", *flKey)
    job.SetenvBool("BufferRequests", true)
    if err := job.Run(); err != nil {
        log.Fatal(err)
    }
```

实现过程中，首先创建一个名为”serveapi”的job，并将flHosts的值赋给job.Args。flHost的作用主要是为Docker Daemon提供使用的协议与监听的地址。随后，Docker Daemon为该job设置了众多的环境变量，如安全传输层协议的环境变量等。最后通过job.Run()运行该serveapi的job。 

由于在eng中key为”serveapi”的handler，value为apiserver.ServeApi，故该job运行时，执行apiserver.ServeApi函数，位于[./docker/api/server/server.go](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1339)。ServeApi函数的作用主要是对于用户定义的所有支持协议，Docker Daemon均创建一个goroutine来启动相应的http.Server，分别为不同的协议服务。 

由于创建并启动http.Server为Docker架构中有关Docker Server的重要内容，《Docker源码分析》系列会在第五篇专文进行分析。 

至此，可以认为Docker Daemon已经完成了serveapi这个job的初始化工作。一旦acceptconnections这个job运行完毕，则会通知init进程Docker Daemon启动完毕，可以开始提供API服务。

**6 总结**
========

本文从源码的角度分析了Docker Daemon的启动，着重分析了mainDaemon()的实现。

Docker Daemon作为Docker架构中的主干部分，负责了Docker内部几乎所有操作的管理。学习Docker Daemon的具体实现，可以对Docker架构有一个较为全面的认识。总结而言，Docker的运行，载体为daemon，调度管理由engine，任务执行靠job。

**7 参考文献**
==========

1.  [微软宣布下一代Windows Server将引入Docker原生支持](http://www.infoq.com/cn/news/2014/10/windows-server-docker)
2.  [sd\_notify, sd\_notifyf](http://%20http://www.freedesktop.org/software/systemd/man/sd_notify.html)
3.  [浅析 Linux 初始化 init 系统，第 3 部分: Systemd](http://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)
4.  [Package os](http://docs.studygolang.com/pkg/os/)
5.  [Docker Reference-Command Line](https://docs.docker.com/reference/commandline/cli/)

