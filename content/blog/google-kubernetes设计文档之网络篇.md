+++
id = "353"

title = "Google Kubernetes设计文档之网络篇"
description = "Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。其从Docker默认网络模型中独立出来形成了一套自己的网络模型，本文将详细介绍。"
tags = ["Kubernetes"]
date = "2014-12-29 15:32:24"
author = "杜军"
banner = "img/blogs/353/k8s.jpg"
categories = ["Kubernetes"]

+++

**摘要**： Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。其从Docker默认网络模型中独立出来形成了一套自己的网络模型，本文将详细介绍。

<!--more-->

**模型和动机**
=========

Kubernetes从Docker默认的网络模型中独立出来形成一套自己的网络模型。该网络模型的目标是：每一个pod都拥有一个扁平化共享网络命名空间的IP，通过该IP，pod就能够跨网络与其它物理机和容器进行通信。一个pod一个IP模型创建了一个干净、反向兼容的模型，在该模型中，从端口分配、网络、域名解析、服务发现、负载均衡、应用配置和迁移等角度，pod都能够被看成虚拟机或物理机。 

另一方面，动态端口分配需要以下方面的支持： 1，固定端口（例如：用于外部可访问服务）和动态分配端口； 2，分割集中分配和本地获取的动态端口； 不过，这不但使调度复杂化（因为端口是一种稀缺资源），而且应用程序的配置也将变得复杂，具体表现为端口冲突、重用和耗尽。 3，使用非标准方法进行域名解析（例如：etcd而不是DNS）； 4，对使用标准域名/地址解析机制的程序（例如：web浏览器）使用代理和/或重定向。 5，除了监控和缓存实例的非法地址/端口变化外，还要监控用户组成员变化以及阻止容器/pod迁移（例如：使用CRIU）。 NAT将地址空间分段的做法引入了额外的复杂性，这将带来诸如破坏自注册机制等问题。 

在一个pod一个IP模型中，从网络角度看，在一个pod中的所有用户容器都像是在同一台宿主机中那样。它们能够在本地访问其它用户容器的端口。暴露给主机网卡的端口是通过普通Docker方式实现的。所有pod中的容器能够通过他们“10”网段（10.x.x.x）的IP地址进行通信。 

除了能够避免上述动态端口分配带来的问题，该方案还能使应用平滑地从非容器环境（物理机或虚拟机）迁移到同一个pod内的容器环境。在同一台宿主机上运行应用程序栈这种场景已经找到避免端口冲突的方法（例如：通过配置环境变量）并使客户端能够找到这些端口。 该方案确实降低了pod中容器之间的隔离性--尽管端口可能存在冲突而且也不存在pod内跨容器的私有端口，但是对于需要自己的端口范围的应用程序可以运行在不同的pod中，而对于需要进行私有通信的进程则可以运行在同一个容器内。另外，该方案假定的前提条件是：在同一个pod中的容器共享一些资源（例如：磁盘卷、处理器、内存等），因此损失部分隔离性也在可接受范围之内。此外，尽管用户能够指定不同容器归属到同一个pod，但一般情况下不能指定不同pod归属于同一台主机。 

当任意一个容器调用SIOCGIFADDR（发起一个获取网卡IP地址的请求）时，它所获得的IP和与之通信的容器看到的IP是一样的--每个pod都有一个能够被其它pod识别的IP。通过无差别地对待容器和pdo内外部的IP和端口，我们创建了一个非NAT的扁平化地址空间。"ip addr show"能够像预期那样正常工作。该方案能够使所有现有的域名解析/服务发现机制：包括自注册机制和分配IP地址的应用程序在容器外能够正常运行（我们应该用etcd、Euraka（用于Acme Air）或Consul等软件测试该方案）。对pod之间的网络通信，我们应该持乐观态度。在同一个pod中的容器之间更倾向于通过内存卷（例如：tmpfs）或IPC（进程间通信）的方式进行通信。 

该模型与标准Docker模型不同。在该模型中，每个容器会得到一个“172”网段（172.x.x.x）的IP地址，而且通过SIOCGIFADDR也只能看到一个“172”网段（172.x.x.x）的IP地址。如果这些容器与其它容器连接，对方容器看到的IP地址与该容器自己通过SIOCGIFADDR请求获取的IP地址不同。简单地说，你永远无法在容器中注册任何东西或服务，因为一个容器不可能通过其私有IP地址被外界访问到。 

我们想到的一个解决方案是增加额外的地址层：以pod为中心的一个容器一个IP模式。每个容器只拥有一个本地IP地址，且只在pod内可见。这将使得容器化应用程序能够更加容易地从物理/虚拟机迁移到pod，但实现起来很复杂（例如：要求为每个pod创建网桥，水平分割/虚拟私有的 DNS），而且可以预见的是，由于新增了额外的地址转换层，将破坏现有的自注册和IP分配机制。

