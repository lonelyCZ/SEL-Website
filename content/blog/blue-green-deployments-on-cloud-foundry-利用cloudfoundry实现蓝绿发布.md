+++

id= "244"

title = "Blue-Green Deployments on Cloud Foundry (利用CloudFoundry实现蓝绿发布)"
description = "Blue-Green Deployments on Cloud Foundry (利用CloudFoundry实现蓝绿发布)"
tags= [ "cloudfoundry" ]
date= "2014-12-02 19:25:20"
author = "丁轶群"
banner= "img/blogs/244/CF-BG-1.png"
categories = [ "cloudfoundry" ]

+++

**前言**
======

原文地址：[Blue-Green Deployments on Cloud Foundry](http://www.mattstine.com/2013/07/10/blue-green-deployments-on-cloudfoundry/) 

We’ll begin with a basic Spring application named ms-spr-demo. This app takes users to a simple web page announcing the ubiquitous “Hello World!” message. We’ll utilize the cf command-line interface to push the application: 

<!--more-->

**这里先向CF PUSH一个很简单的打印“Hello World”的应用：**

```shell
$ cf push --path build/libs/cf-demo.war  
Name> ms-spr-demo  

Instances> 1  

Memory Limit> 512M  

Creating ms-spr-demo... OK  

1: ms-spr-demo  
2: none  
Subdomain> ms-spr-demo  

1: cfapps.io  
2: mattstine.com  
3: none  
Domain> 1  

Creating route ms-spr-demo.cfapps.io... OK  
Binding ms-spr-demo.cfapps.io to ms-spr-demo... OK  

Create services for application?> n  

Save configuration?> y  

Saving to manifest.yml... OK  
Uploading ms-spr-demo... OK  
Starting ms-spr-demo... OK  
-----> Downloaded app package (9.5M)  
Installing java.  
Downloading JDK...  
Copying openjdk-1.7.0_25.tar.gz from the buildpack cache ...  
Unpacking JDK to .jdk  
Downloading Tomcat: apache-tomcat-7.0.41.tar.gz  
Copying apache-tomcat-7.0.41.tar.gz from the buildpack cache ...  
Unpacking Tomcat to .tomcat  
Copying mysql-connector-java-5.1.12.jar from the buildpack cache ...  
Copying postgresql-9.0-801.jdbc4.jar from the buildpack cache ...  
Copying auto-reconfiguration-0.6.8.jar from the buildpack cache ...  
-----> Uploading droplet (48M)  
-----> Uploaded droplet  
Checking ms-spr-demo...  
Staging in progress...  
Staging in progress...  
Staging in progress...  
Staging in progress...  
Staging in progress...  
Staging in progress...  
Staging in progress...  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  1/1 instances: 1 running  
OK  
```


The end result of this `cf push` event is that an application is now serving requests at `http://ms-spr-demo.cfapps.io`. The following graphic shows the state of our system, with the CF Router sending traffic to our application: 

**这个应用的实例将被映射到 `http://ms-spr-demo.cfapps.io` 这个域名上，如下所示：** 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616360/sel/CF-BG-1_nrswoh.png" alt="CF-BG-1" style="zoom:100%;" />
</center>
Next, we make a slight change to our application. Rather that saying “Hello World!” we decide to make it say “Goodbye World!” We build a new war file for the application. Rather than letting `cf` prompt us this time, we’ll make use of the `manifest.yml` file that we saved from our previous push. However, we’ll rename the application and provide a new route. Take a look: 

**接下来，我们对应用源码做了修改，使之打印“Goodbye World”，然后使用第一次部署时生成的`manifest.yml`来进行PUSH，不过，这一次我们将源码部署成一个新应用并映射了新域名：**

```ruby
--- applications: - name: ms-spr-demo-green   memory: 512M   instances: 1
url: ms-spr-demo-green.cfapps.io   path: build/libs/cf-demo.war 
```


As you can see, we’re calling our new application version `ms-spr-demo-green` and we’re mapping it to the URL `ms-spr-demo-green.cfapps.io`. Let’s push the application: 

**下面开始PUSH：**

```ruby
Using manifest file manifest.yml  

Creating ms-spr-demo-green... OK  

1: ms-spr-demo-green  
2: none  
Subdomain> ms-spr-demo-green  

1: cfapps.io  
2: mattstine.com  
3: none  
Domain> 1  

Creating route ms-spr-demo-green.cfapps.io... OK  
Binding ms-spr-demo-green.cfapps.io to ms-spr-demo-green... OK  
Uploading ms-spr-demo-green... OK  
Starting ms-spr-demo-green... OK  
-----> Downloaded app package (9.5M)  
Installing java.  
Downloading JDK...  
Copying openjdk-1.7.0_25.tar.gz from the buildpack cache ...  
Unpacking JDK to .jdk  
Downloading Tomcat: apache-tomcat-7.0.41.tar.gz  
Copying apache-tomcat-7.0.41.tar.gz from the buildpack cache ...  
Unpacking Tomcat to .tomcat  
Copying mysql-connector-java-5.1.12.jar from the buildpack cache ...  
Copying postgresql-9.0-801.jdbc4.jar from the buildpack cache ...  
Copying auto-reconfiguration-0.6.8.jar from the buildpack cache ...  
-----> Uploading droplet (48M)  
-----> Uploaded droplet  
Checking ms-spr-demo-green...  
Staging in progress...  

Staging in progress...  
Staging in progress...  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  0/1 instances: 1 starting  
  1/1 instances: 1 running  
OK  
```

We now have two instances of the application running, each of them using distinct routes: 

**PUSH完成之后，两个应用都运行在CF上并且按照下述方式分配域名：** 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616360/sel/CF-BG-2_pwgfgp.png" alt="CF-BG-3" style="zoom:100%;" />
</center>

Now it’s time for the magic to happen. We’ll map our original URL route (`ms-spr-demo.cfapps.io`) to our “green” instance. This is accomplished very simply by using `cf`: 

**这时，我们将老应用（蓝色）的域名ms-spr-demo.cfapps.io同时映射到新应用（绿色）上：**

```shell
 cf map --app ms-spr-demo-green --host ms-spr-demo --domain cfapps.io Binding ms-spr-demo.cfapps.io to ms-spr-demo-green... OK 
```


The CF router immediately begins to load balance requests between each instance of the application, as shown here:

**此时的域名映射如下图所示：** 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616360/sel/CF-BG-31_e1rpt9.png" alt="CF-BG-3" style="zoom:100%;" />
</center>

Now our router will send requests to `ms-spr-demo.cfapps.io` to both instances of the application, while `ms-spr-demo-green.cfapps.io` only services the “green” instance. Once we determine that all is well, it’s time to stop routing requests to the “blue” instance. This is just as simple using `cf`: 

**这样，ms-spr-demo.cfapps.io将流量导向了蓝绿两个应用，而ms-spr-demo-green.cfapps.io则只将流量导向新部署的绿色应用。这也是我们对绿色应用进行测试和观察的好时机（我们还可以调节蓝色和绿色应用的实例数目来进行流量分配比），一旦我们能够确定绿色应用能够正常的对外服务，接可以将蓝色应用实例切断了：**

```shell
 cf unmap --url ms-spr-demo.cfapps.io --app ms-spr-demo Unbinding ms-spr-demo.cfapps.io from ms-spr-demo... OK 
```


Our “blue” instance is now no longer receiving any web traffic: 
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616361/sel/CF-BG-4_vkjlr5.png" alt="CF-BG-4" style="zoom:100%;" />
</center>

We can now decomission our “blue” instance, or we can leave it around for a period of time in case we decide we need to roll back our changes. The important thing is that our customers suffered absolutely no downtime! 

**这样，ms-spr-demo.cfapps.io ms-spr-demo-green.cfapps.io都将流量导向了绿色应用，我们可以删除旧的蓝色应用或者保留一段时间以防意外情况。最后，我们可以把ms-spr-demo-green.cfapps.io这个测试域名也删除。** **整个发布过程，用户没有感受到任何宕机或者服务重启。** 

附录 Martin Fowler大神提出的蓝绿发布： 

One particular practice associated with Continuous Delivery is `Blue-Green Deployments`. Martin Fowler describes these very well at the link provided, but I’ll summarize briefly here:

*   Cut-over to a new software version is tricky, and must be quick in order to minimize downtime events.
*   Blue-green deployments ensure the parallel existence of two, identical (as possible) production environments.
*   At any given point, only one (e.g. blue) services production traffic.
*   New deploys are made to the other (e.g. green) environment. Final smoke testing is performed here.
*   When green is determined ready, we begin routing traffic to it.
*   We then stop routing traffic to blue.