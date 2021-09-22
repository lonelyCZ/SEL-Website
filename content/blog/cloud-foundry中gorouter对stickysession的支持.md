+++

id= "194"

title = "Cloud Foundry中gorouter对StickySession的支持"
description = "Cloud Foundry作为业界出众的PaaS平台，在应用的可扩展性方面做得非常优秀。具体来讲，在一个应用需要横向伸展的时候，Cloud Foundry可以轻松地帮助用户做好伸展工作，也就是创建出一个应用的多个实例，多个实例地位相等，多个实例共同为用户服务，多个实例共同分担访问压力。大致来说，可以认为是共同分担访问压力，但是也不是针对所有该应用的访问，都进行均衡，分发到不同的应用实例处。譬如：当Cloud Foundry的访问用户访问应用时，第一次的访问，gorouter会将请求分发到应用的某个实例处，但是如果该用户之后的访问都是有状态的，不希望之后的访问会被分发到该应用的其他实例处。针对以上这种情况，Cloud Foundry提供了自己的解决方案，使用StickySession的方式，保证请求依旧分发给指定的应用实例。"
tags= [ "gorouter" , "cloudfoundry" ]
date= "2014-11-21 13:04:13"
author = "孙宏亮"
banner= "img/blogs/209/cloud-foundry.jpg"
categories = [ "cloudfoundry" ]

+++

# 前言

Cloud Foundry作为业界出众的PaaS平台，在应用的可扩展性方面做得非常优秀。 

具体来讲，在一个应用需要横向伸展的时候，Cloud Foundry可以轻松地帮助用户做好伸展工作，也就是创建出一个应用的多个实例，多个实例地位相等，多个实例共同为用户服务，多个实例共同分担访问压力。 

大致来说，可以认为是共同分担访问压力，但是也不是针对所有该应用的访问，都进行均衡，分发到不同的应用实例处。譬如：当Cloud Foundry的访问用户访问应用时，第一次的访问，gorouter会将请求分发到应用的某个实例处，但是如果该用户之后的访问都是有状态的，不希望之后的访问会被分发到该应用的其他实例处。针对以上这种情况，Cloud Foundry提供了自己的解决方案，使用StickySession的方式，保证请求依旧分发给指定的应用实例。

<!--more-->

本文即分析Cloud Foundry中gorouter关于StickySession的实现方式。

该部分内容需要对gorouter有一定的了解，可以参见笔者之前的博文：Cloud Foundry中gorouter源码分析

关于StickySession的信息，gorouter所做的工作，主要分为两个部分：如何给HTTP请求添加StickySession、如何通过StickySession辨别应用的具体实例。

**如何给HTTP请求添加StickySession**
----------------------------

在分析这个问题的时候，首先我们需要提出另一个问题：什么情况下需要给HTTP请求添加StickySession？ 首先，来看这样的一个方法setupStickySession的go语言实现：

```ruby
    func (h *RequestHandler) setupStickySession(endpointResponse *http.Response, endpoint *route.Endpoint) {  
        needSticky := false  
        for _, v := range endpointResponse.Cookies() {  
             if v.Name == StickyCookieKey {  
                needSticky = true  
                break  
            }  
        }  
    
        if needSticky && endpoint.PrivateInstanceId != "" {  
            cookie := &http.Cookie{  
                Name:  VcapCookieId,  
                Value: endpoint.PrivateInstanceId,  
                Path:  "/",  
            }  
    
            http.SetCookie(h.response, cookie)  
        }  
    }  
```


紧接着，查看setupStickySession方法何时被调用的代码：

```ruby
    func (h *RequestHandler) HandleHttpRequest(transport *http.Transport, endpoint *route.Endpoint) (*http.Response, error) {  
        h.transport = transport  
    
        h.setupRequest(endpoint)  
        h.setupConnection()  
    
        endpointResponse, err := transport.RoundTrip(h.request)  
        if err != nil {  
            return endpointResponse, err  
        }  
    
        h.forwardResponseHeaders(endpointResponse)  
    
        h.setupStickySession(endpointResponse, endpoint)  
    
        return endpointResponse, err  
    }  
```


在setupStickySession方法中，可以看到：首先进入一个循环语句块中，当endpointResponse中的cookies中有名为StickyCookieKey的话，将needSticky字段置为true，跳出循环。这里也就回答了什么时候需要给HTTP请求添加StickySession的问题。关于StickyCookieKey的值，可以先看一下gorouter的设置：

```ruby
    const (  
        VcapCookieId    = "__VCAP_ID__"  
        StickyCookieKey = "JSESSIONID"  
    )  
```


可以看到StickyCookieKey的值为JSESSIONID，而JSESSIONID是Tomcat对session id的称呼，在其他容器中就不一定叫JSESSIONID了。因此，如果平台运维者自定义了一种容器的buildpack，而这个容器中对于session id的称呼不为JSESSIONID的话，那么Sticky Session在gorouter中将不能被实现，除非将这部分内容自行进行改写，或者修改该容器对session id的称呼。 

紧接着在setupStickySession中，通过一个判断，来给response创建一个cookie，创建的cookie有Name字段，Value字段以及Path字段。最后通过http.SetCookie(h.response, cookie)来实现对response的设置添加cookie。 读完setupStickySession方法的实现，现在进入调用该方法的HandleHttpRequest方法，该方法的主要工作是：将haproxy代理来的请求，转发给具体相应的DEA上的应用实例，最后构建该请求的response信息，并返回该响应信息。在返回response响应信息的之前，gorouter为该response信息设置了StickySession。 以上便是如何给HTTP请求添加StickySession，以下分析如何通过StickySession辨别应用的具体实例。

**如何通过StickySession辨别应用的具体实例**
------------------------------

当一个请求到达gorouter的时候，gorouter需要首先辨别该请求中是否带有StickySession的信息，如果有的话，，gorouter分析该请求中的相应信息，最终得到指定应用实例的信息，并将请求转发到该指定应用实例。具体实现代码如下： \[plain\] view plaincopy

```ruby
    func (p *proxy) lookup(request *http.Request) (*route.Endpoint, bool) {  
        uri := route.Uri(hostWithoutPort(request))  
    
        // Try choosing a backend using sticky session  
        if _, err := request.Cookie(StickyCookieKey); err == nil {  
            if sticky, err := request.Cookie(VcapCookieId); err == nil {  
                routeEndpoint, ok := p.registry.LookupByPrivateInstanceId(uri, sticky.Value)  
                if ok {  
                    return routeEndpoint, ok  
                }  
            }  
        }  
    
        // Choose backend using host alone  
       return p.registry.Lookup(uri)  
    }  
```


从源码中可以看到，首先从请求中提取出uri，随后分析请求中是否带有StickyCookieKey，也就是JSESSIONID，若有的话，继续分析是否带有VcapCookieID，也就是**VCAP\_ID**，若有的话，那说明是gorouter支持的StickySession，从sticky中找出value属性对应的值，也就是应用PrivateInstanceID，通过该ID寻找出具体对应的是应用的哪个实例，最后返回该应用实例的具体信息，当然包括最终该实例的IP：PORT。如果以上这些StickyCookieKey或者VcapCookieID不存在的话，也就是说该请求不支持StickySession，那么将直接通过uri，寻找某一个应用实例来为该请求服务。 以上便是Cloud Foundry中gorouter对StickySession的支持与实现。 

**转载请注明出处。**
------------
这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。希望本文能够对接触Cloud Foundry v2中gorouter实现StickySession的人有些帮助，如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 