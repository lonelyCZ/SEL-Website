+++

id= "209"

title = "Cloud Foundry中DEA与warden通信完成应用端口监听"
description = "在Cloud Foundry v2版本中，DEA为一个用户应用运行的控制模块，而应用的真正运行都是依附于warden。更具体的来说，是DEA接收到Cloud Controller的请求；DEA发送请求给warden server；warden server创建warden container并将用户应用droplet等环境配置好；DEA发送应用启动请求至warden serve；最后warden container执行启动脚本启动应用。本文主要具体描述，DEA如何与warden交互，以保证最终用户的应用可以成功绑定某一个端口，实现用户应用对外提供服务。 "
tags= [ "dea" , "cloudfoundry" ]
date= "2014-12-02 16:55:44"
author = "孙宏亮"
banner= "img/blogs/209/cloud-foundry.jpg"
categories = [ "cloudfoundry" ]

+++

在Cloud Foundry v2版本中，DEA为一个用户应用运行的控制模块，而应用的真正运行都是依附于warden。更具体的来说，是DEA接收到Cloud Controller的请求；DEA发送请求给warden server；warden server创建warden container并将用户应用droplet等环境配置好；DEA发送应用启动请求至warden serve；最后warden container执行启动脚本启动应用。 

<!--more-->

本文主要具体描述，DEA如何与warden交互，以保证最终用户的应用可以成功绑定某一个端口，实现用户应用对外提供服务。 


DEA在执行启动一个应用的时候，主要做到以下这些部分：promise\_droplet, promise\_container, 其中这两个部分并发完成；promise\_extract\_droplet, promise\_exec\_hook\_script(“before\_start”), promise\_start等。代码如下：

```ruby
    [  
      promise_droplet,  
      promise_container  
    ].each(&:run).each(&:resolve)  
    
    [  
      promise_extract_droplet,  
      promise_exec_hook_script('before_start'),  
      promise_start  
    ].each(&:resolve)  
```

**promise\_droplet:**
---------------------

在这一个环节，DEA主要做的工作是将droplet下载本机，通过droplet\_uri,其中基本的路径在/config/dea.yml中，为base\_dir: /tmp/dea\_ng, 因此最终DEA下载到的droplet存放于DEA组件所在的宿主机上。

**promise\_container:**
-----------------------

该环节的工作主要完成创建一个warden container，随后可以为应用的运行提供一个合适的环境。promise\_container的源码实现如下：

```ruby
    def promise_container  
        Promise.new do |p|
        bind_mounts = [{'src_path' => droplet.droplet_dirname, 'dst_path' => droplet.droplet_dirname}]  
        with_network = true  
        container.create_container(  
            bind_mounts: bind_mounts + config['bind_mounts'],  
            limit_cpu: config['instance']['cpu_limit_shares'],  
            byte: disk_limit_in_bytes,  
            inode: config.instance_disk_inode_limit,  
            limit_memory: memory_limit_in_bytes,  
            setup_network: with_network)  
        attributes['warden_handle'] = container.handle  
        promise_setup_def create_container(params)  
        [:bind_mounts, :limit_cpu, :byte, :inode, :limit_memory, :setup_network].each do |param|  
        raise ArgumentError, "expecting #{param.to_s} parameter to create container" if params[param].nil?  
    end  
    
    with_em do  
        new_container_with_bind_mounts(params[:bind_mounts])  
        limit_cpu(params[:limit_cpu])  
        limit_disk(byte: params[:byte], inode: params[:inode])  
        limit_memory(params[:limit_memory])  
        setup_network if params[:setup_network]  
    end  
    endenvironment.resolve  
        p.deliver  
        end  
    end  
```

可以看到传入的参数主要有：

*   bind\_mounts：完成宿主机文件目录的路径mount到container内部；
*   limit\_cpu：用于限制container的CPU资源分配；
*   byte：磁盘限额；
*   innode：磁盘的innode的限制
*   limit\_memory：内存限额；
*   setup\_network：网络的配置项。

