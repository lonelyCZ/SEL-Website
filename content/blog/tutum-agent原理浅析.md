+++
id = "583"

title = "tutum-agent原理浅析"
description = "tutum-aget是tutum提供的一个开源代理引擎。当你把tutum-agent安装在你本地机器上以后，你就可以把本地的机器节点添加到云端，让tutum来帮你统一进行管理。你随时随地都可以登陆tutum网站获得如下功能。1.启停Docker容器。2.动态伸缩容器实例数量。3.监控容器状态、容器CPU/Memory/Disk/Bandwidth使用量。4.查看容器日志、操作记录。5.与tutum其他服务结合的衍生功能（部署应用、绑定服务等）。"
tags = ["agent"]
date = "2015-06-23 11:14:25"
author = "孙健波"
banner = "img/blogs/583/tutum.jpg"
categories = ["Docker"]

+++
tutum-aget是tutum提供的一个[开源代理引擎](https://github.com/tutumcloud/tutum-agent)。当你把tutum-agent安装在你本地机器上以后，你就可以把本地的机器节点添加到云端，让tutum来帮你统一进行管理。你随时随地都可以登陆tutum网站获得如下功能。

1.  启停Docker容器。
2.  动态伸缩容器实例数量。
3.  监控容器状态、容器CPU\\Memory\\Disk\\Bandwidth使用量
4.  查看容器日志、操作记录
5.  与tutum其他服务结合的衍生功能（部署应用、绑定服务等）

可见，安装了tutum-agent以后相当于把本地的机器加入到了tutum的数据中心进行统一管理。本文将分析tutum-agent的工作原理。





安装
==

tutum-agent通过shell进行安装，安装脚本可以在执行官方的安装指令时下载`curl -Ls https://get.tutum.co/`，也可以在[github源码](https://github.com/tutumcloud/tutum-agent/blob/master/contrib/install-agent.sh)中找到。 这个shell脚本很简单，根据不同linux发行版安装tutum-agent，如果是ubuntu则是执行`apt-get install -yq tutum-agent`，然后生成配置文件`/etc/tutum/agent/tutum-agent.conf`，内容如下。

~~~go
CertCommonName:"<hashID>.node.tutum.io"
DockerHost:"tcp://0.0.0.0:2375"
TutumHost:"https://dashboard.tutum.co/"
TutumToken:"<tokenID>"
TutumUUID:"<UUID>"
~~~

这些配置文件的作用在随后的代码解析中讲解。生成完配置文件后启动该服务。

 `service tutum-agent restart` 

至此shell脚本执行完了，但是安装过程并没有结束，在安装`tutum-agent`时，会因为冲突卸载原先已经安装好的Docker。 

启动后的第一件事就是在TutumHost上注册并验证用户的TutumToken和TutumUUID，获得相应的证书文件，然后从TutumHost上下载Docker的二进制文件，目前Tutum官方提供的是1.5版本的Docker。随后启动Docker daemon，之前下载的证书也主要用在这里。 

启动完Docker以后，就开始下载[Ngrok](https://ngrok.com/)客户端。Ngrok是一个网络代理，它可以让你在防火墙或者NAT映射的私有网络内的服务，让公网可以直接访问。Ngrok的实现方式是，在公网架设一个Ngrok 服务器，私有网络的Ngrok客户端与公网的Ngrok Server保持tcp长连接，用户通过访问Ngrok Server进行代理，从而访问到Ngrok客户端所在机器的服务。使用Ngrok客户端，只需要配置你想要提供服务的应用程序对外服务的端口即可。把这个端neng l口告诉Ngrok客户端，就相当于把本地私有网络下的服务提供给外部。 

至此，所有的服务也就安装完毕了。总结一下，我们实际上安装了tutum-agent、Docker、Ngrok三个应用程序。通过Ngrok把本地的Docker作为服务让公网的其他用户可以使用，tutum-agent在随后会处理包括日志、监控、Docker指令处理等相关内容。

连接
==

安装完Ngrok以后，实际上就相当于建立了穿越NAT网络的隧道，使得公网所在的程序可以直接控制私有网络的Docker服务。Tutum-agent在启动Docker时使用了用户相关的证书，保证了Docker只接受有证书认证的请求，一定程度上保证了本地服务的安全。 

获得了本地Docker服务的控制能力，容器相关的功能的实现自然迎刃而解。Ngrok还会为用户提供命令操作日志以便于审计，所以你可以方便的查看容器的操作记录。 

**建立了连接后我们还需要做什么？** 

通过Ngrok，这一切都变得很简单，但是别忘了，Docker daemon的运行状态需要时刻监控。tutum-agent会对Docker daemon的运行状态周期性的检查，一旦daemon意外推出，可以保证快速检测到并重启。

性能
==

ngrok是一个非常强大的tunnel工具，使用它以后本身带来的代理转发性能损失非常低。 

但是实际上通过ngrok搭建起来的服务，都是应用用户先去访问ngrok服务端，然后再有服务端转发给ngrok客户端，最后处理用户请求并返回。所以，真正让人感知到性能损失的可能是你的ngrok服务端搭建的位置较远，而ngrok客户端与应用用户本身网络较近，这样就容易导致较高的应用访问延迟。

延伸
==

看完了上述内容，可能你会想到，如何构建一个类似Tutum的“Bring your own node”服务，并把它应用到Docker以为的其他项目上。 

第一步，搭建自己的ngrok服务，使得用户可以通过ngrok客户端连接并在任意网络访问本地的服务，可参考博客[“搭建自己的ngrok服务”](http://tonybai.com/2015/03/14/selfhost-ngrok-service/)。

第二步，区分用户。当你的ngrok对外提供服务的时候，会有许多客户端来连接，不同的客户端可能是不同的用户连接的，也有可能是同一个用户的不同应用或不同主机节点。所以你需要在服务端编写自己的判断逻辑，方法很简单。ngrok客户端与服务端建立连接时会生成子域名（或者自定义域名），这个域名一旦建立了连接，就是唯一的，别的用户无法占用，通过这个方法就可以进行最简单的区分。当然，根据你对外提供的服务，你可能还需要通过生成证书来保证服务的安全性。 

第三步，通过API进行操控。当ngrok客户端连接成功后，实际上服务端已经可以连接用户私有网络下服务的端口了，通过端口自然可以访问到服务的API，就基本上可以让用户全局操控自己的服务了。 

第四步，日志、监控、报警与可视化。简单的日志收集分为两部分，一部分自然是ngrok的操作日志，另一部分则是应用相关的日志，需要通过应用API收取。简单来说，监控也分为两部分，一部分是用户主机资源、服务可用性相关的数据监控，另一部分是业务相关的监控。 

基本上，有了这些，一个最基本的服务就完成了。