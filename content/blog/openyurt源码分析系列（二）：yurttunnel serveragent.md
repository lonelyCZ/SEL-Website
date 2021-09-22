+++

id= "16"

title = "openyurt源码分析系列（二）：yurttunnel server/agent"
description = "本文主要对 OpenYurt 中的 Yurttunnel Server/Agent 模块进行了源码分析，内容包括模块启动流程、各个模块组件的工作流程以及通信。"
tags= [ "openyurt"]
date= "2021-05-13 10:00:11"
author = "刘佳文"
banner= "img/blogs/16/process.png"
categories = [ "openyurt" ]

+++

本文主要对 OpenYurt 中的 Yurttunnel Server/Agent 模块进行了源码分析，内容包括模块启动流程、各个模块组件的工作流程以及通信。（源码版本：OpenYurt Version: `0.3.0`	commit: `b3fda60af24ed63d84328417e625ad85c453d4af`）

<!--more-->



## Yurttunnel Server

### 总览

在边缘计算场景中，边缘节点可能处于专有网络中，导致节点（kubelet）无法与云端  kube-apiserver 直接通信。

yurttunnel server 与 yurttunnel agent 主要解决云边网络隔离时，云边通信的问题。每个边缘网络域中设置 agent，通过云端 proxy server 的公网 IP 与 agent 建立长连接，云端组件对边缘节点的访问请求通过 proxy server 转发至目的边缘节点。其中，proxy server 与 agent 之间的通信通过上游项目 apiserver-network-proxy(ANP) 实现。

本质上 yurttunnel 是一个反向通道（即边缘节点向 server 建立连接的时候，同时建立 server 与边缘节点的反向通道）。



### 启动流程

yurttunnel Server 判断是否需要开启 iptables Manager 进行本地 nat 表管理，iptabls Manager 每隔一定时间获取集群中的 configmap 以及 node 信息来更新本地的跳转规则。

创建并启动 certManager，会创建 CSR，将得到证书用于其与 KAS 的安全通信。并维持证书的更新。其中 CSR 的内容为：

```go
// util/certificate/certificate_manager.go

csr := &certificates.CertificateSigningRequest{
		// Username, UID, Groups will be injected by API server.
		TypeMeta: metav1.TypeMeta{Kind: "CertificateSigningRequest"},
		ObjectMeta: metav1.ObjectMeta{
			Name: "",
      GenerateName: "csr-",
		},
		Spec: certificates.CertificateSigningRequestSpec{
			Request: &x509.CertificateRequest{
        Subject: pkix.Name{
          CommonName:   commonName,  // kube-apiserver-kubelet-client
          Organization: organizations,  // []string{ "system:masters", "openyurt:yurttunnel"}
        },
        DNSNames:    dnsNames,
        IPAddresses: ipAddrs,
      }
			Usages:   []certificates.KeyUsage{
				certificates.UsageAny,  // any
			}
		},
	}
```

之后创建并启动 CSRApprover，启动多个协程（2个）处理集群中 yurttunnel server 相关的 CSR（csr 中字段 organizations 列表是否包含了 "openyurt:yurttunnel"）：如果还没有进行批准的，进行人工批准。

根据 kubeconfig 或者 /var/run/secrets/kubernetes.io/serviceaccount/ca.crt 生成 certPool（包含了验证服务器相关证书），通过 certmanager 获取当前证书，再生成  tls 配置文件。

初始化所有的 HandlerWrappers，即如果需要配置 SharedInformerFactory，则配置。其中 HandlerWrappers 的定义如下：

```go
// pkg/yurttunnel/handlerwrapper/handlerwrapper.go

type HandlerWrappers []Middleware

type Middleware interface {
	WrapHandler(http.Handler) http.Handler
	Name() string
}
```

对于需要配置 SharedInformerFactory 的 Middleware，还另外需要实现接口：

```go
// pkg/yurttunnel/handlerwrapper/initializer/initalizer.go

// WantsSharedInformerFactory is an interface for setting SharedInformerFactory
type WantsSharedInformerFactory interface {
	SetSharedInformerFactory(factory informers.SharedInformerFactory) error
}
```

完成 sharedInformerFactory 所有配置之后，启动 sharedInformerFactory，并等待 certManager 创建证书完成。

创建 TunnelServer 并运行，监听本地 0.0.0.0:10263（https 消息）和127.0.0.1:10264（http 消息），将消息格式转换为 ANP Proxy Server 处理的格式并转发至本地运行的 ANP Proxy Server，其将请求转发给对应的 agent。



### 组件成员分析

#### iptable Manager

server 所在的节点每隔一定时间更新本地的 iptables 的 nat 表，主要是增加以下两种 chain：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1620979780/sel/image-20210402131707694_ktzwsp.png"  style="zoom:40%;" />
</center>

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1620979780/sel/image-20210331175234630_osnrz7.png"  style="zoom:40%;" />
</center>



其中 port、node IP 通过获取集群中的 configmap 以及 node 信息来更新。

