+++

id= "76"

title = "Cloud Foundry中gorouter源码分析"
description = "在Cloud Foundry v1版本中，router作为路由节点，转发所有进入Cloud Foundry的请求。由于开发语言为ruby，故router接受并处理并发请求的能力受到语言层的限制。虽然在v1版本中，router曾经有过一定的优化，采用lua脚本代替原先的ruby脚本，由lua来分析请求，使得一部分请求不再经过ruby代码，而直接去DEA访问应用，但是，一旦router暴露在大量的访问请求下，性能依旧是不尽如人意. 为了提高Cloud Foundry router的可用性，Cloud Foundry开源社区不久前推出了gorouter。gorouter采用现阶段比较新颖的go作为编程语言，并重新设计了原有的组件架构。由于go语言本身的特性，gorouter处理并发请求的能力大大超过了router，甚至在同种实验环境下，性能是原先router的20倍左右。"
tags= [ "gorouter" , "cloudfoundry" ]
date= "2014-05-07 10:20:09"
author = "丁轶群"
banner= "img/blogs/76/gorouter-76-1.png"
categories = [ "cloudfoundry" ]

+++

在Cloud Foundry v1版本中，router作为路由节点，转发所有进入Cloud Foundry的请求。由于开发语言为ruby，故router接受并处理并发请求的能力受到语言层的限制。虽然在v1版本中，router曾经有过一定的优化，采用lua脚本代替原先的ruby脚本，由lua来分析请求，使得一部分请求不再经过ruby代码，而直接去DEA访问应用，但是，一旦router暴露在大量的访问请求下，性能依旧是不尽如人意. 

为了提高Cloud Foundry router的可用性，Cloud Foundry开源社区不久前推出了gorouter。gorouter采用现阶段比较新颖的go作为编程语言，并重新设计了原有的组件架构。由于go语言本身的特性，gorouter处理并发请求的能力大大超过了router，甚至在同种实验环境下，性能是原先router的20倍左右。 

<!--more-->

由于gorouter的高性能，笔者也抱着期待的心态去接触go，当然还有gorouter。本文不会从go语言语法的角度入手gorouter，所以有一些go语言的基础再来看本文，是有必要的。本文主要是对gorouter的源码的简单解读，另外还包含一些笔者对gorouter的看法。 

**gorouter的程序组织形式** 
----------

首先，先从gorouter的程序组织形式入手，可见下图：
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616378/sel/gorouter-76-1_uvz43w.png" alt="" style="zoom:80%;" />
</center>



以下简单介绍其中一些重要文件的功能： 


- common：common意指通用，所以该文件夹中也是一些比较通识的概念定义，比如varz，healthz，component等，以及关于项目过程的一些基本操作定义。 
- config：顾名思义，该文件夹中的文件为gorouter组件的配置文件。 
- log：定义gorouter的log形式定义。 
- proxy：作为一个代理处理外界进入Cloud Foundry的所有请求。 
- registry：处理组件或者DEA中应用到gorouter来注册uri的事件，另外还负责请求访问应用时查找应用真实IP，port。 
- route：主要定义在rigistry中需要使用到的三个数据结构：endpoint，pool和uris。 
- router：程序的主入口，main函数所在处。 
- stats：主要负责一些应用记录的状态，还有一些其他零碎的东西，比如定义一个堆。 
- util：其中一般是工具源码，在这里只负责给gorouter进程写pid这件事。 
- varz：主要涉及varz信息的处理，其实就是gorouter组件状态的查阅。 
- router.go: 主要定义了router的数据结构，及其实例初始化的过程，还有最终运行的流程。  

**gorouter的功能**：gorouter的功能主要可以分为三个部分：负责接收Cloud Foundry内部组件及应用uri注册以及注销的请求，负责转发所有外部对Cloud Foundry的访问请求，负责提供gorouter作为一个组件的状态监控。 

