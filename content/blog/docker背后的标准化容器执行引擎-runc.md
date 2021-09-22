+++

id= "633"

title = "Docker背后的标准化容器执行引擎——runC"
description = "本文从OCI组织的成立开始讲起，描述了开放容器格式的标准及其宗旨，这其实就是runC的由来。继而针对具体的runC特性及其启动进行了详细介绍。OCI的成立表明了社区及各大厂商对容器技术的肯定以及加快容器技术发展进步的强烈决心，相信在不久的将来，符合OCI标准的开放容器项目会越来越多，容器技术将更加欣欣向荣地不断前进。"
tags= [ "Docker" , "runc" , "runtime" ]
date= "2015-10-21 19:44:38"
author = "孙健波"
banner= "img/blogs/633/logo_oci.png"
categories = [ "Docker" ]

+++

随着容器技术发展的愈发火热，Linux基金会于2015年6月成立[OCI（Open Container Initiative）](https://www.opencontainers.org/)组织，旨在围绕容器格式和运行时制定一个开放的工业化标准。该组织一成立便得到了包括谷歌、微软、亚马逊、华为等一系列云计算厂商的支持。而runC就是Docker贡献出来的，按照该开放容器格式标准（OCF, Open Container Format）制定的一种具体实现。

<!--more-->

1\. 容器格式标准是什么？
--------------

制定容器格式标准的**宗旨**概括来说就是不受上层结构的绑定，如特定的客户端、编排栈等，同时也不受特定的供应商或项目的绑定，即不限于某种特定操作系统、硬件、CPU架构、公有云等。 该标准目前由[libcontainer](https://github.com/docker/libcontainer)和[appc](https://github.com/appc/spec)的项目负责人（maintainer）进行维护和制定，其规范文档就作为一个项目在Github上维护，地址为[https://github.com/opencontainers/specs](https://github.com/opencontainers/specs)。

### 1.1 容器标准化宗旨

标准化容器的宗旨具体分为如下五条。

*   **操作标准化**：容器的标准化操作包括使用标准容器感觉创建、启动、停止容器，使用标准文件系统工具复制和创建容器快照，使用标准化网络工具进行下载和上传。
    
*   **内容无关**：内容无关指不管针对的具体容器内容是什么，容器标准操作执行后都能产生同样的效果。如容器可以用同样的方式上传、启动，不管是php应用还是mysql数据库服务。
    
*   **基础设施无关**：无论是个人的笔记本电脑还是AWS S3，亦或是Openstack，或者其他基础设施，都应该对支持容器的各项操作。
    
*   **为自动化量身定制**：制定容器统一标准，是的操作内容无关化、平台无关化的根本目的之一，就是为了可以使容器操作全平台自动化。
    
*   **工业级交付**：制定容器标准一大目标，就是使软件分发可以达到工业级交付成为现实。
    

### 1.2 容器标准包（bundle）和配置

一个标准的容器包具体应该至少包含三块部分：

*   `config.json`： 基本配置文件，包括与宿主机独立的和应用相关的特定信息，如安全权限、环境变量和参数等。具体如下：
    *   容器格式版本
    *   rootfs路径及是否只读
    *   各类文件挂载点及相应容器内挂载目录（此配置信息必须与`runtime.json`配置中保持一致）
    *   初始进程配置信息，包括是否绑定终端、运行可执行文件的工作目录、环境变量配置、可执行文件及执行参数、uid、gid以及额外需要加入的gid、hostname、低层操作系统及cpu架构信息。
*   `runtime.json`： 运行时配置文件，包含运行时与主机相关的信息，如内存限制、本地设备访问权限、挂载点等。除了上述配置信息以外，运行时配置文件还提供了“钩子(hooks)”的特性，这样可以在容器运行前和停止后各执行一些自定义脚本。hooks的配置包含执行脚本路径、参数、环境变量等。
*   `rootfs/`：根文件系统目录，包含了容器执行所需的必要环境依赖，如`/bin`、`/var`、`/lib`、`/dev`、`/usr`等目录及相应文件。rootfs目录必须与包含配置信息的`config.json`文件同时存在容器目录最顶层。

### 1.3 容器运行时和生命周期

容器标准格式也要求容器把自身运行时的状态持久化到磁盘中，这样便于外部的其他工具对此信息使用和演绎。该运行时状态以JSON格式编码存储。推荐把运行时状态的json文件存储在临时文件系统中以便系统重启后会自动移除。 基于Linux内核的操作系统，该信息应该统一地存储在`/run/opencontainer/containers`目录，该目录结构下以容器ID命名的文件夹（`/run/opencontainer/containers/<containerID>/state.json`）中存放容器的状态信息并实时更新。有了这样默认的容器状态信息存储位置以后，外部的应用程序就可以在系统上简便地找到所有运行着的容器了。 `state.json`文件中包含的具体信息需要有：

*   版本信息：存放OCI标准的具体版本号。
*   容器ID：通常是一个哈希值，也可以是一个易读的字符串。在`state.json`文件中加入容器ID是为了便于之前提到的运行时hooks只需载入`state.json`就可以定位到容器，然后检测`state.json`，发现文件不见了就认为容器关停，再执行相应预定义的脚本操作。
*   PID：容器中运行的首个进程在宿主机上的进程号。
*   容器文件目录：存放容器rootfs及相应配置的目录。外部程序只需读取`state.json`就可以定位到宿主机上的容器文件目录。

标准的容器生命周期应该包含三个基本过程。

*   容器创建：创建包括文件系统、namespaces、cgroups、用户权限在内的各项内容。
*   容器进程的启动：运行容器进程，进程的可执行文件定义在的`config.json`中，`args`项。
*   容器暂停：容器实际上作为进程可以被外部程序关停(kill)，然后容器标准规范应该包含对容器暂停信号的捕获，并做相应资源回收的处理，避免孤儿进程的出现。

### 1.4 基于开放容器格式（OCF）标准的具体实现

从上述几点中总结来看，开放容器规范的格式要求非常宽松，它并不限定具体的实现技术也不限定相应框架，目前已经有基于OCF的具体实现，相信不久后会有越来越多的项目出现。

*   容器运行时[opencontainers/runc](https://github.com/opencontainers/runc)，即本文所讲的runc项目，是后来者的参照标准。
*   虚拟机运行时[hyperhq/runv](https://github.com/hyperhq/runv)，基于Hypervisor技术的开放容器规范实现。
*   测试[huawei-openlab/oct](https://github.com/huawei-openlab/oct)基于开放容器规范的测试框架。

2\. runC工作原理与实现方式
-----------------

runC的前身实际上是Docker的libcontainer项目，笔者曾经写过一篇文章[《Docker背后的容器管理——Libcontainer深度解析》](http://www.infoq.com/cn/articles/docker-container-management-libcontainer-depth-analysis)专门对libcontainer进行源码分析和解读，感兴趣的读者可以先阅读一下，目前runC也是对libcontainer包的调用，libcontainer包变化并不大。所以此文将不再花费太多笔墨分析其源码，我们将着重讲解其中的变化。

### 2.1 runC从libcontainer的变迁

从本质上来说，容器是提供一个与宿主机系统共享内核但与系统中的其他进程资源相隔离的执行环境。Docker通过调用libcontainer包对namespaces、cgroups、capabilities以及文件系统的管理和分配来“隔离”出一个上述执行环境。同样的，runC也是对libcontainer包进行调用，去除了Docker包含的诸如镜像、Volume等高级特性，以最朴素简洁的方式达到符合OCF标准的容器管理实现。 总体而言，从libcontainer项目转变为runC项目至今，其功能和特性并没有太多变化，具体有如下几点。

1.  把原先的`nsinit`移除，放到外面，命令名称改为`runc`，同样使用[`cli.go`](https://github.com/codegangsta/cli)实现，一目了然。
2.  按照开放容器标准把原先所有信息混在一起的一个配置文件拆分成`config.json`和`runtime.json`两个。
3.  增加了按照开放容器标准设定的容器运行前和停止后执行的`hook`脚本功能。
4.  相比原先的`nsinit`时期的指令，增加了`runc kill`命令，用于发送一个`SIG_KILL`信号给指定容器ID的`init`进程。

总体而言，runC希望包含的特征有:

*   支持所有的Linux namespaces，包括user namespaces。目前user namespaces尚未包含。
*   支持Linux系统上原有的所有安全相关的功能，包括Selinux、 Apparmor、seccomp、cgroups、capability drop、pivot\_root、 uid/gid dropping等等。目前已完成上述功能的支持。
*   支持容器热迁移，通过CRIU技术实现。目前功能已经实现，但是使用起来还会产生问题。
*   支持Windows 10 平台上的容器运行，由微软的工程师开发中。目前只支持Linux平台。
*   支持Arm、Power、Sparc硬件架构，将由Arm、Intel、Qualcomm、IBM及整个硬件制造商生态圈提供支持。
*   计划支持尖端的硬件功能，如DPDK、sr-iov、tpm、secure enclave等等。
*   生产环境下的高性能适配优化，由Google工程师基于他们在生产环境下的容器部署经验而贡献。
*   作为一个正式真实而全面具体的标准存在！

### 2.2 runC是如何启动容器的？

从开放容器标准中我们已经定义了关于容器的两份配置文件和一个依赖包，runc就是通过这些来启动一个容器的。首先我们按照官方的步骤来操作一下。 

runc运行时需要有rootfs，最简单的就是你本地已经安装好了Docker，通过 `docker pull busybox` 下载一个基本的镜像，然后通过 `docker export $(docker create busybox) > busybox.tar` 导出容器镜像的rootfs文件压缩包，命名为`busybox.tar`。

然后解压缩为`rootfs`目录。 `mkdir rootfs tar -C rootfs -xf busybox.tar` 这时我们就有了OCF标准的rootfs目录，需要说明的是，我们使用Docker只是为了获取rootfs目录的方便，runc的运行本身不依赖Docker。 

接下来你还需要`config.json`和`runtime.json`，使用 `runc spec` 可以生成一份标准的`config.json`和`runtime.json`配置文件，当然你也可以按照格式自己编写。 

如果你还没有安装`runc`，那就需要按照如下步骤安装一下，目前`runc`暂时只支持Linux平台。

### 安装`runc`
Create a 'github.com/opencontainers' in your GOPATH/src:

cd github.com/opencontainers git clone https://github.com/opencontainers/runc cd runc make sudo make install \``\` 

最后执行 `runc start` 你就启动了一个容器了。 

可以看到，我们对容器的所有定义，均包含在两份配置文件中，一份简略的`config.json`配置文件类似如下，已用省略号省去部分信息，完整的可以查看[官方github](https://github.com/opencontainers/runc#oci-container-json-format)。 


```yaml
{
  "version": "0.1.0",
  "platform": {
    "os": "linux",
    "arch": "amd64"
  },
  "process": {
    "terminal": true,
    "user": {
      "uid": 0,
      "gid": 0,
      "additionalGids": null
    },
    "args": ["sh"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "TERM=xterm"],
    "cwd": ""
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "shell",
  "mounts": [{
    "name": "proc",
    "path": "/proc"
  }, 
  …… 
  {
    "name": "cgroup",
    "path": "/sys/fs/cgroup"
  }],
  "linux": {
    "capabilities": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
  }
}
```

各部分表示的意思在1.2节中已经讲解，针对具体的内容我们可以看到，版本是0.10，该配置文件针对的是AMD64架构下的Linux系统，启动容器后执行的命令就是`sh`，配置的环境变量有两个，是`PATH`和`TERM`，启动后user的uid和gid都为0，表示进入后是root用户。`cwd`项为空表示工作目录为当前目录。capabilities能力方面则使用白名单的形式，从配置上可以看到只允许三个能力，功能分别为允许写入审计日志、允许发送信号、允许绑定socket到网络端口。 一份简略的`runtime.json`配置则如下，同样用省略号省去了部分内容： 

```yaml
{
  "mounts": {
    "proc": {
      "type": "proc",
      "source": "proc",
      "options": null
    },
    ……
    "cgroup": {
      "type": "cgroup",
      "source": "cgroup",
      "options": ["nosuid", "noexec", "nodev", "relatime", "ro"]
    }
  },
  "hooks": {
    "prestart": null,
    "poststop": null
  },
  "linux": {
    "uidMappings": null,
    "gidMappings": null,
    "rlimits": [{
      "type": "RLIMIT_NOFILE",
      "hard": 1024,
      "soft": 1024
    }],
    "sysctl": null,
    "resources": {
      "disableOOMKiller": false,
      "memory": {
        "limit": 0,
        "reservation": 0,
        "swap": 0,
        "kernel": 0,
        "swappiness": -1
      },
      "cpu": {
        "shares": 0,
        "quota": 0,
        "period": 0,
        "realtimeRuntime": 0,
        "realtimePeriod": 0,
        "cpus": "",
        "mems": ""
      },
      "pids": {
        "limit": 0
      },
      "blockIO": {
        "blkioWeight": 0,
        "blkioWeightDevice": "",
        "blkioThrottleReadBpsDevice": "",
        "blkioThrottleWriteBpsDevice": "",
        "blkioThrottleReadIopsDevice": "",
        "blkioThrottleWriteIopsDevice": ""
      },
      "hugepageLimits": null,
      "network": {
        "classId": "",
        "priorities": null
      }
    },
    "cgroupsPath": "",
    "namespaces": [{
      "type": "pid",
      "path": ""
    }, {
      "type": "network",
      "path": ""
    }, {
      "type": "ipc",
      "path": ""
    }, {
      "type": "uts",
      "path": ""
    }, {
      "type": "mount",
      "path": ""
    }],
    "devices": [{
      "path": "/dev/null",
      "type": 99,
      "major": 1,
      "minor": 3,
      "permissions": "rwm",
      "fileMode": 438,
      "uid": 0,
      "gid": 0
    }, …… 
    {
      "path": "/dev/urandom",
      "type": 99,
      "major": 1,
      "minor": 9,
      "permissions": "rwm",
      "fileMode": 438,
      "uid": 0,
      "gid": 0
    }],
    "apparmorProfile": "",
    "selinuxProcessLabel": "",
    "seccomp": {
      "defaultAction": "SCMP_ACT_ALLOW",
      "syscalls": []
    },
    "rootfsPropagation": ""
  }
}
```

可以看到基本的几项配置分别为挂载点信息、启动前与停止后hooks脚本、然后就是针对Linux的特性支持的诸如用户uid/gid绑定，rlimit配置、namespace设置、cgroups资源限额、设备权限配置、apparmor配置项目录、selinux标记以及seccomp配置。其中，[namespaces](http://www.infoq.com/cn/articles/docker-kernel-knowledge-namespace-resource-isolation)和[cgroups](http://www.infoq.com/cn/articles/docker-kernel-knowledge-cgroups-resource-isolation)笔者均有写文章详细介绍过。 

再下面的工作便都由libcontainer完成了，大家可以阅读这个系列前一篇文章[《Docker背后的容器管理——Libcontainer深度解析》](http://www.infoq.com/cn/articles/docker-container-management-libcontainer-depth-analysis)或者购买书籍[《Docker容器与容器云》](http://item.jd.com/11757034.html)，里面均有详细介绍。 简单来讲，有了配置文件以后，runC就开始借助libcontainer处理以下事情：

*   创建libcontainer构建容器需要使用的进程，称为Process；
*   设置容器的输出管道，这里使用的就是daemon提供的pipes；
*   使用名为Factory的工厂类，通过factory.Create(<容器ID>, <填充好的容器模板container>)创建一个逻辑上的容器，称为Container；
*   执行Container.Start(Process)启动物理的容器；
*   runC等待Process的所有工作都完成。

可以看到，具体的执行者是libcontainer，它是对容器的一层抽象，它定义了Process和Container来对应Linux中“进程”与“容器”的关系。一旦上述物理的容器创建成功，其他调用者就可以通过ID获取这个容器，接着使用Container.Stats得到容器的资源使用信息，或者执行Container.Destory来销毁这个容器。 

综上，runC实际上是把配置交给libcontainer，然后由libcontainer完成容器的启动，而libcontainer中最主要的内容是Process、Container以及Factory这3个逻辑实体的实现原理。runC或者其他调用者只要依次执行“使用Factory创建逻辑容器Container”、“用Process启动逻辑容器Container”即可。

3\. 总结
------

本文从OCI组织的成立开始讲起，描述了开放容器格式的标准及其宗旨，这其实就是runC的由来。继而针对具体的runC特性及其启动进行了详细介绍。笔者在后续的文章中还将针对runC中诸如CRIU热迁移、selinux、apparmor及seccomp配置等特性进行具体的介绍。可以看到OCI的成立表明了社区及各大厂商对容器技术的肯定以及加快容器技术发展进步的强烈决心，相信在不久的将来，符合OCI标准的开放容器项目会越来越多，容器技术将更加欣欣向荣地不断前进。 