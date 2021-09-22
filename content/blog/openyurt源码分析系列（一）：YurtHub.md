+++

id= "15"

title = "openyurt源码分析系列（一）：YurtHub"
description = "YurtHub的功能主要是对客户端Request的反向代理。本文基于Openyurt 0.3版本（commit: `e1c1651405aeeb61f0ad264a9acdf8d08e917a4d`）的源码，提供了对反向代理功能的解析（不包括YurtHub的本地缓存管理：CacheManager，storage和GC）"
tags= [ "openyurt"]
date= "2021-03-24 10:00:11"
author = "张逸飞"
banner= "img/blogs/15/arch.png"
categories = [ "openyurt" ]

+++





YurtHub的功能主要是对客户端Request的反向代理。本文基于Openyurt 0.3版本（commit: `e1c1651405aeeb61f0ad264a9acdf8d08e917a4d`）的源码，提供了对反向代理功能的解析（不包括YurtHub的本地缓存管理：CacheManager，storage和GC）。

<!--more-->



## 一、YurtHubServer源码分析

YurtHubServer是执行ListenAndServe的主体，其主要结构和功能是YurtReverseProxy，即边缘节点的反向代理。YurtHubServer将YurtReverseProxy挂载在几乎所有路径（以`/`为前缀的路径，但也有一些路径另作他用）上，当HTTP请求的URL路径满足条件就会交由YurtReverseProxy处理，但在此之前会对request通过context添加一些额外的信息。

### 挂载的路径和功能

YurtHubServer挂载的路径，方法和handler的对应关系：

| Path                               | Method    | Handler       |
| ---------------------------------- | --------- | ------------- |
| `/v1/token`                        | POST, GET | updateToken   |
| `/v1/healthz`                      | GET       | healthz       |
| `/debug/pprof` and `/debug/pprof/` | ALL       | pprof.Index   |
| `/debug/pprof/profile`             | ALL       | pprof.Profile |
| `/debug/pprof/symbol`              | ALL       | pprof.Symbol  |
| `/debug/pprof/trace`               | ALL       | pprof.Trace   |
| Others                             | ALL       | proxyHandler  |

其中：

updateToken是用来更新节点的证书。

healthz是直接返回StatusOK，用于其它节点对本节点的健康检查。

pprof相关的handler是用来提供profile服务。

proxyHandler提供其主要的反向代理功能。



### proxyHandler功能

YurtHubServer在接收到HTTP request后，在通过YurtReverseProxy转发前，会先对request进行处理。流程如下：

1. 检查是否超过服务器request上限：当pending的request数量超过设置的阈值，拒绝请求。
2. 提取requestInfo：从request中提取具体信息requestInfo，其中包括verb（例如Watch, List），是否是resource请求等信息，并将requestInfo以键值对`requestInfoKey:requestInfo`的形式通过context附加在request上。

3. 附加ClientComponent信息：如果是resource请求，从`request.Header`中提取出`User-Agent`对应的值，以键值对`ProxyClientComponent:component`的形式通过context附加在request上。（component信息表示发送请求的是哪个部件，如kubelet，kube-proxy等）
4. 是否设置Timeout：如果`verb == Watch`，从request中提取Timeout（如果有的话）参数，以context with deadline的形式附加在request上。

5. 是否需要缓存：根据request.Header中`Edge-Cache`键对应的值`needToCache`（bool），以键值对`ProxyReqCanCache:needToCache`的形式通过context	附加在request上。

6. 附加ContentType信息：将request.Header中`Accept`键对应的值`contentType`，以键值对`ProxyReqContentType:contentType`的形式通过context附加在request上。

7. 跟踪response状态码

在这之后才会将request转交给YurtReverseProxy。



其构造handler的函数源码如下：

```go
func (p *yurtReverseProxy) buildHandlerChain(handler http.Handler) http.Handler {
	handler = util.WithRequestTrace(handler)
	handler = util.WithRequestContentType(handler)
	handler = util.WithCacheHeaderCheck(handler)
	handler = util.WithRequestTimeout(handler)
	handler = util.WithRequestClientComponent(handler)
	handler = filters.WithRequestInfo(handler, p.resolver)
	handler = util.WithMaxInFlightLimit(handler, p.maxRequestsInFlight)
	return handler
}
```

这里值得注意的是其执行顺序，当接收到request后，其通过的顺序应该是由下而上的。其中`With*`函数大体都是如下模式：

```go
func With*(handler http.handler, other-parameters) http.Handler{
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		// some statements
        handler.ServeHTTP(w,req)
	})
}
```