- **接受uri注册及注销请求** 当Cloud Foundry内一个组件需要提供HTTP服务的时候，那么这个组件则必须将自己的uri和IP一起注册到gorouter处，典型的有，Cloud Foundry中Service Gateway与Cloud Controller通过HTTP建立连接的，另外Cloud Controller也需要对外提供HTTP服务，所以这些组件必须在gorouter中进行注册，以便可以顺利通信或访问。 除了平台级的组件uri注册，最常见的是应用级的应用uri注册，也就是在Cloud Foundry中新部署应用时，应用所在的DEA会向gorouter发送一个uri，IP和port的注册请求。gorouter收到这个请求后，会添加该记录，并保证可以解析外部的URL访问形式。当然，反过来，当一个应用被删除的时候，为了不浪费Cloud Foundry内部的uri资源，Cloud Foundry会将该uri从gorouter中注销，随即gorouter在节点处删除这条记录。 
- **转发对Cloud Foundry的访问请求** gorouter接受到的访问请求大致可以分为三种：外部请求有：用户对应用的访问请求，用户对Cloud Foundry内部资源的管理请求；内部的请求有：内部组件之间通过HTTP的各类通信。 虽然说请求的类型可以分为三种，但是gorouter对于这些请求的操作都是一致的，找到相应的uri，提取出相应的IP和port，然后进行转发。需要注意的是，在原先版本的router中，router只能接收HTTP请求，然而现在gorouter中，已经考虑了TCP连接，以及websocket。 
- **提供组件监控** Cloud Foundry都有自己的状态监控，可以通过HTTP访问。这主要是每个组件在启动的时候，都作为一个component向Cloud Foundry进行注册，注册的时候带有很多关于自身组件的信息，同时也启动了一个HTTP server。 

**gorouter的初始化及启动流程**和**Router对象实例的创建与初始化**
----------

gorouter的启动过程主要在router.go文件中，在该文件中，首先定义创建一个Router实例的操作并进行初始化，另外还定义了Router实例的开始运行所做的操作。 在router.go文件中，首先需要是Router结构体的定义： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
type Router struct {  
    config     *config.Config  
    ……  
}  
```


随后又定义了Router实例的初始化： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
func NewRouter(c *config.Config) *Router {  
    router := &Router{  
        config: c,  
    }  
        ……  
    return router  
}  
```


还有就是定义了Router实例开始运行时所做的操作： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
func (router *Router) Run() {  
        ……  
}  
```


查看源码可以发现Router结构体有以下几个属性： config：负责传入gorouter所需要的配置信息 proxy：一个代理对象，负责完成请求的转发 mbusClient：作为gorouter中的nats\_client，负责与Cloud Foundry的消息中间件NATS通信 registry：作为gorouter中的注册模块，完成Cloud Foundry中注册或注销请求的处理 varz：处理gorouter自身作为一个组件的状态监控 component：gorouter作为一个组件的信息，将自身的信息存入该component对象 在初始化Router对象实例的时候，都是通过传入的config文件中的配置文件来完成初始化。首先通过创建一个Router实例，并初始化该实例的配置信息： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
router := &Router{  
        config: c,  
    }  
```


然后通过读取该配置属性的信息逐步完成Router实例其他属性的初始化。 创建完Router实例对象router之后，router首先做的是创建一个用来与Cloud Foundry中与nats\_server建立联机的nats\_client: mbusClient： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
router.establishMBus()  
```


然后分别初始化了router对象的registry，varz，proxy： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
    router.registry = registry.NewRegistry(router.config, router.mbusClient)  
    router.registry.StartPruningCycle()  

    router.varz = varz.NewVarz(router.registry)  
    router.proxy = proxy.NewProxy(router.config, router.registry, router.varz)  
```


接着较为重要的是：router.component的创建和执行启动操作： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
router.component = &vcap.VcapComponent{  
        Type:        "Router",  
        Index:       router.config.Index,  
        Host:        host,  
        Credentials: []string{router.config.Status.User, router.config.Status.Pass},  
        Config:      router.config,  
        Varz:        varz,  
        Healthz:     healthz,  
        InfoRoutes: map[string]json.Marshaler{  
            "/routes": router.registry,  
        },  
    }  
    vcap.StartComponent(router.component)  
