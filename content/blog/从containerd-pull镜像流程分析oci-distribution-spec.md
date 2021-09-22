+++
id= "921"

title = "从containerd pull镜像流程分析oci distribution spec"
description = "oci组织成立以来已经形成了关于image和runtime的两个spec。2018年4月，作为与registry交互的镜像分发协议也进入了oci标准化的工作范围。oci以当前被广泛采用的Docker Registry HTTP API V2为基础，构建了oci distribution spec。containerd当前同时支持docker版的和oci版的registry api。为了了解oci所定义的镜像分发协议，本文分析v1.1.0的containerd代码pull流程，tag创建时间为2018年4月23日。"
tags= [ "Docker" , "containerd" ]
date= "2018-08-05T19:03:13"
author = "丁轶群"
banner= "img/blogs/921/dockerarchitecture.png"
categories = [ "containerd" ]

+++

oci组织成立以来已经形成了关于image和runtime的两个spec。[2018年4月](https://www.linuxfoundation.org/press-release/2018/04/open-container-initiative-announces-distribution-specification-project/)，作为与registry交互的镜像分发协议也进入了oci标准化的工作范围。oci以当前被广泛采用的[Docker Registry HTTP API V2](https://github.com/docker/distribution/blob/5cb406d511b7b9163bff9b6439072e4892e5ae3b/docs/spec/api.md)为基础，构建了oci distribution spec。containerd当前同时支持docker版的和oci版的registry api。

<!--more-->

为了了解oci所定义的镜像分发协议，本文分析v1.1.0的containerd代码pull流程，tag创建时间为2018年4月23日。



pull操作的cli命令解释
--------------

关于containerd cli（即ctr）的pull操作指令，由于containerd代码中语焉不详，我们[参照docker官方文档](https://docs.docker.com/engine/reference/commandline/pull/)，pull镜像的指令的操作格式如下：

```shell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

也就是说，我们pull镜像时，可以用 :tag也可以用@digest。  

其中平时不常用的digest形式用来pull从某个layer开始的包含其所有直接间接父layer的镜像，例子如下：

```shell
$ docker pull ubuntu@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
```


其中[digest一般形式如下](https://docs.docker.com/registry/spec/api/)：

```shell
digest      := algorithm ":" hex
algorithm   := /[A-Fa-f0-9_+.-]+/
hex         := /[A-Fa-f0-9]+/
```


另外，根据`containerd/ctr/commands/images/pull.go`，ctr image pull命令还支持几个flag：

1.  registry相关flag：
    2.  skip-verify: 跳过SSL证书验证过程
    3.  plain-http: 是否允许只用http协议（而非https）连接registry
    4.  user：user\[:password\]形式的registry用户名密码，如果flag中没有包含password，会在后续执行中要求用户在console中输入
    5.  refresh：refresh token for authorization server
2.  snapshotter：todo 虽然可以提供自定义值，但目前一律使用default snapshotter
3.  label： 对于像manifest list那些包含子资源的资源，containerd默认会给它们，表达父子资源之间的关系，详见后面对manifest list的处理流程。通过label flag，可以给下载之后给镜像打额外自定义label。
4.  platform： 只下载适应某平台的镜像，比如linux/amd64，linux/arm等
5.  all-platform：下载适用所有平台的镜像

ctr image pull执行流程
------------------

整体流程可以参考[官方流程图](https://github.com/containerd/containerd/blob/master/design/architecture.md) 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605878863/sel/data-flow_ib2p8w.png"  style="zoom:100%;" />
</center>
1. Instruct the Distribution layer to pull a particular image. The distribution layer places the image content into the content store. The image name and root manifest pointers are registered with the metadata store. 
2. Once the image is pulled, the user can instruct the bundle controller to unpack the image into a bundle. Consuming from the content store, layers from the image are unpacked into the snapshot component. 
3. When the snapshot for the rootfs of a container is ready, the bundle controller can use the image manifest and config to prepare the execution configuration. Part of this is entering mounts into the execution config from the snapshot module. 
4. The prepared bundle is then passed off to the runtime subsystem for execution. It reads the bundle configuration to create a running container. 

跟ctr image pull操作执行逻辑定义在`containerd/cmd/ctr/commands/images/pull.go`定义的匿名函数中，步骤如下：

1.  resolve用户需要下载的镜像
2.  从registry pull镜像，把镜像层内容和config保存进content服务，把镜像相关的元数据保存进images元数据服务
3.  unpack进snapshot服务

> 这里说的content、images等服务都是指containerd提供的gRPC服务 containerd/api/services/images/v1/images.proto文件里说images服务can really be considered a "metadata service"。所以上面把它称为“images元数据服务”。

下面主要分析fetch和unpack这两个主要步骤。

### fetch步骤

fetch步骤直接调用了content服务的实现逻辑（调函数，不是调用gRPC接口），即`containerd/cmd/ctr/commands/cotent/fetch.go`里的`Fetch`方法，所以我们实际上也可以在通过执行ctr images fetch来单独执行fetch步骤，并跳过pull命令的unpack部分。 fetch步骤会创建containerd包下的client和remote context对象。  

其中client负责与containerd的gRPC server联系，里面embed了services，即containerd的各种gRPC服务。  

而remote context负责与registry的联系，封装了：

1. base handlers：设置为fetch方法中创建的imagehandler

2. labels：pull命令的flag label的值

3. Platform：pull命令传入的platform、all-platform flag信息

4.  resolver：设置为fetch方法中创建的resolver。  
    
    resolver（`dockerResolver`类型），其中包括pull命令的一些flag，包括credential（user+password），plainHTTP,httpclient，track（NewInMemoryTrack函数创建）。其中的httpclient连接被hardcode了一些配置，比如从环境变量里读取的proxy设置，timeout时间等。
    
5. unpack：没被利用

6. snapshotter：hardcode成native。pull命令有flag，但是这里被hardcode

7. convert schema1：是否需要处理schema1版本的镜像，hardcode成true

fetch流程如下：

### 将reference resolve为oci规范里descriptor

reference只是用户输入的字符串，比如`docker.io/library/redis:alpine`，而descriptor代表了保存在registry中的资源，也可以说是一个待执行的下载任务。

1.  将reference解析为locator和object两部分。reference为用户在ctr cli中输入的希望拉取镜像的名字，比如`ctr image pull docker.io/library/redis:alpine`中的`docker.io/library/redis:alpine`部分，又比如`ctr image pull docker.io/library/redis@sha256:xxx`中的`docker.io/library/redis@sha256:xxx`部分。  
    
    locator代表repo的名称比如前面例子中的`docker.io/library/redis`和`docker.io/library/ubuntu`。而object则指代repo中某个具体的镜像layer，可以用tag标注，也可以用digest标注。比如前面例子中的tag `alpine`和digest `@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2`
    
    > 注意docker cli中registry的host可以省略，但是ctr不会帮我们自动补充。registry的host地址，也就是前面例子中的docker.io，是必须有的。  
    >
    > xxx指代镜像层对应的sha256值，因为长度过长，在本文中用xxx表示  
    >
    > descriptor定义在github.com/opencontainers/images-spec中，属于oci规范的实现
    
2. 根据locator和object信息构建`dockerBase`对象，并将之封装在`dockerFetcher`对象中。`dockerBase`包含一系列成员：
   1.  base：url类型。如过plain-http flag的值为true或者host地址为localhost，url的scheme为http，否则为https。url的Host为registry的host地址，如果是docker.io会被替换为registry-1.docker.io。base的path为`/v2`+locator中的path部分，比如`/library/ubuntu`  
       
       从定义上看base就是为了方便后期组装发送给registry的URL，只包含repo地址。后期发送者的给registry的请求都在此基础上加上一段。
       
   2. client: 设置为resolver中的httpclient

   3. username/secret：resolver中的对应值

3. 根据是否object中是否是digest，定义后续resolve reference过程中要向registry发出的http请求。如果object为digest，先尝试向registry发送`registry-1.docker.io/v2/library/ubuntu/manifest/sha256:xxx`请求，如果未能查询到结果，则再发送`registry-1.docker.io/v2/library/ubuntu/blobs/sha256:xxx`。如果object不是digest，而是tag，则发送的请求形如`registry-1.docker.io/v2/library/redis/manifest/alpine`

4. 根据上一步构建的URL，向registry的发起HTTP协议的查询请求，并根据response构建descriptor对象。descriptor包含3个成员：
   1.  digest为response header"Docker-Content-Digest"的值
   2.  MediaType为response "Content-Type" header的值
   3.  size为response "Content-Length"的值
       
       > 构建descriptor过程不依赖registry返回的response body中信息
       

至此完成了用户输入的reference到descriptor的转化，主要功臣就是`containerd/remotes/docker/resolver.go`中定义的`dockerResolver`对象。它实现了`remotes`包下的`Resolver`接口，负责与registry之间的交互。前面描述的resolve reference成descriptor的流程就在`dockerResolver`的`Resolve`方法中实现。 有了descriptor，我们就有了下载镜像任务的描述，接下来就可以开始按照registry api v2将docker镜像从registry pull到本地，我们继续按照流程走下去。

### 3个handler一场戏：镜像数据下载

假设我们使用ctr image pull docker.io/library/redis:alpine来pull一个redis镜像到本地，在上步中我们已经完成了将reference，也就是docker.io/library/redis:alpine转化为descriptor对象。在这个例子中descriptor包含3个成员：

1.  digest  
    
    为response header"Docker-Content-Digest"的值`sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`
    
2. MediaType  

   response "Content-Type" header的值`application/vnd.docker.distribution.manifest.list.v2+json`

3. size  

   response "Content-Length"的值2035

registry返回这个response给cli的含义就是让我们再发后续的请求给registry，去下载类型为manifest list，大小为2035字节，digest（作为唯一识别标志）为`sha256:xxx`的资源。  

我们知道一个manifest对应一个镜像，会包含镜像中各个层的信息，用户在拿到manifest之后就可以并行下载镜像中包含的各个层。然而这里的Mediatype `application/vnd.docker.distribution.manifest.list.v2+json`并非代表一个manifest，实际它上代表了一个manifest的集合。为什么pull redis:alpine这样一个镜像会对应一个manifest集合，也就是多个manifest呢？那是因为同一个镜像可以对应不同的平台，包括CPU架构、操作系统。所以registry在接收到我们的resolve请求时，通过MediaType为`application/vnd.docker.distribution.manifest.list.v2+json`的response告诉我们这个镜像存在适应多个平台的版本。而客户端可以进一步下载manifest list之后，再去选择下载其中的某个平台的镜像。 `client.go`中的`Pull`方法调用`images.Dispatch(ctx, handler, desc)`，从而启动镜像所有数据，包括元数据和镜像层（元数据和镜像层分别对应镜像manifest中的config和layer部分，详见本文后面对manifest的描述和例子），的下载流程。其中的`desc`参数就是前面步骤resolve出来的descriptor，而这个流程的执行的重大职责交给`handler`参数。`handler`实际上是一个函数变量：

```go
// containerd/images/handlers.go
func(ctx context.Context, desc ocispec.Descriptor) (subdescs []ocispec.Descriptor, err error) {
        var children []ocispec.Descriptor
        for _, handler := range handlers {
            ch, err := handler.Handle(ctx, desc)
            if err != nil {
                if errors.Cause(err) == ErrStopHandler {
                    break
                }
                return nil, err
            }

            children = append(children, ch...)
        }
        return children, nil
}
```


也就是对于每个需要处理的descriptor，顺序调用handler中包含的每个handler（是的，两个变量同一个名字），而每个handler如果在处理descriptor完成之后发现有更多的子 descriptor需要处理，则可以将子descriptor返回出来。最终在一轮调用结束之后，所有handler返回的子descriptor会合并一处，返回给更上一级。所谓的“更上一级”就是`images.Dispatch`，让我们看看它的定义。`images.Dispatch`是一个深度优先的算法，假设有1,2,3，...N个descriptor需要处理，它会先处理1， 然后处理1.1，在处理1.1.1直到1扩散开的子树全处理完，然后开始处理2：

```go
// containerd/images/handlers.go
func Dispatch(ctx context.Context, handler Handler, descs ...ocispec.Descriptor) error {
    eg, ctx := errgroup.WithContext(ctx)
    for _, desc := range descs {//顺序处理每个每个desc

        desc := desc

        eg.Go(func() error {
            desc := desc

            children, err := handler.Handle(ctx, desc)
            if err != nil {
                if errors.Cause(err) == ErrSkipDesc {
                    return nil // don't traverse the children.
                }
                return err
            }

            if len(children) > 0 {//如果一轮handler调用下来后，发现有更多的child，则迭代调用Dispatch
                return Dispatch(ctx, handler, children...)
            }

            return nil
        })
    }

    return eg.Wait()
}
```


现在我们看看整个处理流程的关键问题：1. 每个handler分别做什么，2. handler执行顺序如何排列：

```go
// containerd/client.go
handler = images.Handlers(append(pullCtx.BaseHandlers,
            remotes.FetchHandler(store, fetcher),
            childrenHandler,
        )...)
```


`images.Dispatch(ctx, handler, desc)`中的`handler`参数定义在`client.go`中。这里一共定义了3个handler，其中第一个`BaseHandlers`只是负责将当前的descriptor处理任务登记在正在执行的job列表中，定义如下：

```go
// containerd/cmd/ctr/commands/content/fetch.go
// 第一个handler的定义
func(ctx context.Context, desc ocispec.Descriptor([]ocispec.Descriptor, error) {
        if desc.MediaType != images.MediaTypeDockerSchema1Manifest {
            ongoing.add(desc)
}
```


而第2个handler通过`remotes.FetchHandler(store, fetcher)`构建。从函数的名称及参数看，就知道是利用`fetcher`参数（前面定义的`dockerFetcher`对象)将根据descriptor下载镜像元数据或镜像层，然后保存在`store`参数中（descriptor只是下载任务的描述，保存在content服务中的是按照descriptor下载下来的内容）。我们称第二个handler为fetch handler，这里我们简单分析fetch handler如何实现“读”、“写”这两步骤。

1.  如何从registry读数据  
    
    首先根据descriptor生成需要向registry发送请求的URL。fetch handler会从一系列URL里按照成功可能性的高低逐一挑选后发送出去。一旦能通过其中一个URL得到registry的回复，则停止这个过程。  
    
    URL优先级分几个等级，从高到低分别是：
    
    1.  首先是在构建descriptor时往里面的URLs成员添加的URL，这些URL会被优先考虑。在当前ctr pull redis镜像的例子中，我们并没有在构建descriptor的过程中添加这类URL。
    2.  其次如果需要获取的资源类型为manifest或者manifest list，则优先向registry发送`https://registry-1.docker.io/v2/library/redis/manifests/sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`这样的请求。
    3.  如果如果需要获取的资源类型为manifest或者manifest list，在发送`https://registry-1.docker.io/v2/library/redis/manifests/sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`不成功的情况下，尝试发送`https://registry-1.docker.io/v2/library/redis/blobs/sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`
    4.  如果需要获取的并非是manifest或者manifest list，则直接发送`https://registry-1.docker.io/v2/library/redis/blobs/sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`
    
2. 如何向store里写数据  

   `store`参数类型为`remoteContent`(定义在`content_store.go`中)，fetch handler利用封装在`remoteContent`内部的containerd的content gRPC服务客户端(`ContentClient`对象)，调用containerd的content gRPC服务，实现了`Info`、 `Delete`、 `ReaderAt`等实现镜像数据（下载好的元数据及镜像层内容）在content服务中的读写。在这里，我们跳过与content gRPC服务交互的大部分细节，只简单地说fetch handler从registry获取镜像相关数据之后，会给containerd的content gRPC服务发送`WriteContentRequest`请求，将数据写入content服务中。

以上优先级定义在`containerd/remotes/docker/fetcher.go`中定义的`getV2URLPaths`方法里，感兴趣的读者可以前往阅读。 相对于前两个handler执行逻辑比较直白，与registry定义的交互逻辑比较简单，第三个handler `childrenHandler`则与registry有更为丰富的交互，它通过以下“3步走”的方式构建出来：

```go
// containerd/client.go
// 第三个handler的定义
// Get all the children for a descriptor
childrenHandler := images.ChildrenHandler(store)
// Set any children labels for that content
childrenHandler = images.SetChildrenLabels(store, childrenHandler)
// Filter children by platforms
childrenHandler = images.FilterPlatforms(childrenHandler, pullCtx.Platforms...)
```


其中第一步images.ChildrenHandler(store)构建了一个具备解析manifest list和manifest类型数据的handler，具体流程如下：

```go
// containerd/images/image.go
// Children returns the immediate children of content described by the descriptor.
func Children(ctx context.Context, provider content.Provider, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
    var descs []ocispec.Descriptor
    switch desc.MediaType {
    case MediaTypeDockerSchema2Manifest, ocispec.MediaTypeImageManifest://解析manifest数据
        p, err := content.ReadBlob(ctx, provider, desc.Digest)//从containerd的content服务读取fetch handler写入的manifest
        if err != nil {
            return nil, err
        }

        var manifest ocispec.Manifest
        if err := json.Unmarshal(p, &manifest); err != nil {
            return nil, err
        }
        //将manifest中包含的config和镜像层对应descriptor写入待处理队列中
        descs = append(descs, manifest.Config)
        descs = append(descs, manifest.Layers...)
    case MediaTypeDockerSchema2ManifestList, ocispec.MediaTypeImageIndex://处理manifest list
        p, err := content.ReadBlob(ctx, provider, desc.Digest)//从containerd的content服务读取fetch handler写入的manifest list
        ...
        var index ocispec.Index
        if err := json.Unmarshal(p, &index); err != nil {
            return nil, err
        }
        //将manifest list中包含的多平台镜像的descriptor写入待处理队列
        descs = append(descs, index.Manifests...)
    case MediaTypeDockerSchema2Layer, MediaTypeDockerSchema2LayerGzip,
        MediaTypeDockerSchema2LayerForeign, MediaTypeDockerSchema2LayerForeignGzip,
        MediaTypeDockerSchema2Config, ocispec.MediaTypeImageConfig,
        ocispec.MediaTypeImageLayer, ocispec.MediaTypeImageLayerGzip,
        ocispec.MediaTypeImageLayerNonDistributable, ocispec.MediaTypeImageLayerNonDistributableGzip,
        MediaTypeContainerd1Checkpoint, MediaTypeContainerd1CheckpointConfig:
        // 已经下载好的镜像层和config层，无需进一步挖掘待处理descriptor.
        return nil, nil
        ...
    }

    return descs, nil
}
```


我们继续看`childhandler`是如何manifest list和manifest类型的资源。

#### manifest list处理

在fetch handler的帮助下，我们已经从registry下载了redis:alpine对应的manifest list，具体内容如下，我们可以看到这个manifest list中嵌套了适应多个平台的manifest的descriptor，比如适应linux/amd64、linux/armv6、linux/arm64等。按照前面“3步走”中第一步对`childrenhandler`定义，接下来会从containerd的content gRPC服务中把这个manifest list读取出来，然后把所有的适应各种平台的manifest对应的descriptor全都加入到待处理子descriptor队列中（第3步会按照ctr命令行里的flag过滤删掉无关的descriptor，第一步会先把所有平台的全都加入队列）。

```go
// sample manifest list
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1568,
         "digest": "sha256:d5f25b8d0f6125579cd3ac00a5a6e017ed55721d1b0850a3915da501fe7fd571",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1776,
         "digest": "sha256:b3e481a692c4fce21591fbaa4d7588bc6de5ae65161b3d7417e255dd22cabb71",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v6"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1775,
         "digest": "sha256:8f9dc588a0f8e521923fc61b36b9f54c6c19e8d195e8bbc987515388f4b80e8e",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1775,
         "digest": "sha256:ae54cb5b30f30284eed9796e56ad07e02eb5429a424b3163ed7a3116acf66a33",
         "platform": {
            "architecture": "386",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1776,
         "digest": "sha256:7e3d726348b3c671c82849b930466a2c5d9a653bf8cee20b05d22d30c68a0e75",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1775,
         "digest": "sha256:2398b7389b9ab8710e118aa8fdfc57d63077b8f153add86aaff67eacf825c597",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      }
   ]
}
```


对于manifest list这类包含子descriptor的数据，`childrenHandler`还会在“3步走”的第二步`SetChildrenLabels`中为保存在content服务中的manifest list加上label，以表达manifest list和manifest之间的父子关系。

```go
func SetChildrenLabels(manager content.Manager, f HandlerFunc) HandlerFunc {
    return func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
        children, err := f(ctx, desc)
        if err != nil {
            return children, err
        }

        if len(children) > 0 {//如果在上一步处理结束后发现包含子descriptor
            info := content.Info{
                Digest: desc.Digest,
                Labels: map[string]string{},
            }
            fields := []string{}
            for i, ch := range children {
                info.Labels[fmt.Sprintf("containerd.io/gc.ref.content.%d", i)] = ch.Digest.String()
                fields = append(fields, fmt.Sprintf("labels.containerd.io/gc.ref.content.%d", i))
            }
            //将子descriptor作为descriptor的label
            _, err := manager.Update(ctx, info, fields...)
            if err != nil {
                return nil, err
            }
        }

        return children, err
    }
}
```


经过“3步走”的第二步处理之后，我们查看containerd中的content服务中的数据，会发现digest为`sha256:e57274dac037e5b0c7680717fcaaa0efeffb23430e54e839c50819c9d842a38c`的资源，也就是manifest list，被加上了6个label，逐一对应redis:alpine的manifest list中包含的6个manifest。

```go
containerd.io/gc.ref.content.2=sha256:8f9dc588a0f8e521923fc61b36b9f54c6c19e8d195e8bbc987515388f4b80e8e  
containerd.io/gc.ref.content.3=sha256:ae54cb5b30f30284eed9796e56ad07e02eb5429a424b3163ed7a3116acf66a33  
containerd.io/gc.ref.content.4=sha256:7e3d726348b3c671c82849b930466a2c5d9a653bf8cee20b05d22d30c68a0e75  
containerd.io/gc.ref.content.5=sha256:2398b7389b9ab8710e118aa8fdfc57d63077b8f153add86aaff67eacf825c597  
containerd.io/gc.ref.content.0=sha256:d5f25b8d0f6125579cd3ac00a5a6e017ed55721d1b0850a3915da501fe7fd571  
containerd.io/gc.ref.content.1=sha256:b3e481a692c4fce21591fbaa4d7588bc6de5ae65161b3d7417e255dd22cabb71
```


`childrenHandler`的“3步走”中的第三步，则按照用户调用ctr pull镜像时输入的platform flag来过滤前面第一步中找到的子descriptor，这样下次一迭代调用images.Dispatch时就不会去下载被过滤掉的那些子descriptor对应的资源。第3步逻辑比较简单，这里不再详述。感兴趣的读者可以阅读`images/handlers.go`中定义的`FilterPlatforms`函数。

#### manifest处理

假设当前我们需要下载linux/amd64平台之上的redis:alpine镜像，那么我们会将以下子descriptor从manifest list中解析出来做下一步处理：

1.  mediaType: application/vnd.docker.distribution.manifest.v2+json
2.  size: 1568
3.  digest: sha256:d5f25b8d0f6125579cd3ac00a5a6e017ed55721d1b0850a3915da501fe7fd571
4.  platform:linux/amd64

我们会进入`Dispatch`方法的嵌套调用中，将这个descriptor对应的manifest下载下来，内容如下。我们可以看到在这个manifest中包含了1个config（`"application/vnd.docker.container.image.v1+json"`）和多个镜像层（`application/vnd.docker.image.rootfs.diff.tar.gzip`）对应的descriptor。可以想象`Dispatch`方法抽取子descriptor，并进一步嵌套调用，把config和镜像层分别下载，然后写入content gRPC服务中，这些过程在这里不再描述。

```yaml
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 5108,
      "digest": "sha256:80581db8c700155a91bec6fd6398dad9733135e7c58a19472aa679e8367692ab"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 2206542,
         "digest": "sha256:8e3ba11ec2a2b39ab372c60c16b421536e50e5ce64a0bc81765c2e38381bcff6"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1249,
         "digest": "sha256:1f20bd2a5c234ffab42de6cbf83522946614b21b642a8208dca6b0fd614c31db"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 9071,
         "digest": "sha256:782ff7702b5cd0a7c0109740838c74945fc27e4ce34e1028c24bf73f8249a63a"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 9407568,
         "digest": "sha256:cd719ead7ee305492514a8dfa2afcd0979a16e8192836b4aaed98d8d932973c0"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 98,
         "digest": "sha256:01018940af9a67873ad6737c275cb134372cdf1cda565af58dd14a1e3b85ab2a"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 398,
         "digest": "sha256:3f1bfdda9588f5c0643b485580060b460b21b5331f4760778ef3279680e20966"
      }
   ]
}
```


manifest中的config信息描述如何基于镜像构建容器及其中的进程，内容如下：

```
{
   "architecture":"amd64",
   "config":{
      "Hostname":"",
      "Domainname":"",
      "User":"",
      "AttachStdin":false,
      "AttachStdout":false,
      "AttachStderr":false,
      "ExposedPorts":{
         "6379/tcp":{

         }
      },
      "Tty":false,
      "OpenStdin":false,
      "StdinOnce":false,
      "Env":[
         "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
         "REDIS_VERSION=4.0.10",
         "REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-4.0.10.tar.gz",
         "REDIS_DOWNLOAD_SHA=1db67435a704f8d18aec9b9637b373c34aa233d65b6e174bdac4c1b161f38ca4"
      ],
      "Cmd":[
         "redis-server"
      ],
      "ArgsEscaped":true,
      "Image":"sha256:96b46164c9125001c0f6bc02364784abfe5687875318eb75db86eb924f75e9bc",
      "Volumes":{
         "/data":{

         }
      },
      "WorkingDir":"/data",
      "Entrypoint":[
         "docker-entrypoint.sh"
      ],
      "OnBuild":[

      ],
      "Labels":null
   },
   "container":"f1a03ba1354fa71cae8eece4541fb016712cf065a022466eb9bc97b018c8b577",
   "container_config":{
      "Hostname":"f1a03ba1354f",
      "Domainname":"",
      "User":"",
      "AttachStdin":false,
      "AttachStdout":false,
      "AttachStderr":false,
      "ExposedPorts":{
         "6379/tcp":{

         }
      },
      "Tty":false,
      "OpenStdin":false,
      "StdinOnce":false,
      "Env":[
         "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
         "REDIS_VERSION=4.0.10",
         "REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-4.0.10.tar.gz",
         "REDIS_DOWNLOAD_SHA=1db67435a704f8d18aec9b9637b373c34aa233d65b6e174bdac4c1b161f38ca4"
      ],
      "Cmd":[
         "/bin/sh",
         "-c",
         "#(nop) ",
         "CMD [\"redis-server\"]"
      ],
      "ArgsEscaped":true,
      "Image":"sha256:96b46164c9125001c0f6bc02364784abfe5687875318eb75db86eb924f75e9bc",
      "Volumes":{
         "/data":{

         }
      },
      "WorkingDir":"/data",
      "Entrypoint":[
         "docker-entrypoint.sh"
      ],
      "OnBuild":[

      ],
      "Labels":{

      }
   },
   "created":"2018-07-11T00:57:22.40324836Z",
   "docker_version":"17.06.2-ce",
   "history":[
      {
         "created":"2018-07-06T14:14:06.165546783Z",
         "created_by":"/bin/sh -c #(nop) ADD file:25f61d70254b9807a40cd3e8d820f6a5ec0e1e596de04e325f6a33810393e95a in / "
      },
      {
         "created":"2018-07-06T14:14:06.393355914Z",
         "created_by":"/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:55:42.769600464Z",
         "created_by":"/bin/sh -c addgroup -S redis \u0026\u0026 adduser -S -G redis redis"
      },
      {
         "created":"2018-07-11T00:55:43.769226605Z",
         "created_by":"/bin/sh -c apk add --no-cache 'su-exec\u003e=0.2'"
      },
      {
         "created":"2018-07-11T00:56:55.249363926Z",
         "created_by":"/bin/sh -c #(nop)  ENV REDIS_VERSION=4.0.10",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:56:55.452892851Z",
         "created_by":"/bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-4.0.10.tar.gz",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:56:55.678123654Z",
         "created_by":"/bin/sh -c #(nop)  ENV REDIS_DOWNLOAD_SHA=1db67435a704f8d18aec9b9637b373c34aa233d65b6e174bdac4c1b161f38ca4",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:57:19.918846632Z",
         "created_by":"/bin/sh -c set -ex; \t\tapk add --no-cache --virtual .build-deps \t\tcoreutils \t\tgcc \t\tjemalloc-dev \t\tlinux-headers \t\tmake \t\tmusl-dev \t; \t\twget -O redis.tar.gz \"$REDIS_DOWNLOAD_URL\"; \techo \"$REDIS_DOWNLOAD_SHA *redis.tar.gz\" | sha256sum -c -; \tmkdir -p /usr/src/redis; \ttar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1; \trm redis.tar.gz; \t\tgrep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 1$' /usr/src/redis/src/server.h; \tsed -ri 's!^(#define CONFIG_DEFAULT_PROTECTED_MODE) 1$!\\1 0!' /usr/src/redis/src/server.h; \tgrep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 0$' /usr/src/redis/src/server.h; \t\tmake -C /usr/src/redis -j \"$(nproc)\"; \tmake -C /usr/src/redis install; \t\trm -r /usr/src/redis; \t\trunDeps=\"$( \t\tscanelf --needed --nobanner --format '%n#p' --recursive /usr/local \t\t\t| tr ',' '\\n' \t\t\t| sort -u \t\t\t| awk 'system(\"[ -e /usr/local/lib/\" $1 \" ]\") == 0 { next } { print \"so:\" $1 }' \t)\"; \tapk add --virtual .redis-rundeps $runDeps; \tapk del .build-deps; \t\tredis-server --version"
      },
      {
         "created":"2018-07-11T00:57:20.789646125Z",
         "created_by":"/bin/sh -c mkdir /data \u0026\u0026 chown redis:redis /data"
      },
      {
         "created":"2018-07-11T00:57:21.027940339Z",
         "created_by":"/bin/sh -c #(nop)  VOLUME [/data]",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:57:21.312115762Z",
         "created_by":"/bin/sh -c #(nop) WORKDIR /data",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:57:21.720961412Z",
         "created_by":"/bin/sh -c #(nop) COPY file:9b596974f478088dc2d2bf2906046f6c8872ecff3c716abd89850fd50ec90c47 in /usr/local/bin/ "
      },
      {
         "created":"2018-07-11T00:57:21.956886141Z",
         "created_by":"/bin/sh -c #(nop)  ENTRYPOINT [\"docker-entrypoint.sh\"]",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:57:22.191555711Z",
         "created_by":"/bin/sh -c #(nop)  EXPOSE 6379/tcp",
         "empty_layer":true
      },
      {
         "created":"2018-07-11T00:57:22.40324836Z",
         "created_by":"/bin/sh -c #(nop)  CMD [\"redis-server\"]",
         "empty_layer":true
      }
   ],
   "os":"linux",
   "rootfs":{
      "type":"layers",
      "diff_ids":[
         "sha256:73046094a9b835e443af1a9d736fcfc11a994107500e474d0abf399499ed280c",
         "sha256:9f8f870604a08909589f09337944210db2bf72b2a71f0f707642b3aa9d225f9b",
         "sha256:221a0f51690d5f9063c9a113aa5e1340b50ac4474e7525efb9c60b945589110f",
         "sha256:1b44c45cbb1c6711042de1c2e8785b554da0d25ed03bbd2a6a13fb498eceb6ae",
         "sha256:211a9f3eb69c634f0368994600e7c93df7510f87870da57e51460f177c603ca9",
         "sha256:151bcb0152a97b3a445bc2e2ed29432fc77d662fcb746b05b87d77a8d7bbf023"
      ]
   }
}

```

