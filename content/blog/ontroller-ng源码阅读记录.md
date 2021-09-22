+++
id = "10"

title = "Cloud_Controller_NG源码阅读记录"
description = "Cloud_Controller_NG就是cloud controller next generation的意思。即Cloud Foundry 平台用来管理控制应用和服务的组件。官方文档是这么解释CCNG的作用的： * 维护一个包含应用、服务、配置信息的数据库(CCDB)。 * 在blobstore中存储应用的packages和droplets。 * 通过NATS和其他组件进行通信，包括Droplet Execution Agents (DEAs)、Service Gateways、和 Health Manager（HM）。 * 其他供用户调用的后端API。阅读该组件源码，有助于从应用管理的视角理解cloudfoundry的运行过程。"
tags = ["ccng","cloudfoundry"]
date = "2014-04-03 14:15:12"
author = "孙健波"
banner = "img/blogs/10/cloud_controller_ng.jpg"
categories = ["cloud_controller_ng","Cloudfoundry"]

+++



Cloud\_Controller\_NG就是cloud controller next generation的意思。即Cloud Foundry 平台用来管理控制应用和服务的组件。 

是这么解释CCNG的作用的： \* 维护一个包含应用、服务、配置信息的数据库(CCDB)。 \* 在blobstore中存储应用的packages和droplets。 \* 通过NATS和其他组件进行通信，包括Droplet Execution Agents (DEAs)、Service Gateways、和 Health Manager（HM）。 \* 其他供用户调用的后端API。 

阅读该组件源码，有助于从应用管理的视角理解cloudfoundry的运行过程。 

**说明：** 1. Cloud\_Controller\_NG以下简称CCNG。 2. 本文所阅读的源码版本为github中cf-release中V145 tag下面的CCNG项目源码。

* * *

CCNG各模块概览
=========

首先让我们看一下这张框架图： 