```


最后返回了router实例对象之后，创建与初始化工作即完成了。 


**Router实例对象的运行操作** 
----------

在Router对象的函数run()中，几乎执行了所有的Router实例对象的运行操作。 1.router对象使用mbusClient周期性地去连接Cloud Foundry的nats\_server： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
for {  
    err = router.mbusClient.Connect()  
    if err == nil {  
        break  
    }  
    log.Errorf("Could not connect to NATS: %s", err)  
    time.Sleep(500 * time.Millisecond)  
}  
```


2.router通过mbusClient将自己作为一个component注册到Cloud Foundry: router.RegisterComponent() 3.router订阅其他组件和应用要注册或注销的消息： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
router.SubscribeRegister()  
router.HandleGreetings()  
router.SubscribeUnregister()  
```


4.router通过SendStartMessage()发布router.start的消息： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
router.SendStartMessage()  

// Send start again on reconnect  
router.mbusClient.OnConnect(func() {  
    router.SendStartMessage()  
})  
```


5.周期性的刷新活着的应用的app\_id。 6.等待一个start信息的发送时间，以保证gorouter内registry映射表中已经有路由信息，以便而后在代理外部请求的时候，可以找到路由表的映射关系。所以，很显然大家可以发现，gorouter会将组件或者应用的uri注册信息存放在该自身的内存中，而gorouter关闭的时候，映射表中所有的信息丢失，每当重启的时候，需要通过发送一个start消息，然后靠订阅该消息的组件重新注册uri，从而获取所有的路由关系。 7.以TCP的方式监听本机的一个端口： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
listen, err := net.Listen("tcp", fmt.Sprintf(":%d", router.config.Port))  
if err != nil {  
    log.Fatalf("net.Listen: %s", err)  
}  
```


8.写pid文件：util.WritePidFile(router.config.Pidfile) 9.创建proxy中Server结构体的实例server，并最终一个协程来执行这个server服务于刚才创建的Listen对象： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
server := proxy.Server{Handler: router.proxy}  
go func() {   
    err := server.Serve(listen)  
    if err != nil {  
        log.Fatalf("proxy.Serve: %s", err)  
    }  
}()  
```


以上便是gorouter的router实例在运行时所需要作的操作，当然其中很多模块在实现功能的时候，还定义了其他的函数辅助实现，几乎都在router.go文件的函数定义部分，代码本身不难理解，可以对源码进行仔细阅读。 

**registry模块源码分析** 
----------

registry模块接管的是Cloud Foundry中组件及应用的uri注册或者注销请求。从计算和存储的角度来分析该模块，即可发现该模块完成了请求的处理和自身内存路由表的设计与维护。 首先来分析一下registry的Registry对象： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
type Registry struct {  
    sync.RWMutex  

    *steno.Logger  

    *stats.ActiveApps  
    *stats.TopApps  

    byUri map[route.Uri]*route.Pool  

    table map[tableKey]*tableEntry  

    pruneStaleDropletsInterval time.Duration  
    dropletStaleThreshold      time.Duration  

    messageBus mbus.MessageBus  

    timeOfLastUpdate time.Time  
}  
```


在该对象中，有两个非常重要的属性byUri和table。 可以看到byUri属性是一个map类型，map的key类型为route.Uri,value类型为\*route.pool。那么现在去route/uris.go和route/pool.go中去看一下这些数据结构。uris.go中由定义 type Uri string，那说明它本身就是一个字符串类型，而后可以发现，这是主要域名的形式。而pool类型要稍显复杂，可以理解为是一个路由池，因为在实际情况中，如果DEA上的一个应用由多个实例的话，那么一个uri会对应于多个IP+port的组合。pool拥有一个属性为endpoints，该属性又是一个map类型，key为string，value为Endpoint，具体形式如下： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
type Pool struct {  
    endpoints map[string]*Endpoint  
}  
```


