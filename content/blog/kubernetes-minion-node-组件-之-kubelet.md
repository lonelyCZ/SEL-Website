+++
id = "417"

title = "Kubernetes Minion Node 组件 之 Kubelet"
description = "Kubelet是Google Kubernetes 集群minion工作节点上的一个重要组件.本文将作者阅读代码和亲身的使用经验相结合带你深入理解kubelet. 因为kubernetes代码处在火热迭代开发中,版本日新月异, 本文的源代码来自2014年12月22日github上kubernetes的master最新代码, commit id 119fe37f064905d, 由于kubelet代码量大,变量多,所以文中展示的代码有删节,省略的部分用…表示."
tags = ["Kubernetes"]
date = "2015-01-13 15:50:51"
author = "陈星宇"
banner = "img/blogs/417/k8s_arch.png"
categories = ["Kubernetes"]

+++

Kubelet是Google Kubernetes 集群minion工作节点上的一个重要组件.本文将作者阅读代码和亲身的使用经验相结合带你深入理解kubelet. 因为kubernetes代码处在火热迭代开发中,版本日新月异, 本文的源代码来自2014年12月22日github上kubernetes的master最新代码, commit id 119fe37f064905d, 由于kubelet代码量大,变量多,所以文中展示的代码有删节,省略的部分用...表示. 

<!--more-->

**文章内容分为一下几个部分:**

1.  **Kubelet简介**
2.  **Kubelet源码分析**
3.  **Kubelet 启动docker容器实例分析**

# 1 Kubelet简介 

Kubelet运行在Kubernetes Minion Node上. 它是container agent的继任者（使用golang重写），它是GCE镜像的一部分. Kubelet二进制程序负责维护在特定主机上运行的一组容器。它从配置文件或是从etcd 服务器上同步容器配置清单。容器配置清单是描述一个pod的文件. Kubelet采用一组使用各种机制提供的清单并确保这些清单描述的容器被启动并持续运行. 有以下几种方式提供给kubelet一个容器清单：

1.  文件 通过一个命令行参数传递。此文件每20秒（可配置）重新检查.
2.  HTTP URL 通过命令行参数传递HTTP URL参数。 此端点每20秒检查（也可配置）, 通过查询获得容器清单.
3.  Etcd服务器 Kubelet发现etcd服务器并watch相关的key。watch的etcd key是 /registry/nodes/$（hostname -f）。由于这是一种etcd watch机制，注意到改变并采取相应的行动非常迅速. Kubelet 的组成元素可以从如下代码-Kubelet结构体定义看出:

```go
// Kubelet 结构体描述了kubelet使用到的重要元素.
    type Kubelet struct {
        hostname              string
        dockerClient          dockertools.DockerInterface
        rootDirectory         string
        networkContainerImage string
        podWorkers            *podWorkers
        resyncInterval        time.Duration
        pods                  []api.BoundPod
        
        etcdClient tools.EtcdClient      
        ...
    
        cadvisorClient cadvisorInterface
        ...
    }
```

其中重要的是如下的几个元素:

1.  **dockerClient** : 使用github.com/fsouza/go-dockerclient 作为docker client.
2.  **etcdClient** : 使用 coreos/go-etcd/etcd 作为etcd client.
3.  **rootDirectory** : 维护kubelet文件(volume mounts,etc) 的目录.默认值是/var/lib/kubelet
4.  **podWorkers** : 记录上面有正在运行工作的workers的pod信息
5.  **cAdvisorClient** : google cAdvisor 用来监测minion机器和之上容器的资源使用情况. 使用github.com/google/cadvisor/client

其中还有一个配置Kubelet的结构体,名为KubeletConfig, 他从启动命令行参数构造,最后作为启动kubelet的参数传入,源代码如下,具体可以参见后文的描述.

