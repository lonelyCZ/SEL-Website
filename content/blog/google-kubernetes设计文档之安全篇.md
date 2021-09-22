+++
id = "283"

title = "Google Kubernetes设计文档之安全篇"
description = "摘要：Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。本文为其设计文档系列的第一篇：安全。"
tags = ["Kubernetes"]
date = "2014-12-11 21:04:21"
author = "何思玫"
banner = "img/blogs/283/k8s.jpg"
categories = ["Kubernetes"]

+++

摘要：Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。本文为其设计文档系列的第一篇：安全。

<!--more-->

**1.设计目标**
==========

本文讲述了Kubernetes的容器、API和基础设施在安全方面的设计原则。

1.  保证容器与其运行的宿主机之间有明确的隔离；
2.  限制容器对基础设施或者其它容器造成不良影响的能力；
3.  最小特权原则——限定每个组件只被赋予了执行操作所必需的最小特权，由此确保可能产生的损失达到最小；
4.  通过清晰地划分组件的边界来减少需要加固和加以保护的系统组件数量。

**2.设计要点**
==========

**将etcd中的数据与minion节点和基础设施进行隔离**
-------------------------------

在Kubernetes的设计中，如果攻击者可以访问etcd中的数据，那么他就可以在宿主机上运行任意容器，获得存储在volumes或者pods的任何受保护信息（比如访问口令或者作为环境变量的共享密钥），通过中间人攻击来拦截和重定向运行中的服务流量，或者直接删除整个集群的历史信息。

**Kubernetes设计的基本原则是，对etcd中数据的访问权限应该只赋予某些特定的组件，这些组件或者需要对系统有完整的控制权，或者对系统服务变更请求能执行正确的授权和身份验证操作。**将来，etcd会提供粒度访问控制，但这样的粒度要求有一个管理员能够深刻理解etcd中存储数据的schema，并按照schema设置相应的安全策略。管理员必须能够在策略层面上保证Kubernetes的安全性，而非实现层面；另外，随着时间推移，数据的schema可能产生变化，这样的状况应该被预先考虑以免造成意外的安全泄漏。 

**Kubelet和Kube Proxy都需要与它们特定角色相关的信息**——对于Kubelet，需要的是运行的pods集合的信息；对于Kube Proxy，需要用以负载均衡的服务与端点集合信息。Kubelet同样需要提供运行的pods和历史终止数据的相关信息。Kubelet和Kube Proxy用于加载配置的方式是“wait for changes”的HTTP请求。因此，限制Kubelet和Kube Proxy的权限使其只能访问对应角色所需的信息是可行的。 

**Replication controller和其他future controller的controller manager经过用户授权可以代表其执行对Kubernetes资源的自动化维护。Controller manager访问或修改资源状态的权限应该被严格地限定在它们特定的职责范围之内，而不能访问其他无关角色的信息。**例如，一个replication controller只需要如下权限：创建已知pods配置的副本，设定已经存在的pods的运行状态，或者删除它创建的已存在的pods；而不需要知道pods的内容或者当前状态，亦不需要有访问挂载了volume的pods中的数据的权限。 

Kubernetes pod scheduler负责从pod中读取数据并将其注入pod所在集群的minion节点中。它需要的最低限度的权限有，查看pod ID（用以生成binding）、pod当前状态、分配给pod的资源信息等。Pod scheduler不需要修改pods或查看其它资源的权限，只需要创建binding的权限。Pod scheduler不需要删除binding的权限，除非它接管了宕机机器上原有组件的重定位工作。在这样的情况下，scheduler可能需要对用户或者项目容器信息的读取权限来决定重定位pod的优先位置。 

**原文链接：**[Security in Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/design/security.md)（编译/何思玫 审校/孙宏亮）