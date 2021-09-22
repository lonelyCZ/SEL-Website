+++
id = "394"

title = "Google Kubernetes设计文档之Volumes"
description = "Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。本文描述了Kubernetes中Volumes的使用情况，Volume是一个能够被容器访问的目录。"
tags = ["Kubernetes"]
date = "2015-01-02 10:56:38"
author = "仇臣"
banner = "img/blogs/394/k8s.jpg"
categories = ["Kubernetes"]

+++

**摘要：**Kubernetes是Google开源的容器集群管理系统，构建于Docker之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等功能。本文描述了Kubernetes中Volumes的使用情况，Volume是一个能够被容器访问的目录。

<!--more-->


**Volumes**
===========

本文描述了Kubernetes中Volumes的使用情况，建议在阅读本文前，首先熟悉pods。 

Volume是一个能够被容器访问的目录，它可能还会包含一些数据。Kubernetes Volumes与[Docker Volumes](https://docs.docker.com/userguide/dockervolumes/)类似，但并不完全相同。 

一个Pod会在它的[ContainerManifest](https://developers.google.com/compute/docs/containers/container_vms#container_manifest) 属性中指明其容器需要哪些Volumes。 

容器中的进程可见的文件系统视图由两个源组成：一个单独的Docker image和零个或多个Volumes。[Docker image](https://docs.docker.com/userguide/dockerimages/)位于文件层次结构的根部。所有的Volumes都挂载在Docker image的节点上。Volumes不能挂载在其他的Volumes上，也没有连接其他Volumes的硬链接。Pod中的每个容器都单独地指明了其image挂载的Volume。这会通过VolumeMount属性来确定。

**资源**
======

Volume的存储介质(如硬盘、固态硬盘或内存)是由保存kubelet根目录(一般为/var/lib/kubelet)的文件系统的存储介质决定的。一个EmptyDir或者PersistentDir类型的Volume可以使用多少空间是没有限制的，同时在容器或者pods间也不存在隔离。 

将来，我们预计一个Volume将能够通过使用资源规范来请求一个确定大小的空间；同时，对于包含多种存储介质的集群，将可以选择Volume使用的介质类型。

**Volumes的类型**
==============

Kubernetes现在支持三种类型的Volumes，但将来会支持更多的类型。

**EmptyDir**
------------

一个EmptyDir Volume是在Pod绑定到Node时创建的。当第一条容器命令启动时，它的初始状态为空。在同一个Pod上的容器可以读写EmptyDir中的相同文件。当一个Pod被解绑，在EmptyDir中的数据将永久性删除。 EmptyDir的一些用途如下：

*   暂存空间，例如用于基于磁盘的归并排序或者长计算的检查点；
*   一个目录，由一个内容管理容器填充数据，同时由一个网络服务器容器供应数据。

目前，用户无法控制EmptyDir使用的介质种类。如果Kubelet的配置是使用硬盘，那么所有的EmptyDirectories都将创建在该硬盘上。将来，可以预料的是Pods将可以控制EmptyDir是位于硬盘、固态硬盘还是基于内存的tmpfs上。

**HostDir**
-----------

一个HostDir的Volume将可以访问当前宿主机节点上的文件。 

HostDir的一些用途如下：

*   运行一个需要访问Docker内部结构的容器；可以访问/var/lib/docker这个HostDir；
*   在容器中运行cAdvisor；可以访问/dev/cgroups这个HostDir。

当使用该类型的Volume时，需要格外小心，因为：

*   具有相同配置的pods(例如由同一个podTemplate创建的pods)可能在不同宿主节点上由于宿主机上的目录和文件不同而有着不同的访问结果；
*   当Kubernetes增加资源敏感调度，按其计划，它将不能考虑到HostDir使用的资源。

**GCEPersistentDisk**
---------------------

重要提示：必须创建并格式化一个永久磁盘(PD)才能使用GCEPersistentDisk。 

拥有GCEPersistentDisk的Volume可以访问谷歌计算引擎（Google Compute Engine, GCE）的[永久磁盘](http://cloud.google.com/compute/docs/disks)上的文件。 

使用GCEPersistentDisk时，有一些限制条件：

*   节点(运行kubelet的节点)需要是GCE虚拟机；
*   这些虚拟机需要在相同的GCE项目中，同时被划作PD；
*   避免使用相同Volume来创建多个pods
    *   如果多个pods引用相同的Volume，并且都部署在同一台机器上，不论它们是只读还是可读写的，那么第二个pod的部署都将失败；
    *   只有使用只读加载的文件系统的pod才能创建复制控制器。

**创建一个PD**
----------

在你能够在pod上使用GCE PD前，你需要先创建并格式化它。 

我们正在积极努力得使这个过程更加精简容易。

```shell
DISK_NAME=my-data-disk
    DISK_SIZE=500GB
    ZONE=us-central1-a
    gcloud compute disks create --size=$DISK_SIZE --zone=$ZONE $DISK_NAME
    gcloud compute instances attach-disk --zone=$ZONE --disk=$DISK_NAME --device-name temp-data kubernetes-master
    gcloud compute ssh --zone=$ZONE kubernetes-master \
      --command "sudo mkdir /mnt/tmp && sudo /usr/share/google/safe_format_and_mount /dev/disk/by-id/google-temp-data /mnt/tmp"
    gcloud compute instances detach-disk --zone=$ZONE --disk $DISK_NAME kubernetes-master
```

GCE PD的配置实例：

```yaml
apiVersion: v1beta1
    desiredState:
      manifest:
        containers:
          - image: kubernetes/pause
            name: testpd
            volumeMounts:
              - mountPath: "/testpd"
                name: "testpd"
        id: testpd
        version: v1beta1
        volumes:
          - name: testpd
            source:
              persistentDisk:
                # This GCE PD must already exist and be formatted ext4
                pdName: test
                fsType: ext4
    id: testpd
    kind: Pod
```

**Kubernetes的Volumes使用实例-Walkthrough**
======================================

对于永久存储，首先，我们知道容器的文件系统与容器有着同等的生命周期，所以，我们还需要有永久存储。为了实现这个目的，你需要声明一个Volume，把它作为Pod的一部分，同时将它挂载到一个容器上。

```yaml
apiVersion: v1beta1
    desiredState:
      manifest:
        containers:
          - image: kubernetes/pause
            name: testpd
            volumeMounts:
              - mountPath: "/testpd"
                name: "testpd"
        id: testpd
        version: v1beta1
        volumes:
          - name: testpd
            source:
              persistentDisk:
                # This GCE PD must already exist and be formatted ext4
                pdName: test
                fsType: ext4
    id: testpd
    kind: Pod
```

那么，我们该如何做呢？我们为Pod增加一个Volume：

```yaml
...
    volumes:
      	- name: redis-persistent-storage
        source:
          emptyDir: {}
...
```

然后，我们为该Volume在容器中增加一个对应的挂载路径：

```yaml
...
        volumeMounts:
            # name must match the volume name below
          	- name: redis-persistent-storage
            # mount path within the container
            mountPath: /data/redis
...
```

在Kubernetes中， EmptyDir Volumes在Pod的生命周期内一直可用，该周期比任何一个容器的生命周期都长，所以即使容器失效并重启，我们的永久存储将仍然可用。 

如果你希望挂载一个已经存在于文件系统的目录(例如/var/logs)，你可以直接使用HostDir。 

原文链接：[Kubernetes Volumes](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/volumes.md)（编译/仇臣 审校/张磊）