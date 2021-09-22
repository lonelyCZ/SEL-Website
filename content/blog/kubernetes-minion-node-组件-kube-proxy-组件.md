+++
id = "484"

title = "Kubernetes代码走读之Minion Node 组件 kube-proxy"
description = "Kube-proxy是kubernetes 里运行在minion节点上的一个组件, 它 watch kubernetes 集群里 service 和 endpoints(label是某一特定条件的pods)这两个对象的增加或删除,　并且维护一个service 到 endpoints的映射. 他使用iptables REDIRECT target将对服务的请求流量转接到本地的一个port上,　然后再将流量发到后端,　这样的转发还支持一些策略,　如round robin等，所以我们可以把他看成是一个具有高级功能的反向代理。 本文的内容将分为以下两部分, 源代码来自kubernetes release-0.8.1, 代码有删节，省略的代码或log输出用...表示"
tags = ["Kubernetes"]
date = "2015-01-22 14:16:32"
author = "杜军"
banner = "img/blogs/484/k8s_arch.png"
categories = ["Kubernetes"]

+++

Kube-proxy是kubernetes 里运行在minion节点上的一个组件, 它起的作用是一个服务代理的角色. 本文的内容将分为以下两部分, 源代码来自kubernetes release-0.8.1, 代码有删节，省略的代码或log输出用．．．表示: 

_1 Kube-proxy 简介_

*2 Kube-proxy代码解读*

<!--more-->

# **1 Kube-proxy简介** 

Kube-proxy网络代理运行在每个minion节点上。网上很多人所说这个proxy是kubernetes里的SDN组件，我本人并不这么认为, 我认为可以把他看成是一个高级的反向代理．它的功能反映了定义在每个节点上Kubernetes API中的Services信息，并且可以做简单的TCP流转发或在一组服务后端做round robin的流转发．服务端点目前通过与docker link 兼容的环境变量指定的端口被发现, 这些端口由服务代理打开．目前，用户必须在代理上选择一个端口以暴露服务. 

# **2 Kube-proxy代码解读** 

Kube-proxy代码入口定义在cmd/kube-proxy/proxy.go中, 源代码较长,分段解读如下:

