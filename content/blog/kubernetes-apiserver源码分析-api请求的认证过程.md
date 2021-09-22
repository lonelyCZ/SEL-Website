+++
id = "609"

title = "kubernetes apiserver源码分析——api请求的认证过程"
description = "apiserver相当于是k8集群的一个入口，不论通过kubectl还是使用remote api 直接控制，都要经过apiserver。apiserver说白了就是一个server负责监听指定的端口，之后处理不同的请求。笔者之前希望全面分析一下k8apiserver的源码，后来发现这样并不十分有效，其一没有针对性，其二由于代码本身比较复杂，涉及到的功能较多，面面俱到也不太现实。本文通过分析apiserver的源码解析apiserver启动的时候，参数应该如何设置，相关的机制又是怎样？"
tags = ["Kubernetes","apiserver"]
date = "2015-08-09 11:25:44"
author = "王哲"
banner = "img/blogs/609/k8s_arch.png"
categories = ["Kubernetes"]

+++

**解决什么问题**
----------

笔者之前希望全面分析一下k8apiserver的源码，后来发现这样并不十分有效，其一没有针对性，其二由于代码本身比较复杂，涉及到的功能较多，面面俱到也不太现实。 

于是我们就回到最初的需求，到底需要解决什么问题，第一个问题就是，apiserver启动的时候，使用secure模式，参数应该如何设置，相关的机制又是怎样？ 

<!--more-->

这一部分的issue很多，如果不从源码来分析的话，就只能黑盒化的去尝试各种参数搭配，费时费力，也不确定是否正确，之前就是这样，这几个issue可以供参考： 

https://github.com/GoogleCloudPlatform/kubernetes/issues/10159#issuecomment-113955582 

https://github.com/GoogleCloudPlatform/kubernetes/issues/11000

**apiserver启动过程的代码概览**
----------------------

apiserver相当于是k8集群的一个入口，不论通过kubectl还是使用remote api 直接控制，都要经过apiserver。apiserver说白了就是一个server负责监听指定的端口，之后处理不同的请求，只不过加上的很多控制，k8s项目由那么多大牛构建，作为参考学习，看一下各个组件的源码，想必也是很有帮助的。 

这里分析的是k8s v1.0.0 版本的代码,commit id 为 cd821444dcf3。 

main函数的代码位于./kubernetes/cmd/kube-apiserver：

```go
func main() {
    runtime.GOMAXPROCS(runtime.NumCPU())
    rand.Seed(time.Now().UTC().UnixNano())

    s := app.NewAPIServer()
    s.AddFlags(pflag.CommandLine)

    util.InitFlags()
    util.InitLogs()
    defer util.FlushLogs()

    verflag.PrintAndExitIfRequested()

    if err := s.Run(pflag.CommandLine.Args()); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
```


这一部分主要是进行一些初始化的设置，启动一个apiserver实例，再将其run起来。初始化的各种细节参数暂时不做重点分析，主要关注一下，`app.NewAPIServer()`以及`s.AddFlags(pflag.CommandLine)` 两个函数，New一个apiserver的时候会放入许多默认的初始化参数：

```go
func NewAPIServer() *APIServer {
    s := APIServer{
        InsecurePort:           8080,
        InsecureBindAddress:    util.IP(net.ParseIP("127.0.0.1")),
        BindAddress:            util.IP(net.ParseIP("0.0.0.0")),
        SecurePort:             6443,
        APIRate:                10.0,
        APIBurst:               200,
        APIPrefix:              "/api",
        EventTTL:               1 * time.Hour,
        AuthorizationMode:      "AlwaysAllow",
        AdmissionControl:       "AlwaysAdmit",
        EtcdPathPrefix:         master.DefaultEtcdPathPrefix,
        EnableLogsSupport:      true,
        MasterServiceNamespace: api.NamespaceDefault,
        ClusterName:            "kubernetes",
        CertDirectory:          "/var/run/kubernetes",

        RuntimeConfig: make(util.ConfigurationMap),
        KubeletConfig: client.KubeletConfig{
            Port:        ports.KubeletPort,
            EnableHttps: true,
            HTTPTimeout: time.Duration(5) * time.Second,
        },
    }

    return &s
}
```