> 比如：
>
> server 节点 S1；server 节点 S2，服务端口为 P1；yurthub-node A，服务端口 P1。
>
> 初始化时， iptables 就会建立一个 chain：TUNNEL-PORT-P1，并建立以上形式的规则，在 TUNNEL- PORT 表中建立一条rule：-j TUNNEL-PORT-P1。
>
> 如果一个 server 节点需要发送给 yurthub-node A，A 的接收端口为 P1。之后 server 节点发送给A 的消息就会转发给本地 proxy server 监听的端口。
>
> 如果一个 server 节点需要发送给 server-node B，B 的接收端口为 P1。之后 server 节点发送给A 的消息就会转发给本地 proxy server 监听的端口。
>
> 其中端口（如 P1）、node IP 通过获取集群中的 configmap 以及 node 信息来更新。



#### CertManager

certManager 负责管理 yurttunnel server 与 master 连接的证书。

启动 certManager 之后，certManager 每隔一定时间或者监测到证书 template 发生变化时，certManager 会根据函数 getTemplate 创建 csr（certificate signing request）的 spec.Request 字段：

```go
// pkg/yurttunnel/pki/certmanager/certmanager.go

getTemplate := func() *x509.CertificateRequest {
		return &x509.CertificateRequest{
			Subject: pkix.Name{
				CommonName:   commonName,  // signerName：kube-apiserver-kubelet-client
				Organization: organizations,  // []string{ "system:masters", "openyurt:yurttunnel"}
			},
			DNSNames:    dnsNames,
			IPAddresses: ipAddrs,
		}
	}
```

并 watch 等待处理结果，如果 csr 最终状态为 Approved，则通过 `status.certificate` 字段获取证书，并更新本地证书文件。如果1）出错；2）等待超时；3） csr 的状态为 denied，则继续等待下一次更新。

```go
// util/certificate/certificate_manager.go

func (m *manager) rotateCerts() (bool, error) {
	klog.V(2).Infof("Rotating certificates")

	template, csrPEM, keyPEM, privateKey, err := m.generateCSR()  // create csr
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Unable to generate a certificate signing request: %v", err))
		return false, nil
	}

	// request the client each time
	client, err := m.getClient()
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Unable to load a client to request certificates: %v", err))
		return false, nil
	}

	// Call the Certificate Signing Request API to get a certificate for the
	// new private key.
	req, err := csr.RequestCertificate(client, csrPEM, "", m.usages, privateKey)  
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Failed while requesting a signed certificate from the master: %v", err))
		return false, m.updateServerError(err)
	}

	......
}
```



#### CSRApprover

CSRApprover 用于人工认证（approve）cert Manager 创建的 csr，其数据结构如下：

```go
// pkg/yurttunnel/pki/certmanager/csrapprover.go

// YurttunnelCSRApprover is the controller that auto approve all
// yurttunnel related CSR
type YurttunnelCSRApprover struct {
	csrInformer certv1beta1.CertificateSigningRequestInformer  // 缓存 csr 相关信息
	csrClient   typev1beta1.CertificateSigningRequestInterface
	// RateLimitingInterface is an interface that rate limits items being added to the queue.
	workqueue   workqueue.RateLimitingInterface  // 存放 csr key（<namespace>/<name> 或者 <name>），对 csr 进行认证处理
}

```

在创建 CSRApprover 时，为 csr informer 添加 Add、Update 处理函数：enqueueObj——即将Add 或者 Update 的 csr 的 key（`<namespace>/<name>` 或者 `<name>`） 放入 workqueue 中。

```go
// pkg/yurttunnel/pki/certmanager/csrapprover.go

// NewCSRApprover creates a new YurttunnelCSRApprover
func NewCSRApprover(clientset kubernetes.Interface,
                    csrInformer certinformer.CertificateSigningRequestInformer) *YurttunnelCSRApprover {
	wq := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
	csrInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			enqueueObj(wq, obj)
		},
		UpdateFunc: func(old, new interface{}) {
			enqueueObj(wq, new)
		},
	})
	return &YurttunnelCSRApprover{
		csrInformer: csrInformer,
		csrClient:   clientset.CertificatesV1beta1().CertificateSigningRequests(),
		workqueue:   wq,
	}
}
```

启动 CSRApprover，等待本地 csrInformer 进行同步，同步完成之后，启动 threadiness（默认为2）个协程，每个协程每隔一定时间（1s）就去运行函数 runWorker，直到接收到结束信号，就关闭这个协程。

runWorker 中调用了 processNextItem 方法，主要处理 workqueue 中的 key（csr 的 `<namespace>/<name>` 或者 `<name>`），如果根据 key 能够从 informer 中拿到 obj，检查 csr 的状态，如果状态既不是 approved，也不是 denied，那么尝试人工批准并且更新，如果成功，则这个 key 处理完成；否则，重新加入队列处理。

