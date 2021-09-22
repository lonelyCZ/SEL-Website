+++
id = "399"

title = "Docker源码分析（六）：Docker Daemon网络"
description = "**摘要：**Docker的容器特性和镜像特性已然为Docker实践者带来了诸多效益，然而Docker的网络特性却不能让用户满意。本文从Docker的网络模式入手，分析了Docker Daemon创建网络环境的详细流程，其中着重于分析Docker桥接模式的创建，为之后Docker Container创建网络环境做铺垫。"
tags = ["Docker"]
date = "2015-01-05 10:52:28"
author = "孙宏亮"
banner = "img/blogs/399/docker.jpg"
categories = ["Docker"]

+++



**摘要:** Docker的容器特性和镜像特性已然为Docker实践者带来了诸多效益，然而Docker的网络特性却不能让用户满意。本文从Docker的网络模式入手，分析了Docker Daemon创建网络环境的详细流程，其中着重于分析Docker桥接模式的创建，为之后Docker Container创建网络环境做铺垫。

<!--more-->

**前言**
======

Docker作为一个开源的轻量级虚拟化容器引擎技术，已然给云计算领域带来了新的发展模式。Docker借助容器技术彻底释放了轻量级虚拟化技术的威力，让容器的伸缩、应用的运行都变得前所未有的方便与高效。同时，Docker借助强大的镜像技术，让应用的分发、部署与管理变得史无前例的便捷。然而，Docker毕竟是一项较为新颖的技术，在Docker的世界中，用户并非一劳永逸，其中最为典型的便是Docker的网络问题。

毋庸置疑，对于Docker管理者和开发者而言，如何有效、高效的管理Docker容器之间的交互以及Docker容器的网络一直是一个巨大的挑战。目前，云计算领域中，绝大多数系统都采取分布式技术来设计并实现。然而，在原生态的Docker世界中，Docker的网络却是不具备跨宿主机能力的，这也或多或少滞后了Docker在云计算领域的高速发展。 

工业界中，Docker的网络问题的解决势在必行，在此环境下，很多IT企业都开发了各自的新产品来帮助完善Docker的网络。这些企业中不乏像Google一样的互联网翘楚企业，同时也有不少初创企业率先出击，在最前沿不懈探索。这些新产品中有，Google推出的容器管理和编排开源项目Kubernetes，Zett.io公司开发的通过虚拟网络连接跨宿主机容器的工具Weave，CoreOS团队针对Kubernetes设计的网络覆盖工具Flannel，Docker官方的工程师Jérôme Petazzoni自己设计的SDN网络解决方案Pipework，以及SocketPlane项目等。 

对于Docker管理者与开发者而言，Docker的跨宿主机通信能力固然重要，但Docker自身的网络架构也同样重要。只有深入了解Docker自身的网络设计与实现，才能在这基础上扩展Docker的跨宿主机能力。 

Docker自身的网络主要包含两部分：Docker Daemon的网络配置，Docker Container的网络配置。本文主要分析Docker Daemon的网络。

**Docker Daemon网络分析内容安排**
=========================

本文从源码的角度，分析Docker Daemon在启动过程中，为Docker配置的网络环境，章节安排如下：

1.  Docker Daemon网络配置；
2.  运行Docker Daemon网络初始化任务；
3.  创建Docker网桥。

本文为《Docker源码分析系列》第六篇——Docker Daemon网络篇，第七篇将安排Docker Container网络篇。

**Docker Daemon网络配置**
=====================

Docker环境中，Docker管理员完全有权限配置Docker Daemon运行过程中的网络模式。 关于Docker的网络模式，大家最熟知的应该就是“桥接”的模式。下图为桥接模式下，Docker的网络环境拓扑图（包括Docker Daemon网络环境和Docker Container网络环境）：

<center> 
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605702985/sel/17_emmdxy.jpg" alt="17" style="zoom:50%;" />
</center> 
<center> 图3.1 Docker网络桥接示意图</center> 