可以看到insecure的端口，以及一些默认的监听端口，还有默认的证书存放位置等等，都是一些比较重要的信息。 

启动时候的全部参数通过`s.AddFlags(pflag.CommandLine)`这个函数传入，里面包括了apiserver启动时候的全部参数，这个使用的是`"github.com/spf13/pflag"`这个库，可以具体查看每个相关参数的含义以及初始值。把这些参数的含义弄清，启动的时候把对应的合适的值填进去，作为apiserver的基本使用就基本没问题了。 

之后负责启动的操作都是在run函数中执行，前面初始化的具体细节暂不做分析，这里着重关注一下两部分，一个是master实例的生成：

```go
config := &master.Config{
    EtcdHelper:             helper,
    EventTTL:               s.EventTTL,
    KubeletClient:          kubeletClient,
    ServiceClusterIPRange:  &n,
    EnableCoreControllers:  true,
    EnableLogsSupport:      s.EnableLogsSupport,
    EnableUISupport:        true,
    EnableSwaggerSupport:   true,
    EnableProfiling:        s.EnableProfiling,
    EnableIndex:            true,
    APIPrefix:              s.APIPrefix,
    CorsAllowedOriginList:  s.CorsAllowedOriginList,
    ReadWritePort:          s.SecurePort,
    PublicAddress:          net.IP(s.AdvertiseAddress),
    Authenticator:          authenticator,
    SupportsBasicAuth:      len(s.BasicAuthFile) > 0,
    Authorizer:             authorizer,
    AdmissionControl:       admissionController,
    EnableV1Beta3:          enableV1beta3,
    DisableV1:              disableV1,
    MasterServiceNamespace: s.MasterServiceNamespace,
    ClusterName:            s.ClusterName,
    ExternalHost:           s.ExternalHost,
    MinRequestTimeout:      s.MinRequestTimeout,
    SSHUser:                s.SSHUser,
    SSHKeyfile:             s.SSHKeyfile,
    InstallSSHKey:          installSSH,
    ServiceNodePortRange:   s.ServiceNodePortRange,
}
m := master.New(config)
```

这个是主要是生成master实例对象，各种api请求最后都是通过master对象来处理的。 

还有一个是server启动的时候：

