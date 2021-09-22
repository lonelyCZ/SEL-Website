+++
id = "259"

title = "Docker源码分析（五）：Docker Server的创建"
description = "Docker架构中，Docker Server是Docker Daemon的重要组成部分。Docker Server最主要的功能是：接受用户通过Docker Client发送的请求，并按照相应的路由规则实现路由分发。"
tags = ["Docker"]
date = "2014-12-09 13:00:37"
author = "孙宏亮"
banner = "img/blogs/259/docker.jpg"
categories = ["Docker"]

+++



**1\. Docker Server简介**
=======================

Docker架构中，Docker Server是Docker Daemon的重要组成部分。Docker Server最主要的功能是：接受用户通过Docker Client发送的请求，并按照相应的路由规则实现路由分发。 

<!--more-->

同时，Docker Server具备十分优秀的用户友好性，多种通信协议的支持大大降低Docker用户使用Docker的门槛。除此之外，Docker Server设计实现了详尽清晰的API接口，以供Docker用户选择使用。通信安全方面，Docker Server可以提供安全传输层协议（TLS），保证数据的加密传输。并发处理方面，Docker Daemon大量使用了Golang中的goroutine，大大提高了服务端的并发处理能力。 

本文为《Docker源码分析》系列的第五篇——Docker Server的创建。

**2\. Docker Server源码分析内容安排**
=============================

本文将从源码的角度分析Docker Server的创建，分析内容的安排主要如下：

1.  “serveapi”这个job的创建并执行流程，代表Docker Server的创建；
2.  “serveapi”这个job的执行流程深入分析；
3.  Docker Server创建Listener并服务API的流程分析。

**3\. Docker Server创建流程**
=========================