因此在执行完`handler = util.WithRequestTrace(handler)`后，handler变为`with*`函数的返回值函数，但原先的handler并没有消失，而是在返回值函数中以`handler.ServeHTTP(w,req)`的形式存在。因此`with*`函数相当于在handler执行逻辑上增加了一些执行语句（some statements），对request处理时从最上面的语句开始执行，即从最后的`with*`函数逻辑开始向上执行。



### Links

Next:  [YurtReverseProxy源码分析](#YurtReverseProxy源码分析)

</br>

</br>









## 二、YurtReverseProxy源码分析

### YurtReverseProxy主要功能

代理来自边缘节点（Kubelet，Kube-Proxy等）的REST请求，根据APIServer的健康状况，如果健康则将该请求通过Load Balancer转发到APIServer处理，否则在本地节点Local Proxy中进行处理。最后将reponse返回给请求方。

<img src="https://res.cloudinary.com/rachel725/image/upload/v1616579999/sel/22271616579851_.pic_hd_r9pnx5.png" alt="arch" style="zoom:30%;" />



### YurtReverseProxy结构

YurtReverseProxy内有：

1. LoadBalacer：能判断APIServer的健康状态，并能将请求转发到APIServer处理。
2. LocalProxy：能根据本地缓存情况处理请求。
3. CacheManager：用来管理本地缓存（交由LocalProxy使用，YurtReverseProxy并不直接使用）。
4. RequestInfoResolver：用来解析request得到requestInfo。

结构体如下：

```go
type yurtReverseProxy struct {
	resolver            apirequest.RequestInfoResolver
	loadBalancer        remote.LoadBalancer
	localProxy          *local.LocalProxy
	cacheMgr            cachemanager.CacheManager
	maxRequestsInFlight int // 用来限制pending request数量的上限
	stopCh              <-chan struct{}
}
```



### YurtReverseProxy工作流程

作为YurtHubServer的一个组件，它接受YurtHubServer指定Path（除了`/v1/token`，`/v1/healthz`和一些`/debug`下的路径外）的request，并进行处理。

通过LoadBalancer判断APIServer的健康状况，如果健康则调用LoadBalancer处理请求，否则调用LocalProxy处理请求。

具体源码如下：

```go
func (p *yurtReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	if p.loadBalancer.IsHealthy() {
		p.loadBalancer.ServeHTTP(rw, req)
	} else {
		p.localProxy.ServeHTTP(rw, req)
	}
}
```



### Links

Previous：<a href="#YurtHubServer源码分析">YurtHubServer源码分析</a>

Next:  

1. [LoadBalancer源码分析](#LoadBalancer源码分析)

2. [LocalProxy源码分析](#LocalProxy源码分析)









</br>

</br>



## 三、LoadBalancer源码分析

### LoadBalancer的主要功能

在APIServer处于健康状态时，LoadBalancer接收来自YurtReverseProxy的request，并转发到APIServer。LoadBalancer中记录了多个能处理request的APIServer的地址，在转发request时会采取一定的负载均衡策略。



### 结构

#### LoadBalancer的结构

LoadBalancer内主要有APIServer代理列表（`[]*RemoteProxy`)和负载均衡策略（`loadBalancerAlgo`）。APIServer代理（RemoteProxy）代理一个能处理request的APIServer。LoadBalancer根据设置的负载均衡策略，挑选一个APIServer代理，将request通过APIServer代理转发到APIServer。

```go
type loadBalancer struct {
	backends    []*RemoteProxy
	algo        loadBalancerAlgo
	certManager interfaces.YurtCertificateManager
}
```

#### RemoteProxy的结构

RemoteProxy内置有

- RemoteProxy：利用`net/http/httputil`中的ReverseProxy来完成代理的主要工作。
- CacheManager：利用缓存中的信息修改从APIServer返回的response
- HealthChecker，用来判断APIServer是否健康
- RemoteServer，用来记录APIServer的信息。

```go
// RemoteProxy is an reverse proxy for remote server
type RemoteProxy struct {
	checker      healthchecker.HealthChecker
	reverseProxy *httputil.ReverseProxy
	cacheMgr     cachemanager.CacheManager
	remoteServer *url.URL
	stopCh       <-chan struct{}
}
```



### LoadBalancer的工作流程

1. 根据一定的负载均衡策略选择一个健康的APIServer
2. 将request交APIServer代理处理

源码如下：

```go
func (lb *loadBalancer) ServeHTTP(rw http.ResponseWriter, req *http.Request){
    // 通过策略选择一个APIServer代理
	b := lb.algo.PickOne()
    
    // ...
	// omit code of error handling and logging
    // ...
    
    // 交由APIServer代理处理
	b.ServeHTTP(rw, req)
}
```

APIServer代理收到request后直接调用httputil中的RemoteProxy处理。

```go
func (rp *RemoteProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	rp.reverseProxy.ServeHTTP(rw, req)
}
```

除了默认的处理流程外，将收到response发回前可能还会对response的内容进行处理和修改。主要为：

1. 如果request的操作为"watch"，会在response.Header中添加`Transfer-Encoding:chunked`，使用长连接处理watch请求。
2. 如果request设置了缓存（根据request的context的`ProxyReqCanCache:needToCache`，或是使用其它条件判断），则会调用CacheManager将reponse的内容缓存到本地。



### 负载均衡策略

目前LoadBalancer中内置了两种负载均衡策略：Round-Robin和基于优先级的策略。一个LoadBalancer采用哪种负载均衡策略在创建时由`lbMode`参数指定：`rr`表示Round-Robin，`priority`表示基于优先级的调度策略。默认使用的Round-Robin。LoadBalancer统一通过策略的PickOne()接口来选出一个能够转发请求的APIServer。

**策略结构**

```go
type loadBalancerAlgo interface {
	PickOne() *RemoteProxy
	Name() string
}
```



**Roud-Robin策略原理**

如果没有健康的APIServer，则返回nil。否则通过不回退的轮询来找到第一个健康的APIServer。

轮询部分的源码：

```go
hasFound := false
selected := rr.next // next 是上次轮询的终点位置
// rr.backends是记录的所有APIServer的序列
for i := 0; i < len(rr.backends); i++ {
    selected = (rr.next + i) % len(rr.backends)
    if rr.backends[selected].IsHealthy() {
        hasFound = true
        break
    }
}

if hasFound {
    rr.next = (selected + 1) % len(rr.backends)
    return rr.backends[selected]
}
```



**基于优先级的策略**

使用基于优先级的策略时，需要手动按优先级将APIServer序列排序。策略本身的实现是从头开始的轮询。

实现源码：

```go
// prio.backends是记录的所有APIServer的序列
for i := 0; i < len(prio.backends); i++ {
    if prio.backends[i].IsHealthy() {
    	return prio.backends[i]
    }
}
```



### Links

Previous: [YurtReverseProxy源码分析](#YurtReverseProxy源码分析)







</br>

</br>





## 四、LocalProxy源码分析



### LocalProxy的主要功能

在APIServer非健康状态下，接受来自YurtReverseProxy的request，并根据本地缓存和request的内容进行处理。目前只支持ResourceRequest。处理内容主要是根据ResourceRequest中对资源的操作类型（verb）来更新本地缓存。



### LocalProxy的结构

LocalProxy内有：

1. CacheManager：用来管理本地缓存
2. IsHealthy：是传入的LoadBalancer功能，用来判断APIServer是否健康

结构体如下：

```go
// LocalProxy is responsible for handling requests when remote servers are unhealthy
type LocalProxy struct {
	cacheMgr  manager.CacheManager
	isHealthy IsHealthy
}
```



### LocalProxy的处理流程

1. 过滤非ResourceRequest，并返回BadRequest Response

2. 根据操作类型分别进行处理：

   - **watch：**

     在Response.Header中设置`Transfer-Encoding:chuncked`，以长连接处理watch请求。然而目前LocalProxy并不支持本地处理watch，因此直到Timeout都在周期性探测APIServer是否健康，如果感知到APIserver恢复健康则直接返回nil，下次watch将由LoadBalancer处理。

   - **create：**

     如果请求的resource是event，则调用`CacheManager.CacheResponse`将请求的内容缓存到本地。并总会返回一个response，header和body与request相同，状态码为201(StatusCreated)。

   - **delete || deletecollection：**

     LocalProxy不支持对resource的delete请求，因此返回的Response的Reason为"Forbidden"，状态码为200(StatusOK)。

   - **list || get || update || patch：**

     根据request中的信息生成key，调用`CacheManager.QueryCache`根据key找到缓存中的runtime.object(s)，将其写入response并返回。

服务功能的伪代码如下：

```
if reqestInfo != nil and requestInfo.IsResourceRequest{
	switch requestInfo.Verb{
	case "watch":
		localWatch(request)
	case "create"：
		localPost(request)
	case "delete" or "deletecollection":
		localDelete(request)
	default:
		localReqCache(request)
	}
}
else{
	resp.Write(errors.BadRequest)
}
```



### Links

Previous: [YurtReverseProxy源码分析](#YurtReverseProxy源码分析)
