+++

id= "537"

title = "Docker源码分析（八）：Docker Container网络（下）"
description = "如何使用Docker Container的网络，一直是工业界倍加关心的问题。本文将从Linux内核原理的角度阐述了什么是Docker Container，并对Docker Container 4种不同的网络模式进行了初步的介绍，最终贯穿Docker 架构中的多个模块，如Docker Client、Docker Daemon、execdriver以及libcontainer，深入分析Docker Container网络的实现步骤。"
tags= [ "Docker"]
date= "2015-03-12 20:03:21"
author = "孙宏亮"
banner= "img/blogs/537/docker.jpg"
categories = [ "Docker" ]

+++


**1.Docker Client配置容器网络模式**
---------------------------

Docker目前支持4种网络模式，分别是bridge、host、container、none，Docker开发者可以根据自己的需求来确定最适合自己应用场景的网络模式。 从Docker Container网络创建流程图中可以看到，创建流程第一个涉及的Docker模块即为Docker Client。当然，这也十分好理解，毕竟Docker Container网络环境的创建需要由用户发起，用户根据自身对容器的需求，选择网络模式，并将其通过Docker Client传递给Docker Daemon。

<!--more-->

本节，即从Docker Client源码的角度，分析如何配置Docker Container的网络模式，以及Docker Client内部如何处理这些网络模式参数。 需要注意的是：配置Docker Container网络环境与创建Docker Container网络环境有一些区别。区别是：配置网络环境指用户通过向Docker Client传递网络参数，实现Docker Container网络环境参数的配置，这部分配置由Docker Client传递至Docker Daemon，并由Docker Daemon保存；创建网络环境指，用户通过Docker Client向Docker Daemon发送容器启动命令之后，Docker Daemon根据之前保存的网络参数，实现Docker Container的启动，并在启动过程中完成Docker Container网络环境的创建。 以上的基本知识，理解下文的Docker Container网络环境创建流程。


### **1.1 Docker Client使用**

Docker架构中，用户可以通过Docker Client来配置Docker Container的网络模式。配置过程主要通过docker run命令来完成，实现配置的方式是在docker run命令中添加网络参数。使用方式如下（其中NETWORKMODE为四种网络模式之一，ubuntu为镜像名称，/bin/bash为执行指令）:

docker run -d --net NETWORKMODE ubuntu /bin/bash