其中setup\_network一直设置为true。 在container.create\_container的方法实现中，有以下的方法，如下：

```ruby
    def create_container(params)  
        [:bind_mounts, :limit_cpu, :byte, :inode, :limit_memory, :setup_network].each do |param|  
          raise ArgumentError, "expecting #{param.to_s} parameter to createdef create_container(params)  
        [:bind_mounts, :limit_cpu, :byte, :inode, :limit_memory, :setup_network].each do |param|  
          raise ArgumentError, "expecting #{param.to_s} parameter to create container" if params[param].nil?  
        end  
    
      with_em do  
          new_container_with_bind_mounts(params[:bind_mounts])  
          limit_cpu(params[:limit_cpu])  
          limit_disk(byte: params[:byte], inode: params[:inode])  
          limit_memory(params[:limit_memory])  
          setup_network if params[:setup_network]  
      end  
    end container" if params[param].nil?  
      end  
    
      with_em do  
        new_container_with_bind_mounts(params[:bind_mounts])  
        limit_cpu(params[:limit_cpu])  
        limit_disk(byte: params[:byte], inode: params[:inode])  
        limit_memory(params[:limit_memory])  
        setup_network if params[:setup_network]  
      end  
    end  
```

主要关注一下setup\_network方法，如下：

```ruby
    def setup_network  
        request = ::Warden::Protocol::NetInRequest.new(handle: handle)  
        response = call(:app, request)  
        network_ports['host_port'] = response.host_port  
        network_ports['container_port'] = response.container_port  
    
        request = ::Warden::Protocol::NetInRequest.new(handle: handle)  
        response = call(:app, request)  
        network_ports['console_host_port'] = response.host_port  
        network_ports['console_container_port'] = response.container_port 
    
    end  
```

从代码中可以看到，在setup\_network中，主要完成了两次NetIn操作。对于NetIn操作，需要说明的是，完成的工作是将host主机上的端口映射到container内部的端口。换言之，将host\_ip:port1映射到container\_ip:port2，也就是说如果container在container\_ip上监听的是端口port2，则host机器外部的请求访问host机器，并且端口为port1的时候，host的内核网络栈，会将请求转发给container的port2端口，其中使用的协议为DNAT协议。 因此，在以上的代码中实现了两次NetIn操作，也就是说将container的两个端口映射到了host宿主机，第一个端口用于container内应用的正常占用端口，第二个端口是用来为应用的console功能做服务。虽然container也分配了第二个端口，但是在而后的应用启动等中，该console\_port都没有使用过，可见Cloud Foundry在这里只是预留了接口，但是没有真正利用起来。 以上主要描述了NetIn的功能，以下进入NetIn操作的源码实现。NetIn的源码实现，主要为warden server的部分。其中，是由DEA进程通过warden.sock和warden server建立通信，随后DEA进程发送NetIn请求给warden server，warden server最终处理该请求。 现在进入warden范畴，研究warden如何接收请求，并实现端口的映射。 在warden/lib/warden/server.rb中，大部分代码都是为了完成warden server的运行，在run！方法中，可以看到warden server另外还启动了一个unix domain server，代码如下：

```ruby
    server = ::EM.start_unix_domain_server(unix_domain_path, ClientConnection)  
```

也就是说，warden server会通过整个unix domain server接收从DEA进程发送来的关于warden container的一系列请求，在ClientConnection类中定义，关于这部分请求如何处理的方法。 当unix domain server中ClientConnection类通过receive\_data（data）方法来实现接收请求，代码如下：

```ruby
    def receive_data(data)  
      @buffer << data  
      @buffer.each_request do |request|  
        begin  
            receive_request(request)  
        rescue => e  
            close_connection_after_writing  
            logger.warn("Disconnected client after error")  
            logger.log_exception(e)  
        end  
      end  
    end  
```

从代码中可以看到，当buffer中有请求的时候，通过receive\_request（request）方法来进一步提取请求。再进入receive\_request(request)方法中，可以看到通过process（request）来处理请求。 接着进入真正请求处理的部分，也就时process（request）的实现：