而Endpoint的定义在route/endpoint.go文件中，如下： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
type Endpoint struct {  
    sync.Mutex  

    ApplicationId     string  
    Host              string  
    Port              uint16  
    Tags              map[string]string  
    PrivateInstanceId string  
}  
```


同样的table属性也是map类型，key为tableKey，value为\*tableEntry，随后在相同文件中，有者两个属性的定义： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```ruby
type tableKey struct {  
    addr string  
    uri  route.Uri  
}  

type tableEntry struct {  
    endpoint  *route.Endpoint  
    updatedAt time.Time  
}  
```


关于uri的注册，可以参看Resgister函数： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
func (registry *Registry) Register(uri route.Uri, endpoint *route.Endpoint) {  
    registry.Lock()  
    defer registry.Unlock()  

    uri = uri.ToLower()  

    key := tableKey{  
        addr: endpoint.CanonicalAddr(),  
        uri:  uri,  
    }  

    var endpointToRegister *route.Endpoint  

    entry, found := registry.table[key]  
    if found {  
        endpointToRegister = entry.endpoint  
    } else {  
        endpointToRegister = endpoint  
        entry = &tableEntry{endpoint: endpoint}  

        registry.table[key] = entry  
    }  

    pool, found := registry.byUri[uri]  
    if !found {  
        pool = route.NewPool()  
        registry.byUri[uri] = pool  
    }  

    pool.Add(endpointToRegister)  

    entry.updatedAt = time.Now()  

    registry.timeOfLastUpdate = time.Now()  
}  
```

其中，lock()函数负责将registry对象上锁，随后的defer语句，表示当整个Register函数执行完毕后，有go语言来完成registry对象的解锁操作。 key为一个tableKey的实例，其中addr为"IP:port"形式的string值，同时创建一个Endpoint类型的endpointToRegister，对于需要注册的（uri，endpoint）组合，首先查看table属性中能都找到键为key的记录，如果找到，那说明该key(实为IP+port,uri的组合)已经存在于table中，所以将table中的记录赋值于endpointToRegister；如果没有找到，那说明该key还未存在于table中，属于一个全新的key，需要在table中相应的记录，则首先用请求中的endpoint赋值给endpointToRegister，然后在通过endpoint创建一个endtry对象，并使用语句：registry.table\[key\] = entry来实现最终在table中的存储。当gorouter需要解析域名的时候使用的是byUri数据结构，所以在注册的时候也要对byUri进行操作。首先通过请求中的uri来在byUri中查找是否存在该uri的路由表，如果没有找到的话，则需要新建一个路由池pool，在将整个路由池pool，映射到相应的域名上，也就是请求中的uri。随后还需要给该路由池pool添加endpointToRegister整个对象，由于pool的一条记录本身是map类型的，所以在执行添加时，以endpointToPoint的（IP+port）作为该记录的key，整个endpointToRegister作为value。最后再更新一些其他属性。 以上便是Register函数所做的一些工作，Unregister函数做的工作则是一些注销工作。 随后则是一些关于byUri的查找，主要由以下几种类型： Lookup(uri route.Uri) 通过uri查找，返回pool.Sample(),其实也就是将pool随机返回一条记录，具体可以查看pool的Sample（）函数。 

LookupByPrivateInstanceId(uri route.Uri, p string) 通过 PrivateInstanceID查找，返回pool匹配该PrivateInstanceID的endpoint。 lookupByUri(uri route.Uri) 通过uri查找，返回整个路由池pool。 以上是关于uri的注册或者注销，另外gorouter还会对路由表进行一定的管理，主要是清理一些很陈旧的路由记录。首先在Router实例对象的初始化中就有陈旧路由信息的剪枝：router.registry.StartPruningCycle()，然后通过建立一个协程进行go registry.checkAndPrune(),通过中间一系列的操作之后，执行pruneStaleDroplets()，遍历table对象中所有的记录，并按条件进行剪枝。 