[《Docker源码分析（三）：Docker Daemon启动》](http://www.sel.zju.edu.cn/?p=158)主要分析了Docker Daemon的启动，而在mainDaemon()运行的最后环节，实现了创建并运行名为”serveapi”的job。这一环节的作用是：让Docker Daemon提供API访问服务。实质上，这正是实现了Docker架构中Docker Server的创建与运行。 

从流程的角度来说，Docker Server的创建并运行，代表了”serveapi”这个job的整个生命周期：创建Job实例job，配置job环境变量，以及最终执行该job。本章分三节具体分析这三个不同的阶段。

**3.1 创建名为”serveapi”的job**
--------------------------

Job是Docker架构中Engine内部最基本的任务执行单位，故创建Docker Server这一任务的执行也不例外，需要表示为一个可执行的Job。换言之，需要创建Docker Server，则必须创建一个相应的Job。具体的Job创建形式位于[./docker/docker/daemon.go](https://github.com/docker/docker/blob/v1.2.0/docker/daemon.go#L66)，如下：

```go
job := eng.Job("serveapi", flHosts...)
```

以上代码通过Engine实例eng创建一个Job类型的实例job，job名为”serveapi”，同时用flHost的值来初始化job.Args。flHost的作用是：配置Docker Server监听的协议与监听的地址。 

需要注意的是，《Docker源码分析（三）：Docker Daemon启动》mainDaemon()具体实现过程中，在加载builtins环节已经向eng对象注册了key为”serveapi”的Handler，而该Handler的value为api.ServeApi。因此，在运行名为”serveapi”的job时，会执行该job的Handler，即api.ServeApi。

**3.2 配置job环境变量**
-----------------

创建完Job实例job之后，Docker Daemon为job配置环境参数。在Job实现过程中，为Job配置参数有两种方式：第一，创建Job实例时，用指定参数直接初始化Job的Args属性；第二，创建完Job后，给Job添加指定的环境变量。以下代码则实现了为创建的job配置环境变量：

```go
job.SetenvBool("Logging", true)
job.SetenvBool("EnableCors", *flEnableCors)
job.Setenv("Version", dockerversion.VERSION)
job.Setenv("SocketGroup", *flSocketGroup)

job.SetenvBool("Tls", *flTls)
job.SetenvBool("TlsVerify", *flTlsVerify)
job.Setenv("TlsCa", *flCa)
job.Setenv("TlsCert", *flCert)
job.Setenv("TlsKey", *flKey)
job.SetenvBool("BufferRequests", true)
```

对于以上配置，环境变量的归纳总结如下表：
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605755091/sel/2014-12-09-121005-_E7_9A_84_E5_B1_8F_E5_B9_95_E6_88_AA_E5_9B_BE_y2gays.png" alt="2014-12-09 12:10:05 的屏幕截图" style="zoom:90%;" />
</center>

**3.3 运行job**
-------------

配置完毕job的环境变量，随即执行job的运行函数，具体实现代码如下：

```go
if err := job.Run(); err != nil {
    log.Fatal(err)
}
```


在eng对象中已经注册过key为”serveapi”的Handler，故在运行job的时候，执行这个Handler的value值，相应Handler的value为api.ServeApi。至此，名为”serveapi”的job的生命周期已经完备。下文将深入分析job的Handler，api.ServeApi执行细节的具体实现。

**4\. ServeApi运行流程**
====================

本章将深入分析Docker Server提供API服务的部分，从源码的角度剖析Docker Server的架构设计与实现。 

作为一个监听请求、处理请求的服务端，Docker Server首先明确自身需要为多少种通信协议提供服务，在Docker这个C/S模式的架构中，可以使用的协议无外乎三种:TCP协议，Unix Socket形式，以及fd的形式。随后，Docker Server根据协议的不同，分别创建不同的服务端实例。最后，在不同的服务端实例中，创建相应的路由模块，监听模块，以及处理请求的Handler，形成一个完备的server。 

”serveapi”这个job在运行时，将执行api.ServeApi函数。ServeApi的功能是：循环检查所有Docker Daemon当前支持的通信协议，并对于每一种协议都创建一个goroutine，在这个goroutine内部配置一个服务于HTTP请求的server端。ServeApi的代码实现位于[./docker/api/server/server.go#L1339](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1339-L1368)： 

第一，判断job.Args的长度是否为0，由于通过flHosts来初始化job.Args，故job.Args的长度若为0的话，说明没有Docker Server没有监听的协议与地址，参数有误，返回错误信息。代码如下：

```go
if len(job.Args) == 0 {
    return job.Errorf("usage: %s PROTO://ADDR [PROTO://ADDR ...]", job.Name)
}
```


第二，定义两个变量，protoAddrs代表flHosts的内容；而chError定义了和protoAddrs长度一致的error类型channel管道，chError的作用在下文中会说明。同时还定义了activationLock，这是一个用来同步”serveapi”和”acceptconnections”这两个job执行的channel。在serveapi运行时ServeFd和ListenAndServe的实现中，由于activationLock这个channel中没有内容而阻塞，而当运行”acceptionconnections”这个job时，会首先通知init进程Docker Daemon已经启动完毕，并关闭activationLock，同时也开启了serveapi的继续执行。正是由于activationLock的存在，保证了”acceptconnections”这个job的运行起到通知”serveapi”开启正式服务于API的效果。代码如下：

```go
var (
    protoAddrs = job.Args
    chErrors   = make(chan error, len(protoAddrs))
)
activationLock = make(chan struct{})
```


第三，遍历protoAddrs，即job.Args，将其中的每一项都按照字符串“://”进行分割，若分割后protoAddrParts的长度不为2，则说明协议加地址的书写形式有误，返回job错误；若不为2，则分割获得每一项中的协议protoAddrPart\[0\]与地址protoAddrParts[1](http://www.sel.zju.edu.cn/?p=158)。最后分别创建一个goroutine来执行ListenAndServe的操作。goroutine的运行主要依赖于ListenAndServe(protoAddrParts\[0\], protoAddrParts[1](http://www.sel.zju.edu.cn/?p=158), job)的运行结果，若返回error，则chErrors中有error，当前goroutine执行完毕；若没有返回error，则该goroutine持续运行，持续提供服务。其中最为重要的是ListenAndServe的实现，该函数具体实现了如何创建listener、router以及server，并协调三者进行工作，最终服务于API请求。代码如下：

```go
for _, protoAddr := range protoAddrs {
    protoAddrParts := strings.SplitN(protoAddr, "://", 2)
    if len(protoAddrParts) != 2 {
        return job.Errorf("usage: %s PROTO://ADDR [PROTO://ADDR ...]", job.Name)
    }
    go func() {
        log.Infof("Listening for HTTP on %s (%s)", proto    AddrParts[0], protoAddrParts[1])
        chErrors <- ListenAndServe(protoAddrParts[0], protoAddrParts[1], job)
    }()
}
```


第四，根据chErrors的值运行，若chErrors这个channel中有错误内容，则ServeApi该函数返回；若无错误内容，则循环被阻塞。代码如下：

```go
for i := 0; i < len(protoAddrs); i += 1 {
    err := <-chErrors
    if err != nil {
        return job.Error(err)
    }
}
    return engine.StatusOK
```


至此， ServeApi的运行流程已经详细分析完毕，其中核心部分ListenAndServe的实现，下一章开始深入。

**5\. ListenAndServe实现**
========================

ListenAndServe的功能是：使Docker Server监听某一指定地址，接受该地址上的请求，并对以上请求路由转发至相应的处理函数Handler处。从实现的角度来看，ListenAndServe主要实现了设置一个服务于HTTP的server，该server将监听指定地址上的请求，并对请求做特定的协议检查，最终完成请求的路由与分发。代码实现位于[./docker/api/server/server.go](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1247-L1335)。 

ListenAndServe的实现可以分为以下4个部分：

1.  创建router路由实例；
2.  创建listener监听实例；
3.  创建http.Server；
4.  启动API服务。

ListenAndServe的执行流程如下图：
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605755389/sel/ListenAndServe_uajd36.jpg" alt="ListenAndServe" style="zoom:50%;" />
</center>
<center>图5.1 ListenAndServer执行流程图</center>

下文将按照ListenAndServe执行流程图一一深入分析各个部分。

**5.1 创建router路由实例**
--------------------

首先，ListenAndServe的实现中通过createRouter创建了一个router路由实例。代码实现如下：

```go
r, err := createRouter(job.Eng, job.GetenvBool("Logging"), job.GetenvBool("EnableCors"), job.Getenv("Version"))
if err != nil {
    return err
}
```

createRouter的实现位于[./docker/api/server/server.go#L1094](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1094-L1171)。 

创建router路由实例是一个重要的环节，路由实例的作用是：负责Docker Server对请求进行路由以及分发。实现过程中，主要两个步骤：第一，创建全新的router路由实例；第二，为router实例添加路由记录。

### **5.1.1 创建空路由实例**

实质上，createRouter通过包gorilla/mux实现了一个功能强大的路由器和分发器。如下：

```go
r := mux.NewRouter()
```


NewRouter()函数返回了一个全新的router实例r。在创建Router实例时，给Router对象的两个属性进行赋值，这两个属性为nameRoutes和KeepContext。其中namedRoutes属性为一个map类型，其中key为string类型，value为Route路由记录类型；另外，KeepContext属性为false，表示Docker Server在处理完请求之后，就清除请求的内容，不对请求做存储操作。代码位于[./docker/vendor/src/github.com/gorilla/mux/mux.go#L16](https://github.com/docker/docker/blob/v1.2.0/vendor/src/github.com/gorilla/mux/mux.go#L16-L18)，如下：

```go
func NewRouter() *Router {
    return &Router{namedRoutes: make(map[string]*Route), KeepContext: false}
}
```


可见，以上代码返回的类型为mux.Router。mux.Router会通过一系列已经注册过的路由记录，来为接受的请求做匹配，首先通过请求的URL或者其他条件，找到相应的路由记录，并调用这条路由记录中的执行Handler。mux.Router有以下这些特性：

*   请求可以基于URL 的主机名、路径、路径前缀、shemes、请求头和请求值、HTTP请求方法类型或者使用自定义的匹配规则；
*   URL主机名和路径可以拥有一个正则表达式来表示；
*   注册的URL可以被直接运用，也可以被保留，这样可以保证维护资源的使用；
*   路由记录可以被用以子路由器：如果父路由记录匹配，则嵌套记录只会被用来测试。当设计一个组内的路由记录共享相同的匹配条件时，如主机名、路劲前缀或者其他重复的属性，子路由的方式很有帮助
*   mux.Router实现了http.Handler接口，故和标准的http.ServeMux兼容。

### **5.1.2 添加路由记录**

Router路由实例r创建完毕，下一步工作是为Router实例r添加所需要的路由记录。路由记录存储着用来匹配请求的信息，包括对请求的匹配规则，以及匹配之后的Handler执行入口。 

回到[createRouter](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1094-L1171)实现代码中，首先判断Docker Daemon的启动过程中有没有开启DEBUG模式。通过docker可执行文件启动Docker Daemon，解析flag参数时，若flDebug的值为false，则说明不需要配置DEBUG环境；若flDebug的值为true，则说明需要为Docker Daemon添加DEBUG功能。具体的代码实现如下：

```go
if os.Getenv("DEBUG") != "" {
    AttachProfiler(r)
}
```


AttachProiler(r)的功能是为路由实例r添加与DEBUG相关的路由记录，具体实现位于[./docker/api/server/server.go#L1083](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1083-L1092)，如下：

```go
func AttachProfiler(router *mux.Router) {
    router.HandleFunc("/debug/vars", expvarHandler)
    router.HandleFunc("/debug/pprof/", pprof.Index)
    router.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    router.HandleFunc("/debug/pprof/profile", pprof.Profile)
    router.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    router.HandleFunc("/debug/pprof/heap", pprof.Handler("heap").ServeHTTP)
    router.HandleFunc("/debug/pprof/goroutine", pprof.Handler("goroutine").ServeHTTP)
    router.HandleFunc("/debug/pprof/threadcreate", pprof.Handler("threadcreate").ServeHTTP)
}
```

分析以上源码，可以发现Docker Server使用两个包来完成DEBUG相关的工作：expvar和pprof。包expvar为公有变量提供标准化的接口，使得这些公有变量可以通过HTTP的形式在”/debug/vars”这个URL下被访问，传输时格式为JSON。包pprof将Docker Server运行时的分析数据通过”/debug/pprof/”这个URL向外暴露。这些运行时信息包括以下内容：可得的信息列表、正在运行的命令行信息、CPU信息、程序函数引用信息、ServeHTTP这个函数三部分信息使用情况（堆使用、goroutine使用和thread使用）。 

回到[createRouter函数实现](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1094-L1171)中，完成DEBUG功能的所有工作之后，Docker Docker创建了一个map类型的对象m，用于初始化路由实例r的路由记录。简化的m对象，代码如下：

```go
m := map[string]map[string]HttpApiFunc{
    "GET": {
        ……
        "/images/{name:.*}/get":           getImagesGet,
        ……
    },
    "POST": {
        ……
        "/containers/{name:.*}/copy":    postContainersCopy,
    },
    "DELETE": {
        "/containers/{name:.*}": deleteContainers,
        "/images/{name:.*}":     deleteImages,
    },
    "OPTIONS": {
        "": optionsHandler,
    },
}
```


对象m的类型为map，其中key为string类型，代表HTTP的请求类型，如”GET”，”POST”，”DELETE”等，value为另一个map类型，该map代表的是URL与执行Handler的映射。在第二个map类型中，key为string类型，代表是的请求URL，value为HttpApiFunc类型，代表具体的执行Handler。其中HttpApiFunc类型的定义如下：

```go
type HttpApiFunc func(eng *engine.Engine, version version.Version, w http.ResponseWriter, r *http.Request, vars map[string]string) error
```


完成对象m的定义，随后Docker Server通过该对象m来添加路由实例r的路由记录。对象m的请求方法，请求URL和请求处理Handler这三样内容可以为对象r构建一条路由记录。实现代码。如下：

```go
for method, routes := range m {
    for route, fct := range routes {
        log.Debugf("Registering %s, %s", method, route)
        localRoute := route
        localFct := fct
        localMethod := method

        f := makeHttpHandler(eng, logging, localMethod, localRoute, localFct, enableCors, version.Version(dockerVersion))

        if localRoute == "" {
            r.Methods(localMethod).HandlerFunc(f)
        } else {
            r.Path("/v{version:[0-9.]+}" + localRoute).Methods(localMethod).HandlerFunc(f)
         r.Path(localRoute).Methods(localMethod).HandlerFunc(f)
        }
    }
}
```


以上代码，在第一层循环中，按HTTP请求方法划分，获得请求方法各自的路由记录，第二层循环，按匹配请求的URL进行划分，获得与URL相对应的执行Handler。在嵌套循环中，通过makeHttpHandler返回一个执行的函数f。在返回的这个函数中，涉及了logging信息，CORS信息（跨域资源共享协议），以及版本信息。以下举例说明makeHttpHandler的实现，从对象m可以看到，对于”GET”请求，请求URL为”/info”，则请求Handler为”getInfo”。执行makeHttpHandler的具体代码实现如下：

```go
func makeHttpHandler(eng *engine.Engine, logging bool, localMethod string, localRoute string, handlerFunc HttpApiFunc, enableCors bool, dockerVersion version.Version) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // log the request
        log.Debugf("Calling %s %s", localMethod, localRoute)

        if logging {
            log.Infof("%s %s", r.Method, r.RequestURI)
        }

        if strings.Contains(r.Header.Get("User-Agent"), "Docker-Client/") {
            userAgent := strings.Split(r.Header.Get("User-Agent"), "/")
            if len(userAgent) == 2 && !dockerVersion.Equal(version.Version(userAgent[1])) {
                log.Debugf("Warning: client and server don't have the same version (client: %s, server: %s)", userAgent[1], dockerVersion)
            }
        }
        version := version.Version(mux.Vars(r)["version"])
        if version == "" {
            version = api.APIVERSION
        }
        if enableCors {
            writeCorsHeaders(w, r)
        }

        if version.GreaterThan(api.APIVERSION) {
            http.Error(w, fmt.Errorf("client and server don't have same version (client : %s, server: %s)", version, api.APIVERSION).Error(), http.StatusNotFound)
            return
        }

        if err := handlerFunc(eng, version, w, r, mux.Vars(r)); err != nil {
            log.Errorf("Handler for %s %s returned error: %s", localMethod, localRoute, err)
            httpError(w, err)
        }
    }
}
```


可见makeHttpHandler的执行直接返回一个函数func(w http.ResponseWriter, r \*http.Request) 。在这个func函数的实现中，判断makeHttpHandler传入的logging参数，若为true，则将该Handler的执行通过日志显示，另外通过makeHttpHandler传入的enableCors参数判断是否在HTTP请求的头文件中添加跨域资源共享信息，若为true，则通过writeCorsHeaders函数向response中添加有关CORS的HTTP Header，代码实现位于[./docker/api/server/server.go#L1022](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1022-L1026)，如下：

```go
func writeCorsHeaders(w http.ResponseWriter, r *http.Request) {
    w.Header().Add("Access-Control-Allow-Origin", "*")
    w.Header().Add("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept")
    w.Header().Add("Access-Control-Allow-Methods", "GET, POST, DELETE, PUT, OPTIONS")
}
```


最为重要的执行部分位于handlerFunc(eng, version, w, r, mux.Vars(r))，如以下代码：

```go
if err := handlerFunc(eng, version, w, r, mux.Vars(r)); err != nil {
    log.Errorf("Handler for %s %s returned error: %s", localMethod, localRoute, err)
    httpError(w, err)
}
```


对于”GET”请求类型，”/info”请求URL的请求，由于Handler名为getInfo，也就是说handlerFunc这个形参的值为getInfo，故执行部分直接运行getInfo(eng, version, w, r, mux.Vars(r))，而getInfo的具体实现位于[./docker/api/server/serve.go#L269](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L269-L273)，如下：

```go
func getInfo(eng *engine.Engine, version version.Version, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    w.Header().Set("Content-Type", "application/json")
    eng.ServeHTTP(w, r)
    return nil
}
```

以上makeHttpHandler的执行已经完毕，返回func函数，作为指定URL对应的执行Handler。 

创建完处理函数Handler，需要向路由实例中添加新的路由记录。如果URL信息为空，则直接为该HTTP请求方法类型添加路由记录；若URL不为空，则为请求URL路径添加新的路由记录。需要额外注意的是，在URL不为空，为路由实例r添加路由记录时，考虑了API版本的问题，通过r.Path("/v{version:\[0-9.\]+}" + localRoute).Methods(localMethod).HandlerFunc(f)来实现。 

至此，mux.Router实例r的两部分工作工作已经全部完成：创建空的路由实例r，为r添加相应的路由记录，最后返回路由实例r。

**5.2 创建listener监听实例**
----------------------

路由模块，完成了请求的路由与分发这一重要部分，属于[ListenAndServe实现](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1247-L1335)中的第一个重要工作。对于请求的监听功能，同样需要模块来完成。而在[ListenAndServe实现](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1247-L1335)中，第二个重要的工作就是创建Listener。Listener是一种面向流协议的通用网络监听模块。 

在创建Listener之前，先判断Docker Server允许的协议，若协议为fd形式，则直接通过ServeFd来服务请求；若协议不为fd形式，则继续往下执行。 

在程序执行过程中，需要判断”serveapi”这个job的环境中”BufferRequests”的值，是否为真，若为真，则通过包listenbuffer创建一个Listener的实例l，否则的话直接通过包net创建Listener实例l。具体的代码位于[./docker/api/server/server.go#L1269](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1269-L1273)，如下：

```go
if job.GetenvBool("BufferRequests") {
    l, err = listenbuffer.NewListenBuffer(proto, addr, activationLock)
} else {
    l, err = net.Listen(proto, addr)
}
```

由于在mainDaemon()中创建”serveapi”这个job之后，给job添加环境变量时，已经给”BufferRequets”赋值为true，故使用包listenbuffer创建listener实例。 

Listenbuffer的作用是：让Docker Server可以立即监听指定协议地址上的请求，但是将这些请求暂时先缓存下来，等Docker Daemon全部启动完毕之后，才让Docker Server开始接受这些请求。这样设计有一个很大的好处，那就是可以保证在Docker Daemon还没有完全启动完毕之前，接收并缓存尽可能多的用户请求。 

若协议的类型为TCP，另外job中环境变量Tls或者TlsVerify有一个为真，则说明Docker Server需要支持HTTPS服务，需要为Docker Server配置安全传输层协议（TLS）的支持。为实现TLS协议，首先需要建立一个tls.Config类型实例tlsConfig，然后在tlsConfig中加载证书，认证信息等，最终通过包tls中的NewListener函数，创建出适应于接收HTTPS协议请求的Listener实例l，代码如下：

```go
l = tls.NewListener(l, tlsConfig)
```


至此，创建网络监听的Listener部分已经全部完成。

**5.3 创建http.Server**
---------------------

Docker Server同样需要创建一个Server对象来运行HTTP服务端。在[ListenAndServe实现](https://github.com/docker/docker/blob/v1.2.0/api/server/server.go#L1247-L1335)中第三个重要的工作就是创建http.Server：

```go
httpSrv := http.Server{Addr: addr, Handler: r}
```


其中addr为需要监听的地址，r为mux.Router路由实例。

**5.4 启动API服务**
---------------

创建http.Server实例之后，Docker Server立即启动API服务，使Docker Server开始在Listener监听实例l上接受请求，并对于每一个请求都生成一个新的goroutine来做专属服务。对于每一个请求，goroutine会读取请求，查询路由表中的路由记录项，找到匹配的路由记录，最终调用路由记录中的执行Handler，执行完毕后，goroutine对请求返回响应信息。代码如下：

```go
return httpSrv.Serve(l)
```


至此，ListenAndServer的所有流程已经分析完毕，Docker Server已经开始针对不同的协议，服务API请求。

**6\. 总结**
==========

Docker Server作为Docker Daemon架构中请求的入口，接管了所有Docker Daemon对外的通信。通信API的规范性，通信过程的安全性，服务请求的并发能力，往往都是Docker用户最为关心的内容。本文基于源码，分析了Docker Server大部分的细节实现。希望Docker用户可以初探Docker Server的设计理念，并且可以更好的利用Docker Server创造更大的价值。

**7\. 参考文献**
============

1.  [Golang channels tutorial](http://guzalexander.com/2013/12/06/golang-channels-tutorial.html)
2.  [Package mux](http://www.gorillatoolkit.org/pkg/mux)
3.  [Package expvar](http://docs.studygolang.com/pkg/expvar/)
4.  [Package pprof](http://docs.studygolang.com/pkg/net/http/pprof/)

