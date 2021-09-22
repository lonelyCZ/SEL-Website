+++
id= "573"

title = "玩转Docker镜像"
description = "摘要: Docker是基于Go语言开发，通过分层镜像标准化和内核虚拟化技术，使得应用开发者和运维工程师可以以统一的方式跨平台发布应用。镜像是Docker最核心的技术之一，也是应用发布的标准格式。"
tags= [ "Docker" ]
date= "2014-12-16T16:15:36"
author = "孙宏亮"
banner= "img/blogs/573/docker-image.png"
categories = [ "Docker" ]

+++

**摘要:**Docker是基于Go语言开发，通过分层镜像标准化和内核虚拟化技术，使得应用开发者和运维工程师可以以统一的方式跨平台发布应用。镜像是Docker最核心的技术之一，也是应用发布的标准格式。

<!--more-->


## 前言

Docker是Docker.Inc公司开源的一个基于轻量级虚拟化技术的容器引擎项目,整个项目基于Go语言开发，并遵从Apache 2.0协议。通过分层镜像标准化和内核虚拟化技术，Docker使得应用开发者和运维工程师可以以统一的方式跨平台发布应用，并且以几乎没有额外开销的情况下提供资源隔离的应用运行环境。由于众多新颖的特性以及项目本身的开放性，Docker在不到两年的时间里迅速获得诸多IT厂商的参与，其中更是包括Google、Microsoft、VMware等业界行业领导者。同时，Docker在开发者社区也是一石激起千层浪，许多如我之码农纷纷开始关注、学习和使用Docker，许多企业，尤其是互联网企业，也在不断加大对Docker的投入，大有掀起一场容器革命之势。

## Docker镜像命名解析

