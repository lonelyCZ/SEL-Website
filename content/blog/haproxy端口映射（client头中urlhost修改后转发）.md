+++
id = "99"

title = "Haproxy端口映射（client头中URL/HOST修改后转发）"
description = "CloudFoundry是对域名强依赖的云计算集群，没有域名的话几乎无法访问。但是域名备案等事宜所耗时间较长，在上线较为紧急的情况下，就需要实现直接通过“IP+端口”的形式，在公网访问CF集群上部署的APP。"
tags = ["cloudfoundry","haproxy"]
date = "2014-10-28 17:02:26"
author = "丁轶群"
banner = "img/blogs/99/haproxy.jpg"
categories = ["haproxy"]

+++

CloudFoundry是对域名强依赖的云计算集群，没有域名的话几乎无法访问。但是域名备案等事宜所耗时间较长，在上线较为紧急的情况下，就需要实现直接通过“IP+端口”的形式，在公网访问CF集群上部署的APP。

<!--more-->


解决方案
====

配置两层Haproxy，第一层的Haproxy与公网地址绑定。 对第一层的Haproxy进行配置，把外部通过IP+PORT访问的地址映射到后端第二层Haproxy，并把其访问的http Head修改，把Host字段改成能被Cloudfoundry接受的url字符串。 第二层Haproxy就跟CloudFoundry官方配置相同，作为负载均衡把流量导向下层多个gorouter。

Haproxy的安装：(也可通过源码安装)
=====================

`apt-get install haproxy`

修改基本的配置文件如下：
============

配置文件所在地址：`/etc/haproxy/haproxy.cfg`（用xx.xx.xx.xx代表一个IP地址）

~~~ruby
global
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    stats socket /var/lib/haproxy/stats
    debug
    
defaults
    log global
    option httpclose
    timeout connect 30000ms
    timeout client 300000ms
    timeout server 300000ms

frontend http-in
    mode http
    bind *:81
    reqirep ^Host:\ xx.xx.xx.xx\:81 Host:\ t1.cloud.paas
    option httplog
    option forwardfor
    reqadd X-Forwarded-Proto:\ http
    default_backend http-routers

backend http-routers
    mode http
    reqirep ^Host:\ xx.xx.xx.xx\:81 Host:\ t1.cloud.paas
    balance roundrobin
        server node0 t1.cloud.paas:80 check inter 1000

frontend http-in1
    mode http
    bind *:80
    reqirep ^Host:\ xx.xx.xx.xx\:80 Host:\ t2.cloud.paas
    option httplog
    option forwardfor
    reqadd X-Forwarded-Proto:\ http
    default_backend http-routers1

backend http-routers1
    mode http
    reqirep ^Host:\ xx.xx.xx.xx\:80 Host:\ t2.cloud.paas
    balance roundrobin
        server node0 t1.cloud.paas:80 check inter 1000
~~~

配置文件中涉及到的要点
===========

1.  在global中加入debug可以查看访问的日志信息
2.  frontend表示访问的输入，对应一个backend导向后端服务器。
3.  bind绑定监听的IP和端口
4.  reqirep可以修改http的头，cloudfoundry中gorouter解析域名url的字段是Host，所以我们可以通过reqirep把IP地址换成我们的访问域名地址。
5.  在balance 后面添加haproxy的地址，可以写多个。
6.  最后设置option httpclose。每次关闭连接，保证每次访问的字段都被改变。

（这样做效率会降低很多，但是解决了应急的需求）

部分OPTION选项解析
============

*   option httpclose ：HAProxy会针对客户端的第一条请求的返回添加cookie并返回给客户端，客户端发送后续请求时会发送此cookie到HAProxy，HAProxy会针对此cookie分发到上次处理此请求的服务器上，如果服务器不能忽略此cookie值会影响处理结果。如果避免这种情况配置此选项，防止产生多余的cookie信息。
    
*   option forwardfor ：如果服务器上的应用程序想记录发起请求的客户端的IP地址，需要在HAProxy上配置此选项，这样HAProxy会把客户端的IP信息发送给服务器，在HTTP请求中添加"X-Forwarded-For"字段。
    
*   option originalto ：如果服务器上的应用程序想记录发起请求的原目的IP地址，需要在HAProxy上配置此选项，这样HAProxy会添加"X-Original-To"字段。
    
*   option dontlognull ：保证HAProxy不记录上级负载均衡发送过来的用于检测状态没有数据的心跳包。
    

Haproxy作为服务启动
=============

需要修改一个配置文件`/etc/default/haproxy` 把ENABLED字段的0改为1. 然后通过`service haproxy start`就可以启动。 最后调试完毕后记得把global中的debug字段删掉。

参考内容
====

1.  [Cloud Foundry中gorouter源码分析](http://blog.csdn.net/shlazww/article/details/11974411)
2.  [Haproxy 配置项\\配置实例](http://www.cnblogs.com/dkblog/archive/2012/03/13/2393321.html)
3.  [stackoverflow上的类似问题:haproxy-reqirep-on-host-header-not-forwarding](http://stackoverflow.com/questions/26136239/haproxy-reqirep-on-host-header-not-forwarding)
4.  [haproxy官方文档](http://www.haproxy.org/download/1.4/doc/configuration.txt)