```ruby
    def process(request)  
        case request  
        when Protocol::PingRequest  
          response = request.create_response  
          send_response(response)  
    
        when Protocol::ListRequest  
          response = request.create_response  
          response.handles = Server.container_klass.registry.keys.map(&:to_s)  
          send_response(response)  
    
        when Protocol::EchoRequest  
          response = request.create_response  
          response.message = request.message  
          send_response(response)  
    
        when Protocol::CreateRequest  
          container = Server.container_klass.new  
          container.register_connection(self)  
          response = container.dispatch(request)  
          send_response(response)  
    
        else  
          if request.respond_to?(:handle)  
            container = find_container(request.handle)  
            process_container_request(request, container)  
          else  
            raise WardenError.new("Unknown request: #{request.class.name.split("::").last}")  
          end  
        end  
    rescue WardenError => e  
        send_error(e)  
    rescue => e  
        logger.log_exception(e)  
        send_error(e)  
    end  
```

可见，在warden server中，请求类型可以简单分为5种：PingRequest, ListRequest, EchoRequest, CreateRequest和其他请求，像NetIn请求则属于其他请求中的一种，程序执行进入case语句块的else分支，也就是：

```ruby
if request.respond_to?(:handle)  
  container = find_container(request.handle)  
  process_container_request(request, container)  
else  
  raise WardenError.new("Unknown request: #{request.class.name.split("::").last}")  
end
```

代码清晰可见，warden server首先通过handle找到具体是给哪一个warden container发送请求，然后调用process\_container\_request(request, container)方法。进入process\_container\_request(request, container)方法可以看到：加入请求类型不为StopRequest以及StreamRequest，则进入case语句块的else分支，执行代码：

```ruby
response = container.dispatch(request)  
     send_response(response)  
```

可以看到，是调用了container.dispatch（request）方法才返回了response。 以下进入warden/lib/warden/container/base.rb文件中，该文件的dispatch方法主要实现了warden server接收到请求并预处理之后，如何分发执行具体的请求，代码如下：

```ruby
    def dispatch(request, &blk)  
        klass_name = request.class.name.split("::").last  
        klass_name = klass_name.gsub(/Request$/, "")  
        klass_name = klass_name.gsub(/(.)([A-Z])/) { |m| "#{m[0]}_#{m[1]}" }  
        klass_name = klass_name.downcase  
    
        response = request.create_response  
    
        t1 = Time.now  
    
        before_method = "before_%s" % klass_name  
        hook(before_method, request, response)  
        emit(before_method.to_sym)  
    
        around_method = "around_%s" % klass_name  
        hook(around_method, request, response) do  
            do_method = "do_%s" % klass_name  
        send(do_method, request, response, &blk)  
        end  
    
        after_method = "after_%s" % klass_name  
        emit(after_method.to_sym)  
        hook(after_method, request, response)  
    
        t2 = Time.ndef dispatch(request, &blk)  
        klass_name = request.class.name.split("::").last  
        klass_name = klass_name.gsub(/Request$/, "")  
        klass_name = klass_name.gsub(/(.)([A-Z])/) { |m| "#{m[0]}_#{m[1]}" }  
        klass_name = klass_name.downcase  
    
        response = request.create_response  
    
        t1 = Time.now  
    
        before_method = "before_%s" % klass_name  
        hook(before_method, request, response)  
        emit(before_method.to_sym)  
    
        around_method = "around_%s" % klass_name  
        hook(around_method, request, response) do  
            do_method = "do_%s" % klass_name  
            send(do_method, request, response, &blk)  
        end  
    
        after_method = "after_%s" % klass_name  
        emit(after_method.to_sym)  
        hook(after_method, request, response)  
    
        t2 = Time.now  
    
        logger.info("%s (took %.6f)" % [klass_name, t2 - t1],  
            :request => request.to_hash,  
            :response => response.to_hash)  
    
        response  
     endow  
    
        logger.info("%s (took %.6f)" % [klass_name, t2 - t1],  
            :request => request.to_hash,  
            :response => response.to_hash)  
    
        response  
    end  
```