**proxy模块源码分析**和**server部分** 
----------

可以说作为一个路由节点，proxy是其最为重要的功能，registry这样的模块，其实也是为了能够服务于proxy。对于Cloud Foundry来说，所有通过uri访问Cloud Foundry内部资源的请求，都需要路由节点gorouter作代理。gorouter的proxy模块，首先监听底层的网络端口，然后再将端口发来的请求进行uri解析，最终将请求转发至指定的Cloud Foundry内部节点处。 在proxy模块，实现过程几乎可以从源码中百分百的呈现。从代理流程来讲，proxy模块可以认为是一个方向代理server端，接收所有从nginx发来的请求，并把请求转发至Cloud Foundry内的某些组件处。从实现方式来看，proxy模块建立一条nginx发来请求的连接，根据请求的内部具体信息，做相应的HTTP处理，最终构建该请求的response信息，并通过刚才的连接，将response信息返回给Nginx，当然Nginx最后也会把请求返回给发起请求的用户。其中，刚才提到的相应的HTTP处理，也就是如何将请求发给Cloud Foundry内的某些组件，并接收返回的信息。 粗略划分的话，proxy模块可以分为server端的实现与proxy代理流程的实现。首先从源码入手，第一个需要了解的自然是一些proxy模块中server的重要数据结构，位于server.go文件中。 conn：代表一条连到proxy模块中server上的连接，或者说是从Nginx到gorouter的连接。其中，有需要访问的远程目标的地址，该连接连上的server对象等。 \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片 

```go
	type conn struct {  
		remoteAddr string // network address of remote side  
		server *Server // the Server on which the connection arrived  
		rwc net.Conn // i/o connection  
		lr *io.LimitedReader // io.LimitReader(rwc)  
		buf *bufio.ReadWriter // buffered(lr,rwc), reading from bufio->limitReader->rwc  
		hijacked bool // connection has been hijacked by handler  
	}
```

request：代表请求对象，其中包括http类型中的请求对象，也包含一个response返回信息的对象。 \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片 

```go
	type request struct {  
		*http.Request  
		w *response  
	} 
```

response：代表从server端返回去的一条response的HTTP信息，其中包括这条返回信息返回时的承载的连接，还有很多关于该HTTP响应的属性值。 \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片 

```go
    type response struct {  
		conn *conn  
		reqWantsHttp10KeepAlive bool  
        reqMethod               string  
        reqProtoAtLeast10       bool  
        reqProtoAtLeast11       bool  
        reqExpectsContinue      bool  
        reqContentLength        int64  

        chunking      bool        // using chunked transfer encoding for reply body  
        wroteHeader   bool        // reply header has been written  
        wroteContinue bool        // 100 Continue response was written  
        header        http.Header // reply header parameters  
        written       int64       // number of bytes written in body  
        contentLength int64       // explicitly-declared Content-Length; or -1  
        status        int         // status code passed to WriteHeader  

        closeAfterReply bool  

        requestBodyLimitHit bool  
    }  
```


server：代表接收请求，转发请求的server端，其中包含一个远程地址，另外还有一个非常重要的handler对象，用来处理HTTP请求，如果对Nginx源码熟悉的话，对Handler这个模块应该不会陌生。另外，这里的handler其实就是proxy.go文件中定义的proxy结构体。 \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
	type Server struct {  
		Addr string // TCP address to listen on, ":http" if empty  
		Handler http.Handler // handler to invoke, http.DefaultServeMux if nil  
		ReadTimeout time.Duration // maximum duration before timing out read of the request  
		WriteTimeout time.Duration // maximum duration before timing out write of the response  
		MaxHeaderBytes int // maximum size of request headers, DefaultMaxHeaderBytes if 0  
	}