![Alt text](http://wonderflow.info/wp-content/uploads/2014/02/ccng.png) 
**图1.CCNG架构图byshlallen** 

从ccng架构图中可以看出ccng可以分为以下多个模块： 

1. Stager模块，主要负责与DEA组件的staging部分进行交互； 

2. DEA模块，主要负责与DEA组件进行交互； 

3. Blobstore模块，主要负责创建一个blobstore的存储，以供Cloud Foundry存储应用所需的静态文件； 

4. HealthManager（HM）模块，主要负责与HealthManager组件进行交互； 

5. CCDB模块，负责维护cloud\_controller的数据库； 

6. collector\_registrar模块，负责作为component向Collector组件注册； 

7. router\_registrar模块，负责将cloud controller组件的域名注册至Router组件；

8.  legacy\_api部分，负责接管ccng关于info，bulk以及services等的RESTful请求； 

9. Permission模块，负责各种不同权限用户的注册和认证。 

10. 其他零散模块 

    我们按官方文档给出的组件功能介绍的顺序逐步深入各模块。 

DB模块
----

众所周知，CCDB就是CC的一个postgresql数据库，用于存储CC需要的一些数据。 

在CCNG的rakefile里面，有着CCDB建表的初始化信息，具体的建表内容在`db/migrations/*.rb`中。 

CCNG开始正常运行后，主要调用lib/sequel\_plugins/update\_or\_create.rb里面的函数对以下信息的改变进行更新（更新的代码都在以下各部分的源码中，可以使用全文搜索update\_or\_create函数查看）。 

1. framework：语言运行时框架。就是`"*.war"`包可上传的各种框架，在`/var/vcap/jobs/cloud_controller/config/staging`路径下的各类\*.yml存储。@`lib/cloud_controller/models/framework.rb` 
2. stack ：应用运行的堆环境，默认为lucid64。[stacks](http://docs.cloudfoundry.com/docs/running/architecture/stacks.html)就是一个预先构建的文件系统，包括可运行应用的操作系统环境。@`lib/cloud_controller/models/stack.rb` 
3. runtime：应用可运行语言的运行时环境。运行时环境的具体信息在配置文件`config/runtimes.yml`中。@`lib/cloud_controller/models/runtime.rb` 
4. quota：一些共享信息的更新，包括sevice数量，内存限制等。@`lib/cloud_controller/models/quota_definition.rb` 5. service：存储支持的service信息。@`app/models/services/service_broker.rb`

Blobstore模块
-----------

**注：**blobstor相关源码都@`lib/cloud_controller/blobstore`文件夹下 

Blobstore模块主要负责维护静态文件的存储，这部分文件主要分为三部分： \* package,通俗的讲即应用程序打包。blobstore会把应用的源码进行压缩，变为package。 \* buildpack，通俗的讲即程序build时所需环境打包。CCNG的blobstore又把buildpack又分为admin buildpack,cache buildpack两部分。 \* droplet，通俗的讲即buildpack+package。 

![Alt text](http://wonderflow.info/wp-content/uploads/2014/02/droplet.png) 

**图2.buildpack、droplet、package结构图 by 薛伟** 

从如上的工作内容可知Blobstore具体实现内容为： 1. 文件的压缩，变为package。@`local_app_bits.rb` 运行时调用 `cloud_controller/safe_zipper.rb` 2. 文件指纹收集，避免重复备份。采用SHA1的方式。@`fingerprints_collection.rb` 3. 拷贝package到blobstore进行保存，被`app_bits_package.rb`调用。 4. blobstore中存储的各种包下载地址的创建。@`blobstore_url_generator.rb` 5. blobstore中文件的下载 @`cdb.rb`@`blobstore.rb` 6. blobstore创建的内容主要由@`lib/cloud_controller/dependency_locator.rb`调用

Stager模块
--------

通过blobstore模块的描述我们知道应用的源码和buildpack进行捆绑打包后的最终形式即为droplet，而负责管理这个过程的组件就是Stager模块。 

在Cloud Foundry v1的版本中，整个Cloud Foundry集群只有一个Stager组件。而在Cloud Foundry v2版本中，已经不存在原来独立的Stager组件，而将和Stager功能类似的staging模块设计成DEA的一个子模块。因此，如果v2版本的Cloud Foundry 部署了多个DEA的话，那么该云平台中就会有多个Staging模块。 

正是由于有多个Staging模块的缘故，在ccng处设计了一个与这些Staging进行交互的模块，也就是我们这里谈论的CCNG的stager模块。 

stager模块分为以下三个部分：

1.  **stager\_pool**：stager模块维护的资源池，stager\_poor会保持与DEA中stager模块的通信，维护最优的5个stager的信息，用来接收和分发有关staging任务的请求。@`lib/cloud_controller/stager/stager_poor.rb`
2.  **app\_stager**: stager模块主要负责处理任务的模块。当app上传的接口收到app\_stage命令时，就会调用该模块，该接口@`/lib/cloud_controller/api/app.rb`。该模块会把app和service进行绑定并向nats发送消息让DEA进行staging的过程，成功staging以后就在blobstore里面存储起来。@`lib/cloud_controller/app_stager.rb`
3.  **app\_stager\_task**:CCNG有一个app\_observer模块，它会关注app需要进行的动作，包括update、delete。第2点完成以后，update功能会继续stage，调用app\_stager\_task部分,向nats发送请求。@`lib/cloud_controller/app_stager_task.rb`

一开始看的时候发现第2步和第3步有很多地方函数命名是相同的，觉得很奇怪。后来才知道，第3步就是第2步的延续。所以发送的请求内容是不同的，同时第3步对更新的返回信息有更多的处理。 

可以看一下第2步stage\_request的代码，此时droplet还未形成：

~~~ruby
def staging_request(app)
        {
          :app_id       => app.guid,
          :properties   => staging_task_properties(app),
          :download_uri => LegacyStaging.app_uri(app.guid),
          :upload_uri   => LegacyStaging.droplet_upload_uri(app.guid)
        }
      end
~~~

第3点的stage\_request代码除上述部分外，还要多

~~~ruby
:buildpack_cache_download_uri => @blobstore_url_generator.buildpack_cache_download_url(@app),  
    :buildpack_cache_upload_uri => @blobstore_url_generator.buildpack_cache_upload_url(@app),  
    :start_message => start_app_message,  
    :admin_buildpacks => admin_buildpacks  
~~~

同时上传和下载的地址获取方式也已经不同了，因为droplet已经实际存在了：

~~~ruby
:download_uri => @blobstore_url_generator.app_package_download_url(@app),  
    :upload_uri => @blobstore_url_generator.droplet_upload_url(@app),  
~~~

同时，app\_stage\_task在进行stage之前会从stager\_pool中找出合适的stager来完成staging任务。

DEA模块
-----

DEA模块的功能是负责CCNG与众DEA之间通信。 

在CCNG中，DEA模块也可以大致分为三个部分：

1.  **dea\_client**：主要负责向DEA发送CCNG收到的相关请求。如：应用的启动、暂停、运行，查找具体的instance信息，应用相关修改等等。同时，它也会向HealthManger模块发送请求查询应用状态。@`lib/cloud_controller/dea/dea_client.rb`
2.  **dea\_respondent**：负责相应并完成由DEA主动发送过来的请求。请求主要是当DEA中的应用退出或意外崩溃时，会由DEA通过NATS向CCNG发送应用退出的消息，从而CCNG可以做后续处理工作。@`lib/cloud_controller/dea/dea_respondent.rb`
3.  **dea\_pool**：负责维护Cloud Foundry中所有DEA的资源池。当Cloud Foundry中有新的DEA启动并开始工作时，都会通过NATS向CCNG发布advertise信息，而CCNG则通过发来的该部分信息向dea\_pool中注册DEA信息。当有应用需要启动的时候，CCNG需要找到合适的DEA来完成启动工作，这个时候则由dea\_pool通过各DEA的资源使用情况等其他重要条件完成DEAs的筛选，找出符合条件的DEA。@`lib/cloud_controller/dea/dea_pool.rb`

小结：APP上传流程示意图
-------------

看完了上述几个模块后，我们可以通过下图进行上述知识的整合。

 ![Alt text](http://wonderflow.info/wp-content/uploads/2014/02/cfpush.png) 

**图3.APP上传流程示意图 by 薛伟** 

此图表现的就是执行了`$cf push`命令（原`$vmc push`命令）以后，CF集群运作的过程。圆圈中的数字表示执行顺序。

HealthManager模块
---------------

HealthManager(以下简称HM)模块主要负责与health\_manager建立通信，并完成有关应用健康状态的监控。该部分也可以简单分为两部分：

1.  **HealthManagerClient：**充当CCNG与HM进行通信的client端，通过NATS发消息到HM实现查询指定app的状态、 查找crash的instance、 查看所有app的健康状态这三个功能。@`lib/cloud_controller/health_manager_client.rb`
2.  **HealthManagerRespondent：**负责接收CCNG与HM通信过程中health\_manager发来启动/停止应用的信息。@`lib/cloud_controller/health_manager_respondent.rb`

Service相关模块
-----------

这个模块的主要代码都在`lib/cloud_controller/legacy_api`文件夹下，为什么叫legacy这个名字，我至今没有参透。正如该文件夹的名字所言，文件夹下的文件都是对外提供api的，形式就是一个http对应一个执行函数。该模块涉及到Service的只要有3个，LegacyServiceGateway、LegacyServiceLifecycle和LegacyService:

1.  **LegacyServiceGateway**：service\_gateway对外接口。包括获取所提供的service\_gatway信息并更新、根据条件过滤service\_gateway信息、删除、更新service\_gateway等操作，并在ccdb中进行记录。@`lib/cloud_controller/legacy_api/legacy_service_gateway.rb`
2.  **LegacyService**：service对外的接口，创建、删除、查找service都通过它。@`lib/cloud_controller/legacy_api/legacy_service.rb`
3.  **LegacyServiceLifecycle**：上面针对的是service种类，此处针对的是某个service的一个instance的生命周期的管理。@`lib/cloud_controller/legacy_api/legacy_service_lifecycle.rb`

其他API功能一览
---------

### lib/cloud\_controller/api目录下：

*   info.rb: 展示集群信息，主要是配置文件上写的信息展示
*   crashes.rb：查看崩溃的应用
*   app\_summary.rb：总览各类应用的情况
* **TO BE CONTINUED...**

  


CCNG Model图
-----------

![Alt text](http://10.10.103.47/gitlab/cf_src_docs/raw/master/pictures/model.png)

* * *

从CCNG启动开始的程序细节
==============

整个项目从可执行脚本bin/cloud\_controller开始。 

启动命令为：`$ ./bin/cloud_controller [-c] [-m] [-d]` 

打开bin/cloud\_controller这个文件，前两行就是

~~~shell
$:.unshift(File.expand_path("../../lib", __FILE__))
$:.unshift(File.expand_path("../../app", __FILE__))
~~~

这两行的作用就是把项目中lib和app两个文件夹加入到ruby的path路径中，这样就能在后面直接使用require/load调用了。 

开始代码是这一行：`VCAP::CloudController::Runner.new(ARGV).run!` 

从代码中可以看出调用了Runner模块中的run函数，文件路径为：`lib/cloud_controller/runner.rb` 

在执行run!方法前，先经历了一个初始化。默认rack的环境是生产环境，默认配置文件的路径是config/cloud\_controller.yml，然后根据参数进行调整。 

三个参数分别代表：

*   \-c 设置配置文件读取路径，-c后可指定cloud\_controller.yml，代替默认读取的config下的cloud\_controller.yml为启动配置文件。
    
*   \-d 进行rack\_env设置，把默认的production变成development。程序会执行 register Sinatra::Reloader，动态加载 reload\_path 路径下的\*.rb文件，这样就可以在不需求重启进程情况下，查看修改后结果，加快开发速度
    
*   \-m 数据库初始化，默认进行 rake db:migrate 后，数据库都是空的，加了-m 选项，cc会在启动的时候将一些默认的值写到数据库中，比如：quota定义等等
    

**启动过程**如下:

~~~ruby
def run!
start_cloud_controller
config = @config.dup 
Seeds.write_seed_data(config) if @insert_seed_data
app = create_app(config)
start_thin_server(app, config)
end
~~~

执行create\_app之前会做一些初始化工作，包括：

1. cloud\_controller初始化：创建pid文件，设置logger，创建和db数据库的连接 
2. steno日志对象初始化
3. message\_bus初始化，所有的subscribe和publish都通过这个对象来进行操作 

**附，初始化配置的相关模块：** \* MessageBus \* AccountCapacity \* ResourcePool \* AppPackage \* StagerPool \* AppStager \* LegacyStaging \* DeaPool \* DeaClient \* LegacyBulk \* HealthManagerClient \* Models 

之后通过create\_app生成一个Rack::Builder实例,其中有如下关键代码

~~~ruby
map "/" do
run VCAP::CloudController::Controller.new(config)
end
~~~

map '/'表示，不管接收到什么http请求，都会调用Controller进行处理。do...end模块中[run这个方法](http://rack.rubyforge.org/doc/Rack/Builder.html#method-i-run)的用法属于sinatra框架的特殊语法。 

最后通过start\_thin\_server启动一个http服务，将生成的Rack::Builder实例传递给该服务器，处理来自客户端的请求。thin\_server启动完毕以后，基本上CCNG的启动环节算是结束了。然后进入处理http请求的事件驱动的循环过程。 

**附，框架相关:** \* [rack]((http://rack.rubyforge.org/doc/SPEC.html))是一个用来相应请求的WebServer。 \* [rack]((http://rack.rubyforge.org/doc/SPEC.html))发送过来的请求CCNG采用[sinatra]((ttp://www.sinatrarb.com/intro.html))框架处理

* * *

HTTP请求处理框架
==========

启动的配置以及初始化完毕后，ccng就在thin\_server上以一个rack app的形式运行，事件响应则采用sinatra的架构。

Controller类
-----------

**VCAP::CloudController::Controller**继承于Sinatra::Base，是处理所有http请求的入口。 

一开始，Controller这个类中定义了`before do....end`，这个函数会在第一次次接收到请求的时候执行，主要进行用户请求身份验证操作，判断用户是否合法，如果用户验证失败的话，会返回错误信息码。如果成功验证通过，那么会调用VCAP::CloudController::SecurityContext的set方法设置当前登录用户信息，供后面的陆续操作进行权限判断依据。 

接下来，http请求根据不同的内容被不同的路由规则引导处理。从程序代码的角度看，这些路由主要被分为自定义路由和默认路由两大块。 

通俗的讲： \* 一部分是直接定义在＊.rb文件中，比如 `lib/cloud_controller/legacy_api/*.rb` 下的所有请求都是直接在脚本中定义get或者post请求处理函数，这种是直观就可以看见的 \* 还有一部分，是通过回调`lib/cloud_controller/rest_controller.rb`中的self.rest\_controller这个方法，这个方法里面有一个define\_routes，来定义get，post，delete，put请求，并通过调用api的dispatch函数来进行选择具体的处理函数.

### 自定义路由

自定义路由即文件被载入(require)到Controller模块时，就是rack框架可识别的路由信息。这样自定义的路由信息都被写成类似如下的格式：(以lib/cloud\_controller/api/app\_bits.rb为例)

~~~ruby
module VCAP::CloudController
  rest\_controller :AppBits do
    ......
  end
end
~~~

这其中，起到关键作用的就是rest\_contoller这个宏定义方法，在require文件app\_bits.rb时（或者说执行到这里时），会将其后的内容展开。rest\_controller定义在lib/cloud\_controller/rest\_controller.rb中

~~~ruby
def self.rest_controller(name, &blk)
    # Class.new：生成一个无名的superclass的子类. 若superclass不存在则生成Object的子类.
    # 这里的superclass应该是RestController::ModelController, Base的子类，include了routes
    klass = Class.new RestController::ModelController
    # const_set：在模块中，定义一个名为name且值为value的常数后返回value
    self.const_set name, klass
    # class_eval：添加方法
    # class_eval若带块的话，会把新生成的类传给块参数，然后在类的context中执行该块。
    klass.class_eval &blk

    # 执行了disable_default_routes的类都有自己定义的route
    if klass.default_routes?
    # 否则绝大多数类都有attribute之类的语句
      klass.class_eval do
        define_messages
        define_routes
      end
    end
  end
~~~

对于app\_bits.rb，参数name就是:AppBits，blk就是do...end部分的内容。首先动态生成一个类，然后执行klass.class\_eval将blk中的方法添加为自己的方法。添加方法的同时会执行blk。 

对于app\_bits.rb来说，通过def...end定义的方法不会被执行，如upload、download等。而独立于方法定义之外的是语句可执行的。如

~~~ruby
disable\_default\_routes
put "#{path\_id}/bits", :upload
get "#{path\_id}/download", :download
~~~

等等。 

由此可以看出，这种路由配置实际上是一个方法的执行。以`put "#{path_id}/bits" :upload`为例，其方法是`put`，参数是`"#{path_id}/bits"`和`:upload`。这里的`put`方法也是在`lib/cloud_controller/rest_controller/routes.rb`中定义的

~~~ruby
[:post, :get, :put, :delete].each do |verb|
  define_method(verb) do |*args, &blk|
    (path, method) = *args
  define_route(verb, path, method, &blk)
  end
end
~~~

这里同时定义了4个方法，包括`put`。每一个方法的功能又是定义一个方法。对于`put`，该方法的名字为`put`，参数为`args`和`blk`，在`put "#{path_id}/bits", :upload`中，args是`"#{path_id}/bits", :upload`，blk为`nil`。进一步解析为，path=`"#{path_id}/bits"`，method=`"upload"`。然后执行`define_route("put", "#{path_id}/bits", "upload")`。 define\_route定义在`lib/cloud_controller/rest_controller/routes.rb`中

~~~ruby
def define_route(verb, path, method = nil, &blk)
    opts = {}
    opts[:consumes] = [:json] if [:put, :post].include?(verb)
    klass = self       
    controller.send(verb, path, opts) do |*args|        
      logger.debug "dispatch #{klass} #{verb} #{path}"     
      api = klass.new(@config, logger, env, request.params, request.body, self)
      if method
        # dispatch定义在Base中
        api.dispatch(method, *args)
      else
        blk.yield(api, *args)
      end
    end
  end
~~~

controller就定义在本文件中，`controller.class=Class.send`若是带块调用的话,也会把块原封不动地传给方法，于是就执行`put "#{path_id}/bits" do...end`，这时还不知道如何执行put，所以就展开放在这里，成了rack可读的形式。 send后面有一个参数\*args实际上是:guid，这是。Sinatra框架中，路由范式可以包括具名参数。例如

```ruby
get '/:path/:name' do |n, m|
    "path= #{n}; name= #{m}\n"
end
```

path展开到最底层，只有一个具名参数:guid，于是\*args就是:guid。 

参数env和request是`Sinatra::Base`的成员，其中env包括很多内容，例如`REQUEST_URI，REMOTE_ADDR`等等。因为当前类间接地继承了`Sinatra::Base`，所以可以使用这些变量。 

最后，路由`put "#{path_id}/bits", :upload`就被翻译为，遇到put请求，路径为`"#{path_id}/bits"`，则调用`upload`处理，同时将`:guid`传递给`upload`。

### 默认路由

app.rb及其他一些文件中，没有调用`disable_default_routes`，在执行到rest\_controller时，会调用define\_routes来定义自己的路由。通过define\_routes创建的路由具有一定的相似性，源文件只需给出路径名的一部分，其他部分在define\_routes中动态生成，因此称为默认路由。

```ruby
  def define_routes
    define_standard_routes
    define_to_many_routes
  end
```


`define_standard_routes`和`define_to_many_routes`定义在同名文件中。

```ruby
  def define_standard_routes
    [
      [:post,   path,    :create],
      [:get,    path,    :enumerate],
      [:get,    path_id, :read],
      [:put,    path_id, :update],
      [:delete, path_id, :delete]
    ].each do |verb, path, method|
      define_route(verb, path, method)
    end
  end
```


define\_standard\_routes生成一些相对固定的路由。当有path时，路由为path，方法为post和get；当有path\_id时，路由为path\_id，方法为get，put和delete。path定义在`lib/cloud_controller/rest_controller/base.rb`中

```ruby
  def path
    "#{ROUTE_PREFIX}/#{path_base}"
  end

  # Get and set the base of the path for the api endpoint.
  #
  # @param [String] base path to the api endpoint, e.g. the apps part of
  # /v2/apps/...
  #
  # @return [String] base path to the api endoint
  def path_base(base = nil)
    @path_base = base if base
    @path_base || class_basename.underscore.pluralize
  end

  # basename of the class
  #
  # @return [String] basename of the class
  def class_basename
    self.name.split("::").last
  end
```

当类的源文件中调用了path\_base时，则@path\_base为传入的参数；否则@path\_base为类名做如下变化的结果：将大写字母改为小写，然后将单数形式改为复数形式。例如，对于app.rb，path展开后的形式为/v2/apps，而对于app\_bits.rb，path展开后的形式也为/v2/apps。

 (为什么有/v2？ 因为`ROUTE_PREFIX = "/v2"`) 

path\_id定义在`lib/cloud_controller/rest_controller/model_controller.rb`中。

```ruby
  def path_id
    "#{path}/:guid"
  end
```


其中:guid为具名参数（关于具名参数'named parameters'的意思可参考博客[Just Do It: Learn Sinatra, Part One](http://www.sitepoint.com/just-do-it-learn-sinatra-i/)），根据请求内容而定。 至此，结合前面的分析，define\_standard\_routes的调用结果就显而易见了。例如app.rb，会生成下面这些路由

```ruby
post  "/v2/apps"  create
get  "/va/apps"  enumerate
get  "/v2/apps/:guid"  read
put  "/v2/apps/:guid"  update
delete  "/v2/apps/:guid"  delete
```

create等操作定义在lib/cloud\_controller/rest\_controller/model\_controller.rb中。 

define\_routes接下来还要调用define\_to\_many\_routes。

~~~ruby
def define_to_many_routes        
    to_many_relationships.each do |name, attr|
     
      get "#{path_id}/#{name}" do |api, id|
        api.dispatch(:enumerate_related, id, name)
      end

      put "#{path_id}/#{name}/:other_id" do |api, id, other_id|
        api.dispatch(:add_related, id, name, other_id)
      end

      delete "#{path_id}/#{name}/:other_id" do |api, id, other_id|
        api.dispatch(:remove_related, id, name, other_id)
      end
    end
  end
~~~

如果说define\_standard\_routes用来解析由path和path\_id定义的路由，那么define\_to\_many\_routes就是用来解析通过to\_many定义的路由。 以`lib/cloud_controller/api/space.rb`为例，其中有一段

```ruby
define_attributes do
  attribute  :name,            String
  to_one     :organization
  to_many    :developers
  to_many    :managers
  to_many    :auditors
  to_many    :apps
  to_many    :domains
  to_many    :service_instances
  to_many    :app_events
end
```


当启动时执行到这里时，就会调用define\_attributes方法。define\_attributes定义在lib/cloud\_controller/rest\_controller/model\_controller.rb中

```ruby
  def define_attributes(&blk)
    k = Class.new do
      include ControllerDSL
    end
    # instance_eval：在对象的context中计算字符串expr并返回结果
    k.new(self).instance_eval_r(&blk)
  end
```


于是会执行attribute，to\_one和to\_many这些方法。其中to\_many定义在`lib\cloud_controller\rest_controller\controller_dsl.rb`中。

```ruby
def to_many(name, opts = {})
  to_many_relationships[name] = ToManyAttribute.new(name, opts)
end
```


对于space.rb中的to\_many :apps来说，展开为

```ruby
to_many_relationships[":apps"] = ToManyAttribute.new(":apps", opts)
```

也就是新建一个类，添加到to\_many\_relationships中。根据上下文，to\_many\_relationships是一个Array或一个Hash结构。

执行到define\_to\_many\_routes时，each后的参数name就是apps。api和id是路由范式中的具名参数。 

以路由`"#{path_id}/#{name}"`为例，path\_id中包括两个变量，如果一个请求和该路由匹配，则第一个变量的实际值将赋给api，第二个变量的实际值赋给id。在另外两个路由范式中，:other\_id代表第三个变量。第一个变量是什么还不知道，但通过测试可知，:guid部分的内容会被赋给id，那么第一个具名参数一定在path\_id中。而api在运行过程中代表一个类的实例。例如，在启动时一个路由最终展开为`#{path_id} /v2/spaces/:guid`。当一个URL匹配该路由时，参数值如下

~~~ruby
api=#
id=81243e10-9b28-411e-9ccf-524ab15ce37c
#{name}=apps
~~~

通过define\_standard\_routes定义的路由，其具名参数放在\*args中，数量可以是一个。

处理请求的过程
-------

以客户启动一个APP为例。 

当客户端启动一个app时，发送的请求的格式如下（实际上客户端先检查app的状态，然后才决定是否发送put请求，这里假设app当前状态为stopped，因此客户端需要发送put请求） 

`PUT /v2/apps/:guid?stage_async=true` 

通过前面的分析可知，该请求和一个通过define\_standard\_routes定义的路由匹配，并被`lib/cloud_controller/rest_controller/model_controller.rb`的update方法处理。

~~~ruby
def update(id)
      logger.debug "update: #{id} #{request_attrs}"
      obj = find_id_and_validate_access(:update, id)
      json_msg = self.class::UpdateMessage.decode(body)
      @request_attrs = json_msg.extract(:stringify_keys => true)
      raise InvalidRequest unless request_attrs\
      
      before_modify(obj)
      
      model.db.transaction do
        obj.lock!
        obj.update_from_hash(request_attrs)
      end

      after_modify(obj)

      [HTTP::CREATED, serialization.render_json(self.class, obj, @opts)]
    end
~~~

其参数即是url中的guid。find\_id\_and\_validate\_access会调用`obj = model.find(:guid => id)`得到一个Sequel::Model对象并加以验证，如果合法则返回该对象。 

Sequel 是一个 Ruby 语言的对象映射框架（ORM），其子类Sequel::Model代表一种对象关系映射。而find\_id\_and\_validate\_access返回的Sequel::Model对象相当于数据库中的一行，:guid => id即是查询条件。 

obj.update\_from\_hash根据请求中body中的内容更新数据库。在操作前后分别调用了before\_modify和after\_modify方法，其中before\_modify定义如下

```ruby
def before_modify(app)
  app.stage_async = %w(1 true).include?(params["stage_async"])
end
```


根据put请求的内容，执行before\_modify后app.stage\_async会变为true。当update\_from\_hash这个事物成功后，会调用`lib/cloud_controller/models/app.rb`中的after\_commit，其中涉及到对该标志位的检查。

```ruby
  def after_commit
    super
    react_to_saved_changes(previous_changes || {})
  end

  def react_to_saved_changes(changes)
    if changes.has_key?(:state)
      react_to_state_change
    elsif changes.has_key?(:instances)
      delta = changes[:instances][1] - changes[:instances][0]
      react_to_instances_change(delta)
    end
  end
```


在after\_commit中调用react\_to\_saved\_changes，执行数据库更新后应该做的一些操作。react\_to\_saved\_changes首先检查changes这个参数，看其是否包含:state关键字。当app在数据库中的状态由stopped变为started时，changes会记录该变化，于是执行react\_to\_state\_change。

```ruby
  def react_to_state_change
    if started?
      stage_if_needed do
        DeaClient.start(self)
        send_droplet_updated_message
      end
    elsif stopped?
      DeaClient.stop(self)
      send_droplet_updated_message
    end
  end
```

react\_to\_state\_change判断状态是不是“STARTED”，结果为真，于是执行if后的语句。如果需要，则先执行stage操作，然后调用DeaClient.start和send\_droplet\_updated\_message。send\_droplet\_updated\_message向nats中发送droplet.updated消息，DeaClient.start开始启动一个app的过程。

~~~ruby
def start(app)
        start_instances_in_range(app, (0...app.instances))
        app.routes_changed = false
      end
      
      def start_instances_in_range(app, indices)
        start_instances_with_message(app, indices, {})
      end
      
      def start_instances_with_message(app, indices, message_override)
        msg = start_app_message(app)

        indices.each do |idx|
          msg[:index] = idx
          dea_id = dea_pool.find_dea(app.memory, app.stack.name, app.guid)
          if dea_id
            dea_publish("#{dea_id}.start", msg.merge(message_override))
            dea_pool.mark_app_staged(dea_id: dea_id, app_id: app.guid)
          else
            logger.error "no resources available #{msg}"
          end
        end
      end
~~~

start\_instances\_with\_message的第一个参数是app名字，第二个参数是实例个数，controller从dea\_pool中找到一个dea，通过nats向主题`#{dea_id}.start`发送消息。`#{dea_id}`对应的dea收到消息后，就会执行启动app的操作。

* * *

一些大家可能感兴趣的细节
============

cloud\_controller.yml文件
-----------------------

1.  CCNG统一使用8181作为统一对外开放的端口，原CC的端口为9022.
2.  info包含了如下默认信息，包括版本号、描述等。其中，support\_address默认为"http://support.cloudfoundry.com" description默认为 "VMware's Cloud Application Platform"
3.  db包含了和CCDB的最大连接数和超时时间的设置.
4.  staging字段包含了最大stage时间，默认为120s，如果上传的应用大，需要修改配置.

TO BE CONTINUED...
------------------

* * *

参考文献
====

*   [Cloud\_Controller\_NG github源码](https://github.com/cloudfoundry/cloud_controller_ng)
*   [Cloud\_Controller\_ng源码分析](http://blog.sina.com.cn/s/blog_c0cc23d90101otot.html)
*   [cloud controller v2源码解析](http://blog.csdn.net/tibelf/article/details/13295443)
* [Cloud Foundry中cloud\_controller\_ng架构简析](http://blog.csdn.net/shlazww/article/details/18887607)

  