镜像是Docker最核心的技术之一，也是应用发布的标准格式。无论你是用docker pull image，或者是在Dockerfile里面写FROM image，从Docker[官方Registry](https://registry.hub.docker.com/)下载镜像应该是Docker操作里面最频繁的动作之一了。那么在我们执行docker pull image时背后到底发生了什么呢？在回答这个问题前，我们需要先了解下docker镜像是如何命名的，这也是Docker里面比较容易令人混淆的一块概念：Registry，Repository, Tag and Image。 

下面是在本地机器运行docker images的输出结果： 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616358/sel/1_s4nci0.jpg" alt="" style="zoom:70%;" />
</center>


我们可以发现我们常说的“ubuntu”镜像其实不是一个镜像名称，而是代表了一个名为ubuntu的Repository，同时在这个Repository下面有一系列打了tag的Image，Image的标记是一个GUID，为了方便也可以通过Repository:tag来引用。 

那么Registry又是什么呢？Registry存储镜像数据，并且提供拉取和上传镜像的功能。Registry中镜像是通过Repository来组织的，而每个Repository又包含了若干个Image。

*   Registry包含一个或多个Repository
*   Repository包含一个或多个Image
*   Image用GUID表示，有一个或多个Tag与之关联

那么在哪里指定Registry呢？让我们再拉取一个更完整命名的镜像吧： 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616357/sel/2_jc6nzs.jpg" alt="" style="zoom:100%;" />
</center>

上面我试图去拉取一个ubuntu镜像，并且指定了Registry为我本机搭建的私有Registry。下面是Docker CLI中pull命令的代码片段 (docker/api/client/command.go中的CmdPull函数) 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616358/sel/3_qj27qz.png" alt="" style="zoom:100%;" />
</center>
在运行时，上面的taglessRemote变量会被传入localhost:5000/ubuntu。上面代码试图从taglessRemote变量中解析出Registry的地址，在我们的例子中，它是localhost:5000。 

那我们回过头再来看看下面这个耳熟能详的pull命令背后的故事吧： 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616358/sel/4_xjbqzj.jpg" alt="" style="zoom:100%;" />
</center>


我们跟着上面的示例代码，进一步进入解析函数ResolveRepositoryName的定义代码片段(docker/registry/registry.go) 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616358/sel/5_iwkcnt.png" alt="" style="zoom:70%;" />
</center>



我们发现，Docker CLI会判断传入的taglessRemote参数的第一部分中是否包含’.’或者':’，如果存在则认为第一部分是Registry地址，否则会使用Docker官方默认的Registry（即index.docker.io其实这里是一个Index Server，和Registry的区别留在后面再去深究吧），即上面代码中高亮的部分。背后的故事还没有结束，如果你向DockerHub上传过镜像，应该记得你上传的镜像名称格式为user-name/repository:tag，这样用户Bob和用户Alice可以有相同名称的Repository，通过用户名前缀作为命名空间隔离，比如Bob/ubuntu和Alice/ubuntu。官方镜像是通过用户名library来区分的，具体代码片段如下(docker/api/client/command.go中的CmdPull函数) 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616359/sel/6_e4vw7c.png" alt="" style="zoom:100%;" />
</center>


我们回过头再去看Docker命令行中解析Tag的逻辑吧(docker/api/client/command.go中的CmdPull函数)： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616359/sel/7_ijptio.png" alt="" style="zoom:100%;" />
</center>


代码会试着在用户输入的Image名称中找’ : ‘后面的tag,如果不存在，会使用默认的‘DEFAULTTAG’，即‘latest’。 

也就是说在我们的例子里面，命令会被解析为下面这样（注意，下面的命令不能直接运行，因为Docker CLI不允许明确指定官方Registry地址） 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616359/sel/8_pam9dz.jpg" alt="" style="zoom:100%;" />
</center>




## 配置Registry Mirror

Docker之所以这么吸引人，除了它的新颖的技术外，围绕官方Registry（Docker Hub）的生态圈也是相当吸引人眼球的地方。在Docker Hub上你可以很轻松下载到大量已经容器化好的应用镜像，即拉即用。这些镜像中，有些是Docker官方维护的，更多的是众多开发者自发上传分享的。而且你还可以在Docker Hub中绑定你的代码托管系统（目前支持Github和Bitbucket）配置自动生成镜像功能，这样Docker Hub会在你代码更新时自动生成对应的Docker镜像，是不是很方便？ 

不幸的是Docker Hub并没有在国内放服务器或者用国内的CDN，下载个镜像20分钟最起码，我等码农可耗不起这么长时间，老板正站在身后催着我们搬运代码呢。为了克服跨洋网络延迟，一般有两个解决方案：一是使用私有Registry，另外是使用Registry Mirror，我们下面一一展开聊聊.

### 方案一

就是搭建或者使用现有的私有Registry，通过定期和Docker Hub同步热门的镜像，私有Registry上保存了一些镜像的副本，然后大家可以通过docker pull private-registry.com/user-name/ubuntu:latest，从这个私有Registry上拉取镜像。因为这个方案需要定期同步Docker Hub镜像，因此它比较适合于使用的镜像相对稳定，或者都是私有镜像的场景。而且用户需要显式的映射官方镜像名称到私有镜像名称，私有Registry更多被大家应用在企业内部场景。私有Registry部署也很方便，可以直接在Docker Hub上下载Registry镜像，即拉即用，具体部署可以参考[官方文档](https://github.com/docker/docker-registry)。 

### 方案二

是使用Registry Mirror，它的原理类似于缓存，如果镜像在Mirror中命中则直接返回给客户端，否则从存放镜像的Registry上拉取并自动缓存在Mirror中。最酷的是，是否使用Mirror对Docker使用者来讲是透明的，也就是说在配置Mirror以后，大家可以仍然输入docker pull ubuntu来拉取Docker Hub镜像，除了速度变快了，和以前没有任何区别。 

了以更便捷的方式对接Docker Hub生态圈，使用Registry Mirror自然成为我的首选。接下来我就和大家一起看看Docker使用Mirror来拉取镜像的过程。下面的例子，我使用的是由**[DaoCloud](http://www.daocloud.io/)**提供的Registry Mirror服务，在申请开通Mirror服务后你会得到一个Mirror地址，然后我们要做的就是把这个地址配置在Docker Server启动脚本中，重启Docker服务后Mirror配置就生效了（如何获得Mirror服务可以参考本篇文章的附录） Ubuntu下配置Docker Registry Mirror的命令如下：

```shell
sudo echo “DOCKER_OPTS=\”\$DOCKER_OPTS –registry-mirror=http://your-id.m.daocloud.io -d\”” >> /etc/default/docker
sudo service docker restart
```


如果你是用的Boot2Docker，配置命令为：

```shell
# 进入Boot2Docker Start Shell，并执行
sudo su
echo “EXTRA_ARGS=\”–registry-mirror=http://your-id.m.daocloud.io\”” >> /var/lib/boot2docker/profile
exit
# 重启Boot2Docker
```

配置好Registry Mirror后，就可以拉取Docker镜像了，经我测试，使用DaoCloud的Mirror后，拉取常见镜像的速度可以达到1.5M左右，具体速度在你的网络环境可能会略有不同。 

我们来看看配置了Registry Mirror后，Docker拉取镜像的过程吧。首先是CLI拉取镜像命令代码片段(docker/api/client/command.go中的CmdPull函数) 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616360/sel/9_aako7n.png" alt="" style="zoom:100%;" />
</center>


首先，Docker CLI会试图获得授权，在我们的例子中会向https://index.docker.io/v1请求认证，认证完成后，认证服务器会返回一个对应的Token。注意，这里用户认证与配置的Registry Mirror完全无关，这样我们就不用担心使用Mirror的安全问题了。接着Docker CLI会调用Docker Server（即Docker daemon程序）的创建镜像命令，Docker Server随之会执行具体的拉取镜像动作，代码片段如下(docker/graph/pull.go的pullRepository函数) 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616359/sel/10_y6vtwb.png" alt="" style="zoom:70%;" />
</center>



从代码中可以发现，如果配置了Registry Mirror，Docker Server会首先从Mirror中拉取镜像，如果Mirror拉取失败会退而求其次从镜像中指定的Registry拉取。大家又可以松口气了，就算配置的Registry Mirror失效，也不会影响用户拉取镜像，只不过速度就。。。 

镜像拉下来后，就可以运行容器了 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616360/sel/11_ayahi9.png" alt="" style="zoom:100%;" />
</center>


## 附录

下面我简单介绍下如何在DaoCloud申请一个Mirror服务，首先登陆[DaoCloud主页](http://www.daocloud.io/) 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605873640/sel/12_p7jbt1.jpg" alt="12" style="zoom:50%;" /> 
</center>




点击**”立刻注册“**，简单填写个人信息后，随即登陆并自动跳转到”控制台“，按照提示点击**”启动你的加速器“**按钮。 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605880323/sel/13_jkj5m2.png" alt="" style="zoom:60%;" />
</center>


启动成功后，你就拥有了一个你专用的Registry Mirror地址了，加速器链接就是你要设置”--registry-mirror“的地址。目前每个用户有10G的加速流量（Tips：如果流量不够用可以邀请好友获得奖励流量，邀请越多奖励越多哦） 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605880440/sel/14_lfqopk.png" alt="" style="zoom:60%;" />
</center>


最后，要感谢国内存储行业领先企业**[七牛云存储](http://http//www.qiniu.com/)**在存储和CDN方面提供的大力支持，正因为有了像七牛这样技术领先又热心促进互联网生态发展的企业的积极参与，我们才能给开发者提供更多高质量的服务。 

## 结语

今天和大家一起聊了聊Docker在拉取镜像时如何解析镜像和执行拉取动作的，以及如何通过设置Registry Mirror克服网络延时，加速拉取过程。涉及到的代码只集中在Docker CLI和Docker Server，在很多方面并没有展开，比如Registry是如何响应以及如何和Index Server联动的，只能留给下次再和大家详细探讨了。 转载自:[DaoCloud](http://blog.daocloud.io) 