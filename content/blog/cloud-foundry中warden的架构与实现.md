+++

id= "217"

title = "Cloud Foundry中warden的架构与实现"
description = "在Cloud Foundry中，当应用开发者的应用由Cloud Foundry的组件DEA来运行时，应用的资源隔离与控制显得尤为重要，而warden的存在很好得解决了这个问题。 Cloud Foundry中warden项目的首要目的是提供一套简易的接口来管理隔离的环境，这些隔离的环境可以被称为“容器”，他们可以在CPU使用，内存使用，磁盘使用以及设备访问权限方面做相应的限制。  "
tags= [ "dea" , "cloudfoundry" ]
date= "2014-12-02 17:19:09"
author = "孙宏亮"
banner= "img/blogs/217/CF-waden-1.jpeg"
categories = [ "cloudfoundry" ]

+++

在Cloud Foundry中，当应用开发者的应用由Cloud Foundry的组件DEA来运行时，应用的资源隔离与控制显得尤为重要，而warden的存在很好得解决了这个问题。 

Cloud Foundry中warden项目的首要目的是提供一套简易的接口来管理隔离的环境，这些隔离的环境可以被称为“容器”，他们可以在CPU使用，内存使用，磁盘使用以及设备访问权限方面做相应的限制。 

<!--more-->

本文将从四个方面进行探讨分析warden的实现：

1.  warden的功能介绍及框架实现
2.  warden框架的对外接口及实现
3.  warden框架的内部模块及实现
4.  warden的运行示例


**warden的功能介绍及框架实现**
--------------------

### **warden功能介绍**