```

源码本身是从结构体入手，并进行方法定义，为了便于理解，以下采用请求流程的方式对源码进行解读。 要想对server结构有初步的了解，那必须从server的函数Server（）入手。简单代码形式如下（已省略部分异常处理等代码）： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
func (srv *Server) Serve(l net.Listener) error {  
    defer l.Close()  
    var tempDelay time.Duration // how long to sleep on accept failure  
    for {  
        rw, e := l.Accept()  
        ……  
        c, err := srv.newConn(rw)  
        ……  
        go c.serve()  
    }  
    panic("not reached")  
}  
```


该Serve（）函数的发起者为Server实例对象，传入的参数为对端口的监听对象。在执行该函数的时候，defer方法显性的定义了关于函数执行完毕后所需要处理的后续工作。关于server的具体执行操作，不难理解的服务器端需要不断轮询端口，并对端口处发来的请求进行相应的处理。在这里的代码实现即为一个for循环，在该循环中，首先server实例对象accept一个连接。这里的原理和socket的实现很类似，首先作为一个server端，先去监听listen，某一个端口，然后去accept这个端口发来的连接请求，也就是说，一旦有连接请求发来的话，server便会去accept该请求；然后作为一个client端，所需要做的操作就是去给server端的某一端口发送连接请求，如果有server监听了这个端口，那么它可以accept该连接请求；最后双方可以通信。 首先，server通过代码 rw, e := l.Accept() 实现对监听端口请求的接受；然后server再对这个net.Conn类型的rw，进行处理，最后生成一个新的连接，也就是上面涉及到的conn结构对象；接着，server创建一个协程来完成这条连接上的请求。可以发现的是，由于在gorouter中server对象只有一个，所以所有的请求都是经过这个server的，那在for循环中，server会接受很多的请求，创建很多的连接，然后对于对于一个连接上的请求，又会创建一个协程来完成，如果不借助协程的高并发处理能力，几乎不能应对大负载。 以上是对Server实例对象执行时的代码入口的解读，当真正处理请求的时候，是在go c.serve()处，其中go代表这开辟一个协程，c.serve()则是处理的具体实现。 以下是对serve（）函数的分析，首先来看函数中的主要代码： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
func (c *conn) serve() {  
    defer func() {  
        ……  
        }  
    }()  

    for {  
        req, w, err := c.readRequest()  
        ……  
        // Expect 100 Continue support  
        if req.expectsContinue() {  
            ……  
        } else if req.Header.Get("Expect") != "" {  
            ……  
        }  

        handler := c.server.Handler  
        ……  
        handler.ServeHTTP(w, req.Request)  
        ……  
        req.finishRequest()  
        if w.closeAfterReply {  
            break  
        }  
    }  
    c.close()  
}  
```


defer关键字依旧是表示随后定义的函数是做来为serve（）方法作善后处理。接着是一个for循环，在该for循环中，首先从连接中读取一个请求，然后对该请求的某些属性进行查阅并处理，接着创建一个handler，最后由该handler来处理HTTP请求，并结束一个请求，如果该请求是一个一次连接，那么关于该连接，如果该请求处于长连接上，则继续for循环的下一次迭代，继续从连接中读取请求并处理。 现在我们涉及函数中具体实现。 在for循环中，首先读取连接中的请求，代码形式为：req, w, err := c.readRequest(), 该实现，返回两个对象，一个为读取的请求，另一个为需要生成的response。在readRequest（）函数中，代码的形式为（只显示主要部分）： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
func (c *conn) readRequest() (r *request, w *response, err error) {  
    ……  
    var req *http.Request  
    if req, err = http.ReadRequest(c.buf.Reader); err != nil {  
        ……  
    }  
    c.lr.N = noLimit  

    req.RemoteAddr = c.remoteAddr  

    w = new(response)  
    w.conn = c  

    r = new(request)  
    r.Request = req  
    r.w = w  

    ……  
    return r, w, nil  
}  
```