~~~go
func main() {
        flag.Parse()
        util.InitLogs()
        defer util.FlushLogs()
    
        if err := util.ApplyOomScoreAdj(*oomScoreAdj); err != nil {
            glog.Info(err)
        }
~~~

首先

**1\. 使用flag pkg初始化命令行参数到相应的变量, 如etcd\_servers选项** 

**2\. 初始化log** 

**3\. 应用oomScoreAdj参数到/proc/self/oom\_score\_adj文件**.  oom\_score\_adj 是-1000到1000的数值, 用来表征进程当发生OOM(out of memory)时系统对该进程的行为,值越低越不容易被杀死.默认值是-899. 

**4\. 使用下列两个函数新建两个重要的数据结构 _ServiceConfig_ 和 _EndpointsConfig_**

~~~go
serviceConfig := config.NewServiceConfig()
endpointsConfig := config.NewEndpointsConfig()
~~~

由于proxy和kubernetes Service概念关系很大, 强烈建议读者访问[kubernetes Service](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md)官方文档了解其基本概念. 先介绍ServiceConfig结构体和相关操作NewServiceConfig(), 源代码如下, 定义在pkg/proxy/config/config.go中

```go
type ServiceConfig struct {
        mux     *config.Mux
        bcaster *config.Broadcaster
        store   *serviceStore
}
func NewServiceConfig() *ServiceConfig {
    updates := make(chan struct{})
    store := &serviceStore{updates: updates, services: make(map[string]map[string]api.Service)}
    mux := config.NewMux(store)
    bcaster := config.NewBroadcaster()
    go watchForUpdates(bcaster, store, updates)
    return &ServiceConfig{mux, bcaster, store}
}
```


ServiceConfig结构体跟踪记录Service配置信息的变化,他接受通过channel传递的Sevice上"set",　"add",　"remove"的操作,　并使用相应的handler函数处理这些变化. ServiceConfig通过组合的方式将config.Mux, config.Broadcaster和serviceStore类集成起来. 并启动一个goroutine watchForUpdates(bcaster, store, updates), watchForUpdates()函数源代码如下：

~~~go
func watchForUpdates(bcaster *config.Broadcaster, accessor config.Accessor, updates <-chan struct{}) {
        for _ = range updates {
            bcaster.Notify(accessor.MergedState())
        }
    }
~~~

他的作用是当updates channel有可用的信息时,　调用bcaster.Notify()函数以广播的方式将MergedState()处理后的serivces信息通知到各个Listener. 其中MergeState实际调用的是(s \*serviceStore) MergedState(); bcaster.Notify()函数源码如下,　它定义在pkg/util/config/config.go中

~~~go
// 通知所有listener.
func (b *Broadcaster) Notify(instance interface{}) {
    b.listenerLock.RLock()
    listeners := b.listeners
    b.listenerLock.RUnlock()
    for _, listener := range listeners {
        listener.OnUpdate(instance)
    }
}
~~~

listener 相关的结构体源代码如下, 可以看出_golang interface_的飘逸灵活用法:

~~~go
type Listener interface {
        // OnUpdate 在每次object 有变化时被调用.
        OnUpdate(instance interface{})
    }
    
    // ListenerFunc 是一个函数类型，他以下面的方式实现了OnUpdate()函数.
    type ListenerFunc func(instance interface{})
    
    func (f ListenerFunc) OnUpdate(instance interface{}) {
        f(instance)
    }
~~~

NewServiceConfig函数最后返回这个新建的ServiceConfig结构体. 

EndpointsConfig 结构体类似于ServiceConfig结构体, 相关的操作也与Service极其相似,也是采用组合的方式将config.Mux, config.Broadcaster, endpointsStore组合起来, 并马上启动一个goroutine watchForUpdates(),调用的和之前的watchForUpdates相同. 

**5\. 新建loadBalancerRR和代理proxier**

~~~go
loadBalancer := proxy.NewLoadBalancerRR()
proxier := proxy.NewProxier(loadBalancer, net.IP(bindAddress), iptables.New(exec.New(), protocol))
if proxier == nil {
    glog.Fatalf("failed to create proxier, aborting")
}
~~~

5.1 先介绍loadBalancer,源代码定义在pkg/proxy/roundrobin.go中,如下:

~~~go
// LoadBalancerRR is 是一个使用round robin方式的负载均衡器的实现.
    type LoadBalancerRR struct {
        lock          sync.RWMutex
        endpointsMap  map[string][]string　//记录每个service的后端endpoint映射
        rrIndex       map[string]int //记录每个service下一次round robin轮到的后端index
        serviceDtlMap map[string]serviceDetail　//保存服务的名字和细节的映射
    }
    // NewLoadBalancerRR 返回一个新的LoadBalancerRR结构体.
    func NewLoadBalancerRR() *LoadBalancerRR {
        return &LoadBalancerRR{
            endpointsMap:  make(map[string][]string),
            rrIndex:       make(map[string]int),
            serviceDtlMap: make(map[string]serviceDetail),
        }
    }
~~~

5.2 proxier源代码定义在pkg/proxy/proxier中,　如下

~~~go
// Proxier 结构体实现了一个简单的tcp代理
type Proxier struct {
    loadBalancer LoadBalancer
    mu           sync.Mutex // 保护下面的serviceMap数据结构
    serviceMap   map[string]*serviceInfo
    listenIP     net.IP
    iptables     iptables.Interface
    hostIP       net.IP
}
// NewProxier 新建并返回一个Proxier结构体
func NewProxier(loadBalancer LoadBalancer, listenIP net.IP, iptables iptables.Interface) *Proxier {
    ...
    hostIP, err := chooseHostInterface()
    ...
    // 清楚旧的iptables rules.
    iptablesDeleteOld(iptables)
    // 初始化 iptables.
    if err := iptablesInit(iptables); err != nil {
        glog.Errorf("Failed to initialize iptables: %v", err)
        return nil
    }
    
    if err := iptablesFlush(iptables); err != nil {
        glog.Errorf("Failed to flush iptables: %v", err)
        return nil
    }
    return &Proxier{
        loadBalancer: loadBalancer,
        serviceMap:   make(map[string]*serviceInfo),
        listenIP:     listenIP,
        iptables:     iptables,
        hostIP:       hostIP,
    }
}
~~~

如上面的代码注释所说NewProxier函数初始化主机上的iptables信息. proxier组成的元素主要包含刚刚创建的loadBalancer对象和一个iptables的执行器,他负责根据service的变化信息更新iptables设置.

**6\. 将刚刚新建的proxier和loadBalancer分别向servicesConfig和endpointsConfig绑定**

~~~go
serviceConfig.RegisterHandler(proxier)
endpointsConfig.RegisterHandler(loadBalancer)
~~~

他的作用是当service或endpoint的配置信息发生变化时，就调用proxier或loadbalancer的相关函数． 

下面先介绍serviceConfig.RegisterHandler(proxier),源代码如下:

~~~go
func (c *ServiceConfig) RegisterHandler(handler ServiceConfigHandler) {
    c.bcaster.Add(config.ListenerFunc(func(instance interface{}) {
        handler.OnUpdate(instance.([]api.Service))
    }))
}
~~~

这个函数的功能是将proxier的OnUpdate函数注册添加到上文说的serviceConfig的bcaster的Listener\[\] slice里,当从channel收到新的可用信息,　实际调用的是下面的OnUpdate()函数即Proxier对象的OnUpdate(). 这个函数很重要，由于代码段比较长,下面分段解释，他定义在pkg/proxy/proxier.go 中. 

首先看一下传进参数services的值

~~~go
func (proxier *Proxier) OnUpdate(services []api.Service) {
    glog.V(4).Infof("Received update notice: %+v", services)
~~~

从后端的一台minion机器上的kube-proxy.log查看一下传进来的services信息如下:

~~~go
I0112 09:48:02.876275   23642 proxier.go:443] Received update notice: [... ObjectMeta:{Name:redis-master ... 
Spec:{Port:6379 Protocol:TCP Selector:map[name:redis-master] PortalIP:11.1.1.83 ProxyPort:0 CreateExternalLoadBalancer:false 
PublicIPs:[] ContainerPort:{Kind:0 IntVal:6379 StrVal:} SessionAffinity:}]
~~~

如上展示的是kubernetes官方guest-example的redis-master的service, 他的重要参数是Spec这个字段，他包含service portal信息. 再继续往下进行,下面的代码段将刚收到的services信息和本地已有的通过info, exists := proxier.getServiceInfo(service.Name) 函数获得的info作比较. 如果service有更改,　则在后台重启或使用addServiceOnPort()函数新增代理:

~~~go
activeServices := util.StringSet{}
        for _, service := range services {
            activeServices.Insert(service.Name)
            info, exists := proxier.getServiceInfo(service.Name)
            serviceIP := net.ParseIP(service.Spec.PortalIP)
            
            //service 信息完全不变，不做任何操作
            if exists && info.portalPort == service.Spec.Port && info.portalIP.Equal(serviceIP) {
                continue
            }
            // service 信息发生改变，如portalPort或portalIＰ发生变化时，则重启
            if exists && (info.portalPort != service.Spec.Port || !info.portalIP.Equal(serviceIP) || !ipsEqual(service.Spec.PublicIPs, info.publicIP)) {
                glog.V(4).Infof("Something changed for service %q: stopping it", service.Name)
                err := proxier.closePortal(service.Name, info)
                ...
                err = proxier.stopProxy(service.Name, info)
                ...
            }
            ...
            info, err := proxier.addServiceOnPort(service.Name, service.Spec.Protocol, service.Spec.ProxyPort, udpIdleTimeout)
~~~

其中核心的addServiceOnPort()函数源代码如下, 他的最主要的作用是为每个service开启一个代理socket, 其中分配一个随机的port号为portNum, 并启动一个goroutine sock.ProxyLoop() 进行代理循环，这个函数定义在pkg/proxy/proxier.go中，它实现了一个round robin代理的逻辑，供读者自行阅读，这里由于篇幅所限不赘述:

~~~go
func (proxier *Proxier) addServiceOnPort(service string, protocol api.Protocol, proxyPort int, timeout time.Duration) (*serviceInfo, error) {
    // 新建一个代理socket
    sock, err := newProxySocket(protocol, proxier.listenIP, proxyPort)
    ...
    _, portStr, err := net.SplitHostPort(sock.Addr().String())
    ...
    portNum, err := strconv.Atoi(portStr)
    ...
    //将sessionAffinityType设置为None, 默认没有sessionAffinity
    si := &serviceInfo{
        proxyPort:           portNum,
        protocol:            protocol,
        socket:              sock,
        timeout:             timeout,
        sessionAffinityType: api.AffinityTypeNone,
        stickyMaxAgeMinutes: 180,
    }
   　//将新建的serviceInfo保存进proxier的serviceMap数据结构里
    proxier.setServiceInfo(service, si)

    glog.V(1).Infof("Proxying for service %q on %s port %d", service, protocol, portNum)
    go func(service string, proxier *Proxier) {
        defer util.HandleCrash()
        sock.ProxyLoop(service, proxier)
    }(service, proxier)

    return si, nil
}
~~~

观察后台kube-proxy log 如下, 49263 就是个随机分配的port：

~~~go
I0114 14:50:18.388765   21305 proxier.go:427] Proxying for service "redis-master" on TCP port 49623
~~~

回到OnUpdate函数里，之后将最新的service信息保存到info数据, 　由于proxier.getServiceInfo(service.Name) 返回的是指针, 所以最后实际上保存到了proxier.serviceMap这个数据结构里.

~~~go
info.portalIP = serviceIP
info.portalPort = service.Spec.Port
info.publicIP = service.Spec.PublicIPs
info.sessionAffinityType = service.Spec.SessionAffinity
info.stickyMaxAgeMinutes = 180
~~~

最后调用openPortal函数打开servicePortal

~~~go
err = proxier.openPortal(service.Name, info)
~~~

他实际调用的是如下的openOnePortal函数,他的源码如下,定义在pkg/proxy/proxier.go中

~~~go
func (proxier *Proxier) openOnePortal(portalIP net.IP, portalPort int, protocol api.Protocol, proxyIP net.IP, proxyPort int, name string) error {
        // 处理containers之间的流量.
        args := proxier.iptablesContainerPortalArgs(portalIP, portalPort, protocol, proxyIP, proxyPort, name)
        existed, err := proxier.iptables.EnsureRule(iptables.TableNAT, iptablesContainerPortalChain, args...)
        ...
        if !existed {
            glog.Infof("Opened iptables from-containers portal for service %q on %s %s:%d", name, protocol, portalIP, portalPort)
        } 
        ...
    }
~~~

他的作用是保证根据上文的service信息添加了正确的iptables rule,　并将相应的规则添加到iptables NAT表里, 最后调用的是pkg/util/iptables/iptables.go中的run函数,源代码如下:

~~~go
func (runner *runner) run(op operation, args []string) ([]byte, error) {
    iptablesCmd := runner.iptablesCommand()

    fullArgs := append([]string{string(op)}, args...)
    glog.V(1).Infof("running iptables %s %v", string(op), args)
    return runner.exec.Command(iptablesCmd, fullArgs...).CombinedOutput()

}
~~~

从后台的kube-proxy log中可以看到下面的iptables操作

~~~go
I0112 12:38:03.870741    7284 iptables.go:169] running iptables -N [KUBE-PROXY -t nat]
 I0112 12:38:03.877783    7284 iptables.go:169] running iptables -C [PREROUTING -t nat -j KUBE-PROXY]
 I0112 12:38:03.885954    7284 iptables.go:169] running iptables -C [OUTPUT -t nat -j KUBE-PROXY]
 I0112 12:38:03.893237    7284 iptables.go:169] running iptables -C [KUBE-PROXY -t nat -m comment --comment redis-master -p tcp -m tcp -d 11.1.1.83/32 --dport 6379 -j REDIRECT --to-ports 37292]
 I0112 12:38:03.904520    7284 iptables.go:169] running iptables -C [KUBE-PROXY -t nat -m comment --comment redisslave -p tcp -m tcp -d 11.1.1.176/32 --dport 6379 -j REDIRECT --to-ports 37334]
