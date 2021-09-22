+++

id= "147"

title = "Docker源码分析（二）：Docker Client创建与命令执行"
description = "如今，Docker作为业界领先的轻量级虚拟化容器管理引擎，给全球开发者提供了一种新颖、便捷的软件集成测试与部署之道。在团队开发软件时，Docker可以提供可复用的运行环境、灵活的资源配置、便捷的集成测试方法以及一键式的部署方式。可以说，Docker的优势在简化持续集成、运维部署方面体现得淋漓尽致，它完全让开发者从前者中解放出来，把精力真正地倾注在开发上。"
tags= [ "Docker"]
date= "2014-12-02 13:01:39"
author = "孙宏亮"
banner= "img/blogs/147/docker-2-1.jpg"
categories = [ "Docker" ]

+++

**1\. 前言**
==========

如今，Docker作为业界领先的轻量级虚拟化容器管理引擎，给全球开发者提供了一种新颖、便捷的软件集成测试与部署之道。在团队开发软件时，Docker可以提供可复用的运行环境、灵活的资源配置、便捷的集成测试方法以及一键式的部署方式。可以说，Docker的优势在简化持续集成、运维部署方面体现得淋漓尽致，它完全让开发者从前者中解放出来，把精力真正地倾注在开发上。 

<!--more-->