可见，执行该函数的时候，首先通过http的函数ReadRequest（）来实现从连接c中读取请求，然后通过该请求，分别创建一个request对象和response对象。需要注意的是代码w.conn=c，也就是在说创建玩reponse对象后，对对象属性初始化时，将response的连接熟悉感conn，依旧赋值为c，那么当server将该请求转发给Cloud Foundry内部组件处理后收到回复，并对回复再进行处理，来完成这里的response重写后，依旧通过之前的连接发回去。这样的实现显得更加高效，之前我一直在考虑，想nginx之类的反向代理服务器，接收请求转发请求，接收回复转发回复的流程，如果都需要重新创建连接来完成的话，连接的开销会巨大，对于一些有长连接需求的http请求，重建连接的机制会显得非常笨重。在这里的go语言实现中，由于自定义了response的结构体，又轻松地实现了连接捆绑，所以不需要考虑连接的重新创建，但是这样的方式肯定也会付出一定的代价，比如说内存的消耗等，因为每个response实例对象中，都会存放一个连接信息。 在serve（）函数可以看到，在读取连接请求readRequest（）后，对请求进行一些处理之后，会创建一个handler对象来实现HTTP请求的处理，由于该部分的实现在proxy.go文件中，所以本文稍后即会涉及，简单来讲就是给后台做代理，将请求发给后台，并接收后台的回复。 当获得后台的响应请求后，server随即执行finishRequest（）函数，其主要的功能就是将返回的后台回复，写入需要返回给用户的response对象中。然后判断response对象中的属性closeAfterReply，如果为真，则表示之前的请求是一个一次请求，该请求表明，自身发出之后接收到回复之后，不会再发起请求，就算有，也情愿是在创建一个连接来实现，所以程序跳出for循环，关闭连接；如果为假的话，那说明请求需要在一条长连接上进行操作，换言之，在请求的回复发给用户后，用户还会有请求通过这条连接发给server，这样的话，无需关闭连接，只需有server继续对这条连接执行readRequest（）函数即可。 

**proxy部分** 
----------

相比较而言，proxy部分做的工作要比server部分少一些，它主要的工作就是解析请求的uri和转发请求。 关于解析请求的uri的工作，在函数Lookup（）中实现： \[plain\] view plaincopy在CODE上查看代码片派生到我的代码片

```go
func (proxy *Proxy) Lookup(request *http.Request) (*route.Endpoint, bool) {  
    uri := route.Uri(hostWithoutPort(request))  

    if _, err := request.Cookie(StickyCookieKey); err == nil {  
        if sticky, err := request.Cookie(VcapCookieId); err == nil {  
            routeEndpoint, ok := proxy.Registry.LookupByPrivateInstanceId(uri, sticky.Value)  
            if ok {  
                return routeEndpoint, ok  
            }  
        }  
    }  
    return proxy.Registry.Lookup(uri)  
}  
```


首先，找到请求中的host，然后对于该请求，检查是否有StickyCookieKey，如果有的话，直接从中获取sticky，再通过uri和sticky.value的组合找到相应的routeEndpoint，也就是请求的backend。这里可以稍微解释一下StickyCookieKey的作用。一旦一个请求中含有该cookie，而且能被解析到相应的uri和sticky值，也就是说这个请求，希望被处理的时候，能继续被上次处理过这个用户发出的请求的app instance上，这样的话，可以避免一些不必要的数据冲突等，或者减少DEA中 app instance的负载。如果没有找到cookie的话，那么proxy就老老实实通过host来找到相应的ip：port，如果一个host有多个instance实例的话，proxy会通过某种策略来决策由哪个Instance来服务。 在转发请求的时候，实现在函数ServeHTTP（）方法中，在实现过程中，我们需要清楚其中的几个重要的方面即可：构建一个responseWriter并初始化某些属性，判断请求的类型并分别处理（TCP、websocket和HTTP），若为HTTP类型则通过transport.RoundTrip方法发送请求并接收响应，最后填写reponseWriter和设置cookie。这一部分的代码随着gorouter版本的更新会有一些形式上的不同，但是主要的功能和思想都是一致的。 以上就是对gorouter一些模块的源码分析。

**转载请注明出处。**
------------
这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 