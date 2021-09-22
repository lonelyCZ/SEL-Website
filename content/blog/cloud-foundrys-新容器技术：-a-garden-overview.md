+++

id= "226"

title = "Cloud Foundry’s 新容器技术： A Garden Overview"
description = "CloudFoundry（CF）中很早就使用了VMware研发的Warden容器来负责应用的资源分配隔离和实例调度。可惜的是，这一本来可以成为业界标准和并掀起一阵革命的容器PaaS技术却因为Pivotal的方针路线上的种种原因被后来居上Docker吊打至今。最近CFer有醒悟的迹象，在Warden上进行了大量改进和升级，本文就来一窥CF新容器技术的一些要点。"
tags= [ "Docker" , "cloudfoundry" ]
date= "2014-12-02 18:52:46"
author = "丁轶群"
banner= "img/blogs/226/CF-RQ-1.png"
categories = [ "cloudfoundry" ]

+++



## 前言

编译自：

[Cloud Foundry’s Container Technology: A Garden Overview](http://blog.pivotal.io/cloud-foundry-pivotal/features/cloud-foundry-container-technology-a-garden-overview) 

[Containers in Cloud Foundry: warden meets libcontainer](http://underlap.blogspot.com/2014/06/warden-meets-libcontainer.html) 

CloudFoundry（CF）中很早就使用了VMware研发的Warden容器来负责应用的资源分配隔离和实例调度。可惜的是，这一本来可以成为业界标准和并掀起一阵革命的容器PaaS技术却因为Pivotal的方针路线上的种种原因被后来居上Docker吊打至今。最近CFer有醒悟的迹象，在Warden上进行了大量改进和升级，本文就来一窥CF新容器技术的一些要点。

<!--more-->


**Warden和Garden**
-----------------

Warden背景：

 "CloudFoundry’s container technology is provided by Warden,which was created by VMware’s Pieter Noorduis and others.Warden is a subtle combination of Ruby code, a core written inC, and shell scripts to configure the host and containers." 

在此前的WardenDEA中，在每个安装好的DEA上都会运行Warden服务（Ruby写的，调用大量shell来配置host），用来管理Cgroup，Namespaces和以及进程管理。同时，Warden容器的感知和状态监控也由此服务来负责。作为一个C/S结构的服务，Warden使用了谷歌的protobuf协议来负责交互。每个容器内部都运行一个wshd daemon（C语言写的）来负责容器内的管理比如启动应用进程，输出日志和错误等等。这里需要注意的正是由于使用了protobuf，warden对外的交互部分强依赖于wardenprotocol，使得warden对开发者的易用性大打折扣。 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616361/sel/CF-RQ-1_drqy4s.png" alt="" style="zoom:100%;" />
</center>


Wardenstructure

在CloudFoundry的下一代PaaS项目Diego中，Pivotal团队对于Warden进行了基于Golang的重构，并建立了一个独立的项目Garden。在Garden中，容器管理的功能被从server代码里分离出来，即server部分只负责接收协议请求，而原先的容器管理则交给backend组件，包括将接收到的请求映射成为Linux（假如是Linux backend的话）操作。值得注意的是：这样backend架构再次透露出了warden跨平台的野心，可以想象一旦Windowsbackend被社区（比如IronFoundry）贡献出来后的威力。更重要的是，RESTful风格的API终于被引入到了Garden里面，原作者说是为了实验和测试，但实际上Docker最成功的一点正是友好的API和以此为基础的扩展能力。 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616361/sel/CF-RQ-2_w7xi13.png" alt="" style="zoom:100%;" />
</center>


Gardenstructure

**Namespaces**
--------------

容器化应用依然通过namespaces来定义它所能使用的资源。最简单的例子，应用的运行需要监听指定的端口，而传统方法中这个端口就必须在全局的host网络namespaces上可见。为了避免应用互相之间出现端口冲突，Garden服务就需要设置一组namepaces来隔离每个应用的IP和port（即网络namespace）。需要再次强调，容器化的应用资源隔离不同于传统的虚拟化技术，虽然我们在讲容器，但是我们并没有去创建“什么”，而是为实实在在运行着的应用进程划分属于它自己的“命名空间”。 Garden使用了除用户namespace之外的所有namespace技术。具体实现是使用挂载namespace的方法来用用户目录替换原host的root文件系统（使用pivot\_root指令），然后unmount这个root文件系统使得从容器不会直接访问到该目录 备注：Linux在很早之前就支持了namespaces技术，从一开始为文件系统挂载点划分namespace，到最新的为用户添加namespace，具体演化参见：[Articles on Linux namespaces](http://lwn.net/)

**ResourceControl**
-------------------

被限制在运行在namespaces中的应用可以在这个“匿名的操作系统环境“中自由的使用类似于CPU和MEM这样的资源，但是应用仍然是直接访问系统设备的。Linux提供了一系列controlgroups来将进程划分为层级结构的组然后将它们限制到不同的约束中。这些约束由cgroup中的resourcecontrollers来实现并负责与kernel子系统进行交互。举个例子：memoryresource controller可以限制一个controlgroup中的进程能够在真实内存中使用的页数，从而确保这些进程在超出限制后被停止。 Garden使用了五种资源控制：cpuset(CPUs and memory nodes) , cpu (CPU bandwidth), cpuacct (CPUaccounting), devices (device access), and memory(memoryusage)，并通过这些资源控制堆每一个容器设置一个controlgroup。所以容器中的进程将被限制在resourcecontrollers指定的资源数下运行（严格地说cpuacct仅统计CPUusage，并不做出具体限制）。 此外，Garden还使用setrlimit系统调用来控制容器中进程的资源使用；使用setquota来为容器中的用户设置配额。这一点上也同Warden相同。

**NetworkingFacilities**
------------------------

简单来说，每个容器都运行在独立的网络namespace中，Garden负责控制进出容器的网络流量。Garden会创建一对veth（虚拟网卡）设备，为它们分配IP，然后将其中的一个放到容器的网络namespace中。接下来Garden会设置IP路由表来保证IP包能够正确地传入或传出容器。最后，网络包过滤规则会为容器创建防火墙来限制inbound和outbound的流量。这一点上与Warden原先的网络相同。 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616362/sel/CF-RQ-3_jvk0vq.png" alt="" style="zoom:100%;" />
</center>

Containernetworking

**RootFile System**
-------------------

Warden是允许用户自己指定一个root文件系统来供容器使用的。而Garden不仅继承了这个功能，更重要的是还能支持从Docker image来构建rootfs。同时，Garden在rootfs中引入了rw权限的层，所以容器可以叠加自己的rootfs而不影响其他容器（比如container A是warden rootfs，而container B是docker rootfs）

**GardenAPI**
-------------

新的Garden API同样是基于Google protobuf的，包括如下所有操作：

*   Capacity– returns the memory and disk capacity of the host machine
*   Create– creates a container and returns its handle (a string which identifies the container)
*   Info –returns information about a specified container such as its IPaddress and a list of processes running in the container
*   Run –spawns a process in the container and streams its output back to theclient
*   Attach– starts streaming the output of a specified process ina specified container back to the client
*   List –lists all container handles
*   LimitBandwidth,LimitCpu, LimitDisk, LimitMemory – adjusts the limits of aspecified container for network bandwidth, CPU shares,disk usage, and memory usage, respectively
*   NetIn –maps a port on the host machine to a port in the specified container
*   NetOut– whitelists outbound network traffic from the specified containerto a specified network and/or port
*   StreamIn– copies data into a specified file in the specifiedcontainer’s file system
*   StreamOut– copies data out of a specified file in the specifiedcontainer’s file system
*   Ping –checks that the garden server is running
*   Stop –terminates all processes in a specified container but leaves thecontainer around (instopped state)
*   Destroy– destroys the specified container

**Merge Warden with Docker?**
-----------------------------

与此同时，Pivotal还计划通过更通用的驱动层来将Docker直接merge到现有的系统中。主要实现方式是为前面所说的backend添加libcontainer-specific backend，如下图所示： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616363/sel/CF-RQ-4_ujgd92.png" alt="" style="zoom:80%;" />
</center>



这样，用户的请求将通过这个backend翻译成libcontainer API来启动Docker镜像——比如将用户的应用run在Docker而不是Warden容器中。但是这个feature是作为一个长期计划与Garden并行的，主要原因应该是libcontainer的API并没有完全ready，目前还在采用很初级的替代方案。 综上，CF的新容器技术向后来者Docker展示了友好的一面，并且把握住了Docker image这一核心value而不是费力不讨好地支持所有Docker feature。但是这此的改动还是一次被动的应对，恐怕只有Windows backend和libcontainer backend的及时release才能迅速扭转局面。