然而，把Docker的功能发挥到极致，并非一件易事。在深刻理解Docker架构的情况下，熟练掌握Docker Client的使用也非常有必要。前者可以参阅[《Docker源码分析》系列之Docker架构篇](http://www.sel.zju.edu.cn/?p=112)，而本文主要针对后者，从源码的角度分析Docker Client，力求帮助开发者更深刻的理解Docker Client的具体实现，最终更好的掌握Docker Client的使用方法。即本文为《Docker源码分析》系列的第二篇——Docker Client篇。

**2\. Docker Client源码分析章节安排**
=============================

本文从源码的角度，主要分析Docker Client的两个方面：创建与命令执行。前四章安排如下： 第一章为前言，介绍Docker的作用以及研究Docker Client的必要性。 第二章介绍部分章节安排。 第三章从Docker Client的创建入手，进行源码分析，主要分为三小节。 在3.1节中，分析如何通过docker命令，解析出命令行flag参数，以及docker命令中的请求参数。 在3.2节中，分析如何处理具体的flag参数信息，并收集Docker Client所需的配置信息。 在3.3节中，分析如何创建一个Docker Client。 第四章在已有Docker Client的基础上，分析如何执行docker命令，分为两小节。 在4.1节中，分析如何解析docker命令中的请求参数，获取请求的类型。 在4.2节中，分析Docker Client如何将执行具体的请求命令，最终将请求发送至Docker Server。

**3\. Docker Client的创建**
========================

Docker Client的创建，实质上是Docker用户通过可执行文件docker，与Docker Server建立联系的客户端。以下分三个小节分别阐述Docker Client的创建流程。 以下为整个docker源代码运行的流程图： 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616369/sel/docker-2-1_pigueq.jpg" alt="" style="zoom:50%;" />
</center>



上图通过流程图的方式，使得读者更为清晰的了解Docker Client创建及执行请求的过程。其中涉及了诸多源代码中的特有名词，在下文中会一一解释与分析。

### **3.1. Docker命令的flag参数解析**

众所周知，在Docker的具体实现中，Docker Server与Docker Client均由可执行文件docker来完成创建并启动。那么，了解docker可执行文件通过何种方式区分两者，就显得尤为重要。 对于两者，首先举例说明其中的区别。Docker Server的启动，命令为docker -d或docker --daemon=true；而Docker Client的启动则体现为docker --daemon=false ps、docker pull NAME等。 可以把以上Docker请求中的参数分为两类：第一类为命令行参数，即docker程序运行时所需提供的参数，如: -D、--daemon=true、--daemon=false等；第二类为docker发送给Docker Server的实际请求参数，如：ps、pull NAME等。 

对于第一类，我们习惯将其称为flag参数，在go语言的标准库中，同时还提供了一个[flag包](https://github.com/docker/docker/blob/master/pkg/mflag/flag.go)，方便进行命令行参数的解析。 交待以上背景之后，随即进入实现Docker Client创建的源码，位于[./docker/docker/docker.go](https://github.com/docker/docker/blob/v1.2.0/docker/docker.go)，在该go文件中，包含了整个Docker的main函数，也就是整个Docker（不论Docker Daemon还是Docker Client）的运行入口。部分main函数代码如下：

```go
func main() {
        if reexec.Init() {
            return
        }
        flag.Parse()
        // FIXME: validate daemon flags here
        ……
}
```

在以上代码中，首先判断reexec.Init()方法的返回值，若为真，则直接退出运行，否则的话继续执行。查看位于./docker/reexec/reexec.go中[reexec.Init()](https://github.com/docker/docker/blob/v1.2.0/reexec/reexec.go#L23)的定义，可以发现由于在docker运行之前没有任何的Initializer注册，故该代码段执行的返回值为假。 紧接着，main函数通过调用flag.Parse()解析命令行中的flag参数。查看源码可以发现Docker在[./docker/docker/flag.go](https://github.com/docker/docker/blob/v1.2.0/docker/docker.go)中定义了多个flag参数，并通过init函数进行初始化。代码如下：

```go
var (
    flVersion     = flag.Bool([]string{"v", "-version"}, false, "Print version information and quit")
    flDaemon      = flag.Bool([]]string{"d", "-daemon"}, false, "Enable daemon mode")
    flDebug       = flag.Bool([]string{"D", "-debug"}, false, "Enable debug mode")
    flSocketGroup = flag.String([]string{"G", "-group"}, "docker", "Group to assign the unix socket specified by -H when running in daemon mode use '' (the empty string) to disable setting of a group")
    flEnableCors  = flag.Bool([]string{"#api-enable-cors", "-api-enable-cors"}, false, "Enable CORS headers in the remote API")
    flTls         = flag.Bool([]string{"-tls"}, false, "Use TLS; implied by tls-verify flags")
    flTlsVerify   = flag.Bool([]string{"-tlsverify"}, false, "Use TLS and verify the remote (daemon: verify client, client: verify daemon)")

// these are initialized in init() below since their default values depend on dockerCertPath which isn't fully initialized     until init() runs
    flCa    *string
    flCert  *string
    flKey   *string
    flHosts []string
    )

    func init() {
    flCa = flag.String([]string{"-tlscacert"}, filepath.Join(dockerCertPath, defaultCaFile), "Trust only remotes providing a certificate signed by the CA given here")
    flCert = flag.String([]string{"-tlscert"}, filepath.Join(dockerCertPath, defaultCertFile), "Path to TLS certificate file")
    flKey = flag.String([]string{"-tlskey"}, filepath.Join(dockerCertPath, defaultKeyFile), "Path to TLS key file")
    opts.HostListVar(&flHosts, []string{"H", "-host"}, "The socket(s) to bind to in daemon mode\\nspecified using one or more tcp://host:port, unix:///path/to/socket, fd://\* or fd://socketfd.")
    }
```


这里涉及到了Golang的一个特性，即init函数的执行。在Golang中init函数的特性如下：

*   init函数用于程序执行前包的初始化工作，比如初始化变量等；
*   每个包可以有多个init函数；
*   包的每一个源文件也可以有多个init函数；
*   同一个包内的init函数的执行顺序没有明确的定义；
*   不同包的init函数按照包导入的依赖关系决定初始化的顺序；
*   init函数不能被调用，而是在main函数调用前自动被调用。

因此，在main函数执行之前，Docker已经定义了诸多flag参数，并对很多flag参数进行初始化。定义的命令行flag参数有：flVersion、flDaemon、flDebug、flSocketGroup、flEnableCors、flTls、flTlsVerify、flCa、flCert、flKey、flHosts等。 以下具体分析flDaemon：

*   定义：flDaemon = flag.Bool(\[\]string{"d", "-daemon"}, false, "Enable daemon mode")
*   flDaemon的类型为Bool类型
*   flDaemon名称为”d”或者”-daemon”，该名称会出现在docker命令中
*   flDaemon的默认值为false
*   flDaemon的帮助信息为”Enable daemon mode”
*   访问flDaemon的值时，使用指针\* flDaemon解引用访问

在解析命令行flag参数时，以下的语言为合法的：

*   \-d, --daemon
*   \-d=true, --daemon=true
*   \-d=”true”, --daemon=”true”
*   \-d=’true’, --daemon=’true’

当解析到第一个非定义的flag参数时，命令行flag参数解析工作结束。举例说明，当执行docker命令docker --daemon=false --version=false ps时，flag参数解析主要完成两个工作：

*   完成命令行flag参数的解析，名为-daemon和-version的flag参数flDaemon和flVersion分别获得相应的值，均为false；
*   遇到第一个非flag参数的参数ps时，将ps及其之后所有的参数存入flag.Args()，以便之后执行Docker Client具体的请求时使用。

如需深入学习flag的解析，可以参见源码[命令行参数flag的解析](https://github.com/docker/docker/blob/master/pkg/mflag/flag.go)。

### **3.2. 处理flag信息并收集Docker Client的配置信息**

有了以上flag参数解析的相关知识，分析Docker的main函数就变得简单易懂很多。通过总结，首先列出源代码中处理的flag信息以及收集Docker Client的配置信息，然后再一一对此分析：

*   处理的flag参数有：flVersion、flDebug、flDaemon、flTlsVerify以及flTls；
*   为Docker Client收集的配置信息有：protoAddrParts(通过flHosts参数获得，作用为提供Docker Client与Server的通信协议以及通信地址)、tlsConfig(通过一系列flag参数获得，如_flTls、_flTlsVerify，作用为提供安全传输层协议的保障)。

随即分析处理这些flag参数信息，以及配置信息。 在flag.Parse()之后的代码如下：

```go
if *flVersion {
    showVersion()
    return
}
```


不难理解的是，当经过解析flag参数后，若flVersion参数为真时，调用showVersion()显示版本信息，并从main函数退出；否则的话，继续往下执行。

```go
if *flDebug {
    os.Setenv("DEBUG", "1")
}
```


若flDebug参数为真的话，通过os包的中Setenv函数创建一个名为DEBUG的系统环境变量，并将其值设为”1”。继续往下执行。

```go
if len(flHosts) == 0 {
    defaultHost := os.Getenv("DOCKER_HOST")
    if defaultHost == "" || *flDaemon {
        // If we do not have a host, default to unix socket
        defaultHost = fmt.Sprintf("unix://%s", api.DEFAULTUNIXSOCKET)
    }
    if _, err := api.ValidateHost(defaultHost); err != nil {
        log.Fatal(err)
    }
    flHosts = append(flHosts, defaultHost)
}
```


以上的源码主要分析内部变量flHosts。flHosts的作用是为Docker Client提供所要连接的host对象，也为Docker Server提供所要监听的对象。 分析过程中，首先判断flHosts变量是否长度为0，若是的话，通过os包获取名为DOCKER_HOST环境变量的值，将其赋值于defaultHost。若defaultHost为空或者flDaemon为真的话，说明目前还没有一个定义的host对象，则将其默认设置为unix socket，值为api.DEFAULTUNIXSOCKET，该常量位于[./docker/api/common.go](https://github.com/docker/docker/blob/v1.2.0/api/common.go#L17)，值为"/var/run/docker.sock"，故defaultHost为”unix:///var/run/docker.sock”。验证该defaultHost的合法性之后，将defaultHost的值追加至flHost的末尾。继续往下执行。

```go
if *flDaemon {
    mainDaemon()
    return
}
```


若flDaemon参数为真的话，则执行mainDaemon函数，实现Docker Daemon的启动，若mainDaemon函数执行完毕，则退出main函数，一般mainDaemon函数不会主动终结。由于本章节介绍Docker Client的启动，故假设flDaemon参数为假，不执行以上代码块。继续往下执行。

```go
if len(flHosts) > 1 {
    log.Fatal("Please specify only one -H")
protoAddrParts := strings.SplitN(flHosts[0], "://", 2)
```


以上，若flHosts的长度大于1的话，则抛出错误日志。接着将flHosts这个string数组中的第一个元素，进行分割，通过”://”来分割，分割出的两个部分放入变量protoAddrParts数组中。protoAddrParts的作用为解析出与Docker Server建立通信的协议与地址，为Docker Client创建过程中不可或缺的配置信息之一。

```go
var (
    cli       *client.DockerCli
    tlsConfig tls.Config
)
tlsConfig.InsecureSkipVerify = true
```


由于之前已经假设过flDaemon为假，则可以认定main函数的运行是为了Docker Client的创建与执行。在这里创建两个变量：一个为类型是client.DockerCli指针的对象cli，另一个为类型是tls.Config的对象tlsConfig。并将tlsConfig的InsecureSkipVerify属性设置为真。TlsConfig对象的创建是为了保障cli在传输数据的时候，遵循安全传输层协议(TLS)。安全传输层协议(TLS) 用于两个通信应用程序之间保密性与数据完整性，该协议有两层组成：TLS记录协议和TLS握手协议。tlsConfig是Docker Client创建过程中可选的配置信息。

```go
// If we should verify the server, we need to load a trusted ca
if *flTlsVerify {
    *flTls = true
    certPool := x509.NewCertPool()
    file, err := ioutil.ReadFile(*flCa)
    if err != nil {
        log.Fatalf("Couldn't read ca cert %s: %s", *flCa, err)
    }
    certPool.AppendCertsFromPEM(file)
    tlsConfig.RootCAs = certPool
    tlsConfig.InsecureSkipVerify = false
}
```


若flTlsVerify这个flag参数为真的话，则说明需要验证server端的安全性，tlsConfig对象需要加载一个受信的ca文件。该ca文件的路径为\*flCA参数的值，最终完成tlsConfig对象中RootCAs属性的赋值，并将InsecureSkipVerify属性置为假。

```go
// If tls is enabled, try to load and send client certificates
if *flTls || *flTlsVerify {
    _, errCert := os.Stat(*flCert)
    _, errKey := os.Stat(*flKey)
    if errCert == nil && errKey == nil {
        *flTls = true
        cert, err := tls.LoadX509KeyPair(*flCert, *flKey)
        if err != nil {
            log.Fatalf("Couldn't load X509 key pair: %s. Key encrypted?", err)
        }
        tlsConfig.Certificates = []tls.Certificate{cert}
    }
}
```


如果flTls和flTlsVerify两个flag参数中有一个为真，则说明需要加载以及发送client端的证书。最终将证书内容交给tlsConfig的Certificates属性。 至此，flag参数已经全部处理，并已经收集完毕Docker Client所需的配置信息。之后的内容为Docker Client如何实现创建并执行。

### **3.3. Docker Client的启动**

Docker Client的创建其实就是在已有配置参数信息的情况，通过Client包中的NewDockerCli方法创建一个实例cli，源码实现如下：

```go
if *flTls || *flTlsVerify {
    cli = client.NewDockerCli(os.Stdin, os.Stdout, os.Stderr, protoAddrParts[0], protoAddrParts[1], &tlsConfig)
} else {
    cli = client.NewDockerCli(os.Stdin, os.Stdout, os.Stderr, protoAddrParts[0], protoAddrParts[1], nil)
}
```


如果flag参数flTls为真或者flTlsVerify为真的话，则说明需要使用TLS协议来保障传输的安全性，故创建Docker Client的时候，将TlsConfig参数传入；否则的话，同样创建Docker Client，只不过TlsConfig为nil。 关于Client包中的NewDockerCli函数的实现，可以具体参见[./docker/api/client/cli.go](https://github.com/docker/docker/blob/v1.2.0/api/client/cli.go#L81)。

```go
func NewDockerCli(in io.ReadCloser, out, err io.Writer, proto, addr string, tlsConfig *tls.Config) *DockerCli {
    var (
        isTerminal = false
        terminalFd uintptr
        scheme     = "http"
    )

    if tlsConfig != nil {
        scheme = "https"
    }

    if in != nil {
        if file, ok := out.(*os.File); ok {
            terminalFd = file.Fd()
            isTerminal = term.IsTerminal(terminalFd)
        }
    }

    if err == nil {
        err = out
    }
    return &DockerCli{
        proto:      proto,
        addr:       addr,
        in:         in,
        out:        out,
        err:        err,
        isTerminal: isTerminal,
        terminalFd: terminalFd,
        tlsConfig:  tlsConfig,
        scheme:     scheme,
    }
}
```


总体而言，创建DockerCli对象较为简单，较为重要的DockerCli的属性有proto：传输协议；addr：host的目标地址，tlsConfig：安全传输层协议的配置。若tlsConfig为不为空，则说明需要使用安全传输层协议，DockerCli对象的scheme设置为“https”，另外还有关于输入，输出以及错误显示的配置，最终返回该对象。 通过调用NewDockerCli函数，程序最终完成了创建Docker Client，并返回main函数继续执行。

**4\. Docker命令执行**
------------------

main函数执行到目前为止，有以下内容需要为Docker命令的执行服务：创建完毕的Docker Client，docker命令中的请求参数（经flag解析后存放于flag.Arg()）。也就是说，需要使用Docker Client来分析docker 命令中的请求参数，并最终发送相应请求给Docker Server。

### **4.1. Docker Client解析请求命令**

Docker Client解析请求命令的工作，在Docker命令执行部分第一个完成，直接进入main函数之后的[源码部分](https://github.com/docker/docker/blob/v1.2.0/docker/docker.go#L102)：

```go
if err := cli.Cmd(flag.Args()...); err != nil {
    if sterr, ok := err.(*utils.StatusError); ok {
        if sterr.Status != "" {
            log.Println(sterr.Status)
        }
        os.Exit(sterr.StatusCode)
    }
    log.Fatal(err)
}
```


查阅以上源码，可以发现，正如之前所说，首先解析存放于flag.Args()中的具体请求参数，执行的函数为cli对象的Cmd函数。进入[./docker/api/client/cli.go的Cmd函数](https://github.com/docker/docker/blob/v1.2.0/api/client/cli.go#L51)：

```go
// Cmd executes the specified command
func (cli *DockerCli) Cmd(args ...string) error {
    if len(args) > 0 {
        method, exists := cli.getMethod(args[0])
        if !exists {
            fmt.Println("Error: Command not found:", args[0])
            return cli.CmdHelp(args[1:]...)
        }
        return method(args[1:]...)
    }
    return cli.CmdHelp(args...)
}
```


由代码注释可知，Cmd函数执行具体的指令。源码实现中，首先判断请求参数列表的长度是否大于0，若不是的话，说明没有请求信息，返回docker命令的Help信息；若长度大于1的话，说明有请求信息，则首先通过请求参数列表中的第一个元素args\[0\]来获取具体的method的方法。如果上述method方法不存在，则返回docker命令的Help信息，若存在的话，调用具体的method方法，参数为args[1](http://www.infoq.com/cn/articles/docker-command-line-quest)及其之后所有的请求参数。 

还是以一个具体的docker命令为例，docker –daemon=false –version=false pull Name。通过以上的分析，可以总结出以下操作流程： (1) 解析flag参数之后，将docker请求参数”pull”和“Name”存放于flag.Args(); (2) 创建好的Docker Client为cli，cli执行cli.Cmd(flag.Args()…); 在Cmd函数中，通过args\[0\]也就是”pull”,执行cli.getMethod(args\[0\])，获取method的名称； (3) 在getMothod方法中，通过处理最终返回method的值为”CmdPull”; (4) 最终执行method(args\[1:\]…)也就是CmdPull(args\[1:\]…)。

### **4.2. Docker Client执行请求命令**

上一节通过一系列的命令解析，最终找到了具体的命令的执行方法，本节内容主要介绍Docker Client如何通过该执行方法处理并发送请求。 由于不同的请求内容不同，执行流程大致相同，本节依旧以一个例子来阐述其中的流程，例子为：docker pull NAME。 Docker Client在执行以上请求命令的时候，会执行CmdPull函数，传入参数为args\[1:\]...。源码具体为[./docker/api/client/command.go中的CmdPull函数](https://github.com/docker/docker/blob/v1.2.0/api/client/commands.go#L1183)。 以下逐一分析CmdPull的源码实现。 

(1)

```go
cmd := cli.Subcmd("pull", "NAME[:TAG]", "Pull an image or a repository from the registry") 
```


通过cli包中的Subcmd方法定义一个类型为Flagset的对象cmd； 

(2)

```go
tag := cmd.String([]string{"#t", "#-tag"}, "", "Download tagged image in a repository") 
```


给cmd对象定义一个类型为String的flag，名为”#t”或”#-tag”，初始值为空。 

(3)

```go
if err := cmd.Parse(args); err != nil { 
    return nil
}
```


将args参数进行解析，解析过程中，先提取出是否有符合tag这个flag的参数，若有，将其给赋值给tag参数，其余的参数存入cmd.NArg();若无的话，所有的参数存入cmd.NArg()中。 

(4)

```go
if cmd.NArg() != 1 {
    cmd.Usage()
    return nil 
} 
```


判断经过flag解析后的参数列表，若参数列表中参数的个数不为1，则说明需要pull多个image，pull命令不支持，则调用错误处理方法cmd.Usage()，并返回nil。

(5)

```go
var (
    v = url.Values{} 
    remote = cmd.Arg(0) 
)
v.Set("fromImage", remote) 
if *tag == "" {
    v.Set("tag", *tag) 
} 
```


创建一个map类型的变量v，该变量用于存放pull镜像时所需的url参数；随后将参数列表的第一个值赋给remote变量，并将remote作为键为fromImage的值添加至v；最后若有tag信息的话，将tag信息作为键为”tag”的值添加至v。 

(6)

```go
remote, _ = parsers.ParseRepositoryTag(remote) 
// Resolve the Repository name from fqn to hostname + name
hostname, _, err := registry.ResolveRepositoryName(remote) 
if err != nil {
    return err
} 
```


通过remote变量解析出镜像所在的host地址，以及镜像的名称。 

(7)

```go
cli.LoadConfigFile()
// Resolve the Auth config relevant for this server 
authConfig := cli.configFile.ResolveAuthConfig(hostname)
```


通过cli对象获取与Docker Server通信所需要的认证配置信息。 

(8)

```go
pull := func(authConfig registry.AuthConfig) error {
    buf, err := json.Marshal(authConfig) 
    if err != nil { 
        return err 
    } 
    registryAuthHeader := []string{ 
        base64.URLEncoding.EncodeToString(buf),
    } 
    return cli.stream("POST", "/images/create?"+v.Encode(), nil, cli.out, map[string][]string{
        " X-Registry-Auth": registryAuthHeader, .
        })
    } 
```


定义一个名为pull的函数，传入的参数类型为registry.AuthConfig，返回类型为error。函数执行块中最主要的内容为：cli.stream(……)部分。该部分具体发起了一个给Docker Server的POST请求，请求的url为"/images/create?"+v.Encode()，请求的认证信息为：map\[string\]\[\]string{"X-Registry-Auth": registryAuthHeader,}。 

(9)

```go
if err := pull(authConfig); err != nil {
    if strings.Contains(err.Error(), "Status 401") {
        fmt.Fprintln(cli.out, "\\nPlease login prior to pull:")
        if err := cli.CmdLogin(hostname); err != nil {
            return err 
        }
        authConfig := cli.configFile.ResolveAuthConfig(hostname)
        return pull(authConfig) 
        }
        return err 
} 
```


由于上一个步骤只是定义pull函数，这一步骤具体调用执行pull函数，若成功则最终返回，若返回错误，则做相应的错误处理。若返回错误为401，则需要先登录，转至登录环节，完成之后，继续执行pull函数，若完成则最终返回。 以上便是pull请求的全部执行过程，其他请求的执行在流程上也是大同小异。总之，请求执行过程中，大多都是将命令行中关于请求的参数进行初步处理，并添加相应的辅助信息，最终通过指定的协议给Docker Server发送Docker Client和Docker Server约定好的API请求。

**5\. 总结**
----------

本文从源码的角度分析了从docker可执行文件开始，到创建Docker Client，最终发送给Docker Server请求的完整过程。 笔者认为，通过学习与理解Docker Client相关的源码实现，不仅可以让用户熟练掌握Docker命令的使用，还可以使得用户在特殊情况下有能力修改Docker Client的源码，使其满足自身系统的某些特殊需求，以达到定制Docker Client的目的，最大发挥Docker开放思想的价值。

**6\. 参考文献**
------------

1.  [深入浅出Docker（二）：Docker命令行探秘](http://www.infoq.com/cn/articles/docker-command-line-quest)
2.  [The Go Programming Language](http://docs.studygolang.com/pkg/)
3.  [GO标准库—命令行参数解析flag包](http://blog.studygolang.com/2013/02/%E6%A0%87%E5%87%86%E5%BA%93-%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%82%E6%95%B0%E8%A7%A3%E6%9E%90flag/)
4.  [Docker Reference-Command Line](https://docs.docker.com/reference/commandline/cli/)