```go
if secureLocation != "" {
    secureServer := &http.Server{
        Addr:           secureLocation,
        Handler:        apiserver.MaxInFlightLimit(sem, longRunningRE, apiserver.RecoverPanics(m.Handler)),
        ReadTimeout:    ReadWriteTimeout,
        WriteTimeout:   ReadWriteTimeout,
        MaxHeaderBytes: 1 << 20,
        TLSConfig: &tls.Config{
            // Change default from SSLv3 to TLSv1.0 (because of POODLE vulnerability)
            MinVersion: tls.VersionTLS10,
        },
    }

    if len(s.ClientCAFile) > 0 {
        clientCAs, err := util.CertPoolFromFile(s.ClientCAFile)
        if err != nil {
            glog.Fatalf("unable to load client CA file: %v", err)
        }
        // Populate PeerCertificates in requests, but don't reject connections without certificates
        // This allows certificates to be validated by authenticators, while still allowing other auth types
        secureServer.TLSConfig.ClientAuth = tls.RequestClientCert
        // Specify allowed CAs for client certificates
        secureServer.TLSConfig.ClientCAs = clientCAs
    }

    glog.Infof("Serving securely on %s", secureLocation)
    go func() {
        defer util.HandleCrash()
        for {
            if s.TLSCertFile == "" && s.TLSPrivateKeyFile == "" {
                s.TLSCertFile = path.Join(s.CertDirectory, "apiserver.crt")
                s.TLSPrivateKeyFile = path.Join(s.CertDirectory, "apiserver.key")
                // TODO (cjcullen): Is PublicAddress the right address to sign a cert with?
                alternateIPs := []net.IP{config.ServiceReadWriteIP}
                alternateDNS := []string{"kubernetes.default.svc", "kubernetes.default", "kubernetes"}
                // It would be nice to set a fqdn subject alt name, but only the kubelets know, the apiserver is clueless
                // alternateDNS = append(alternateDNS, "kubernetes.default.svc.CLUSTER.DNS.NAME")
                if err := util.GenerateSelfSignedCert(config.PublicAddress.String(), s.TLSCertFile, s.TLSPrivateKeyFile, alternateIPs, alternateDNS); err != nil {
                    glog.Errorf("Unable to generate self signed cert: %v", err)
                } else {
                    glog.Infof("Using self-signed cert (%s, %s)", s.TLSCertFile, s.TLSPrivateKeyFile)
                }
            }
            // err == systemd.SdNotifyNoSocket when not running on a systemd system
            if err := systemd.SdNotify("READY=1\n"); err != nil && err != systemd.SdNotifyNoSocket {
                glog.Errorf("Unable to send systemd daemon sucessful start message: %v\n", err)
            }
            if err := secureServer.ListenAndServeTLS(s.TLSCertFile, s.TLSPrivateKeyFile); err != nil {
                glog.Errorf("Unable to listen for secure (%v); will try again.", err)
            }
            time.Sleep(15 * time.Second)
        }
    }()
}
http := &http.Server{
    Addr:           insecureLocation,
    Handler:        apiserver.RecoverPanics(m.InsecureHandler),
    ReadTimeout:    ReadWriteTimeout,
    WriteTimeout:   ReadWriteTimeout,
    MaxHeaderBytes: 1 << 20,
}
if secureLocation == "" {
    // err == systemd.SdNotifyNoSocket when not running on a systemd system
    if err := systemd.SdNotify("READY=1\n"); err != nil && err != systemd.SdNotifyNoSocket {
     glog.Errorf("Unable to send systemd daemon sucessful start message: %v\n", err)
    }
}
glog.Infof("Serving insecurely on %s", insecureLocation)
glog.Fatal(http.ListenAndServe())
```

大致看一下这部分代码，首先是生成一个http.Server对象`secureServer`，设置好相关的启动参数，之后会新启一个goroutine，如果有ca文件，说明要使用https的方式，就把ca文件也一并加载进来，之后在新的goroutine中通过`secureServer.ListenAndServeTLS`启动secureserver，使用https的方式来监听指定的secure端口。 

之后还会生成一个`http.Server`实例 http，这个就是采用insecure的方式，最后通过`http.ListenAndServe()`来启动。 

对比两种启动方式，可以看到，它们加载的handler都来自与之前生成的master实例m，一个是`m.Handler`，另一个是`m.InsecureHandler`。采用`m.Handler`的时候会多一些额外的处理，这个暂不分析，总是这个Handler中存放的就是这个server去进行处理的各种路由和对应的实现方式。

**api认证部分的实现**
--------------

在上面提到的run函数中，可以找到认证组件的实现：

```go
authenticator, err := apiserver.NewAuthenticator(s.BasicAuthFile, s.ClientCAFile, s.TokenAuthFile, s.ServiceAccountKeyFile, s.ServiceAccountLookup, helper)
```


之后在生成master实例的时候，这个认证器`authenticator`会作为Master实例的初始参数传入：

```go
config := &master.Config{
......
    Authenticator:          authenticator,
    ......
}
m := master.New(config)
```


这几个参数的使用也是之前issue里提到的问题比较多的地方，下面大致了解一下生成认证器的这几个参数，具体的使用在后面再进行具体的说明。

*   s.BasicAuthFile:指定basicauthfile文件所在的位置，当这个参数不为空的时候，会开启basicauth的认证方式，这是一个.csv文件，三列分别是password,username,useruid。
    
*   s.ClientCAFile：用于给客户端签名的根证书，当这个参数不为空的时候，会启动https的认证方式，会通过这个根证书对客户端的证书进行身份认证。
    
*   s.TokenAuthFile：用于指定token文件所在的位置，当这个参数不为空的时候，会采用token的认证方式，token文件也是csv的格式，三列分别是"token,username,useruid"。
    