运行以上命令时，首先创建一个Docker Client，然后Docker Client会解析整条命令的请求内容，接着解析出为run请求，意为运行一个Docker Container，最终通过Docker Client端的API接口，调用CmdRun函数完成run请求执行。（详情可以查阅[《Docker源码分析》系列的第二篇——Docker Client篇](http://www.sel.zju.edu.cn/?p=147)）。 Docker Client解析出run命令之后，立即调用相应的处理函数CmdRun进行处理关于run请求的具体内容。CmdRun的作用主要可以归纳为三点：

*   解析Docker Client传入的参数，解析出config、hostconfig和cmd对象等；
*   发送请求至Docker Daemon，创建一个container对象，完成Docker Container启动前的准备工作；
*   发送请求至Docker Daemon，启动相应的Docker Container（包含创建Docker Container网络环境创建）。

### **1.2 runconfig包解析**

CmdRun函数的实现位于[./docker/api/client/commands.go](https://github.com/docker/docker/blob/v1.2.0/api/client/commands.go#L1990)。CmdRun执行的第一个步骤为：通过runconfig包中ParseSubcommand函数解析Docker Client传入的参数，并从中解析出相应的config，hostConfig以及cmd对象，实现代码如下:

```go
    config, hostConfig, cmd, err := runconfig.ParseSubcommand
    (cli.Subcmd("run", "[OPTIONS] IMAGE [COMMAND] [ARG...]", 
    "Run a command in a new container"), args, nil) 
```

其中，config的类型为Config结构体，hostConfig的类型为HostConfig结构体，两种类型的定义均位于runconfig包。Config与HostConfig类型同用以描述Docker Container的配置信息，然而两者之间又有着本质的区别，最大的区别在于两者各自的作用范畴：

*   Config结构体：描述Docker Container独立的配置信息。独立的含义是：Config这部分信息描述的是容器本身，而不会与容器所在host宿主机相关；
*   HostConfig结构体：描述Docker Container与宿主机相关的配置信息。

#### **1.2.1 Config结构体**

Config结构体描述Docker Container本身的属性信息，这些信息与容器所在的host宿主机无关。结构体的定义如下：

```go
    type Config struct {
        Hostname        string
        Domainname      string
        User            string
        Memory          int64  // Memory limit (in bytes)
        MemorySwap      int64  // Total memory usage (memory + swap); set \`-1' to disable swap
        CpuShares       int64  // CPU shares (relative weight vs. other containers)
        Cpuset          string // Cpuset 0-2, 0,1
        AttachStdin     bool
        AttachStdout    bool
        AttachStderr    bool
        PortSpecs       []string // Deprecated - Can be in the format of 8080/tcp
        ExposedPorts    map[nat.Port]struct{}
        Tty             bool // Attach standard streams to a tty, including stdin if it is not closed.
        OpenStdin       bool // Open stdin
        StdinOnce       bool // If true, close stdin after the 1 attached client disconnects.
        Env             []string
        Cmd             []string
        Image           string // Name of the image as it was passed by 
    the operator (eg. could be symbolic)
        Volumes         map[string]struct{}
        WorkingDir      string
        Entrypoint      []string
        NetworkDisabled bool
        OnBuild         []string
    }
```

Config结构体中各属性的详细解释如下表： 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616365/sel/CONTAINER1_pklwyf.png" alt="" style="zoom:100%;" />
</center>


#### **1.2.2 HostConfig结构体**

HostConfig结构体描述Docker Container与宿主机相关的属性信息，结构体的定义如下：

```go
    type HostConfig struct {
        Binds           []string
        ContainerIDFile string
        LxcConf         []utils.KeyValuePair
        Privileged      bool
        PortBindings    nat.PortMap
        Links           []string
        PublishAllPorts bool
        Dns             []string
        DnsSearch       []string
        VolumesFrom     []string
        Devices         []DeviceMapping
        NetworkMode     NetworkMode
        CapAdd          []string
        CapDrop         []string
        RestartPolicy   RestartPolicy
    }
```

Config结构体中各属性的详细解释如下表： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616366/sel/CONTAINER2_q6f7tm.png" alt="" style="zoom:100%;" />
</center>


#### **1.2.3 runconfig解析网络模式**

讲述完Config与HostConfig结构体之后，回到runconfig包中分析如何解析与Docker Container网络模式相关的配置信息，并将这部分信息传递给config实例与hostConfig实例。 runconfig包中的ParseSubcommand函数调用parseRun函数完成命令请求的分析，实现代码位于[./docker/runconfig/parse.go#L37-L39](https://github.com/docker/docker/blob/v1.2.0/runconfig/parse.go#L37-L39)，如下：

```go
    func ParseSubcommand(cmd *flag.FlagSet, args []string,
    sysInfo *sysinfo.SysInfo) (*Config, *HostConfig, *flag.FlagSet, error) {
            return parseRun(cmd, args, sysInfo)
    }
```

进入parseRun函数即可发现：该函数完成了四方面的工作：

*   定义与容器配置信息相关的[flag参数](https://github.com/docker/docker/blob/v1.2.0/runconfig/parse.go#L42-L97)；
*   解析docker run命令后紧跟的请求内容，将请求内容全部保存至flag参数中，余下的内容一个为镜像image名，另一个为需要在容器内执行的cmd命令；
*   通过flag参数验证参数的有效性，并处理得到Config结构体与HostConfig结构体需要的属性值；
*   创建并初始化Config类型实例config、HostConfig类型实例hostConfig，最总返回config、hostConfig与cmd。

本文主要分析Docker Container的网络模式，而parseRun函数中有关容器网络模式的flag参数有flNetwork与flNetMode，两者的定义分别位于[./docker/runconfig/parse.go#L62](https://github.com/docker/docker/blob/v1.2.0/runconfig/parse.go#L62)与[./docker/runconfig/parse.go#L75](https://github.com/docker/docker/blob/v1.2.0/runconfig/parse.go#L75)，如下：

```go
    flNetwork = cmd.Bool([]string{"#n", "#-networking"}, true, "Enable networking for this container")

    flNetMode = cmd.String([]string{"-net"}, "bridge", "Set the Network mode for the container\n'bridge': creates a new network stack for the container on the docker bridge\n'none': no networking for this container\n'container:<name|id>': reuses another container network stack\n'host': use the host network stack inside the container. Note: the host mode gives the container full access to local system services such as D-bus and is therefore considered insecure.")
```

可见flag参数flNetwork表示是否开启容器的网络模式，若为true则开启，说明需要给容器创建网络环境；否则不开启，说明不给容器赋予网络功能。该flag参数的默认值为true，另外使用该flag的方式为：在docker run之后设定--networking或者-n，如：

```go
    docker run --networking true ubuntu /bin/bash
```

另一个flag参数flNetMode则表示为容器设定的网络模式，共有四种选项，分别是：bridge、none、container:<name|id>和host。四种模式的作用上文已经详细解释，此处不再赘述。使用该flag的方式为：在docker run之后设定--net，如：

```go
    Docker run --net host ubuntu /bin/bash
```

用户使用docker run启动容器时设定了以上两个flag参数（--networking和--net），则runconfig包会解析出这两个flag的值。最终，通过flag参数flNetwork，得到Config类型实例config的属性NetworkDisabled；通过flag参数flNetMode，得到HostConfig类型实例hostConfig的属性NetworkMode。 函数parseRun返回config、hostConfig与cmd，代表着runconfig包解析配置参数工作的完成，CmdRun得到返回内容之后，继续向下执行。

### **1.3 CmdRun执行**

在runconfig包中已经将有关容器网络模式的配置置于config对象与hostConfig对象，故在CmdRun函数的执行中，更多的是基于config对象与hostConfig参数处理配置信息，而没有其他的容器网络处理部分。 CmdRun后续主要工作是：利用Docker Daemon暴露的RESTful API接口，将docker run的请求发送至Docker Daemon。以下是CmdRun执行过程中Docker Client与Docker Daemon的简易交互图。 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616366/sel/CONTAINER3_zckdf2.jpg" alt="" style="zoom:50%;" />
</center>


图1.1 CmdRun中Docker Client与Docker Daemon交互图

具体分析CmdRun的执行流程可以发现：在解析config、hostConfig与cmd之后，Docker Client首先发起请求create container。若Docker Daemon节点上已经存在该容器所需的镜像，则立即执行create container操作并返回请求响应；Docker Client收到响应后，再发起请求start container。若容器镜像还不存在，Docker Daemon返回一个404的错误，表示镜像不存在；Docker Client收到错误响应之后，再发起一个请求pull image，使Docker Daemon首先下载镜像，下载完毕之后Docker Client再次发起请求create container，Docker Daemon创建完毕之后，Docker Client最终发起请求start container。 总结而言，Docker Client负责创建容器请求的发起。关于Docker Container网络环境的配置参数存储于config与hostConfig对象之中，在请求create container和start container发起时，随请求一起发送至Docker Daemon。

**2.Docker Daemon创建容器网络流程**
---------------------------

Docker Daemon接收到Docker Client的请求大致可以分为两次，第一次为create container，第二次为start container。这两次请求的执行过程中，都与Docker Container的网络相关。以下按照这两个请求的执行，具体分析Docker Container网络模式的创建。Docker Daemon如何通过Docker Server解析RESTful请求，并完成分发调度处理，在《Docker源码分析》系列的第五篇——Docker Server篇中已经详细分析过，本文不再赘述。

### **2.1创建容器并配置网络参数**

Docker Daemon首先接收并处理create container请求。需要注意的是：create container并非创建了一个运行的容器，而是完成了以下三个主要的工作：

*   通过runconfig包解析出create container请求中与Docker Container息息相关的config对象；
*   在Docker Daemon内部创建了与Docker Container对应的container对象；
*   完成Docker Container启动前的准备化工作，如准备所需镜像、创建rootfs等。

创建容器过程中，Docker Daemon首先通过runconfig包中ContainerConfigFromJob函数，解析出请求中的config对象，解析过程代码如下：

config := runconfig.ContainerConfigFromJob(job)

至此，Docker Client处理得到的config对象，已经传递至Docker Daemon的config对象，config对象中已经含有属性NetworkDisabled具体值。 处理得到config对象之后，Docker Daemon紧接着创建container对象，并为Docker Container作相应的准备工作。具体的实现代码位于[./docker/daemon/create.go#L73-L78](https://github.com/docker/docker/blob/v1.2.0/daemon/create.go#L73-L78)，如下：

```go
    if container, err = daemon.newContainer(name, config, img); err != nil {
        return nil, nil, err
    }
    if err := daemon.createRootfs(container, img); err != nil {
        return nil, nil, err
    }
```

与Docker Container网络模式配置相关的内容主要位于创建container对象中。newContainer函数的定义位于[./docker/daemon/daemon.go#L516-L550](https://github.com/docker/docker/blob/v1.2.0/daemon/daemon.go#L516-L550)，具体的container对象如下：

```go
    container := &Container{
        ID:              id,
        Created:         time.Now().UTC(),
        Path:            entrypoint,
        Args:            args, //FIXME: de-duplicate from config
        Config:          config,
        hostConfig:      &runconfig.HostConfig{},
        Image:           img.ID, // Always use the resolved image id
        NetworkSettings: &NetworkSettings{},
        Name:            name,
        Driver:          daemon.driver.String(),
        ExecDriver:      daemon.execDriver.Name(),
        State:           NewState(),    
    }
```

在container对象中，config对象直接赋值给container对象的Config属性，另外hostConfig属性与NetworkSeeetings属性均为空。其中hostConfig对象将在start container请求执行过程中被赋值，NetworkSettings类型的作用是描述容器的网络具体信息，定义位于[./docker/daemon/network\_settings.go#L11-L18](https://github.com/docker/docker/blob/v1.2.0/daemon/network_settings.go#L11-L18)，代码如下：

```go
    type NetworkSettings struct {
            IPAddress   string
            IPPrefixLen int
            Gateway     string
            Bridge      string
            PortMapping map[string]PortMapping // Deprecated
            Ports       nat.PortMap
    }
```

Networksettings类型的各属性的详细解释如下表： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616365/sel/CONTAINER4_g4iqnw.png" alt="" style="zoom:100%;" />
</center>

总结而言，Docker Daemon关于create container请求的执行，先实现了容器配置信息从Docker Client至Docker Daemon的转移，再完成了启动容器前所需的准备工作。

### **2.2启动容器之网络配置**

创建容器阶段，Docker Daemon创建了容器对象container，container对象内部的Config属性含有NetworkDisabled。创建容器完成之后，Docker Daemon还需要接收Docker Client的请求，并执行start container的操作，即启动容器。 启动容器过程中，Docker Daemon首先通过runconfig包中ContainerHostConfigFromJob函数，解析出请求中的hostConfig对象，解析过程代码如下：

```go
    hostConfig := runconfig.ContainerHostConfigFromJob(job) 
```

至此，Docker Client处理得到的hostConfig对象，已经传递至Docker Daemon的hostConfig对象，hostConfig对象中已经含有属性NetworkMode具体值。 容器启动的所有工作，均由以下的Start函数来完成，代码位于[./docker/daemon/start.go#L36-L38](https://github.com/docker/docker/blob/v1.2.0/daemon/start.go#L36-L38)，如下：

```go
    if err := container.Start(); err != nil {
            return job.Errorf("Cannot start container %s: %s", name, err)
    }
```

Start函数实现了容器的启动。更为具体的描述是：Start函数实现了进程的启动，另外在启动进程的同时为进程设定了命名空间（namespace），启动完毕之后为进程完成了资源使用的控制，从而保证进程以及之后进程的子进程都会在同一个命名空间内，且受到相同的资源控制。如此一来，Start函数创建的进程，以及该进程的子进程，形成一个进程组，该进程组处于资源隔离和资源控制的环境，我们习惯将这样的进程组环境称为容器，也就是这里的Docker Container。 回到Start函数的执行，位于[./docker/daemon/container.go#L275-L320](https://github.com/docker/docker/blob/v1.2.0/daemon/container.go#L275-L320)。Start函数执行过程中，与Docker Container网络模式相关的部分主要有三部分：

*   initializeNetwork()，初始化container对象中与网络相关的属性；
*   populateCommand，填充Docker Container内部需要执行的命令，Command中含有进程启动命令，还含有容器环境的配置信息，也包括网络配置；
*   container.waitForStart()，实现Docker Container内部进程的启动，进程启动之后，为进程创建网络环境等。

#### **2.2.1初始化容器网络配置**

容器对象container中有属性hostConfig，属性hostConfig中有属性NetworkMode，初始化容器网络配置initializeNetworking()的主要工作就是，通过NetworkMode属性为Docker Container的网络作相应的初始化配置工作。 Docker Container的网络模式有四种，分别为：host、other container、none以及bridge。initializeNetworking函数的执行完全覆盖了这四种模式。 initializeNetworking()函数的实现位于[./docker/daemon/container.go#L881-L933](https://github.com/docker/docker/blob/v1.2.0/daemon/container.go#L881-L933)。

##### **2.2.1.1 初始化host网络模式配置**

Docker Container网络的host模式意味着容器使用宿主机的网络环境。虽然Docker Container使用宿主机的网络环境，但这并不代表Docker Container可以拥有宿主机文件系统的视角，而host宿主机上有很多信息标识的是网络信息，故Docker Daemon需要将这部分标识网络的信息，从host宿主机添加到Docker Container内部的指定位置。这样的网络信息，主要有以下两种：

*   host宿主机的主机名（hostname）；
*   host宿主机上/etc/hosts文件，用于配置IP地址以及主机名。

其中，宿主机的主机名hostname用于创建container.Config中的Hostname与Domainname属性。 另外，Docker Daemon在Docker Container的rootfs内部创建hostname文件，并在文件中写入Hostname与Domainname；同时创建hosts文件，并写入host宿主机上/etc/hosts内的所有内容。

##### **2.2.1.2 初始化other container网络模式配置**

Docker Container的other container网络模式意味着：容器使用其他已经创建容器的网络环境。 Docker Daemon首先判断host网络模式之后，若不为host网络模式，则继续判断Docker Container网络模式是否为other container。如果Docker Container的网络模式为other container（假设使用的-net参数为--net=container:17adef，其中17adef为容器ID）。Docker Daemon所做的执行操作包括两部分。 第一步，从container对象的hostConfig属性中找出NetworkMode，并找到相应的容器，即17adef的容器对象container，实现代码如下：

```go
    nc, err := container.getNetworkedContainer()
```

第二步，将17adef容器对象的HostsPath、ResolveConfPath、Hostname和Domainname赋值给当前容器对象container，实现代码如下：

```go
    container.HostsPath = nc.HostsPath
    container.ResolvConfPath = nc.ResolvConfPath
    container.Config.Hostname = nc.Config.Hostname
    container.Config.Domainname = nc.Config.Domainname
```

##### **2.2.1.3 初始化none网络模式配置**

Docker Container的none网络模式意味着不给该容器创建任何网络环境，容器只能使用127.0.0.1的本机网络。 Docker Daemon通过config属性的DisableNetwork来判断是否为none网络模式。实现代码如下：

```go
    if container.daemon.config.DisableNetwork {
            container.Config.NetworkDisabled = true
            return container.buildHostnameAndHostsFiles("127.0.1.1")
    }
```

##### **2.2.1.4 初始化bridge网络模式配置**

Docker Container的bridge网络模式意味着为容器创建桥接网络模式。桥接模式使得Docker Container创建独立的网络环境，并通过“桥接”的方式实现Docker Container与外界的网络通信。 初始化bridge网络模式的配置，实现代码如下：

```go
    if err := container.allocateNetwork(); err != nil {
            return err
    }
```
return container.buildHostnameAndHostsFiles(container.NetworkSettings.IPAddress) 

以上代码完成的内容主要也是两部分：第一，通过allocateNetwork函数为容器分配网络接口设备需要的资源信息（包括IP、bridge、Gateway等），并赋值给container对象的NetworkSettings；第二，为容器创建hostname以及创建Hosts等文件。

#### **2.2.2创建容器Command信息**

Docker在实现容器时，使用了Command类型。Command在Docker Container概念中是一个非常重要的概念。几乎可以认为Command是Docker Container生命周期的源头。Command的概念会贯穿以后的《Docker源码分析》系列，比如Docker Daemon与dockerinit的关系，dockerinit和entrypoint.sh的关系，entrypoint.sh与CMD的关系，以及namespace在这些内容中扮演的角色。 简单来说，Command类型包含了两部分的内容：第一，运行容器内进程的外部命令exec.Cmd；第二，运行容器时启动进程需要的所有环境基础信息：包括容器进程组的使用资源、网络环境、使用设备、工作路径等。通过这两部分的内容，我们可以清楚，如何启动容器内的进程，同时也清楚为容器创建什么样的环境。 首先，我们先来看Command类型的定义，位于[./docker/daemon/execdriver/driver.go#L84](https://github.com/docker/docker/blob/v1.2.0/daemon/execdriver/driver.go#L84)，通过分析Command类型以及其他相关的数据结构类型，可以得到以下简要类型关系图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616366/sel/CONTAINER5_eenmhn.jpg" alt="" style="zoom:50%;" />
</center>

图 2.1 Command类型关系图

从Command类型关系图中可以看到，Command类型中重新包装了exec.Cmd类型，即代表需要创建的进程具体的外部命令；同时，关于网络方面的属性有Network，Network的类型为指向Network类型的指针；关于Docker Container资源使用方面的属性为Resources，从Resource的类型来看，Docker目前能做的资源限制有4个维度，分别为内存，内存+Swap，CPU使用，CPU核使用；关于挂载的内容，有属性Mounts；等等。 

简单介绍Command类型之后，回到Docker Daemon启动容器网络的源码分析。在Start函数的执行流程中，紧接initializeNetworking()之后，与Docker Container网络相关的是populateCommand环节。populateCommand的函数实现位于[./docker/daemon/container.go#L191-L274](https://github.com/docker/docker/blob/v1.2.0/daemon/container.go#L191-L273)。

上文已经提及，populateCommand的作用是创建包execdriver的对象Command，该Command中既有启动容器进程的外部命令，同时也有众多为容器环境的配置信息，包括网络。 本小节，更多的分析populateCommand如何填充Command对象中的网络信息，其他信息的分析会在《源码分析系列》的后续进行展开。 Docker总共有四种网络模式，故populateCommand自然需要判断容器属于哪种网络模式，随后将具体的网络模式信息，写入Command对象的Network属性中。查验Docker Container网络模式的代码位于[./docker/daemon/container.go#L204-L227](https://github.com/docker/docker/blob/v1.2.0/daemon/container.go#L204-L227)，如下：

```go
    parts := strings.SplitN(string(c.hostConfig.NetworkMode), ":", 2)
        switch parts[0] {
        case "none":
        case "host":
            en.HostNetworking = true
        case "bridge", "": // empty string to support existing containers
            if !c.Config.NetworkDisabled {
                network := c.NetworkSettings
                en.Interface = &execdriver.NetworkInterface{
                    Gateway:     network.Gateway,
                    Bridge:      network.Bridge,
                    IPAddress:   network.IPAddress,
                    IPPrefixLen: network.IPPrefixLen,
                }
            }
        case "container":
            nc, err := c.getNetworkedContainer()
            if err != nil {
                return err
            }
            en.ContainerID = nc.ID
        default:
            return fmt.Errorf("invalid network mode: %s", c.hostConfig.NetworkMode)
        }
```

populateCommand首先通过hostConfig对象中的NetworkMode判断容器属于哪种网络模式。该部分内容涉及到execdriver包中的Network类型，可参见Command类型关系图中的Network类型。若为none模式，则对于Network对象（即en，\*execdriver.Network）不做任何操作。若为host模式，则将Network对象的HostNetworking置为true；若为bridge桥接模式，则首先创建一个NetworkInterface对象，完善该对象的Gateway、Bridge、IPAddress和IPPrefixLen信息，最后将NetworkInterface对象作为Network对象的Interface属性的值；若为other container模式，则首先通过getNetworkedContainer()函数获知被分享网络命名空间的容器，最后将容器ID，赋值给Network对象的ContainerID。

由于bridge模式、host模式、none模式以及other container模式彼此互斥，故Network对象中Interface属性、ContainerID属性以及HostNetworking三者之中只有一个被赋值。当Docker Container的网络查验之后，populateCommand将en实例Network属性的值，传递给Command对象。 至此，populateCommand关于网络方面的信息已经完成配置，网络配置信息已经成功赋值于Command的Network属性。。

#### **2.2.3启动容器内部进程**

当为容器做好所有的准备与配置之后，Docker Daemon需要真正意义上的启动容器。根据Docker Daemon启动容器流程涉及的Docker模块中可以看到，这样的请求，会被发送至execdriver，再经过libcontainer，最后实现真正启动进程，创建完容器。 回到Docker Daemon的启动容器，daemon包中start函数的最后一步即为执行container.waitForStart()。waitForStart函数的定义位于[./docker/daemon/container.go#L1070-L1082](https://github.com/docker/docker/blob/v1.2.0/daemon/container.go#L1070-L1082)，代码如下：

```go
    func (container *Container) waitForStart() error {
            container.monitor = newContainerMonitor(container, container.hostConfig.RestartPolicy)
            
            select {
            case <-container.monitor.startSignal:
            case err := <-utils.Go(container.monitor.Start):
                return err
            }

            return nil
    }
```

以上代码运行过程中首先通过newContainerMonitor返回一个初始化的containerMonitor对象，该对象中带有容器进程的重启策略（RestartPolicy）。这里简单介绍containerMonitor对象。总体而言，containerMonitor对象用以监视容器中第一个进程的执行。如果containerMonitor中指定了一种进程重启策略，那么一旦容器内部进程没有启动成功，Docker Daemon会使用重启策略来重启容器。如果在重启策略下，容器依然没有成功启动，那么containerMonitor对象会负责重置以及清除所有已经为容器准备好的资源，例如已经为容器分配好的网络资源（即IP地址），还有为容器准备的rootfs等。 

waitForStart()函数通过container.monitor.Start来实现容器的启动，进入[./docker/daemon/monitor.go#L100](https://github.com/docker/docker/blob/v1.2.0/daemon/monitor.go#L100)，可以发现启动容器进程位于[./docker/daemon/monitor.go#L136](https://github.com/docker/docker/blob/v1.2.0/daemon/monitor.go#L136)，代码如下：

```go
    exitStatus, err = m.container.daemon.Run(m.container, pipes, m.callback) 
```

以上代码实际调用了daemon包中的Run函数，位于[./docker/daemon/daemon.go#L969-L971](https://github.com/docker/docker/blob/v1.2.0/daemon/daemon.go#L969-L971)，代码如下：

```go
    func (daemon *Daemon) Run(c *Container, pipes *execdriver.Pipes, startCallback execdriver.StartCallback) (int, error) {
            return daemon.execDriver.Run(c.command, pipes, startCallback)
    }
```

最终，Run函数中调用了execdriver中的Run函数来执行Docker Container的启动命令。 至此，网络部分在Docker Daemon内部的执行已经结束，紧接着程序运行逻辑陷入execdriver，进一步完成容器启动的相关步骤。

**3.execdriver网络执行流程**
----------------------

Docker架构中execdriver的作用是启动容器内部进程，最终启动容器。目前，在Docker中execdriver作为执行驱动，可以有两种选项：lxc与native。其中，lxc驱动会调用lxc工具实现容器的启动，而native驱动会使用[Docker官方发布的libcontainer](https://github.com/docker/libcontainer/tree/v1.2.0)来启动容器。 

Docker Daemon启动过程中，execdriver的类型默认为native，故本文主要分析native驱动在执行启动容器时，如何处理网络部分。 在Docker Daemon启动容器的最后一步，即调用了execdriver的Run函数来执行。通过分析Run函数的具体实现，关于Docker Container的网络执行流程主要包括两个环节： (1) 创建libcontainer的Config对象 (2) 通过libcontainer中的namespaces包执行启动容器 将execdriver.Run函数的运行流程具体展开，与Docker Container网络相关的流程，可以得到以下示意图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616367/sel/CONTAINER6_fcpd7f.jpg" alt="" style="zoom:60%;" />
</center>


图3.1 execdriver.Run执行流程图

### **3.1创建libcontainer的Config对象**

Run函数位于./docker/daemon/execdriver/native/driver.go#L62-L168，进入Run函数的实现，立即可以发现该函数通过createContainer创建了一个container对象，代码如下：

```go
    container, err := d.createContainer(c) 
```

其中c为Docker Daemon创建的execdriver.Command类型实例。以上代码的createContainer函数的作用是：使用execdriver.Command来填充libcontainer.Config。 libcontainer.Config的作用是，定义在一个容器化的环境中执行一个进程所需要的所有配置项。createContainer函数的存在，使用Docker Daemon层创建的execdriver.Command，创建更下层libcontainer所需要的Config对象。这个角度来看，execdriver更像是封装了libcontainer对外的接口，实现了将Docker Daemon认识的容器启动信息转换为底层libcontainer能真正使用的容器启动配置选项。libcontainer.Config类型与其内部对象的关联图如下： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616369/sel/CONTAINER7_anm6gp.jpg" alt="" style="zoom:60%;" />
</center>

图3.2 libcontainer.Config类型关系图

进入createContainer的源码实现部分，位于[./docker/daemon/execdriver/native/create.go#L23-L77](https://github.com/docker/docker/blob/v1.2.0/daemon/execdriver/native/create.go#L23-L77)，代码如下：

```go
    func (d *driver) createContainer(c *execdriver.Command) (*libcontainer.Config, error) {
        container := template.New()
        ……
        if err := d.createNetwork(container, c); err != nil {
            return nil, err
        }
        ……
        return container, nil
    }
```

#### **3.1.1 libcontainer.Config模板实例**

从createContainer函数的实现以及execdriver.Run执行流程图中都可以看到，createContainer所做的第一个操作就是即为执行template.New()，意为创建一个libcontainer.Config的实例container。其中，template.New()的定义位于[./docker/daemon/execdriver/native/template/default\_template.go](https://github.com/docker/docker/blob/v1.2.0/daemon/execdriver/native/template/default_template.go)，主要的作用为返回libcontainer关于Docker Container的默认配置选项。 Template.New()的代码实现如下：

```go
    func New() *libcontainer.Config {
        container := &libcontainer.Config{
            Capabilities: []string{
                "CHOWN",
                "DAC_OVERRIDE",
                "FSETID",
                "FOWNER",
                "MKNOD",
                "NET_RAW",
                "SETGID",
                "SETUID",
                "SETFCAP",
                "SETPCAP",
                "NET_BIND_SERVICE",
                "SYS_CHROOT",
                "KILL",
                "AUDIT_WRITE",
            },
            Namespaces: map[string]bool{
                "NEWNS":  true,
                "NEWUTS": true,
                "NEWIPC": true,
                "NEWPID": true,
                "NEWNET": true,
            },
            Cgroups: &cgroups.Cgroup{
                Parent:          "docker",
                AllowAllDevices: false,
            },
            MountConfig: &libcontainer.MountConfig{},
        }

        if apparmor.IsEnabled() {
            container.AppArmorProfile = "docker-default"
        }

        return container
    }
```

关于该libcontainer.Config默认的模板对象，从源码实现中可以看到，首先设定了Capabilities的默认项，如CHOWN、DAC\_OVERRIDE、FSETID等；其次又将Docker Container所需要的设定的namespaces添加默认值，即需要创建5个NAMESPACE，如NEWNS、NEWUTS、NEWIPC、NEWPID和NEWNET，其中不包括user namespace，另外与网络相关的namespace为NEWNET；最后设定了一些关于cgroup以及apparmor的默认配置。 Template.New()函数最后返回类型为libcontainer.Config的实例container，该实例中只含有默认配置项，其他的配置项的添加需要createContainer的后续操作来完成。

#### **3.1.2 createNetwork实现**

在createContainer的实现流程中，为了完成container对象（类型为libcontainer.Config）的完善，最后有很多步骤，如与网络相关的createNetwork函数调用，与Linux内核Capabilities相关的setCapabilities函数调用，与cgroups相关的setupCgroups函数调用，以及与挂载目录相关的setupMounts函数调用等。本小节主要分析createNetwork如何为container对象完善网络配置项。 createNetwork函数的定义位于[./docker/daemon/execdriver/native/create.go#L79-L124](https://github.com/docker/docker/blob/v1.2.0/daemon/execdriver/native/create.go#L79-L124)，该函数主要利用execdriver.Command中Network属性中的内容，来判断如何创建libcontainer.Config中Network属性（关于两中Network属性，可以参见图3.1和图3.2）。由于Docker Container的4种网络模式彼此互斥，故以上Network类型中Interface、ContainerID与HostNetworking最多只有一项会被赋值。 由于execdriver.Command中Network的类型定义如下：

```go
    type Network struct {
        Interface      *NetworkInterface
        Mtu            int              
        ContainerID    string            
        HostNetworking bool             
    }
```

分析createNetwork函数，其具体实现可以归纳为4部分内容： (1) 判断网络是否为host模式； (2) 判断是否为bridge桥接模式； (3) 判断是否为other container模式； (4) 为Docker Container添加loopback网络设备。 首先来看execdriver判断是否为host模式的代码：

```go
    if c.Network.HostNetworking {
        container.Namespaces["NEWNET"] = false
        return nil
    }
```

当execdriver.Command类型实例中Network属性的HostNetworking为true，则说明需要为Docker Container创建host网络模式，使得容器与宿主机共享同样的网络命名空间。关于host模式的具体介绍中，已经阐明，只须在创建进程进行CLONE系统调用时，不传入CLONE\_NEWNET参数标志即可实现。这里的代码正好准确的验证了这一点，将container对象中NEWNET的Namespace设为false，最终在libcontainer中可以达到效果。 再看execdriver判断是否为bridge桥接模式的代码：

```go
    if c.Network.Interface != nil {
            vethNetwork := libcontainer.Network{
                Mtu:        c.Network.Mtu,
                Address:    fmt.Sprintf("%s/%d", c.Network.Interface.IPAddress, 
    c.Network.Interface.IPPrefixLen),
                Gateway:    c.Network.Interface.Gateway,
                Type:       "veth",
                Bridge:     c.Network.Interface.Bridge,
                VethPrefix: "veth",
            }
            container.Networks = append(container.Networks, &vethNetwork)
        }
```

当execdriver.Command类型实例中Network属性的Interface不为nil值，则说明需要为Docker Container创建bridge桥接模式，使得容器使用隔离的网络环境。于是这里为类型为libcontainer.Config的container对象添加Networks属性vethNetwork，网络类型为“veth”，以便libcontainer在执行时，可以为Docker Container创建veth pair。 接着来看execdriver判断是否为other container模式的代码：

```go
    if c.Network.ContainerID != "" {
            d.Lock()
            active := d.activeContainers[c.Network.ContainerID]
            d.Unlock()

            if active == nil || active.cmd.Process == nil {
                return fmt.Errorf("%s is not a valid running container to join", c.Network.ContainerID)
            }
            cmd := active.cmd

            nspath := filepath.Join("/proc", fmt.Sprint(cmd.Process.Pid), "ns", "net")
            container.Networks = append(container.Networks, &libcontainer.Network{
                Type:   "netns",
                NsPath: nspath,
            })
        }
```

当execdriver.Command类型实例中Network属性的ContainerID不为空字符串时，则说明需要为Docker Container创建other container模式，使得创建容器使用其他容器的网络环境。实现过程中，execdriver需要首先在activeContainers中查找需要被共享网络环境的容器active；并通过active容器的启动执行命令cmd找到容器第一进程在宿主机上的PID；随后在proc文件系统中找到该进程PID的关于网络namespace的路径nspath；最后为类型为libcontainer.Config的container对象添加Networks属性，Network的类型为“netns”。 此外，createNetwork函数还实现为Docker Container创建一个loopback回环设备，以便容器可以实现内部通信。实现过程中，同样为类型libcontainer.Config的container对象添加Networks属性，Network的类型为“loopback”，代码如下：

```go
    container.Networks = []*libcontainer.Network{
            {
                Mtu:     c.Network.Mtu,
                Address: fmt.Sprintf("%s/%d", "127.0.0.1", 0),
                Gateway: "localhost",
                Type:    "loopback",
            },
        }
```

至此，createNetwork函数已经把与网络相关的配置，全部创建在类型为libcontainer.Config的container对象中了，就等着启动容器进程时使用。

### **3.2 调用libcontainer的namespaces启动容器**

回到execdriver.Run函数，创建完libcontainer.Config实例container，经过一系列其他方面的处理之后，最终execdriver执行namespaces.Exec函数实现启动容器，container对象依然是namespace.Exec函数中一个非常重要的参数。这一环节代表着execdriver把启动Docker Container的工作交给libcontainer，以后的执行陷入libcontainer。 调用namespaces.Exec的代码位于[./docker/daemon/execdriver/native/driver.go#L102-L127](https://github.com/docker/docker/blob/v1.2.0/daemon/execdriver/native/driver.go#L102-L127)，为了便于理解，简化的代码如下：

```go
    namespaces.Exec(container, c.Stdin, c.Stdout, c.Stderr, c.Console,
    c.Rootfs, dataPath, args, parameter_1, parameter_2)
```

其中parameter\_1为定义的函数，如下：

```go
    func(container *libcontainer.Config, console, rootfs, dataPath, 
    init string, child *os.File, args []string) *exec.Cmd {
            c.Path = d.initPath
            c.Args = append([]string{
                DriverName,
                "-console", console,
                "-pipe", "3",
                "-root", filepath.Join(d.root, c.ID),
                "--",
            }, args...)

            // set this to nil so that when we set the clone flags anything else is reset
            c.SysProcAttr = &syscall.SysProcAttr{
                Cloneflags: uintptr(namespaces.GetNamespaceFlags(container.Namespaces)),
            }
            c.ExtraFiles = []*os.File{child}

            c.Env = container.Env
            c.Dir = c.Rootfs

            return &c.Cmd
        }
```

同样的，parameter_2也为定义的函数，如下：

```go
    func() {
            if startCallback != nil {
                c.ContainerPid = c.Process.Pid
                startCallback(c)
            }
        }
```

Parameter_1以及parameter_2这两个函数均会在libcontainer的namespaces中发挥很大的重要。 至此，execdriver模块的执行部分已经完结，Docker Daemon的程序运行逻辑陷入libcontainer。

**4.libcontainer实现内核态网络配置**
---------------------------

libcontainer是一个Linux操作系统上容器的实现包。libcontainer指定了创建一个容器时所需要的配置选项，同时它利用Linux namespace和cgroup等技术为使用者提供了一套Golang原生态的容器实现方案，并且没有使用任何外部依赖。用户借助libcontainer，可以感受到众多操纵namespaces，网络等资源的便利。 当execdriver调用libcontainer中namespaces包的Exec函数时，libcontainer开始发挥其实现容器功能的作用。Exec函数位于[./libcontainer/namespaces/exec.go#L24-L113](https://github.com/docker/libcontainer/blob/v1.2.0/namespaces/exec.go#L24-L113)。

本文更多的关心Docker Container的网络创建，因此从这个角度来看Exec的实现可以分为三个步骤： (1) 通过createCommand创建一个Golang语言内的exec.Cmd对象； (2) 启动命令exec.Cmd，执行容器内第一个进程； (3) 通过InitializeNetworking函数为容器进程初始化网络环境。 以下详细分析这三个部分，源码的具体实现。

### **4.1创建exec.Cmd**

提到exec.Cmd，就不得不提Go语言标准库中的包os以及包os/exec。前者提供了与平台无关的操作系统功能集，后者则提供了功能集里与命令执行相关的部分。 首先来看一下在Go语言中exec.Cmd的定义，如下：

```go
    type Cmd struct {
            Path string                   //所需执行命令在系统中的路径
            Args []string                  //传入命令的参数
            Env []string                   //进程运行时的环境变量
            Dir string                     //命令运行的工作目录
            Stdin io.Reader
            Stdout io.Writer
            Stderr io.Writer
            ExtraFiles []*os.File             //进程所需打开的文件描述符资源
            SysProcAttr *syscall.SysProcAttr   //可选的操作系统属性
            Process *os.Process            //代表Cmd启动后，操作系统底层的具体进程
            ProcessState *os.ProcessState    //进程退出后保留的信息
    }
```

清楚Cmd的定义之后，再来分析namespaces包的Exec函数中，是如何来创建exec.Cmd的。在Exec函数的实现过程中，使用了以下代码实现Exec.Cmd的创建：

```go
    command := createCommand(container, console, rootfs, dataPath, 
    os.Args[0], syncPipe.Child(), args) 
```

其中createCommand为namespace.Exec函数中传入的倒数第二个参数，类型为CreateCommand。而createCommand只是namespaces.Exec函数的形参，真正的实参则为execdriver调用namespaces.Exec时的参数parameter_1，即如下代码：

```go
    func(container *libcontainer.Config, console, rootfs, dataPath,
    init string, child *os.File, args []string) *exec.Cmd {
            c.Path = d.initPath
            c.Args = append([]string{
                DriverName,
                "-console", console,
                "-pipe", "3",
                "-root", filepath.Join(d.root, c.ID),
                "--",
            }, args...)

            // set this to nil so that when we set the clone flags anything else is reset
            c.SysProcAttr = &syscall.SysProcAttr{
                Cloneflags: uintptr(namespaces.GetNamespaceFlags(container.Namespaces)),
            }
            c.ExtraFiles = []*os.File{child}

            c.Env = container.Env
            c.Dir = c.Rootfs

            return &c.Cmd
        }
```

熟悉exec.Cmd的定义之后，分析以上代码的实现就显得较为简单。为Cmd赋值的对象有Path，Args，SysProcAttr，ExtraFiles，Env和Dir。其中需要特别注意的是Path的值d.initPath，该路径下存放的是dockerinit的二进制文件，Docker 1.2.0版本下，路径一般为“/var/lib/docker/init/dockerinit-1.2.0”。另外SysProcAttr使用以下的代码来赋值：

```go
    &syscall.SysProcAttr{
            Cloneflags: uintptr(namespaces.GetNamespaceFlags(container.Namespaces)),
        }
```

syscall.SysProAttr对象中的Cloneflags属性中，即保留了libcontainer.Config类型的实例container中的Namespace属性。换言之，通过exec.Cmd创建进程时，正是通过Cloneflags实现Clone系统调用中传入namespace参数标志。 回到函数执行中，在函数的最后返回了c.Cmd，命令创建完毕。

### **4.2启动exec.Cmd创建进程**

创建完exec.Cmd，当然需要将该执行命令运行起来，namespaces.Exec函数中直接使用以下代码实现进程的启动：

```go
    if err := command.Start(); err != nil {
            return -1, err
        }
```

这一部分的内容简单直接， Start()函数用以完成指定命令exec.Cmd的启动执行，同时并不等待其启动完毕便返回。Start()函数的定义位于os/exec包。 进入os/exec包，查看Start()函数的实现，可以看到执行过程中，会对command.Process进行赋值，此时command.Process中会含有刚才启动进程的PID进程号，该PID号属于在宿主机pid namespace下，而并非是新创建namespace下的PID号。

### **4.3为容器进程初始化网络环境**

上一环节实现了容器进程的启动，然而却还没有为之配置相应的网络环境。namespaces.Exec在之后的InitializeNetworing中实现了为容器进程初始化网络环境。初始化网络环境需要两个非常重要的参数：container对象以及容器进程的Pid号。类型为libcontainer.Config的实例container中包含用户对Docker Container的网络配置需求，另外容器进程的Pid可以使得创建的网络环境与进程新创建的namespace进行关联。 namespaces.Exec中为容器进程初始化网络环境的代码实现位于[./libcontainer/namespaces/exec.go#L75-L79](https://github.com/docker/libcontainer/blob/v1.2.0/namespaces/exec.go#L75-L79)，如下：

```go
    if err := InitializeNetworking(container, command.Process.Pid, syncPipe, &networkState); err != nil {
            command.Process.Kill()
            command.Wait()
            return -1, err
        }
```

InitializeNetworing的作用很明显，即为创建的容器进程初始化网络环境。更为底层的实现包含两个步骤： (1) 先在容器进程的namespace外部，创建容器所需的网络栈； (2) 将创建的网络栈迁移进入容器的net namespace。 IntializeNetworking的源代码实现位于[./libcontainer/namespaces/exec.go#L176-L187](https://github.com/docker/libcontainer/blob/v1.2.0/namespaces/exec.go#L176-L187)，如下：

```go
    func InitializeNetworking(container *libcontainer.Config, nspid int, pipe *syncpipe.SyncPipe, networkState *network.NetworkState) error {
        for _, config := range container.Networks {
            strategy, err := network.GetStrategy(config.Type)
            if err != nil {
                return err
            }
            if err := strategy.Create((*network.Network)(config), nspid, networkState); err != nil {
                return err
            }
        }
        return pipe.SendToChild(networkState)
    }
```

以上源码实现过程中，首先通过一个循环，遍历libcontainer.Config类型实例container中的网络属性Networks；随后使用GetStrategy函数处理Networks中每一个对象的Type属性，得出Network的类型，这里的类型有3种，分别为“loopback”、“veth”、“netns”。除host网络模式之外，loopback对于其他每一种网络模式的Docker Container都需要使用；veth针对bridge桥接模式，而netns针对other container模式。 得到Network类型的类型之后，libcontainer创建相应的网络栈，具体实现使用每种网络栈类型下的Create函数。以下分析三种不同网络栈各自的创建流程。

#### **4.3.1 loopback网络栈的创建**

Loopback是一种本地环回设备，libcontainer创建loopback网络设备的实现代码位于[./libcontainer/network/loopback.go#L13-L1](https://github.com/docker/libcontainer/blob/v1.2.0/network/loopback.go#L13-L15)，如下：

```go
    func (l *Loopback) Create(n *Network, nspid int, networkState *NetworkState) error {
            return nil
    }
```

令人费解的是，libcontainer在loopback设备的创建函数Create中，并没有作实质性的内容，而是直接返回nil。 其实关于loopback设备的创建，要回到Linux内核为进程新建namespace的阶段。当libcontainer执行command.Start()时，由于创建了一个新的网络namespace，故Linux内核会自动为新的net namespace创建loopback设备。当Linux内核创建完loopback设备之后，libcontainer所做的工作即只要保留loopback设备的默认配置，并在Initialize函数中实现启动该设备。

#### **4.3.2 veth网络栈的创建**

Veth是Docker Container实际使用的网络策略之一，其使用网桥docker0并创建veth pair虚拟网络设备对，最终使一个veth配置在宿主机上，而另一个veth安置在容器网络namespace内部。 libcontainer中实现veth策略的代码非常通俗易懂，代码位于[./libcontainer/network/veth.go#L19-L50](https://github.com/docker/libcontainer/blob/v1.2.0/network/veth.go#L19-L50)，如下：

```go
    name1, name2, err := createVethPair(prefix)
        if err != nil {
            return err
        }
        if err := SetInterfaceMaster(name1, bridge); err != nil {
            return err
        }
        if err := SetMtu(name1, n.Mtu); err != nil {
            return err
        }
        if err := InterfaceUp(name1); err != nil {
            return err
        }
        if err := SetInterfaceInNamespacePid(name2, nspid); err != nil {
            return err
        }
```

主要的流程包含的四个步骤： (1) 在宿主机上创建veth pair； (2) 将一个veth附加至docker0网桥上； (3) 启动第一个veth； (4) 将第二个veth附加至libcontainer创建进程的namespace下。 使用Create函数实现veth pair的创建之后，在Initialize函数中实现将网络namespace中的veth改名为“eth0”，并设置网络设备的MTU等。

#### **4.3.3 netns网络栈的创建**

netns专门为Docker Container的other container网络模式服务。netns完成的工作是将其他容器的namespace路径传递给需要创建other container网络模式的容器使用。 libcontainer中实现netns策略的代码位于[./libcontainer/network/netns.go#L17-L20](https://github.com/docker/libcontainer/blob/v1.2.0/network/netns.go#L17-L20)，如下：

```go
    func (v *NetNS) Create(n *Network, nspid int, networkState *NetworkState) error {
            networkState.NsPath = n.NsPath
            return nil
    }
```

使用Create函数先将NsPath赋给新建容器之后，在Initialize函数中实现将网络namespace的文件描述符交由新创建容器使用，最终实现两个Docker Container共享同一个网络栈。 通过Create以及Initialize的实现之后，Docker Container相应的网络栈环境即已经完成创建，容器内部的应用进程可以使用不同的网络栈环境与外界或者内部进行通信。关于Initialize函数何时被调用，需要清楚Docker Daemon与dockerinit的关系，以及如何实现Docker Daemon进程与dockerinit进程跨namespace进行通信，这部分内容会在《Docker源码分析》系列后续专文分析。

**5.总结**
--------

如何使用Docker Container的网络，一直是工业界倍加关心的问题。本文将从Linux内核原理的角度阐述了什么是Docker Container，并对Docker Container 4种不同的网络模式进行了初步的介绍，最终贯穿Docker 架构中的多个模块，如Docker Client、Docker Daemon、execdriver以及libcontainer，深入分析Docker Container网络的实现步骤。 

目前，若只谈论Docker，那么它还是只停留在单host宿主机的场景上。如何面对跨host的场景、如何实现分布式Docker Container的管理，目前为止还没有一个一劳永逸的解决方案。再者，一个解决方案的存在，总是会适应于一个应用场景。Docker这种容器技术的发展，大大改善了传统模式下使用诸如虚拟机等传统计算单位存在的多数弊端，却在网络方面使得自身的使用过程中存在瑕疵。希望本文是一个引子，介绍Docker Container网络，以及从源码的角度分析Docker Container网络之后，能有更多的爱好者思考Docker Container网络的前世今生，并为Docker乃至容器技术的发展做出贡献。

**7.参考文献**
----------

[http://docs.studygolang.com/pkg/os/exec/#Cmd](http://docs.studygolang.com/pkg/os/exec/#Cmd) 

[https://github.com/docker/libcontainer/tree/v1.2.0](https://github.com/docker/libcontainer/tree/v1.2.0) 

[https://github.com/docker/libcontainer/issues/323](https://github.com/docker/libcontainer/issues/323) 