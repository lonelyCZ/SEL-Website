+++
id = "972"

title = "knative serving开发环境搭建"
description = "knative serving开发环境搭建。本文基官方knative serving的开发环境搭建文档，加自己的实践，给出以下knative serving开发环境的搭建方案。开始之前请确保已按上面的开发环境搭建文档搭好Kubernetes(1.10以上)和istio。"
tags = ["istio","Kubernetes","serverless"]
date = "2018-10-11 20:11:02"
author = "丁轶群"
banner = "img/blogs/972/knative serving.png"
categories = ["Kubernetes","service mesh"]

+++

本文基官方knative serving的[开发环境搭建文档](https://github.com/knative/serving/blob/master/DEVELOPMENT.md)，加自己的实践，给出以下knative serving开发环境的搭建方案。  

开始之前请确保已按上面的开发环境搭建文档搭好Kubernetes(1.10以上)和istio。

安装开源docker registry
-------------------

knative serving组件采用ko实现自动化的镜像构建，并将构建好的镜像上传到指定镜像仓库中。因此我们需要安装一个简单的开发用私有镜像仓库。  

按照[这个docker官方文档](https://docs.docker.com/registry/deploying)，下载并启动镜像仓库：

```shell
docker run -d -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 -p 5000:5000   --name registry registry:2
```

只有制定了bind address为0.0.0.0才能让kubernetes集群中的其他节点也能访问到该registry。  

注意上面的镜像仓库默认对外提供http服务，如果需要提供https服务，则首先准备好https证书，然后按照[这篇docker官方文档](https://docs.docker.com/registry/deploying/#run-an-externally-accessible-registry)配置镜像仓库。由于我们手头没有https证书，因此在这里不配置镜像仓库的https服务。

修改所有需要访问私有镜像仓库的docker daemon配置文件
--------------------------------

docker daemon默认采用https的方式访问镜像仓库，为了让docker daemon使用http访问镜像仓库，需要按照[这篇docker官方文档](https://docs.docker.com/registry/insecure/#deploy-a-plain-http-registry)配置docker daemon：

1.  在/etc/docker/daemon.json添加如下内容
    
    ```yaml
    {
      "insecure-registries" : ["<镜像仓库的ip地址>:5000"]
    }
    ```
    
2.  如果是已经运行中的docker daemon，则需要如下指令重启docker daemon：
    
    ```shell
    systemctl daemon-reload
    systemctl restart docker
    ```
    
    

修改ko源码，让它用http协议访问镜像仓库
----------------------

ko默认采用https访问镜像仓库，并且无法通过配修改，只能通过修改源码。

1.  下载ko源码
    
    ```shell
    mkdir -p ~/go/src/github.com/google
    cd ~/go/src/github.com/google
    git clone https://github.com/google/go-containerregistry.git
    ```
    
2.  修改`serving/pkg/name/repository.go`第91行
    
    ```go
    reg, err := NewSecureRegistry(registry, strict)
    ```
    
    
    改为
    
    ```go
    reg, err := NewInsecureRegistry(registry, strict)
    ```
    
3.  重新编译ko工具
    
    ```shell
    cd ~/go/src/github.com/google/go-containerregistry/cmd/ko
    go build
    ```
    
    
    完成后得到新的ko可执行文件
    

修改ko配置
------

修改ko保存镜像的repo+namespace名称：

```shell
export KO_DOCKER_REPO=<镜像仓库的ip地址地址>:5000/knative
```


修改ko build镜像用到的基础镜像，在未来运行ko的目录下新建`.ko.yaml`文件，并添加如下内容：

```yaml
defaultBaseImage: <镜像仓库的ip地址>:5000/distroless/base:latest
```


把镜像push到私有仓库上
-------------

下载gcr.io/distroless/base:latest，执行docker tag把镜像重命名为 前面的名字，即`<镜像仓库的ip地址>:5000/distroless/base:latest`

部署knative serving组件
-------------------

在Kubernetes masster上git clone knative serving代码，然后在master上执行如下命令来部署knative-serving组件

```shell
./ko apply -f ~/go/src/github.com/knative/serving/config/
```


应该可以在几秒钟内完成，结束之后执行`kubectl get pods -n knative-serving`，可以得到部署成功的pod，状态为RUNNING：

1.  activator-547f9c88f-7kpnq
2.  autoscaler-6f497c9777-22fz8
3.  controller-fd8c88d89-k66c5
4.  webhook-85b7c954f7-zl65q
