+++
id = "27"

title = "基于KubeEdge的边缘节点分组管理设计与实现"
description = "KubeEdge 1.11版本提供了“边缘节点分组管理”新特性，抽象出了跨地域的应用部署模型。该模型将边缘节点按地区划分为节点组，并将应用所需资源打包成一个整体在节点组上进行部署，降低了边缘应用生命周期管理的复杂度，有效减少运维成本。"
tags = [ "kubeedge" ]
date = "2022-08-26 20:09:00"
author = "鲍玥&张逸飞"
banner = "img/blogs/27/overview.png"
categories = [ "kubeedge" ]

+++

KubeEdge 1.11版本提供了“边缘节点分组管理”新特性，抽象出了跨地域的应用部署模型。该模型将边缘节点按地区划分为节点组，并将应用所需资源打包成一个整体在节点组上进行部署，降低了边缘应用生命周期管理的复杂度，有效减少运维成本。

<!--more-->

## 1. 边缘应用跨地域部署面临的挑战

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662090171/sel/640_euw291.png" 
    alt="边缘应用跨地域部署示意图" style="zoom:75%;"/>
    <figcaption>图1 边缘应用跨地域部署示意图</figcaption>
</figure>

在边缘计算场景中，边缘节点通常分布在不同的地理区域，这些区域中的节点有着计算资源、网络结构和硬件平台等属性上的差异。如图1所示，边缘节点部署在杭州、北京和上海等地域，各地域边缘节点的规模不同，不同地域网络不互通，以及不同区域镜像仓库也是不同的，如北京的节点无法通过IP直接访问其他区域的节点。因此在部署边缘应用的时候，通常需要为每个这样的地理区域维护一个Deployment，对于资源少的区域减少副本数量，对于局域网中的节点需要把镜像地址改为本地镜像仓库的地址，同样也需要为每个地区管理单独的Service资源，来解决跨地域节点之间的访问问题。然而随着地理区域和应用数量的增长，对应用的管理会变得越来越复杂，运维成本也随之增加。基于以上背景，KubeEdge提供了边缘节点分组管理能力，来解决在跨地域应用部署中运维复杂度的问题。

## 2. 边缘节点分组管理设计与实现

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098569/sel/640_dwtdtv.png" 
    alt="边缘节点分组整体概览" style="zoom:75%;"/>
    <figcaption>图2 边缘节点分组整体概览</figcaption>
</figure>

如图2所示，边缘节点分组特性的整体设计图，主要由节点分组、边缘应用和流量闭环三个部分的内容组成，下面会就以上各个部分详细展开。

### 2.1 节点分组（NodeGroup）

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098587/sel/640_p5mzkr.png" 
    alt="节点分组示例" style="zoom:75%;"/>
    <figcaption>图3 节点分组示例</figcaption>
</figure>

根据边缘节点的地理分布特点，可以把同一区域的边缘节点分为一组，将边缘节点以节点组的形式组织起来，同一节点同时只能属于一个节点组。节点分组可以通过matchLabels字段，指定节点名或者节点的Label两种方式对节点进行选择。节点被包含到某一分组后，会被添加上apps.kubeedge.io/belonging-to:nodegroup的Label。

### 2.2 边缘应用（EdgeApplication）

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098596/sel/640_r6de7o.png" 
    alt="边缘应用EdgeApplication的组成" style="zoom:75%;"/>
    <figcaption>图4 边缘应用EdgeApplication的组成</figcaption>
</figure>

边缘应用用于将应用资源打包，按照节点组进行部署，并满足不同节点组之间的差异化部署需求。该部分引入了一个新的CRD: EdgeApplication，主要包括两个部分：

1. Workload Templates。主要包括边缘应用所需要的资源模板，例如Deployment Template、Service Template和ConfigMap Template等；
2. WorkloadScopes。主要针对不同节点组的需求，用于资源模板的差异化配置，包括副本数量差异化配置（Replicas Overrider）和镜像差异化配置（Image Overrider），其中Image Overrider包括镜像仓库地址、仓库名称和标签。

对于应用主体，即Deployment，会根据Deployment Template以及差异化配置Overrider生成每组所需的Deployment版本，通过调整nodeSelector将其分别部署到指定分组中。对于应用依赖的其他资源，如ConfigMap和Service，则只会在集群中通过模板创建一个相应的资源。边缘应用会对创建的资源进行生命周期管理，当删除边缘应用时，所有创建的资源都会被删除。

### 2.3 流量闭环

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098607/sel/640_kjultw.png" 
    alt="流量闭环示意图" style="zoom:75%;"/>
    <figcaption>图5 流量闭环示意图</figcaption>
</figure>