*   s.ServiceAccountKeyFile：当不为空的时候，采用ServiceAccount的认证方式，这个其实是一个公钥密钥。注释里说要包含：PEM-encoded x509 RSA private or public key，发送过来的信息是在客户端使用对应的私钥加密过的，服务端使用指定的公钥来解密信息。
    
*   s.ServiceAccountLookup：这个参数值一个bool值，默认为false，如果为true的话，就会从etcd中取出对应的ServiceAccount与传过来的信息进行对比验证，反之则不会。
    
*   helper：这是一个用于与etcd交互的客户端实例，具体生成过程这里不进行具体分析。
    

下面结合认证器的具体生成过程对这些参数的使用进行具体分析，先总体看一下认证器部分的代码结构：

```go
func NewAuthenticator(basicAuthFile, clientCAFile, tokenFile, serviceAccountKeyFile string, serviceAccountLookup bool, helper tools.EtcdHelper) (authenticator.Request, error) {
    var authenticators []authenticator.Request

    if len(basicAuthFile) > 0 {
        basicAuth, err := newAuthenticatorFromBasicAuthFile(basicAuthFile)
        if err != nil {
            return nil, err
        }
        authenticators = append(authenticators, basicAuth)
    }

    if len(clientCAFile) > 0 {
        certAuth, err := newAuthenticatorFromClientCAFile(clientCAFile)
        if err != nil {
            return nil, err
        }
        authenticators = append(authenticators, certAuth)
    }

    if len(tokenFile) > 0 {
        tokenAuth, err := newAuthenticatorFromTokenFile(tokenFile)
        if err != nil {
            return nil, err
        }
        authenticators = append(authenticators, tokenAuth)
    }

    if len(serviceAccountKeyFile) > 0 {
        serviceAccountAuth, err := newServiceAccountAuthenticator(serviceAccountKeyFile, serviceAccountLookup, helper)
        if err != nil {
            return nil, err
        }
        authenticators = append(authenticators, serviceAccountAuth)
    }
    fmt.Println("the length of authticator:", len(authenticators))
    switch len(authenticators) {
    case 0:
        return nil, nil
    case 1:
        return authenticators[0], nil
    default:
        return union.New(authenticators...), nil
    }
}
```


结合上面的分析，这部分的代码结构就比较清楚了，返回的结果是一个`authenticator.Request`对象数组，每一个元素都是一个认证器，根据传入的参数是否为空来判断最后要生成多少个认证器，最后的union.New函数实际上返回的就是一个authenticator.Request数组：

```go
// unionAuthRequestHandler authenticates requests using a chain of authenticator.Requests
type unionAuthRequestHandler []authenticator.Request

// New returns a request authenticator that validates credentials using a chain of authenticator.Request objects
func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
    return unionAuthRequestHandler(authRequestHandlers)
}
```


我们可以看一下authenticator.Request接口的实现：

```go
// Request attempts to extract authentication information from a request and returns
// information about the current user and true if successful, false if not successful,
// or an error if the request could not be checked.
type Request interface {
    AuthenticateRequest(req *http.Request) (user.Info, bool, error)
}
```


其中的方法 `AuthenticateRequest`的主要功能就是把userinfo从request中提取出来，并返回是否认证成功，以及对应的错误信息。

**生成带有认证器的handler**
-------------------

下面我们直接跳到对于api请求的认证部分，看一下当某个请求过来的时候，apiserver是如何对其进行认证的，具体代码在/pkg/master/master.go的`func (m *Master) init(c *Config)`函数中：

```go
// Install Authenticator
    if c.Authenticator != nil {
        authenticatedHandler, err := handlers.NewRequestAuthenticator(m.requestContextMapper, c.Authenticator, handlers.Unauthorized(c.SupportsBasicAuth), handler)
        if err != nil {
            glog.Fatalf("Could not initialize authenticator: %v", err)
        }
        handler = authenticatedHandler
    }   }
        handler = authenticatedHandler
    }
```