由于Cloud Foundry v1中DEA组件运行应用程序时，自身设计存在一定的缺陷，即同一个DEA上运行的应用不能很好的实现运行过程中资源的隔离与限制，故在Cloud Foundry v2中引入了warden这一模块。 warden专门接收DEA组件发送的关于应用的管理请求，在处理这部分管理请求时，借助轻量级虚拟化技术，将宿主机操作系统进行虚拟化，在容器内部执行请求的具体内容。warden的具体使用效果为应用程序之间互不感知，资源间完成隔离，各自的资源使用存在上限。假设Cloud Foundry不存在应用程序资源的隔离与限制机制，则在同一个DEA上运行的多个应用程序，在负载增加的时候，会出现竭力竞争资源的情况，当资源消耗殆尽时，大大降低应用程序的可用性与安全性。 在资源隔离与限制方面，warden主要提供3个维度的用户自定义隔离与限制：内存、磁盘、网络带宽；另外warden还提供以下维度的资源隔离与限制，但仅提供默认值，不提供用户自定义设置：CPU、CPUACCT、Devices。 同时，warden作为一个虚拟化容器，还提供众多的API命令，供用户完成对warden container的管理。主要的命令如下：copy in、copy out、create、destroy、echo、error 、info、limit\_bandwidth、limit\_disk、limit\_memory、limit\_cpu、link 、list、message、net in、net out、ping、run、spawn、stop和stream等 。这些命令的功能介绍可以简单参见：[James Bayer对于warden与docker的比较文档](https://docs.google.com/document/d/1DDBJlLJ7rrsM1J54MBldgQhrJdPS_xpc9zPdtuqHCTI/edit?pli=1#)。

### **warden框架实现**

在涉及warden框架的具体实现时，需要先申明和warden相关的多个概念：

*   warden：在Cloud Foundry中实现应用资源隔离与控制的框架，其中包括，warden\_client、warden\_server、warden\_protocol和warden container；
*   warden server：warden框架中server端的实现，主要负责接收client端请求，以及请求的处理执行；
*   warden client：warden框架中client端的实现，被Cloud Foundry中被dea\_ng组件调用，实现给warden\_server发送具体请求；
*   warden protocol：warden框架中定义warden\_client与warden\_server通信时的消息请求协议；
*   warden container：warden框架中管理与运行应用程序的容器，资源的隔离与限制以容器为单位。

warden框架的实现为典型的C/S架构，如下图： 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616362/sel/CF-waden-1_a4liwi.jpg" alt="CF-waden-1" style="zoom:90%;" />
</center>


**warden框架的对外接口及实现**
--------------------

虽然warden模块是Cloud Foundry中不可或缺的一部分，但是如果不借助Cloud Foundry的话，warden依然可以用来管理warden container，并在container内部运行应用程序等。 若warden运行在Cloud Foundry内部，则dea\_ng组件内嵌warden\_client，并以warden\_client与warden\_server建立通信，分发应用的管理请求；若warden单独存在，则可以通过warden的REPL（Read-Eval-Print Loop）命令行工具瑞与warden\_server进行通信，用户通过命令行发起container的管理请求。本章将以以上两个方式阐述warden框架的对外接口及实现。

### **warden与dea\_ng通信**

warden在Cloud Foundry中的使用，几乎完全是和dea\_ng一起捆绑使用。在部署dea\_ng时，不论Cloud Foundry集群中安装了多个dea\_ng组件，每个dea\_ng组件所在的节点上都会安装一个warden，由此可见warden与dea\_ng的存在为一一对应关系。 以下是warden与dea\_ng的交互示意图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616362/sel/CF-waden-2_awtv1e.jpg" alt="CF-waden-2" style="zoom:100%;" />
</center>



由以上示意图可知，从dea\_ng接受请求，分发container请求，主要分为以下几个步骤：

1.  dea\_ng通过消息中间件NATS获取app的管理请求；
2.  dea\_ng根据请求类型，并通过Warden::Protocol协议创建出相对应的container请求；
3.  dea\_ng通过已经和warden\_server建立连接的waren\_client发送container请求。

### **warden与REPL命令行交互**

warden也可以单独安装在某个机器上，当需要管理warden时，可以通过REPL命令行的方式，启动一个进程，创建warden\_client，并负责接收用户在命令行输入的warden container管理命令，然后通过warden\_client给warden\_server发送请求。 从上可知，REPL和dea\_ng与warden的通信方式几乎相同，区别仅仅在两者的使用方式。以下是warden与repl命令行交互的示意图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616362/sel/CF-waden-3_xkj0dp.jpg" alt="CF-waden-3" style="zoom:100%;" />
</center>


**warden框架的内部模块及实现**
--------------------

上文已经提及warden框架为C/S架构，抽象而言，它的运行包括：1.通过warden\_client给warden\_server发送container request；2.由warden\_server接收请求并处理；3.warden\_server针对warden container执行请求。 以下是warden框架的简单示意图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616364/sel/CF-waden-5_ve7me0.jpg" alt="CF-waden-4" style="zoom:50%;" />
</center>



### **warden\_server的框架实现**

warden\_server的主要功能为接收warden\_client的请求，并分发处理。warden\_server的具体实现为，通过EventMachine启动一个server，并监听本地的一个unix domain socket文件，最终将所有接收到的请求转发给ClientConnection句柄来处理。ClientConnection首先从监听的sock文件中读取数据，然后从读出的数据中读取请求，随后辨别请求的类型，最后通过请求的类型，执行相应的操作。如果请求针对某一个具体的warden container，则需要从container注册过的registry中先找出相应的warden container，随后对该warden container执行相应的shell 脚本。 warden\_server在disptach请求的时候，主要是完成对请求的分发。请求类型主要可以分为两类，一类为针对容器内部的具体操作，比如针对创建warden container请求执行新建container操作，对指定container执行运行某任务的操作，对指定container执行销毁操作等：；另一类为容器信息的获取，比如对warden container的ping操作，获取所有的container hanles信息，对warden执行echo命令等。以下阐述warden\_server的请求类型。

### **warden container的实现**

warden\_server主要负责管理warden container的管理，包括创建，设置，销毁等。当warden container创建完毕并且设置完毕后，由warden container负责自身的运行。运行在warden container内部的应用程序，由于warden container提供资源隔离和控制的环境，从而实现应用程序的资源隔离与限制。 warden的资源隔离与限制是整个Cloud Foundry的核心，其中warden container的资源隔离与限制主要通过Linux内核的cgroup机制，quota以及tc（traffic controller）来实现。 这一部分将从warden container的文件结构、生命周期，网络配置入手，阐述warden container的部分实现。

### **warden container的文件结构**

warden container可以认为是一个简易版的操作系统，故也存在自己内部的文件目录，刚创建完毕的时候，其内部的文件结构主要有以下6个文件夹：/bin、/etc、/jobs、/lib、/run、/tmp、/mnt。 其中/bin文件下主要是一些可执行文件，比如：wshd，wsh，iomux-spawn，iomux-link等，该部分可执行文件的用途，下文或以后会进行阐述。 /etc则是存放与主机相关的文件和目录。这些文件包括系统的配置文件，包括系统的主机名，网络设置参数等。 /jobs存放从warden container外部传输进来的job信息。 /libs存放系统重要的库文件等。 /run存放容器内运行的任务。 /tmp主要存放一些临时文件，该文件下主要有一个rootfs文件夹，rootfs文件夹下有一个精简的系统。 /mnt主要用于mount文件目录，并存放容器创建时添加的设备文件。

### **warden container的生命周期**

warden container的生命周期主要包括，创建、使用、删除这三个流程，其中在使用过程中会涉及众多有关warden container的具体操作。 首先是warden container的创建。warden server 收到创建warden container的请求之后，通过执行脚本来实现创建容器。这部分脚本的作用可以概括为以下几部分：初始化文件系统、配置容器属性、配置容器网络以及运行warden container中最重要的守护进程wshd。在创建完warden container之后，关于每一个warden container，都会生成一个container的handle，便于container的记录。 随后是warden container的使用。warden container的使用包括两种情况，一种情况是用户请求通过warden\_client,然后经过warden\_server进行管理，比如用户对warden container进行配置资源限制，用户给warden container内部传输文件等；另一种情况是当warden container内部运行着web应用时，warden container外部用户通过应用提供的服务访问内部应用。在第一种情况中，都是通过warden container 内部的washd进程fork出shell进程，而用户命令通过wsh传输给wshd，wshd又将命令交由shell执行。 在warden container的生命周期中还包括其自身的删除。删除过程中，首先让wshd给container内部所有进程发送一个pkill -TERM请求，并等待进程的结束，若没有结束，则直接杀死进程，随后wshd进程退出，并清除container文件目录。

### **warden container的网络配置**

warden container的网络配置如下图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616363/sel/CF-waden-4_ektifo.jpg" alt="CF-waden-5" style="zoom:80%;" />
</center>



首先，warden为每一个容器创建一个虚拟网卡，并通过warden的宿主机作端口映射。如宿主机对于本地指定端口上的请求，通过DNAT转发至指定warden container内部。当warden container对外发送请求时，通过自身的虚拟网卡对外发送请求，发送至宿主机另外虚拟出的虚拟网卡上，并将该虚拟网卡作为container的网关，进行请求处理，最终通过宿主机发送至外网。 以上便是warden架构的简要介绍。 转载请注明出处。 这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。希望本文能够对接触Cloud Foundry v2中warden模块的人有些帮助，如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 