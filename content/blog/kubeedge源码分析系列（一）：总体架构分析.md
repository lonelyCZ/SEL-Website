+++
id = "989"

title = "kubeedge源码分析系列（一）：总体架构分析"
description = "kubeedge是华为在KubeCon CloudNativeCon China 2018上发布的面向边缘环境容器管理平台。kubeedge能够接入云端Kubernetes集群，使得边缘端应用的管理可以跟云端应用的管理一样，采用广为熟知的Kubernetes API。SEL实验室准备用一系列文章分析kubeedge的总体架构和各个模块的内部实现机制，本文为系列文章的第一篇，分析kubeedge的版本为1.1后的commit `31e562bc`。"
tags = ["cloudnative","edgecomputing","kubeedge"]
date = "2018-11-29 16:55:37"
author = "丁轶群"
banner = "img/blogs/989/kubeedge_arch.png"
categories = ["Kubernetes"]

+++

kubeedge是华为在KubeCon CloudNativeCon China 2018上发布的面向边缘环境容器管理平台。kubeedge能够接入云端Kubernetes集群，使得边缘端应用的管理可以跟云端应用的管理一样，采用广为熟知的Kubernetes API。 

<!--more-->

SEL实验室准备用一系列文章分析kubeedge的总体架构和各个模块的内部实现机制，本文为系列文章的第一篇，分析kubeedge的版本为1.1后的commit `31e562bc`。



总体架构，kubeedge中的各种模块
-------------------

kubeedge由多个模块（Module，beehive微服务框架中的概念，见后描述）组成，根据不同运行模式，包括edge或site两个模式，可以向华为自己的微服务框架beehive注册运行不同的模块。 

在edge模式下注册模块由`kubeedge/edge/cmd/app`包下的`registerModules`函数完成，注册的模块包括：

1.  `devicetwin`
2.  `edged`
3.  `edgehub`
4.  `eventbus`
5.  `edgemesh`
6.  `metamanager`
7.  `servicebus`
8.  `test`

> registerModules还调用了本地数据库的`InitBDManager`函数

而在site模式下，启动的模块包括：

1.  edged
2.  edgecontroller
3.  metamanager

> site模式下的`registerModules`同样也调用了本地数据库的`InitBDManager`函数

各模块的作用在edgecore的cobra命令描述中：

1.  DeviceTwin is responsible for storing device status and syncing device status to the cloud. It also provides query interfaces for applications.
2.  Edged is an agent that runs on edge nodes and manages containerized applications and devices.
3.  Edgehub is a web socket client responsible for interacting with Cloud Service for the edge computing (like Edge Controller as in the KubeEdge Architecture). This includes syncing cloud-side resource updates to the edge, and reporting edge-side host and device status changes to the cloud.
4.  EventBus is a MQTT client to interact with MQTT servers (mosquito), offering publish and subscribe capabilities to other components.
5.  MetaManager is the message processor between edged and edgehub. It is also responsible for storing/retrieving metadata to/from a lightweight database (SQLite).
6.  ServiceBus is a HTTP client to interact with HTTP servers (REST), offering HTTP client capabilities to components of cloud to reach HTTP servers running at edge.

模块的定义与分组
--------

kubeedge中的模块实现beehive中的模块定义

```go
//beehive/pkg/core/module.go
type Module interface {
    Name() string
    Group() string
    Start(c *context.Context)
    Cleanup()
}
```


其中功能的`Name`和`Group`方法决定了模块所属分组，前面提到的各模块分组情况如下(stub除外)：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605790473/sel/43181605790418_.pic_qgbgqa.png" style="zoom:90%;" />
</center>

模块的初始化
------

各模块的`Register`函数调用beehive的`Register`函数将模块注册到beehive中。根据配置文件中模块是否被启用（`modules.enabled`），beehive将模块加入内部的`modules` map或者`disabledModules` map中加以管理。 

edgecore或edgesite在完成模块注册之后，会调用beehive的`Run`函数启动各模块（`StartModules`）并监听信号（`GracefulShutdown`）以准备好正常关闭。

### StartModules

`StartModules`函数创建模块间通讯机制，并启动协程调用每个模块的`Start`函数。

### GracefulShutdown

GracefulShutdown函数监听系统signal，当接收到如下这些signal时，调用每个模块的`CleanUp`函数

1.  syscall.SIGINT
2.  syscall.SIGHUP
3.  syscall.SIGTERM
4.  syscall.SIGQUIT
5.  syscall.SIGILL
6.  syscall.SIGTRAP
7.  syscall.SIGABRT

模块间通讯
-----

beehive采用golang的channel方式实现模块间通讯，未来可能有基于unix socket的松耦合通讯方式。  

通讯方式包括“单播”和“组播”两种方式，即可以将消息单独发给某个模块，也可以把消息发给模块组（即前面说的`edged`、`hub`、`bus`等组）。 

beehive采用context管理分组与模块间通讯，当使用channel为通讯方式时，`ChannelContext`实现了与context相关的两个接口： `ModuleContext`和`MessageContext`。

```go
// beehive/pkg/core/context/context_channel.go
type ModuleContext interface {
    AddModule(module string)
    AddModuleGroup(module, group string)
    Cleanup(module string)
}
```


`ChannelContext`对`ModuleContext`的实现中，`AddModule`为一个模块创建默认buffer大小为1024的channel，在自己的`channels`成员中将模块名字映射到该channel，用于“单播”。`AddModuleGroup`将一个模块对应的channel添加到所属group内，也就是将`ChannelContext`的`typeChannels[group][module]`设置为模块对应channel，用于“组播”。

```go
// beehive/pkg/core/context/context_channel.go
channels     map[string]chan model.Message
typeChannels map[string]map[string]chan model.Message
```

`ChannelContext`在`MessageChannel`接口的实现中，实现了模块间消息的同步与异步发送，单播组播等。

```go
// beehive/pkg/core/context/context_channel.go
type MessageContext interface {
    // async mode
    Send(module string, message model.Message)
    Receive(module string) (model.Message, error)
    // sync mode
    SendSync(module string, message model.Message, timeout time.Duration) (model.Message, error)
    SendResp(message model.Message)
    // group broadcast
    Send2Group(moduleType string, message model.Message)
    Send2GroupSync(moduleType string, message model.Message, timeout time.Duration) error
}
```