~~~

使用命令$ iptables -t nat -L 可以看到结果如下:

~~~go
Chain KUBE-PROXY (2 references)
target     prot opt source               destination         
REDIRECT   tcp  --  anywhere             11.1.1.83            /* redis-master */ tcp dpt:6379 redir ports 37292
REDIRECT   tcp  --  anywhere             11.1.1.176           /* redisslave */ tcp dpt:6379 redir ports 37334
REDIRECT   tcp  --  anywhere             11.1.1.1             /* redis-master2 */ tcp dpt:6379 redir ports 41074
REDIRECT   tcp  --  anywhere             11.1.1.169           /* kubernetes */ tcp dpt:https redir ports 54541
REDIRECT   tcp  --  anywhere             11.1.1.91            /* kubernetes-ro */ tcp dpt:http redir ports 35201
~~~

可以看到minion主机上的iptable新增了KUBE-PROXY CHAIN , 并且采用的是REDIRECT target. 这个CHAIN有两个引用, 分别是PREROUTING 和 OUTPUT CHAIN. 最后的结果是容器里的应用如果想通过11.1.1.176：6379 访问redisslave服务时,流量被REDIRECT到了本机的37334端口, 通过这个本地37334端口，再使用刚刚说到的每个service提供的代理socket导向到真正后端. 如下面的终端输出所示.这里的 11.1.1.176相当于一个虚拟的ip. 这个ip范围可在启动 kube-apiserver时指定portal\_net参数指定.