通过流量闭环的能力，将服务流量限制在同一节点组内，在一个节点组中访问Service时，后端总是在同一个节点组中。当使用EdgeApplication中的Service Template创建Service时，会为Service添上service-topology:range-nodegroup的annotation，KubeEdge云上组件CloudCore会根据该annotation对Endpoint和Endpointslice进行过滤，滤除不在同一节点组内的后端，之后再下发到边缘节点。

此外，在下发集群中默认的Master Service “Kubernetes”所关联的Endpoint和Endpointslice时，会将其维护的IP地址修改为边缘节点MetaServer地址，用户在边缘应用中list/watch集群资源时，可以兼容K8s流量访问方式，实现无缝迁移和对接。

## 3. 实现原理与设计理念

在这个部分，我们会分享一下边缘节点分组管理特性的设计理念，并结合KubeEdge整体架构，详细介绍一下我们的实现原理。

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098625/sel/640_rk2cbk.png" 
    alt="设计理念" style="zoom:75%;"/>
    <figcaption>图6 设计理念</figcaption>
</figure>

我们希望给用户提供一个统一的运维入口，原本我们需要维护各个地区的Deployment，如果需要进行增删改查操作，我们需要对每个地区的Deployment都执行一遍相同的操作，不仅增加了运维成本，还容易引入人为操作的错误。边缘节点分组管理特性通过引入EdgeApplication CRD，统一了Deployment等资源的运维入口。

另外我们需要提供更大的扩展可能性，在内部实现中，我们统一使用了Unstructured结构，降低与特定资源的耦合度，方便后续添加其他资源。另外为了不干涉原生资源和流程，我们降低与Kubernetes Reconciliation的耦合度，可以保证Deployment等资源操作过程的原生性。

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098636/sel/640_bixgqp.png" 
    alt="节点组和边缘应用实现" style="zoom:75%;"/>
    <figcaption>图7 节点组和边缘应用实现</figcaption>
</figure>

在边缘节点分组管理特性中，我们引入了两个CRD，分别是节点组NodeGroup和边缘应用EdgeApplication。在NodeGroup Reconciliation中，NodeGroup Controller用于监听NodeGroup CRD的变化，并对节点的apps.kubeedge.io/belonging-to:nodegroup Label进行增删改等操作，同时，加入节点组的节点，会上报状态到NodeGroup CRD中，我们就可以通过查询NodeGroup直接查看节点组内所有节点的状态。

EdgeApplication Reconciliation与NodeGroup Reconciliation类似，由EdgeApplication Controller来监听EdgeApplication CRD的变化，对相应资源进行增删改等操作，同时对应资源会上报状态到EdgeApplication CRD中。

<figure align="center">
    <img
    src="https://res.cloudinary.com/rachel725/image/upload/v1662098653/sel/640_vv8ou3.png" 
    alt="整体架构" style="zoom:75%;"/>
    <figcaption>图8 整体架构</figcaption>
</figure>

如图8所示，是最终的整体架构图。在边缘节点分组管理特性中，我们引入了新的组件ControllerManager，其中包括了刚才我们介绍的NodeGroup Controller和EdgeApplication Controller，在CloudCore中引入了新的模块EndpointSlice Filter，用于实现流量闭环的能力。

图中蓝色区域是前面已经介绍了的节点分组和边缘应用的内容，在这里再重点介绍一下Service Template实现流量闭环能力的过程。首先在EdgeApplication CRD中加入Service的模板，在创建边缘应用时，Service range-nodegroup资源也会随之生成，同时控制面会自动为其创建EndpointSlice。EndpointSlice会通过KubeEdge的云边通道下发到边缘节点，CloudCore中的EndpointSlice Filter会进行过滤，保证下发到同一节点组内的边缘节点，由此可以保证边缘上的客户端访问始终在一个节点组内。

对于用户来说，图8中紫色的线表达了用户需要维护的资源。首先用户需要维护NodeGroup，来管理节点组中的节点；其次，用户需要维护EdgeApplication资源，通过EdgeApplication来实现对各个地域边缘应用的生命周期管理。

## 4. 发展规划

目前KubeEdge社区已经实现了Deployment、Service和ConfigMap等资源的打包以及流量闭环的能力，并且支持资源的部分状态收集；未来将继续拓展边缘节点分组的能力，实现边缘网关，支持StatefulSet等更多资源，逐步完善应用状态收集，并在Kubectl中支持更友好的资源展现形式。欢迎大家能够加入KubeEdge社区，一起完善与增强KubeEdge边缘节点分组等方面的能力。

**本文作者：**

KubeEdge社区Member：华为云 鲍玥 ; 浙江大学SEL实验室 张逸飞 