然而，“桥接”是Docker网络模式中最为常用的模式。除此之外，Docker还为用户提供了更多的可选项，下文将对此一一说来。

**Docker Daemon网络配置接口**
-----------------------

Docker Daemon每次启动的过程中，都会初始化自身的网络环境，这样的网络环境最终为Docker Container提供网络通信服务。 

Docker管理员配置Docker的网络环境，可以在Docker Daemon启动时，通过Docker提供的接口来完成。换言之，可以使用docker二进制可执行文件，运行docker -d并添加相应的flag参数来完成。 

其中涉及的flag参数有EnableIptables、EnableIpForward、BridgeIface、BridgeIP以及InterContainerCommunication。该五个参数的定义位于[./docker/daemon/config.go](https://github.com/docker/docker/blob/v1.2.0/daemon/config.go#L51-L52)，具体代码如下：

~~~go
flag.BoolVar(&config.EnableIptables, []string{"#iptables", "-iptables"}, true, "Enable Docker's addition of iptables rules")
flag.BoolVar(&config.EnableIpForward, []string{"#ip-forward", "-ip-forward"}, true, "Enable net.ipv4.ip_forward")
flag.StringVar(&config.BridgeIP, []string{"#bip", "-bip"}, "", "Use this CIDR notation address for the network bridge's IP, not compatible with -b")
flag.StringVar(&config.BridgeIface, []string{"b", "-bridge"}, "", "Attach containers to a pre-existing network bridge\nuse 'none' to disable container networking")
flag.BoolVar(&config.InterContainerCommunication, []string{"#icc", "-icc"}, true, "Enable inter-container communication")
~~~

以下介绍这5个flag的作用：

*   EnableIptables：确保Docker对于宿主机上的iptables规则拥有添加权限；
*   EnableIpForward：确保net.ipv4.ip\_forward可以使用，使得多网络接口设备模式下，数据报可以在网络设备之间转发；
*   BridgeIP：在Docker Daemon启动过程中，为网络环境中的网桥配置CIDR网络地址；
*   BridgeIface：为Docker网络环境指定具体的通信网桥，若BridgeIface的值为”none”，则说明不需要为Docker Container创建网桥服务，关闭Docker Container的网络能力；
*   InterContainerCommunication：确保Docker容器之间可以完成通信。

除了Docker会使用到的5个flag参数之外，Docker在创建网络环境时，还使用一个DefaultIP变量，如下：

~~~go
opts.IPVar(&config.DefaultIp, []string{"#ip", "-ip"}, "0.0.0.0", "Default IP address to use when binding container ports")
~~~

该变量的作用是：当绑定容器的端口时，将DefaultIp作为默认使用的IP地址。 

具备了以上Docker Daemon的网络背景知识，以下着重举例分析使用BridgeIP和BridgeIface，在启动Docker Daemon时进行网络配置：

![18](https://res.cloudinary.com/rachel725/image/upload/v1605705210/sel/18_sbcq6r.png)


深入理解BridgeIface与BridgeIP，并熟练使用相应的flag参数，即做到了如何配置Docker Daemon的网络环境。需要特别注意的是，Docker Daemon的网络与Docker Container的网络存在很大的区别。Docker Daemon为Docker Container创建网络的大环境，Docker Container的网络需要Docker Daemon的网络提供支持，但不唯一。举一个形象的例子，Docker Daemon可以创建docker0网桥，为之后Docker Container的桥接模式提供支持，然而Docker Container仍然可以根据用户需求创建自身网络，其中Docker Container的网络可以是桥接模式的网络，同时也可以直接共享使用宿主机的网络接口，另外还有其他模式，会在《Docker源码分析》系列的第七篇——Docker Container网络篇中详细介绍。

**Docker Daemon网络初始化**
----------------------

正如上一节所言，Docker管理员可以通过与网络相关的flag参数BridgeIface与BridgeIP，来为Docker Daemon创建网路环境。最简单的，Docker管理员通过执行”docker -d”就已经完成了运行Docker Daemon，而Docker Daemon在启动的时候，根据以上两个flag参数的值，创建相应的网络环境。 

Docker Daemon网络初始化流程图如下：
<center> 
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605705380/sel/19_wgpy5h.jpg" alt="19" style="zoom:50%;" />
</center>

<center> 图 3.2 Docker Daemon网络初始化流程图 </center>

Docker Daemon网络初始化的流程总体而言，主要是根据解析flag参数来决定到底建立哪种类型的网络环境。从流程图中可知，Docker Daemon创建网络环境时有两个分支，不难发现分支代表的分别是：为Docker创建一个网络驱动、以及对Docker的网络不做任何的操作。 

以下参照Docker Daemon网络初始化流程图具体分析实现步骤。

### **启动Docker Daemon传递flag参数**

用户启动Docker Daemon，并在命令行中选择性的传入所需要的flag参数。

### **解析网络flag参数**

flag包对命令行中的flag参数进行解析，其中和Docker Daemon网络配置相关的flag参数有5个，分别是：EnableIptables、EnableIpForward、BridgeIP、BridgeIface以及InterContanierCommunication，各个flag参数的作用上文已有介绍。

### **预处理flag参数**

预处理与网络配置相关的flag参数信息，包括检测配置信息的兼容性、以及判断是否创建Docker网络环境。

首先检验是否会出现彼此不兼容的配置信息，源码位于[./docker/daemon/daemon.go#L679-L685](https://github.com/docker/docker/blob/v1.2.0/daemon/daemon.go#L679-L685)。

这部分的兼容信息有两种。第一种是BridgeIP和BridgeIface配置信息的兼容性，具体表现为用户启动Docker Daemon时，若同时指定了BridgeIP和BridgIface的值，则出现兼容问题。原因为这两者属于互斥对，换言之，若用户指定了新建网桥的设备名，那么该网桥已经存在，无需指定网桥的IP地址BridgeIP；若用户指定了新建网桥的网络IP地址BridgeIP，那么该网桥肯定还没有新建成功，则Docker Daemon在新建网桥时使用默认网桥名“docker0”。具体如下：

~~~go
// Check for mutually incompatible config options
if config.BridgeIface != "" && config.BridgeIP != "" {
    return nil, fmt.Errorf("You specified -b & --bip, mutually exclusive options. Please specify only one.")
}
~~~

第二种是EnableIptables和InterContainerCommunication配置的兼容性，具体是指不能同时指定这两个flag参数为false。原因很简单，如果指定InterContainerCommunication为false，则说明Docker Daemon不允许创建的Docker容器之间互相进行通信。但是为了达到以上目的，Docker正是使用iptables过滤规则。因此，再次设定EnableIptables为false，关闭iptables的使用，即出现了自相矛盾的结果。代码如下：

~~~go
if !config.EnableIptables && !config.InterContainerCommunication {
    return nil, fmt.Errorf("You specified --iptables=false with --icc=false. ICC uses iptables to function. Please set --icc or --iptables to true.")
    }
~~~

检验完系统配置信息的兼容性问题，Docker Daemon接着会判断是否需要为Docker Daemon配置网络环境。判断的依据为BridgeIface的值是否与DisableNetworkBridge的值相等，DisableNetworkBridge在[./docker/daemon/config.go#L13](https://github.com/docker/docker/blob/v1.2.0/daemon/config.go#L13)中被定义为const量，值为字符串”none”。因此，若BridgeIface为”none”，则DisableNetwork为true，最终Docker Daemon不会创建网络环境；若BridgeIface不为”none”，则DisableNetwork为false，最终Docker Daemon需要创建网络环境（桥接模式）。

### **确定Docker网络模式**

Docker网络模式由配置信息DisableNetwork决定。由于在上一环节已经得出DisableNetwork的值，故这一环节可以确定Docker网络模式。该部分的源码实现位于[./docker/daemon/daemon.go#L792-L805](https://github.com/docker/docker/blob/v1.2.0/daemon/daemon.go#L792-L805)，如下：

```go
if !config.DisableNetwork {
    job := eng.Job("init_networkdriver")

    job.SetenvBool("EnableIptables", config.EnableIptables)
    job.SetenvBool("InterContainerCommunication", config.InterContainerCommunication)
    job.SetenvBool("EnableIpForward", config.EnableIpForward)
    job.Setenv("BridgeIface", config.BridgeIface)
    job.Setenv("BridgeIP", config.BridgeIP)
    job.Setenv("DefaultBindingIP", config.DefaultIp.String())

    if err := job.Run(); err != nil {
        return nil, err
    }
}
```
若DisableNetwork为false，则说明需要创建网络环境，具体的模式为创建Docker网桥模式。创建网络环境的步骤为：

1.  创建名为”init\_networkdriver”的job；
2.  为该job配置环境变量，设置的环境变量有EnableIptables、InterContainerCommunication、EnableIpForward、BridgeIface、BridgeIP以及DefaultBindingIP；
3.  运行job。

运行”init\_network”即为创建Docker网桥，这部分内容将会在下一节详细分析。 

若DisableNetwork为true。则说明不需要创建网络环境，网络模式属于none模式。 

以上便是Docker Daemon网络初始化的所有流程。

**创建Docker网桥**
--------------

Docker的网络往往是Docker开发者最常提起的话题。而Docker网络中最常使用的模式为bridge桥接模式。本小节将详细分析创建Docker网桥的创建流程。 

创建Docker网桥的实现通过”init\_network”这个job的运行来完成。”init\_network”的实现为InitDriver函数，位于[./docker/daemon/networkdriver/bridge/driver.go#L79](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L79)，运行流程如下：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605707027/sel/20_esuvnz.jpg" alt="20" style="zoom:55%;" />
</center>
<center>图3.3 Docker Daemon创建网桥流程图</center>

### **提取环境变量**

在InitDriver函数的实现过程中，Docker首先提取”init\_networkdriver”这个job的环境变量。这样的环境变量共有6个，各自的作用在上文已经详细说明。具体的实现代码为：

~~~go
var (
    network        *net.IPNet
    enableIPTables = job.GetenvBool("EnableIptables")
    icc            = job.GetenvBool("InterContainerCommunication")
    ipForward      = job.GetenvBool("EnableIpForward")
    bridgeIP       = job.Getenv("BridgeIP")
)

if defaultIP := job.Getenv("DefaultBindingIP"); defaultIP != "" {
    defaultBindingIP = net.ParseIP(defaultIP)
}

bridgeIface = job.Getenv("BridgeIface")
~~~

### **确定Docker网桥设备名**

提取job的环境变量之后，Docker随即确定最终使用网桥设备的名称。为此，Docker首先创建了一个名为usingDefaultBridge的bool变量，含义为是否使用默认的网桥设备，默认值为false。接着，若环境变量中bridgeIface的值为空，则说明用户启动Docker时，没有指定特定的网桥设备名，因此Docker首先将usingDefaultBridge置为true，然后使用默认的网桥设备名DefaultNetworkBridge，即docker0；若bridgeIface的值不为空，则判断条件不成立，继续往下执行。这部分的代码实现为：

~~~go
usingDefaultBridge := false
if bridgeIface == "" {
    usingDefaultBridge = true
    bridgeIface = DefaultNetworkBridge
}
~~~



### **查找bridgeIface网桥设备**

确定Docker网桥设备名bridgeIface之后，Docker首先通过bridgeIface设备名在宿主机上查找该设备是否真实存在。若存在，则返回该网桥设备的IP地址，若不存在，则返回nil。实现代码位于[./docker/daemon/networkdriver/bridge/driver.go#L99](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L99)，如下：

~~~go
addr, err := networkdriver.GetIfaceAddr(bridgeIface)
~~~

GetIfaceAddr的实现位于`./docker/daemon/networkdriver/utils.go`，实现步骤为：首先通过Golang中net包的InterfaceByName方法获取名为bridgeIface的网桥设备，会得出以下结果：

*   若名为bridgeIface的网桥设备不存在，直接返回error；
*   若名为bridgeIface的网桥设备存在，返回该网桥设备的IP地址。

需要强调的是：GetIfaceAddr函数返回error，说明当前宿主机上不存在名为bridgeIface的网桥设备。而这样的结果会有两种不同的情况：第一，用户指定了bridgeIface，那么usingDefaultBridge为false，而该bridgeIface网桥设备在宿主机上不存在；第二，用户没有指定bridgeIface，那么usingDefaultBridge为true，bridgeIface名为docker0，而docker0网桥在宿主机上也不存在。 

当然，若GetIfaceAddr函数返回的是一个IP地址，则说明当前宿主机上存在名为bridgeIface的网桥设备。这样的结果同样会有两种不同的情况：第一，用户指定了bridgeIface，那么usingDefaultBridge为false，而该bridgeIface网桥设备在宿主机上已经存在；第二，用户没有指定bridgeIface，那么usingDefaultBridge为true，bridgeIface名为docker0，而docker0网桥在宿主机上也已经存在。第二种情况一般是：用户在宿主机上第一次启动Docker Daemon时，创建了默认网桥设备docker0，而后docker0网桥设备一直存在于宿主机上，故之后在不指定网桥设备的情况下，重启Docker Daemon，会出现docker0已经存在的情况。 

以下两小节将分别从bridgeIface已创建与bridgeIface未创建两种不同的情况分析。

### **bridgeIface已创建的情况**

Docker Daemon所在宿主机上bridgeIface的网桥设备存在时，Docker Daemon仍然需要验证用户在配置信息中是否为网桥设备指定了IP地址。 用户启动Docker Daemon时，假如没有指定bridgeIP参数信息，则Docker Daemon使用名为bridgeIface的原有的IP地址。 当用户指定了bridgeIP参数信息时，则需要验证：指定的bridgeIP参数信息与bridgeIface网桥设备原有的IP地址信息是否匹配。若两者匹配，则验证通过，继续往下执行；若两者不匹配，则验证不通过，抛出错误，显示“bridgeIP与已有网桥配置信息不匹配”。该部分内容位于[./docker/daemon/networkdriver/bridge/driver.go#L119-L129](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L119-L129)，代码如下：

~~~go
network = addr.(*net.IPNet)
// validate that the bridge ip matches the ip specified by BridgeIP
if bridgeIP != "" {
    bip, _, err := net.ParseCIDR(bridgeIP)
    if err != nil {
        return job.Error(err)
    }
    if !network.IP.Equal(bip) {
        return job.Errorf("bridge ip (%s) does not match existing bridge configuration %s", network.IP, bip)
    }
}
~~~



### **bridgeIface未创建的情况**

Docker Daemon所在宿主机上bridgeIface的网桥设备未创建时，上文已经介绍将存在两种情况：

*   用户指定的bridgeIface未创建；
*   用户未指定bridgeIface，而docker0暂未创建。

当用户指定的bridgeIface不存在于宿主机时，即没有使用Docker的默认网桥设备名docker0，Docker打印日志信息“指定网桥设备未找到”，并返回网桥未找到的错误信息。代码实现如下：

~~~go
if !usingDefaultBridge {
    job.Logf("bridge not found: %s", bridgeIface)
    return job.Error(err)
}
~~~

当使用的默认网桥设备名，而docker0网桥设备还未创建时，Docker Daemon则立即实现创建网桥的操作，并返回该docker0网桥设备的IP地址。代码如下：

~~~go
// If the iface is not found, try to create it
job.Logf("creating new bridge for %s", bridgeIface)
if err := createBridge(bridgeIP); err != nil {
    return job.Error(err)
}

job.Logf("getting iface addr")
addr, err = networkdriver.GetIfaceAddr(bridgeIface)
if err != nil {
    return job.Error(err)
}
network = addr.(*net.IPNet)
~~~

创建Docker Daemon网桥设备docker0的实现，全部由createBridge(bridgeIP)来实现，createBridge的实现位于[./docker/daemon/networkdriver/bridge/driver.go#L245](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L245)。 createBridge函数实现过程的主要步骤为：

1.  确定网桥设备docker0的IP地址；
2.  通过createBridgeIface函数创建docker0网桥设备，并为网桥设备分配随机的MAC地址；
3.  将第一步中已经确定的IP地址，添加给新创建的docker0网桥设备
4.  启动docker0网桥设备。

以下详细分析4个步骤的具体实现。 首先Docker Daemon确定docker0的IP地址，实现方式为判断用户是否指定bridgeIP。若用户未指定bridgeIP，则从Docker预先准备的IP网段列表addrs中查找合适的网段。具体的代码实现位于[./docker/daemon/networkdriver/bridge/driver.go#L257-L278](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L257-L278)，如下：

~~~go
if len(bridgeIP) != 0 {
    _, _, err := net.ParseCIDR(bridgeIP)
    if err != nil {
        return err
    }
    ifaceAddr = bridgeIP
} else {
    for _, addr := range addrs {
        _, dockerNetwork, err := net.ParseCIDR(addr)
        if err != nil {
            return err
        }
        if err := networkdriver.CheckNameserverOverlaps(nameservers, dockerNetwork); err == nil {
            if err := networkdriver.CheckRouteOverlaps(dockerNetwork); err == nil {
                ifaceAddr = addr
                break
            } else {
                log.Debugf("%s %s", addr, err)
            }
        }
    }
}
~~~



其中为网桥设备准备的候选网段地址addrs为：

~~~go
addrs = []string{
        "172.17.42.1/16", // Don't use 172.16.0.0/16, it conflicts with EC2 DNS 172.16.0.23
        "10.0.42.1/16",   // Don't even try using the entire /8, that's too intrusive
        "10.1.42.1/16",
        "10.42.42.1/16",
        "172.16.42.1/24",
        "172.16.43.1/24",
        "172.16.44.1/24",
        "10.0.42.1/24",
        "10.0.43.1/24",
        "192.168.42.1/24",
        "192.168.43.1/24",
        "192.168.44.1/24",
}
~~~

通过以上的流程的执行，可以确定找到一个可用的IP网段地址，为ifaceAddr；若没有找到，则返回错误日志，表明没有合适的IP地址赋予docker0网桥设备。 

第二个步骤通过createBridgeIface函数创建docker0网桥设备。createBridgeIface函数的实现如下：

~~~go
func createBridgeIface(name string) error {
    kv, err := kernel.GetKernelVersion()
    // only set the bridge's mac address if the kernel version is > 3.3
    // before that it was not supported
    setBridgeMacAddr := err == nil && (kv.Kernel >= 3 && kv.Major >= 3)
    log.Debugf("setting bridge mac address = %v", setBridgeMacAddr)
    return netlink.CreateBridge(name, setBridgeMacAddr)
}
~~~



以上代码通过宿主机Linux内核信息，确定是否支持设定网桥设备的MAC地址。若Linux内核版本大于3.3，则支持配置MAC地址，否则则不支持。而Docker在不小于3.8的内核版本上运行才稳定，故可以认为内核支持配置MAC地址。最后通过netlink的CreateBridge函数实现创建docker0网桥。 

Netlink是Linux中一种较为特殊的socket通信方式，提供了用户应用间和内核进行双向数据传输的途径。在这种模式下，用户态可以使用标准的socket API来使用netlink强大的功能，而内核态需要使用专门的内核API才能使用netlink。 

Libcontainer的netlink包中CreateBridge实现了创建实际的网桥设备，具体使用系统调用的代码如下：

~~~go
syscall.Syscall(syscall.SYS_IOCTL, uintptr(s), SIOC_BRADDBR, uintptr(unsafe.Pointer(nameBytePtr)))
~~~

创建完网桥设备之后，为docker0网桥设备配置MAC地址，实现函数为setBridgeMacAddress。 

第三个步骤是为创建docker0网桥设备绑定IP地址。上一步骤仅完成了创建名为docker0的网桥设备，之后仍需要为docker0网桥设备绑定IP地址。具体代码实现为：

~~~go
if netlink.NetworkLinkAddIp(iface, ipAddr, ipNet); err != nil {
        return fmt.Errorf("Unable to add private network: %s", err)
}
~~~

NetworkLinkAddIP的实现同样位于libcontainer中的netlink包，主要的功能为：通过netlink机制为一个网络接口设备绑定一个IP地址。 第四个步骤是启动docker0网桥设备。具体实现代码为：

~~~go
if err := netlink.NetworkLinkUp(iface); err != nil {
        return fmt.Errorf("Unable to start network bridge: %s", err)
    }
~~~

NetworkLinkUp的实现同样位于libcontainer中的netlink包，功能为启动docker网桥设备。 至此，docker0网桥历经确定IP、创建、绑定IP、启动四个环节，createBridge关于docker0网桥设备的工作全部完成。

### **获取网桥设备的网络地址**

创建完网桥设备之后，网桥设备必然会存在一个网络地址。网桥网络地址的作用为：Docker Daemon在创建Docker Container时，使用该网络地址为Docker Container分配IP地址。 Docker使用代码network = addr.(\*net.IPNet)获取网桥设备的网络地址。

### **配置Docker Daemon的iptables**

创建完网桥之后，Docker Daemon为容器以及宿主机配置iptables，包括为container之间所需要的link操作提供支持，为host主机上所有的对外对内流量制定传输规则等。该部分详情可以参看[《Docker源码分析（四）：Docker Daemon之NewDaemon实现》](http://www.sel.zju.edu.cn/?p=165)。代码位于[./docker/daemon/networkdriver/bridge/driver/driver.go#L133](https://github.com/docker/docker/blob/v1.2.0/daemon/networkdriver/bridge/driver.go#L133)，如下：

~~~go
// Configure iptables for link support
if enableIPTables {
        if err := setupIPTables(addr, icc); err != nil {
            return job.Error(err)
        }
}

// We can always try removing the iptables
if err := iptables.RemoveExistingChain("DOCKER"); err != nil {
        return job.Error(err)
}

if enableIPTables {
        chain, err := iptables.NewChain("DOCKER", bridgeIface)
        if err != nil {
            return job.Error(err)
        }
        portmapper.SetIptablesChain(chain)
}
~~~

### **配置网络设备间数据报转发功能**

在Linux系统上，数据包转发功能是被默认禁止的。数据包转发，就是当host主机存在多个网络设备时，如果其中一个接收到数据包，并需要将其转发给另外的网络设备。通过修改/proc/sys/net/ipv4/ip\_forward的值，将其置为1，则可以保证系统内数据包可以实现转发功能，代码如下：

~~~go
if ipForward {
        // Enable IPv4 forwarding
        if err := ioutil.WriteFile("/proc/sys/net/ipv4/ip_forward", []byte{'1', '\n'}, 0644); err != nil {
            job.Logf("WARNING: unable to enable IPv4 forwarding: %s\n", err)
        }
}
~~~



### **注册网络Handler**

创建Docker Daemon网络环境的最后一个步骤是：注册4个与网络相关的Handler。这4个Handler分别是allocate\_interface、release\_interface、allocate\_port和link，作用分别是为Docker Container分配网络设备，回收Docker Container网络设备、为Docker Container分配端口资源、以及为Docker Container间执行link操作。 至此，Docker Daemon的网络环境初始化工作全部完成。

**总结**
======

在工业界，Docker的网络问题备受关注。Docker的网络环境可以分为Docker Daemon网络和Docker Container网络。本文从Docker Daemon的网络入手，分析了大家熟知的Docker 桥接模式。 

Docker的容器技术以及镜像技术，已经给Docker实践者带来了诸多效益。然而Docker网络的发展依然具有很大的潜力。下一篇Docker Container网络篇，将会带来更为灵活的Docker网络配置。 

5 参考文献 [LINUX netlink机制](http://www.cnblogs.com/iceocean/articles/1594195.html) [Package net](http://docs.studygolang.com/pkg/net/) 