首先提取出请求的类型名，如果是NetIn请求的话，提取出来的请求的类型名称为net\_in，随后构建出方法明do\_method，也就是do\_net\_in，接着就通过send(do\_method, request, response, &blk)方法，将参数发送给do\_net\_in方法。 

现在就是进入warden/lib/warden/container/features/net.rb文件中, do\_net\_in方法实现如下：

```ruby
    def do_net_in(request, response)  
        if request.host_port.nil?  
          host_port = self.class.port_pool.acquire  
    
          #Use same port on the container side as the host side if unspecified  
          container_port = request.container_port || host_port  
    
          # Port may be re-used after this container has been destroyed  
          @resources["ports"] << host_port  
          @acquired["ports"] << host_port  
        else  
          host_port = request.host_port  
          container_port = request.container_port || host_port  
        end  
    
        _net_in(host_port, container_port)  
    
        @resources["net_in"] ||= []  
        @resources["net_in"] << [host_port, container_port]  
    
        response.host_port      = host_port  
        response.container_port = container_port  
    rescue WardenError  
      self.class.port_pool.release(host_port) unless request.host_port  
      raise  
    end  
```

可见，如果请求端口没有指定的话，那么就使用代码host\_port = self.class.port\_pool.acquire来获取端口号，默认情况下，container端口号与host端口号保持一致，有了这两个端口号之后，执行代码\_net\_in(host\_port, container\_port)， 真正实现端口映射，如下：

```ruby
    def _net_in(host_port, container_port)  
      sh File.join(container_path, "net.sh"), "in", :env => {  
        "HOST_PORT"      => host_port,  
        "CONTAINER_PORT" => container_port,  
      }  
    end  
```

可以清晰的看到是，使用了容器内部的net.sh脚本来实现端口映射。现在进入warden/root/linux/skeleton/net.sh脚本，进入参数为in的执行部分：

```ruby
    "in")  
        if [ -z "${HOST_PORT:-}" ]; then  
            echo "Please specify HOST_PORT..." 1>&2  
            exit 1  
        fi  
        if [ -z "${CONTAINER_PORT:-}" ]; then  
          echo "Please specify CONTAINER_PORT..." 1>&2  
          exit 1  
        fi  
        iptables -t nat -A ${nat_instance_chain} \  
          --protocol tcp \  
          --destination "${external_ip}" \  
          --destination-port "${HOST_PORT}" \  
          --jump DNAT \  
          --to-destination "${network_container_ip}:${CONTAINER_PORT}"  
        ;;  
```

可见，该脚本这部分的功能是在host主机创建一条DNAT规则，使得host主机上所有HOST\_PORT端口上的网络请求，都转发至network\_container\_ip:CONTAINER\_PORT上，也就是完成了目标地址IP转变。

**promise\_extract\_droplet:**
------------------------------

该环节主要完成的是让container运行脚本，使得container容器将位于host主机的droplet文件，解压至container内部，代码内容如下：

```ruby
    def promise_extract_droplet  
        Promise.new do |p|  
          script = "cd /home/vcap/ && tar zxf #{droplet.droplet_path}"  
          container.run_script(:app, script)  
          p.deliver  
        end  
    end  
```


**promise\_exec\_hook\_script('before\_start')：**
-------------------------------------------------

这部分主要完成的功能是让容器运行名为before\_start的脚本，在老的版本中，该部分的设置默认为空。

**promise\_start:**
-------------------

这部分主要完成的是应用的启动。其中创建容器的时候，返回的端口号，会被DEA保存。最终，当DEA启动应用的时候，由于DEA会将端口号作为参数传递给应用程序的启动脚本，因此当应用启动时会自动去监听已经为它设置的端口，即完成端口监听。 

**转载请注明出处。** 
----------

这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。希望本文能够对接触Cloud Foundry v2中warden模块以及应用端口映射的人有些帮助，如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 