~~~go
type KubeletConfig struct {
        EtcdClient              tools.EtcdClient
        DockerClient            dockertools.DockerInterface
        CAdvisorPort            uint
        Address                 util.IP
        ...
        AllowPrivileged         bool
        HostnameOverride        string
        RootDirectory           string
        ... 
        ManifestURL             string
        FileCheckFrequency      time.Duration
        HttpCheckFrequency      time.Duration
        Hostname                string
        ...
    }
~~~

# 2 Kubelet源码分析 

下面介绍一些kubelet里用到的关键的数据结构和代码段 , 对于kubernetes的新手, 这个还是很有帮助的, 这也给需要hack kubelet的人提供了代码的入口. 代码入口定义在cmd/kubelet/kubelet.go的main函数, 如下:

~~~go
func main() {
        flag.Parse()
        util.InitLogs()
        defer util.FlushLogs()
        rand.Seed(time.Now().UTC().UnixNano())
    
        ...
    
        kcfg := standalone.KubeletConfig{
            Address:                 address,
            ...
            AllowPrivileged:         *allowPrivileged,
            HostnameOverride:        *hostnameOverride,
            RootDirectory:           *rootDirectory,
            ConfigFile:              *config,
            ManifestURL:             *manifestURL,
            FileCheckFrequency:      *fileCheckFrequency,
            HttpCheckFrequency:      *httpCheckFrequency,
            NetworkContainerImage:   *networkContainerImage,
            ...
            DockerClient:            kubelet.ConnectToDockerOrDie(*dockerEndpoint),
            EtcdClient:              kubelet.EtcdClientOrDie(etcdServerList, *etcdConfigFile),
        }
    
        standalone.RunKubelet(&kcfg)
        // runs forever
        select {}
    }
~~~

使用默认的参数和用户自定义的命令行启动参数构造好KubeletConfig这个结构体后, 就调用standalone.RunKubelet(&kcfg)进行真正的server端工作. RunKubelet函数定义在pkg/standalone/standalone.go中, 源码如下:

~~~go
func RunKubelet(kcfg *KubeletConfig) {
        kubelet.SetupEventSending(kcfg.AuthPath, kcfg.ApiServerList)
        
        ...
    
        cfg := makePodSourceConfig(kcfg)
        k := createAndInitKubelet(kcfg, cfg)
        // process pods and exit.
        if kcfg.Runonce {
            if _, err := k.RunOnce(cfg.Updates()); err != nil {
                glog.Errorf("--runonce failed: %v", err)
            }
        } else {
            startKubelet(k, cfg, kcfg)
        }
    }
~~~

其中重要的函数有makePodSourceConfig(kcfg), 他的源代码如下:

~~~go
func makePodSourceConfig(kc *KubeletConfig) *config.PodConfig {
        
        cfg := config.NewPodConfig(config.PodConfigNotificationSnapshotAndUpdates)
    
        ...
    
        // 使用 etcd源获得pod的配置清单
        if kc.EtcdClient != nil {
            glog.Infof("Watching for etcd configs at %v", kc.EtcdClient.GetCluster())
            config.NewSourceEtcd(config.EtcdKeyForHost(kc.Hostname), kc.EtcdClient, cfg.Channel(kubelet.EtcdSource))
        }
        return cfg
    }
~~~

他的作用是从KubeletConfig构造PodConfig. 其中一个很重要的参数config.PodConfigNotificationSnapshotAndUpdates指示了容器的操作的信息: 　

1.  当有容器变动时发送的是UPDATE 信息 　
2.  当有容器增加或删除时发送的是SET 信息, 这个参数之后会用到.

在这里我们使kubelet采用etcd server获得pod配置参数信息, 这种模式最常见, 还要注意的是NewSourceEtcd函数的参数是cfg.Channel(kubelet.EtcdSource)函数, 他不仅仅返回一个chan<- interface{} 供NewSourceEtcd使用, 背后还进行了大量额外的操作, 它调用了Mux的Channel(),源代码如下:

~~~go
// 该函数返回一个可以接收配置信息更新的 channel, 相同的源只会返回同一个channel, Mux结构内部存有一个map数据结构即sources
     // 记录源名字和channel 的匹配信息
    func (m *Mux) Channel(source string) chan interface{} {
        if len(source) == 0 {
            panic("Channel given an empty name")
        }
        m.sourceLock.Lock()
        defer m.sourceLock.Unlock()
        channel, exists := m.sources[source]
        if exists {
            return channel
        }
        newChannel := make(chan interface{})
        m.sources[source] = newChannel
        go util.Forever(func() { m.listen(source, newChannel) }, 0)
        return newChannel
    }
    func (m *Mux) listen(source string, listenChannel <-chan interface{}) {
        for update := range listenChannel {
            m.merger.Merge(source, update)
        }
    }
~~~

除了返回channel以外, 他的额外操作是启动了一个goroutine不停地调用如上的listen函数, 他监听newChannel这一Channel, 一旦该channel有可用的信息, 就调用Merge(source, update), 它的源代码如下,定义在pkg/kubelet/config/config.go中:

~~~go
func (s *podStorage) Merge(source string, change interface{}) error {
        s.updateLock.Lock()
        defer s.updateLock.Unlock()
    
        adds, updates, deletes := s.merge(source, change)
  
        switch s.mode {
        ...
    
        case PodConfigNotificationSnapshotAndUpdates:
            if len(updates.Pods) > 0 {
                s.updates  0 || len(adds.Pods) > 0 {
                s.updates <- kubelet.PodUpdate{s.MergedState().([]api.BoundPod), kubelet.SET, source}
            }    
       ...
        return nil
    }
~~~

该函数综合各种source的pod配置变化信息, 并调用merge(source,change)函数, 并根据不同的mode , 把更新信息发送到s.updates channel. 在这里就用到了之前所说的PodConfigNotificationSnapshotAndUpdates参数. 

回到之前所说的config.NewSourceEtcd方法, 这里面很重要的一个数据结构是sourceEtcd结构体和相关的操作NewSourceEtcd()和Run()方法, 源代码如下:

~~~go
type sourceEtcd struct {
        key     string
        helper  tools.EtcdHelper
        updates chan<- interface{}
    }
    
    func NewSourceEtcd(key string, client tools.EtcdClient, updates chan<- interface{}) {
        helper := tools.EtcdHelper{
            client,
            latest.Codec,
            tools.RuntimeVersionAdapter{latest.ResourceVersioner},
        }
        source := &sourceEtcd{
            key:     key,
            helper:  helper,
            updates: updates,
        }
        glog.V(1).Infof("Watching etcd for %s", key)
        go util.Forever(source.run, time.Second)
    }
    
    func (s *sourceEtcd) run() {
        watching := s.helper.Watch(s.key, 0)
        for {
            select {
            case event, ok := <-watching.ResultChan():
                if !ok {
                    return
                }
                ...
                pods, err := eventToPods(event)
                ...
    
                glog.V(4).Infof("Received state from etcd watch: %+v", pods)
                s.updates <- kubelet.PodUpdate{pods, kubelet.SET, kubelet.EtcdSource}
            }
        }
    }
~~~

NewSourceEtcd的功能是启动一个goroutine, watch相应的key如/registry/nodes/127.0.0.1/boundpod. 使用etcd 的watch机制使得相应的value变化可以第一时间通过eventToPods函数将其更新到sourceEtcd 里的 updates 这个channel. 其中etcd相关的函数如s.helper.Watch定义在pkg/tools下, 里面封装了非常详尽的高级etcd操作. 

我们查看一个运行在ubuntu server的kubernetes 集群的kubelet的log. 它使用本地的127.0.0.1:4001 的etcd client, 并watch /registry/nodes/127.0.0.1/boundpods 这个key.

~~~go
I0105 14:02:22.112762   11455 standalone.go:220] Watching for etcd configs at [http://127.0.0.1:4001]
I0105 14:02:22.113057   11455 etcd.go:58] Watching etcd for /registry/nodes/127.0.0.1/boundpods
~~~

简而言之,可以理解为通过etcd获得理想的pod状态信息, 并将其更新到s.updates 这个channel里, 它是一个只写的interface channel, 里面实际传送的是kubelet.PodUpdate这个结构体. 我们可以从log 里一窥究竟,可以发现里面包含了非常详尽的pod信息.我们发现这个名为podtest的pod内含master1 和master2 两个容器, 信息包含容器的镜像名称, 暴露端口信息, 容器volume,还有name全部大写的环境变量 信息如KUBERNETES\_SERVICE\_HOST.由于log信息量大, 已用"..."取代重复部分.

~~~go
I0105 14:02:22.161784   11455 etcd.go:81] Received state from etcd watch: 
[ ... Containers: [ {Name:master1 Image:10.10.103.215:5000/redis Command:[] WorkingDir: Ports:[{Name: HostPort:6388 ContainerPort:6379 Protocol:TCP HostIP:}] 
Env:[{Name:KUBERNETES_SERVICE_HOST Value:11.1.1.196} ...] Memory:0 CPU:100 VolumeMounts:[]...}
~~~

再回到代码调用的最顶层, MakePodSourceConfig之后调用createAndInitKubelet函数构造真正的kubelet结构体, 并同时启动goroutine进行容器和镜像的garbageCollection, cAdvisor监控并初始化HealthChecking包括TCP, HTTP, EXEC健康检查. 最后使用startKubelet()函数启动 Kubelet server.

~~~go
func startKubelet(k *kubelet.Kubelet, cfg *config.PodConfig, kc *KubeletConfig) {
            // start the kubelet
            go util.Forever(func() { k.Run(cfg.Updates()) }, 0)
    
            // start the kubelet server
            if kc.EnableServer {
                go util.Forever(func() {
                    kubelet.ListenAndServeKubeletServer(k, net.IP(kc.Address), kc.Port, kc.EnableDebuggingHandlers)
            }, 0)
        }
    }
~~~


其中第一个goroutine 后台持续运行k.Run函数, 他的参数是cfg.Updates()函数的返回值, 他包含最新的来自etcd的理想pod配置信息. 经过其他一些设置, 最终调用如下面的syncLoop函数,代码如下. 他的作用是处理pod配置信息的变化, 并通过handler.SyncPods(kl.pods)函数保持实际minion的运行状态和期望状态一致. 其中期望状态就是kl.pods参数. SyncHandler是一个golang interface, 实际传入的参数是kubelet, 它定义了SyncPods方法. 源代码如下, 定义在pkg/kubelet/kubelet.go里.

~~~go
func (kl *Kubelet) syncLoop(updates <-chan PodUpdate, handler SyncHandler) {
        for {
            select {
            case u := <-updates:
                switch u.Op {
                case SET:
                    glog.V(3).Infof("SET: Containers changed")
                    kl.pods = u.Pods
                    kl.pods = filterHostPortConflicts(kl.pods)
                case UPDATE:
                    glog.V(3).Infof("Update: Containers changed")
                    kl.pods = updateBoundPods(u.Pods, kl.pods)
                    kl.pods = filterHostPortConflicts(kl.pods)
                default:
                    panic("syncLoop does not support incremental changes")
                }
            case <-time.After(kl.resyncInterval):
                glog.V(4).Infof("Periodic sync")
                if kl.pods == nil {
                    continue
                }
            }
    
            err := handler.SyncPods(kl.pods)
            if err != nil {
                glog.Errorf("Couldn't sync containers: %v", err)
            }
        }
    }
~~~

SyncPods()函数的功能包括通过docker client查验docker daemon当前运行的container的信息, 启动未启动的容器, 杀掉不需要的容器, 移除没有容器关联的volume. 

该函数比较复杂, 具体调用syncPod()函数完成最后的任务. 源代码定义在pkg/kubelet/kubelet.go中. 后文再具体介绍. 另外一个goroutine是kubelet.ListenAndServeKubeletServer(k, net.IP(kc.Address), kc.Port, kc.EnableDebuggingHandlers)这个函数, 他初始化一个自定义路由的http server用来接收外界发给他的HTTP 请求,包括外界对podInfo的查询, 这台minion上绑定的pods的信息等. 　　　 

其中具体的handler处理函数可以接收外界对podInfo的查询, 这台minion上绑定的pods的信息等都定义在pkg/kubelet/server.go中, 具体的安装Handler函数源代码如下.

~~~go
　// 安装相应uri的处理函数
    func (s *Server) InstallDefaultHandlers() {
        healthz.InstallHandler(s.mux)
        s.mux.HandleFunc("/podInfo", s.handlePodInfoOld)
        s.mux.HandleFunc("/api/v1beta1/podInfo", s.handlePodInfoVersioned)
        s.mux.HandleFunc("/boundPods", s.handleBoundPods)
        s.mux.HandleFunc("/stats/", s.handleStats)
        s.mux.HandleFunc("/spec/", s.handleSpec)
    }
    
    func (s *Server) InstallDebuggingHandlers() {
        s.mux.HandleFunc("/run/", s.handleRun)
    
        s.mux.HandleFunc("/logs/", s.handleLogs)
        s.mux.HandleFunc("/containerLogs/", s.handleContainerLogs)
    }
~~~

# 3 Kubelet 启动Docker container 实例分析 

这里我以启动一个内含redis-master和一个sshd两个容器的pod为例. manifest 如下:

~~~go
{
      "id": "podtest",
      "kind": "Pod",
      "apiVersion": "v1beta1",
      "desiredState": {
        "manifest": {
          "version": "v1beta1",
          "id": "redis-master",
          "containers": [{
            "name": "master1",
                  "image": "10.10.103.215:5000/redis",
            "cpu": 100,
            "ports": [{
              "containerPort": 6379,
              "hostPort": 6388
            }]},
            {"name": "master2",
                  "image": "10.10.103.215:5000/sshd",
            "cpu": 100,
            "ports": [{
              "containerPort": 22,
              "hostPort": 8888
            }]}
            ]
        }
      },
      "labels": {
        "name": "redis-master"
      }
    }
~~~

如前文所说当kubelet通过etcd发现新的podtest pod启动请求时, 后端log如下所示.

~~~go
I0105 14:02:22.161784   11455 etcd.go:81] Received state from etcd watch: [ {TypeMeta:{Kind: APIVersion:} ObjectMeta:{Name:podtest Namespace:default SelfLink:/api/v1beta1/boundPods/podtest UID:569fcaa3-940a-11e4-a3b2-000c2992ee7a ResourceVersion: CreationTimestamp:2015-01-04 20:08:05 +0800 CST Labels:map[] Annotations:map[]} Spec:{Volumes:[] 
    Containers:[
       {Name:master1 Image:10.10.103.215:5000/redis ...} 
       {Name:master2 Image:10.10.103.215:5000/sshd ...]
       RestartPolicy:{Always:0xfe7a78 OnFailure: Never:} NodeSelector:map[] Host:}}]
~~~

(kl \*Kubelet) SyncLoop() 通过updates channel收到这个信息 SET: Conainers changed , 调用 (kl \*Kubelet)SyncPods(pods \[\]api.BoundPod), pods参数记录需要的理想pod状态信息. syncPods函数执行的具体的过程如下: 　　　

1.  首先先通过docker client 查询目前正在运行的所用容器的信息, 当然这里可能包含非kubelet manage的用户自己启动的容器. 　　　
2.  之后遍历pods 查看是否需要有要启动的容器, 代码如下图所示

~~~go
// 检测是否有容器需要启动
    for ix := range pods {
        pod := &pods[ix]
        podFullName := GetPodFullName(pod)
        uuid := pod.UID
        desiredPods[uuid] = empty{}
    
        // 将net容器加入要启动的desiredContainers数据结构里
        desiredContainers[podContainer{podFullName, uuid, networkContainerName}] = empty{}
        for _, cont := range pod.Spec.Containers {
            desiredContainers[podContainer{podFullName, uuid, cont.Name}] = empty{}
        }
    
        // 异步启动podWorker 进行syncPod()
        kl.podWorkers.Run(podFullName, func() {
            err := kl.syncPod(pod, dockerContainers)
            if err != nil {
                glog.Errorf("Error syncing pod, skipping: %v", err)
                record.Eventf(pod, "", "failedSync", "Error syncing pod, skipping: %v", err)
            }
        })
    }
~~~

其中重要的一个参数netowrkContainerName是一个string, 值为”net”, 它使用的image的名字是 kubernetes/pause. 该image是kubernetes独有的一个镜像, 每启动一个pod都会附加启动这样一个容器, 并且他先于你需要真正提供服务的容器运行. 可以使用docker image 命令查看kubernetes/pause的具体信息, 可以看到他的大小只有200KB+, 是非常非常小的一个镜像.他里面只有一个汇编程序/pause 在容器运行时运行, 他的作用就只是简单的等待. 由他接管了整个pod的网络.

~~~shell
vcap@ubuntu:~$ docker images

REPOSITORY                TAG    IMAGE ID     CREATED    VIRTUAL SIZE

10.10.103.215:5000/redis latest 5c41055e6eaf  5 weeks ago  427.6 MB

10.10.103.215:5000/sshd  latest 692ffdd5ad2a  3 months ago 439.3 MB

kubernetes/pause         latest 6c4579af347b  5 months ago 239.8 kB

kubernetes/pause         go     6c4579af347b  5 months ago 239.8 kB
~~~

之后再把提供服务的用户需要的容器加入desiredContainers这个map数据结构里. 再运行kubelet里的podWorkers.Run方法,源代码如下:

~~~go
func (self *podWorkers) Run(podFullName string, action func()) {
        self.lock.Lock()
        defer self.lock.Unlock()
    
        // 如果已经有工作正在运行，不启动新工作.
        if self.workers.Has(podFullName) {
            return
        }
        self.workers.Insert(podFullName)
    
        // goroutine启动action工作.
        go func() {
            defer util.HandleCrash()
            action()
    
            self.lock.Lock()
            defer self.lock.Unlock()
            self.workers.Delete(podFullName)
        }()
    }
~~~

其实podWorkers是一个比较简单的结构体 ,他内含一个sync.Mutex和一个StringSet类型的works变量, works里面存储的是所谓的podFullName. 可以把podFullName看做是一个sync Job 的id.它是由GetPodFullName()函数返回, 保证在所有配置源中都有独一无二的名字, 因此可以区分每个sync job. 其中很重要的一点是异步启动了一个goroutine运行action()函数, 也就是kubelet对象的syncPod()函数, 执行完成后再删除掉特定的podFullName代表工作完成. 　 

下面重点介绍syncPod函数,它定义在pkg/kubelet/kubelet.go中,源代码如下,由于源代码很长,这里就简单介绍一下操作流程:

1.  首先确保特殊net容器已经启动, 若没有启动则先杀死pod里所有容器, 再重新启动net容器.
2.  根据pod配置文件准备需要挂载的volume, 供后文启动容器时使用.
3.  之后再对每个pod里的容器计算一个expectedHash值,使用的方法是adler32/hash pkg里的函数.再查询docker 后台相同名字的容器的hash. 若hash不同,则杀掉原来的容器并杀掉所关联的net容器.若hash相同则不操作.
4.  查验容器的restartPolicy, 如果容器挂了不指定重启策略,则什么都不做,这适用于不运行daemon进程的容器.
5.  调用docker client下载容器镜像.
6.  调用 docker client 运行容器, 使用的函数是kubelet 的 runContainer().源代码如下:

~~~go
containerID, err := kl.runContainer(pod, &container, podVolumes, "container:"+string(netID))
    if err != nil {
       
        glog.Errorf("Error running pod %s container %s: %v", podFullName, container.Name, err)
        continue
    }
~~~

**这里很重要的一点是这个容器和pod net容器共用net namespace,如第四个参数所示**. 其他的如端口映射或挂载volume设置都在runContainer()函数中定义.这里不再赘述. Pod 启动完毕使用 docker ps 看到结果如下,由于每行输出比较多,可能会有所混乱

~~~shell
vcap@ubuntu:~$ docker ps
    CONTAINER ID        IMAGE                             COMMAND                CREATED             STATUS              PORTS                                          NAMES
    5d1ad2fe030f        10.10.103.215:5000/redis:latest   "redis-server /etc/r   19 hours ago        Up 19 hours                                                        k8s_master1.ffde9eab_podtest.default.etcd_569fcaa3-940a-11e4-a3b2-000c2992ee7a_fa7b90b4
    b98b0802d38c        10.10.103.215:5000/sshd:latest    "/usr/sbin/sshd -D"    19 hours ago        Up 19 hours                                                        k8s_master2.994d9dd9_podtest.default.etcd_569fcaa3-940a-11e4-a3b2-000c2992ee7a_1c28f740
    17bc7f7d8480        kubernetes/pause:go               "/pause"               19 hours ago        Up 19 hours         0.0.0.0:6388->6379/tcp, 0.0.0.0:8888->22/tcp   k8s_net.d01ea6ed_podtest.default.etcd_569fcaa3-940a-11e4-a3b2-000c2992ee7a_a3c24aea
~~~

可以看到对于这个redis-server pod有一个附属网络容器,镜像是kubernetes/pause , 由他暴露出6379 redis端口, 当使用docker inspect 时也会发现所有的网络设置, 如容器IP, 端口映射都记录在net这个容器里. 

# **总结**

1.  Kubelet 里面充分使用goroutine 这一golang的feature做并发, 并使用channel在goroutine间传递信息. 简而言之, 如下图所示, kubelet使用到了一个etcd相关的goroutine 1 及时处理相应key值发生的变化, 并把变化信息通过updates channel告诉podConfig这边的用于合并pod 配置的goroutine 2, 这个goroutine最后把变化更新到podConfig下的updates channel. 最后一个goroutine 3根据这个channel信息并与机器上实际的docker 容器运行状态做比较, 更新容器到理想状态. 总体而言为了封装, channel作为函数参数或是结构体field在多个函数和结构体中被使用. 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605793895/sel/kubegoroutine_daa5rr.png" style="zoom:80%;" />
</center>


2.  当我们想更多的看到kubelet代码里的log , 或是想通过自己打log记录一些关键数据结构的值时 , 我们可以在kubelet的启动参数后加如 - -v=4 .这样就会使能所有kubelet的复杂的debug信息都会输出到log文件中(如使用ubuntu upstart启动的是/var/log/upstart/kubelet.log), 供开发者review 查看.具体使用可以参见golang glog 这个google推出的记录log的pkg, 这里不再赘述.
    
3.  Kubelet 启动一个pod时总是会新建一个名为net的容器, 后续的用户container 都公用他的net namspace.暴露的端口也都有它负责.这样就保证了当用户的容器因为不可抗力原因重启, 整个pod的网络信息如ip, port等信息不变,这也是kubelet的一个重要特性,防止了只使用普通docker容器重启ip会变化的麻烦.
    
4.  Kubelet总体架构图如下: 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605793942/sel/minon_E6_9E_B6_E6_9E_84_fvn856.png" style="zoom:50%;" />
</center>