**当前实现**
========

Google计算引擎（GCE）集群配置了[高级路由](https://cloud.google.com/compute/docs/networking#routing)，使得每个虚拟机都有额外的256个可路由IP地址。这些是除了分配给虚拟机的通过NAT用于访问互联网的“主”IP之外的IP。该实现在Docker外部创建了一个叫做cbr0的网桥（为了与docker0网桥区别开），该网桥只负责对从容器内部流向外部的网络流量进行NAT转发。 

目前，从“主”IP（即互联网，如果制定了正确的防火墙规则的话）发到映射端口的流量由Docker的用户模式进行代理转发。未来，端口转发应该由Kubelet或Docker通过`iptables`进行：[Issue #15](https://github.com/GoogleCloudPlatform/kubernetes/issues/15)。 

启动Docker时附加参数：DOCKER\_OPTS="--bridge cbr0 --iptables=false"。 

并用SaltStack在每个node中创建一个网桥，代码见container\_bridge.py：

```python
cbr0:
  container_bridge.ensure:
  - cidr: {{ grains['cbr-cidr'] }}
...
grains:
  roles:
  - kubernetes-pool
  cbr-cidr: $MINION_IP_RANGE
```
在GCE中，我们让以下IP地址可路由： 

```shell
gcloud compute routes add "${MINION_NAMES[$i]}" \
  --project "${PROJECT}" \
  --destination-range "${MINION_IP_RANGES[$i]}" \
  --network "${NETWORK}" \
  --next-hop-instance "${MINION_NAMES[$i]}" \
  --next-hop-instance-zone "${ZONE}" &
```
以上代码中的`MINION_IP_RANGES`是以10.开头的24位网络号IP地址空间（10.x.x.x/24）。 

尽管如此，GCE本身并不知道这些IP地址的任何信息。 

这些IP地址不是外部可路由的，因此，那些有与外界通信需求的容器需要使用宿主机的网络。如果外部流量要转发给虚拟机，它只会被转发给虚拟机的主IP（该IP不会被分配给任何一个pod），因此我们使用docker的-p标记把暴露的端口映射到主网卡上。该方案的带来的副作用是不允许两个不同的pod暴露同一个端口（更多相关讨论见[Issue #390](https://github.com/GoogleCloudPlatform/kubernetes/issues/390)）。 

我们创建了一个容器用于管理pod的网络命名空间，该网络容器内有一个本地回环设备和一块虚拟以太网卡。所有的用户容器从该pod的网络容器那里获取他们的网络命名空间。 

Docker在它的“container”网络模式下从网桥（我们为每个节点上创建了一个网桥）那里分配IP地址，具体步骤如下： 

1，使用最小镜像创建一个普通容器（从网络角度）并运行一条永远阻塞的命令。这不是一个用户定义的容器，给它一个特别的众所周知的名字。

*   创建一个新的网络命名空间（netns）和本地回路设备
*   创建一对新的虚拟以太网设备并将它绑定到刚刚创建的网络命名空间
*   从docker的IP地址空间中自动分配一个IP

2，在创建用户容器时指定网络容器的名字作为它们的“net”参数。Docker会找到在网络容器中运行的命令的PID并将它绑定到该PID的网络命名空间去。

**其他网络实现例子**
============

其他实现方案用于在GCE外部实现一个pod一个IP模型的目标。

*   [OpenVSwitch with GRE/VxLAN](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/ovs-networking.md)
*   [Flannel](https://github.com/coreos/flannel#flannel)

**挑战和未来的工作**
============

**Docker API**
--------------

目前，docker inspect并不展示容器的网络配置信息，因为它们从另一个容器提取该信息。该信息应该以某种方式暴露出来。

**外部IP分配**
----------

我们希望能够从Docker外部分配IP地址（[Docker issue #6743](https://github.com/dotcloud/docker/issues/6743)），这样我们就不必静态地分配固定大小的IP范围给每个节点，而且即使网络容器重启也能保持容器IP地址固定不变（[Docker issue #2801](https://github.com/dotcloud/docker/issues/2801)），这样就使得pod易于迁移。目前，如果网络容器挂掉，所有的用户容器必须关闭再重启，因为网络容器重启后其网络命名空间会发生变化而且任何一个随后重启的用户容器会加入新的网络命名空间进而导致无法与对方容器通信。另外，IP地址的改变将会引发DNS缓存/生存时间问题。外部IP分配方案会简化DNS支持（见下文）。

**域名解析，服务发现和负载均衡**
------------------

除了利用第三方的发现机制来启用自注册外，我们还希望自动建立DDNS（[Issue #146](https://github.com/GoogleCloudPlatform/kubernetes/issues/146)）。hostname方法，$HOSTNAME等应该返回一个pod的名字（[Issue #298](https://github.com/GoogleCloudPlatform/kubernetes/issues/298)），gethostbyname方法应该要能够解析其他pod的名字。我们可能会建立一个DNS解析服务器来做这些事情（[Docker issue #2267)](https://github.com/dotcloud/docker/issues/2267)），所以我们不必动态更新/etc/hosts文件。 

端点目前是通过环境变量获取的。[Docker链接兼容](https://docs.docker.com/userguide/dockerlinks/)变量和kubernetes指定变量（{NAME}\_SERVICE\_HOST和{NAME}\_SERVICE\_BAR）都是支持的，并会被解析成由服务代理打开的端口。事实上，我们并不使用[docker特使模式](https://docs.docker.com/articles/ambassador_pattern_linking/)来链接容器因为我们不需要应用程序在配置阶段识别所有客户端。虽然如今的服务都是由服务代理管理的，但是这是一个应用程序不应该依赖的实现细节，客户端应该使用服务入口IP（以上环境变量会被解析成该IP）。然而，一个扁平化的服务命名空间无法伸缩而且环境变量不允许动态更新，这使得通过应用隐性的顺序限制进行服务部署变得复杂。我们打算在DNS中为每个服务注册入口IP，并且希望这种方式能够成为首选解决方案。 

我们也希望适应其它的负载均衡方案（例如：HAProxy），非负载均衡服务（[Issue #260](https://github.com/GoogleCloudPlatform/kubernetes/issues/260)）和其它类型的分组（例如：线程池等）。提供监测应用到pod地址的标记选择器的能力使有效地监控用户组成员状态变得可能，这些监控数据将直接被服务发现机制消费或同步。用于监听/取消监听事件的事件钩（[Issue #140](https://github.com/GoogleCloudPlatform/kubernetes/issues/140)）将使上述目标更容易实现。

**外部可路由性**
----------

我们希望跨节点的容器间网络通信使用pod IP地址。假设节点A的容器IP地址空间是10.244.1.0/24，节点B的容器的IP地址空间是10.244.2.0/24，且容器A1的的IP地址是10.244.1.1，容器B1的IP地址是10.244.2.1。现在，我们希望容器A1能够直接与容器B1通信而不通过NAT，那么容器B1看到IP包的源IP应该是10.244.1.1而不是节点A的主宿主机IP。这意味着我们希望关闭用于容器之间（以及容器和虚拟机之间）通信的NAT。 

我们也希望从外部互联网能够直接路由到pod。然而，我们还无法提供额外的用于直接访问互联网的容器IP。因此，我们不将外部IP映射到容器IP。我们通过让终点不是内部网络（!10.0.0.0/8）的流量经过主宿主机IP走NAT来解决这个问题，这样与因特网通信时就能通过GCE网络实现1：1NAT转换。类似地，从因特网流向内部网络的流量也能够经宿主机IP实现NAT转换/代理。 因此，让我们用三个用例场景结束上面的讨论：

1.  容器->容器或容器虚拟机。该场景需要直接使用以10.开头的地址（10.x.x.x）并且不需要用到NAT。
2.  容器->因特网。容器IP需要映射到主宿主机IP好让GCE知道如何向外发送这些流量。这里就有两层NAT：容器IP->内部宿主机IP->外部主机IP。第一层结合IP表发生在客户容器上，第二层则是GCE网络的一部分。第一层（容器IP->内部宿主机IP）进行了动态端口分配，而第二层的端口映射是1：1的。
3.  因特网->容器。该场景的流量必须通过主宿主机IP而且理想状态下也拥有两层NAT。但是，目前的流量路径是：外部主机IP -> 内部主机IP -> Docker) -> (Docker -> 容器IP)，即Docker是中间代理。一旦（[issue #15](https://github.com/GoogleCloudPlatform/kubernetes/issues/15)）被解决，那么就应该是：外部主机IP -> 内部主机IP -> 容器IP。但是，为了实现后者，就必须为每个管理的端口建立端口转发的防火墙（iptables）。

如果我们有办法让外部IP路由到集群中的pod，我们就可以采用为每个pod创建一个新的宿主机网卡别名的方案。该方案能够消除使用宿主机IP地址带来的调度限制。 

**IPv6** 

IPv6将会是一个有意思的选项，但我们还没法使用它。支持IPv6已经被提到Docker的日程上了： [Docker issue #2974](https://github.com/dotcloud/docker/issues/2974), [Docker issue #6923](https://github.com/dotcloud/docker/issues/6923), [Docker issue #6975](https://github.com/dotcloud/docker/issues/6975)。另外，主流云服务提供商（例如：AWS EC2，GCE）目前也不支持直接给虚拟机绑定IPv6地址。尽管如此，我们将很高兴接受在物理机上运行Kubernetes的用户提交的代码。:) 

**原文链接：**[Networking](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/design/networking.md)（编译/杜军 审校/张磊）