~~~shell
vcap@224:~$ nc -zv 11.1.1.176 6379
Connection to 11.1.1.176 6379 port [tcp/*] succeeded!
~~~

再使用proxier.loadBalancer.NewService函数

~~~go
proxier.loadBalancer.NewService(service.Name, info.sessionAffinityType, info.stickyMaxAgeMinutes)
~~~

源代码定义在pkg/proxy/roundrobin.go中,源代码如下:

~~~go
func (lb *LoadBalancerRR) NewService(service string, sessionAffinityType api.AffinityType, stickyMaxAgeMinutes int) error {
        if stickyMaxAgeMinutes == 0 {
            stickyMaxAgeMinutes = 180 //单位分钟
    　　　}
        if _, exists := lb.serviceDtlMap[service]; !exists {
            lb.serviceDtlMap[service] = *newServiceDetail(service, sessionAffinityType, stickyMaxAgeMinutes)
            glog.V(4).Infof("NewService.  Service does not exist.  So I created it: %+v", lb.serviceDtlMap[service])
        }
        return nil
    }
~~~

可以看到一个很重要的操作是维护serviceDtlMap这样一个golang map数据结构, key是相应service name. 这里sessionAffinityType有基于clientIPAddress的方式，也有最普通的None模式，也就是常用的round robin方式无sessionAffinity, 如下面的log就是创建了一个名为redis-master的sessionAffinityType为 None的 service.

~~~go
I0112 12:37:53.824530    7284 roundrobin.go:83] NewService.  Service does not exist.  So I created it: {name:redis-master sessionAffinityType:None sessionAffinityMap:map[] stickyMaxAgeMinutes:180}
~~~

OnUpdate函数最后的活动是对未活动的service, 即要被删除掉的service 关闭portal,　删除指定的iptables rule,　关闭代理套接字.　代码如下:

~~~go
　　proxier.mu.Lock()
　　defer proxier.mu.Unlock()
　　for name, info := range proxier.serviceMap {
        if !activeServices.Has(name) {
            ...
            err := proxier.closePortal(name, info)
            ...
            err = proxier.stopProxyInternal(name, info)
            ...
        }
    }
~~~

**7.回到main函数，下一步指定变化的源 ,一般采用etcd\_server ,即调用下面代码的else部分**

~~~go
if clientConfig.Host != "" {
        ...
    } else {
        var etcdClient *etcd.Client

        // 创建 etcd client
        if len(etcdServerList) > 0 {
            ...
            etcdClient = etcd.NewClient(etcdServerList)
        } ...
        
        if etcdClient != nil {
            glog.Infof("Using etcd servers %v", etcdClient.GetCluster())

            config.NewConfigSourceEtcd(etcdClient,
                serviceConfig.Channel("etcd"),
                endpointsConfig.Channel("etcd"))
        }
    }
~~~

流程为 7.1 先使用给定的命令行参数etcd\_servers 创建一个新的etcdClient对象, 使用的pkg是coreos的github.com/coreos/go-etcd/etcd pkg. 查看后端kube-proxy的log如下

~~~go
I0107 10:30:25.561301    5921 proxy.go:117] Using etcd servers [http://127.0.0.1:4001]
~~~

7.2 之后使用config.NewConfigSourceEtcd(etcdClient, serviceConfig.Channel("etcd"), endpointsConfig.Channel("etcd"))函数. 注意到这个函数的后两个参数是serviceConfig.Channel("etcd")和endpointsConfig.Channel("etcd")的返回值. 这两个函数除了make创建出相应的channel之外还做了一些额外的操作值得注意! 下面介绍其中的一个serviceConfig.Channel("etcd"),　endpointsConfig.Channel("etcd")雷同.

~~~go
func (c *ServiceConfig) Channel(source string) chan ServiceUpdate {
    ch := c.mux.Channel(source)
    serviceCh := make(chan ServiceUpdate)
    go func() {
        for update := range serviceCh {
            ch <- update
        }
        close(ch)
    }()
    return serviceCh
}
~~~

他调用Mux里的Channel函数创建一个新的channel ch,　并且创建一个传送ServiceUpdate 的channel serviceCh, 一旦serviceCh有可用的信息如update就将他写入ch. 之后再回来看一下ConfigSourceEtcd结构体和相关的NewConfigSourceEtcd函数，它的源代码定义在pkg/proxy/config/etcd.go, 代码如下:

~~~go
type ConfigSourceEtcd struct {
        client           *etcd.Client
        serviceChannel   chan ServiceUpdate
        endpointsChannel chan EndpointsUpdate
        interval         time.Duration
}
 　func NewConfigSourceEtcd(client *etcd.Client, serviceChannel chan ServiceUpdate, endpointsChannel chan EndpointsUpdate) ConfigSourceEtcd {
        config := ConfigSourceEtcd{
            client:           client,
            serviceChannel:   serviceChannel,
            endpointsChannel: endpointsChannel,
            interval:         2 * time.Second,
        }
        go config.Run()
        return config
    }
~~~

NewConfigSourceEtcd作用是创建一个新的ConfigSourceEtcd结构体, 这个结构体里封装了etcdClient和两个传输信息的Channel: 1 serviceChannel 2 endpointsChannel.并启动一个goroutine config.Run()函数.源代码如下:

```go
// Run函数watch　etcd 上与service 和endpoint相关的key的变化，并将新的处理好的serviceUpdate和endpointUpate值输出到指定channel
// 如刚刚提到的1 serviceChannel 2 endpointsChannel.
func (s ConfigSourceEtcd) Run() {
　　　...
    for {
    　　// 通过查询etcd上/registry/services/specs下的相关key获得service 和endpoint列表
        services, endpoints, err = s.GetServices()
        if err != nil {
            ...
    　　} else {
         　// 传输到相应channel，如刚刚说到的(c *ServiceConfig) Channel函数里的所谓serviceCh　channel
            if len(services) > 0 {
                serviceUpdate := ServiceUpdate{Op: SET, Services: services}
                s.serviceChannel <- serviceUpdate
            }
            ...

        }
        time.Sleep(30 * time.Second)
    }
}
```

**8.启动health监控**

```go
if *healthz_port > 0 {
            go util.Forever(func() {
                err := http.ListenAndServe(bindAddress.String()+":"+strconv.Itoa(*healthz_port), nil)
                if err != nil {
                    glog.Errorf("Starting health server failed: %v", err)
                }
            }, 5*time.Second)
        }
```

目前什么http的HandleFunc都没有,访问healthz\_port会遇到404 not found error. 

**9.最后调用SyncLoop()**

```go
// Just loop forever for now...
    proxier.SyncLoop()
```

其源代码定义在pkg/proxy/proxier.go中，如下：

```go
func (proxier *Proxier) SyncLoop() {
    for {
        select {
        // 每隔syncInterval　默认值是５s进行一次，保证iptables设置及portal正常工作
        // 并清理除AffinityTypeNone之外的超时session
        case <-time.After(syncInterval):
            glog.V(2).Infof("Periodic sync")
            if err := iptablesInit(proxier.iptables); err != nil {
                glog.Errorf("Failed to ensure iptables: %v", err)
            }
            proxier.ensurePortals()
            proxier.cleanupStaleStickySessions()
        }
    }
}
```

**总结:** 

1 每个Kubernetes节点都运行一个kube-proxy这么一个服务代理,它 watch kubernetes 集群里 service 和 endpoints(label是某一特定条件的pods)这两个对象的增加或删除,　并且维护一个service 到 endpoints的映射. 他使用iptables REDIRECT target将对服务的请求流量转接到本地的一个port上,　然后再将流量发到后端,　这样的转发还支持一些策略,　如round robin等，所以我们可以把他看成是一个具有高级功能的反向代理。 

2 下面为Kube-proxy内部goroutine及channel 使用图，以service信息传递为例 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605835425/sel/serviceChannel_sxqhle.png" alt="serviceChannel" style="zoom:67%;" />
</center>