```go
// pkg/yurttunnel/pki/certmanager/csrapprover.go

func (yca *YurttunnelCSRApprover) runWorker() {
	for yca.processNextItem() {
	}
}

func (yca *YurttunnelCSRApprover) processNextItem() bool {
	// Get blocks until it can return an item to be processed. If shutdown = true,
	// the caller should end their goroutine. You must call Done with item when you
	// have finished processing it.
	key, quit := yca.workqueue.Get()
	if quit {
		return false
	}
	csrName, ok := key.(string)
	if !ok {
		yca.workqueue.Forget(key)
		runtime.HandleError(
			fmt.Errorf("expected string in workqueue but got %#v", key))
		return true
	}

	defer yca.workqueue.Done(key)

	csr, err := yca.csrInformer.Lister().Get(csrName) 
	if err != nil { 
		runtime.HandleError(err)
		if !apierrors.IsNotFound(err) { 
			yca.workqueue.AddRateLimited(key)
		}
		return true
	}

	if err := approveYurttunnelCSR(csr, yca.csrClient); err != nil {  // 检查 csr 的状态，尝试人工批准它
		runtime.HandleError(err)
		enqueueObj(yca.workqueue, csr)
		return true
	}
	return true
}
```

csrApprover 和 certManager 的合作流程为：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1620979780/sel/image-20210402112715032_ocnw4d.png"  style="zoom:30%;" />
</center>

#### anpTunnelServer

1）创建  http.Server，监听地址为 udsSockFile（/tmp/interceptor-proxier.sock），handler 为 anpserver.Tunnel，其结构为：

```go
// pkg/server/tunnel.go
type Tunnel struct {
	Server *ProxyServer
}

func (t *Tunnel) ServeHTTP(w http.ResponseWriter, r *http.Request) {
...
}
```

其中成员 ProxyServer 为 ANP Proxy Server，负责管理与 yurttunnel Agent 之间的长连接，并将请求转发给对应的 agent。

2）创建 http.Server，监听地址为0.0.0.0:10263（负责监听 master 发来的 https 消息），TLSConfig 为之前生成的 tls 配置文件，handler 为 wrappedHandler，其作用为将监听到的消息封装成 anp proxy server 需要的格式，并将其转发至 anpserver.Tunnel。其核心 handler 为 RequestInterceptor，并且包装了之前已经初始化的 HandlerWrappers。其中 RequestInterceptor 结构为：

```go
// pkg/yurttunnel/server/interceptor.go

// ReqRequestInterceptor intercepts http/https requests sent from the master,
// prometheus and metric server, setup proxy tunnel to kubelet, sends requests
// through the tunnel and sends responses back to the master
type RequestInterceptor struct {
	contextDialer func(addr string, header http.Header, isTLS bool) (net.Conn, error)  // 与本地"/tmp/interceptor-proxier.sock" 建立连接
}

// ServeHTTP will proxy the request to the tunnel and return response from tunnel back
// to the client
func (ri *RequestInterceptor) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// 1. setup the tunnel
	tunnelConn, err := ri.contextDialer(r.Host, r.Header, r.TLS != nil)
	if err != nil {
		klogAndHttpError(w, http.StatusServiceUnavailable,
			"fail to setup the tunnel: %s", err)
		return
	}
	defer tunnelConn.Close()

	// 2. proxy the request to tunnel
	if err := r.Write(tunnelConn); err != nil {
		klogAndHttpError(w, http.StatusServiceUnavailable,
			"fail to write request to tls connection: %s", err)
		return
	}

	// 3.1 handling the upgrade request
	if httpstream.IsUpgradeRequest(r) {
		serveUpgradeRequest(tunnelConn, w, r)
		return
	}

	// 3.2 handling the request
	serveRequest(tunnelConn, w, r)
}
```

3）创建 http.Server，监听地址为127.0.0.1:10264（负责监听 master 发来的 http 消息），handler 为 wrappedHandler。

消息传递路径如下：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1620965218/sel/image-20210402132602123_x6gelt.png"  style="zoom:30%;" />
</center>

## Yurttunnel Agent

### 启动流程：

通过unnel server 相关的 service、ep、cloud nodes，获取集群中一个可用的 yurttunnel server 的连接地址（host:port）。

创建并启动 certManager，会创建 CSR，将得到证书用于其与 yurttunnel server 的安全通信。并维持证书的更新。

根据  /var/run/secrets/kubernetes.io/serviceaccount/ca.crt 生成 certPool（包含了验证服务器相关证书），通过 certmanager 获取当前证书，再生成  tls 配置文件。

创建 TunnelAgent 并运行，其本质是一个 anpagent。





## 通信流程



<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1620965036/sel/image-20210402114614715_shmtdg.png"  style="zoom:30%;" />
</center>





## 参考资料

[OpenYurt 深度解读：如何构建 Kubernetes 原生云边高效协同网络？](https://mp.weixin.qq.com/s?__biz=MzUzNzYxNjAzMg==&mid=2247495998&idx=1&sn=0fce8d2e736f911cecf1a35d83c5bd26&chksm=fae6faf1cd9173e787c8cb1e5f704083c24497ecd64451e0eb39855164d155f6f70b05089e88&cur_album_id=1410601643242307586&scene=190#rd)