实现细节暂不讨论，从功能上讲，这一段就是对handler进行一层包装，生成一个带有认证器的handler。 其中`handlers.Unauthorized(c.SupportsBasicAuth)`函数是一个返回Unauthorized信息的函数，如果认证失败，这个函数就会被调用。 

我们大致看一下NewRequestAuthenticator函数：

```go
func NewRequestAuthenticator(mapper api.RequestContextMapper, auth authenticator.Request, failed http.Handler, handler http.Handler) (http.Handler, error) {
    return api.NewRequestContextFilter(
        mapper,
        http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
            user, ok, err := auth.AuthenticateRequest(req)
            if err != nil || !ok {
                if err != nil {
                    glog.Errorf("Unable to authenticate the request due to an error: %v", err)
                }
                failed.ServeHTTP(w, req)
                return
            }

            if ctx, ok := mapper.Get(req); ok {
                mapper.Update(req, api.WithUser(ctx, user))
            }

            handler.ServeHTTP(w, req)
        }),
    )
}
```


可以看到HandleFunc中调用的函数，就是要调用我们之前提到的AuthenticateRequest函数，使用其提取用户信息，判断验证是否成功，如果有错误或者认证失败，返回Unauthorized新的的函数就会被调用。结合之前的分析，我们只要把每种认证器的AuthenticateRequest函数分析一下，就可以了解认证操作的具体实现过程了。

**每种认证操作的具体实现过程**
-----------------

结合上面的`NewAuthenticator`源码可以知道，最多一共有五种authenticators:即`basicAuth`、`certAuth`、`tokenAuth`、`serviceAccountAuth`，还有通过Union.New生成的`unionAuthRequestHandler`，下面我们结合每个认证器的生成过程具体看一下每个`authenticators`的`AuthenticateRequest`函数：

*   **unionAuthRequestHandler实例**

~~~go
func (authHandler unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (user.Info, bool, error) { 
  var errlist []error for _, 
  currAuthRequestHandler := range authHandler { 
    info, ok, err := currAuthRequestHandler.AuthenticateRequest(req) 
    if err != nil { 
            errlist = append(errlist, err) 
            continue
     }
     if ok {
         return info, true, nil
     }
  }     
  return nil, false, errors.NewAggregate(errlist) 
}
~~~

结合之前的NewAuthenticator可以看到，当authenticators数目大于1的时候，会生成`unionAuthRequestHandler`实例，之后会遍历其中的元素，调用每一个元素的`AuthenticateReques`方法，**只要其中有一种认证方式成功，最后认证就会返回true**。

*   **basicAuth：**

bacisAuth的认证比较直接，就是把信息从.csv文件中读取出来，返回一个PasswordAuthenticator结构，其中包含一个map:`users map[string]*userPasswordInfo`,具体验证的时候，就从map中读取已有信息，比较用户名和密码，可以看出这种认证方式确实比较基础，仅仅做了基本的认证。

```go
func newAuthenticatorFromBasicAuthFile(basicAuthFile string) (authenticator.Request, error) {
  basicAuthenticator, err := passwordfile.NewCSV(basicAuthFile) 
  if err != nil { 
    return nil, err 
  }  
	return basicauth.New(basicAuthenticator), nil
}
```


 首先会生成一个basicAuthenticator实例（PasswordAuthenticator对象），之后会将这个对象转化为Authenticator实例，里面包含authenticator.Password接口，PasswordAuthenticator实现了这个接口。

```go
type Authenticator struct { auth authenticator.Password }
```


具体的AuthenticateRequest的调用代码比较简单：

```go
func (a *Authenticator) AuthenticateRequest(req *http.Request) (user.Info, bool, error) { 
  auth := strings.TrimSpace(req.Header.Get("Authorization")) 
  if auth == "" { 
    return nil, false, nil 
  } 
  parts := strings.Split(auth, " ") 
  if len(parts) < 2 || strings.ToLower(parts\[0\]) != "basic" { 
    return nil, false, nil 
  }
  payload, err := base64.StdEncoding.DecodeString(parts[1])
  if err != nil {
      return nil, false, err
  }

  pair := strings.SplitN(string(payload), ":", 2)
  if len(pair) != 2 {
      return nil, false, errors.New("malformed basic auth header")
  }
  username := pair[0]
  password := pair[1]
  return a.auth.AuthenticatePassword(username, password)
}
```


主要就是从request的Header中提取出Authorization字段的信息，用basic作为分隔，之后根据`:`作为分隔，提取出用户名和密码，调用AuthenticatePassword进行检验。这里的a.auth是之前传过来的PasswordAuthenticator对象，可以看下具体的这对象的AuthenticatePassword的实现：

```go
func (a *PasswordAuthenticator) AuthenticatePassword(username, password string) (user.Info, bool, error){ 
	user, ok := a.users[username] 
	if !ok { 
		return nil, false, nil 
	} 
	if user.password != password { 
		return nil, false, nil 
	} 
	return user.info, true, nil 
}
```


就是根据username把info从map中提取出来进行检验，比较简单。

*   **certAuth：**

这里首先要声明一点， **https仅仅是认证方式的一种，secureport可以是https的也可以不是https的，不要把这两个弄混。**关于golang中https的使用的基本内容以及相关证书的生成可以参考[之前这个文章](http://wangzhezhe.github.io/blog/2015/08/05/httpsandgolang/)。 

ca文件指定了之后，说明要使用https的方式，这里是cafile是给客户端证书签名的根证书，用于https握手的时候对客户端进行身份认证。正常情况下还要指定服务端的.key和.crt文件，这里默认的就是使用双向认证的方式，在服务端启动的时候，要把对应的证书也加进去，分别用到的是`tls-cert-file`以及`tls-private-key-file`这个两个参数，如果这两个参数没指定的话，证书就会使用自签名的方式被自动生成，放在`CertDirectory: "/var/run/kubernetes"`目录下。 

这里具体验证的操作使用的是包含x509验证对象的AuthenticateRequest函数（./plugin/pkg/auth/authenticator/request/x509/x509.go），遵循的也是通常的https认证原理，具体细节不在此讨论。 

使用ca认证的时候，只要是ca签名过的证书都可以通过验证，这个时候ca的安全性就比较重要了，在某些使用场景中，证书应该如何分发问题可能是需要考虑的。

*   tokenAuth：

是用token的方式，具体代码的结构与basic auth file的方式比较类似，代码不再赘述，主要功能是先从指定的.csv文件中把信息加载进来，存在服务端TokenAuthenticator实例的一个tokens的map中`tokens map[string]*user.DefaultInfo`，之后用户信息发送过来，会从Authorization中提取出携带token值，只不过这里标记token的关键字使用的是"bearer"，把token值提取出来之后，进行对比，看是否ok。

*   serviceAccountAuth：

saAuth实际上是token auth的变形，这里用到的是jwt(json web token)来进行具体的操作，具体的功能可以参考这个[文章](http://www.sel.zju.edu.cn/?p=588)，本质上来说，saAuth也是一个token认证，只不过这个token是把一些信息加密（签名）之后生成的。大致介绍一下jwt，具体格式可以参考[这个文章](http://haomou.net/2014/08/13/2014_web_token/)这里从实现的角度进行一些分析:

```go
func newServiceAccountAuthenticator(keyfile string, lookup bool, helper tools.EtcdHelper) (authenticator.Request, error) { publicKey, err := serviceaccount.ReadPublicKey(keyfile) if err != nil { return nil, err }

var serviceAccountGetter serviceaccount.ServiceAccountTokenGetter
if lookup {
    // If we need to look up service accounts and tokens,
    // go directly to etcd to avoid recursive auth insanity
    serviceAccountGetter = serviceaccount.NewGetterFromEtcdHelper(helper)
}

tokenAuthenticator := serviceaccount.JWTTokenAuthenticator([]*rsa.PublicKey{publicKey}, lookup, serviceAccountGetter)
return bearertoken.New(tokenAuthenticator), nil 
}
```

可以看到，首先将publickey提取出来，之后如果lookup参数为true，会根据etcdhelper生成一个serviceAccountGetter,否则使用默认的serviceAccountGetter,用来从etcd中取具体的sa和secret，secret可以理解为某些敏感信息：

```go
type ServiceAccountTokenGetter interface { GetServiceAccount(namespace, name string) (*api.ServiceAccount, error) GetSecret(namespace, name string) (*api.Secret, error) }
```


我们先看最后一部分`bearertoken.New(tokenAuthenticator)`,返回的Authenticator结构的AuthenticateRequest方法就和tokenauth中的一样，从Authorization字段中提取出bearer token，之后使用接口中的方法`a.auth.AuthenticateToken(token)`进行验证，这里实际执行AuthenticateToken方法的是tokenAuthenticator对象（jwtTokenAuthenticator实例），`func (j *jwtTokenAuthenticator) AuthenticateToken(token string)`函数代码较长，就不在赘述，其主要的功能是使用之前提取出来的公钥密钥对信息进行解密，得到`parsedToken`:

```go
type Token struct { 
Raw string // The raw token. Populated when you Parse a token 
Method    SigningMethod // The signing method used or to be used 
Header     map[string]interface{} // The first segment of the token       
Claims       map[string]interface{} // The second segment of the token     
Signature string // The third segment of the token. Populated when you Parse a token 
Valid bool // Is the token valid? Populated when you Parse/Verify a token }
```

之后提取出其中的Claims信息并进行检验，看是否符合要求，如果look up字段为true的话，就会根据标记在Claims中的namespace ，secretName ，serviceAccountName , 利用之前生成的ServiceAccountTokenGetter从etcd中取出已设置好的serviceaccount以及secret来进行身份验证，验证通过之后会返回user信息。 

ServiceAccount的相关部分代码还在不断完善中，这里只分析了大概逻辑，相比其他方式serviceaccount还是挺好使用的，相当于是token的升级版，由于信息加密的原因，比仅仅使用token安全了不少，下面是参考k8相关代码生成的使用serviceacount方式发送api的方式，实际使用中，后面的发送api的部分直接使用源码中自己的kubectl，设置好对应的BearerToken字段即可：

```go
package main


import ( //"crypto/rsa" "crypto/tls" "fmt" "github.com/GoogleCloudPlatform/kubernetes/pkg/serviceaccount" "github.com/dgrijalva/jwt-go" "io/ioutil" "net/http" )

const ( test1 = "test1" test2 = "test2" )

func main() {

// Create the token
token := jwt.New(jwt.SigningMethodRS256)
// Set some claims
token.Claims[test1] = "zjusel"
token.Claims[test2] = "zjusel"

// Sign and get the complete encoded token as a string

//客户端用私钥进行加密 服务端用公钥进行解密
privateKey, err := serviceaccount.ReadPrivateKey("sa.key")

fmt.Println(privateKey)
if err != nil {
    fmt.Println(err.Error())
    return
}

tokenString, err := token.SignedString(privateKey)
if err != nil {
    fmt.Println(err.Error())
}
fmt.Println("token string: ", tokenString)

tr := &http.Transport{
    TLSClientConfig: &tls.Config{
        InsecureSkipVerify: true},
    DisableCompression: true,
}

client := &http.Client{Transport: tr}

url := "https://10.10.105.34:8081/api/v1/nodes"

reqest, err := http.NewRequest("GET", url, nil)
reqest.Header.Set("Content-Type", "application/x-www-form-urlencoded")
reqest.Header.Set("Authorization", "bearer "+tokenString)
resp, err := client.Do(reqest)

if err != nil {
    fmt.Println(err.Error())
    return
}

body, _ := ioutil.ReadAll(resp.Body)
fmt.Println(string(body))

}
```



**总结**
------

通过上面的分析，相信对kube-apiserver启动时候的身份验证部分的参数已经可以做到“心中有数”，即使有不清楚的地方，至少也可以做到按图索骥，从源码的角度分析参数应该如何设置，在实际使用中可以根据不同的场景使用合适的方式进行认证。