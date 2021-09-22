+++

id= "14"

title = "kubeedge源码分析系列（五）：DeviceTwin模块详解"
description = "本篇主要从DeviceTwin组件的源码出发，剖析DeviceTwin模块的内部实现，同时也对其四个子模块（membership、communication、device和twin）的具体执行逻辑进行具体分析。"
tags= [ "kubeedge"]
date= "2021-01-25 10:05:11"
author = "毛金勇"
banner= "img/blogs/14/devicetwin设计原理.png"
categories = [ "kubeedge" ]

+++



本篇主要从DeviceTwin组件的源码出发，剖析DeviceTwin模块的内部实现，同时也对其四个子模块（membership、communication、device和twin）的具体执行逻辑进行具体分析。

<!--more-->



## 1. 概述

首先，明确DeviceTwin模块在KubeEdge架构中所处的位置，如下图所示：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1611580230/sel/kubeedge-devicetwin-1_nge9uz.png" alt="kubeedge设备管理链路" style="zoom:70%;" />
</center>

其中，黄色部分是KubEedge设备管理链路相关的组件。

DeviceTwin组件主要负责设备相关的处理工作，比如同步设备的收集的实时数据、绑定设备与边缘节点的关系，等等。其内部又分为四个子模块，分别是membership、communication、device和twin。如下图所示：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1611580230/sel/kubeedge-devicetwin-2_zdqifk.png" alt="devicetwin设计原理" style="zoom:70%;" />
</center>

在进一步理解DeviceTwin之前，这里需要读者先明确的几个概念，即：

* **设备属性(device attribute/property)**：设备属性可以理解为设备的元数据（或称之为静态属性），负责描述设备的详细信息，定义好之后一般是不会变的。比如说温度计设备，它的作用是负责采集环境温度。显然，这种设备一般会有一个名为“temperature”的属性，该属性的类型也许是“float”类型，表示温度的数据类型，等等。
* **设备状态(device status)**：设备状态指的是设备是否在线，一般有“online”、“offline”和“unknown”3种定义。
* **设备孪生(device twin)**：设备孪生则是设备的动态属性，表示具体设备的专有实时数据，例如灯的开/关状态、温度计真实采集到的温度值，等等。在设备孪生（twin）中，进一步定义了“desired value(期望值)”和“reported value(真实值)”。其中，“desired value”指的是控制面希望设备达到的状态，比如，用户远程打开灯，由用户发出的指令即属于期望值；而“reported value”指的是设备上报给控制面的真实值，比如，温度计采集的温度值。需要注意的是，并不是每种设备都必须存在“desired value”，但一般都会有“reported value”。对于灯这类读写（readwrite）设备而言，我们既可以读取其上报的真实值（开或关），也可以控制它的状态（开或关）；而对于温度计这类只读（readonly）设备而言，我们只能读取其上报的真实值，而无需设置其期望值。

明确以上3个概念是理解KubeEdge中Device API设计的核心，也有助于理解DeviceTwin组件的源码。

总的来说，DeviceTwin组件负责沟通协调四个子模块的工作，具体包含4个方面，即：1）同步设备数据；2）注册并启动子模块；3）根据消息类型分别向子模块进行消息分发；4）对子模块进行健康检查。更具体的，四个子模块的作用分别如下：

* **Membership Module**：该模块主要负责绑定新加入的设备与指定的边缘节点（其实就是NodeSelector的体现）。比如温度传感器关联在边缘节点node-A上，蓝牙音箱关联在了节点node-B上，如果云端要控制蓝牙音箱，那就要把数据准确的推到node-B上。
* **Twin Module**：该模块主要负责所有设备孪生相关的操作。比如，设备孪生更新（device twin update）、设备孪生获取（device twin get）和设备孪生同步至云端（device twin sync-to-cloud）。
* **Communication Module**：该模块主要负责各个子模块之间的通信。
* **Device Module**：该模块主要负责执行设备相关的操作，比如处理设备状态（device status）更新和设备属性（device attribute）更新。

（PS. 如果不搞清楚“设备属性”、“设备状态”和“设备孪生”的概念，那么初学者很容易混淆Twin Module和Device Module这两个子模块的功能职责。事实上，我不太明白如此设计的原因是什么，总觉得这部分的设计稍有欠妥）

此外，DeviceTwin模块会在本地数据库（SQLite）中创建3张表，即Device Table、Device Attribute Table和Device Twin Table，分别记录设备的基本信息、设备的属性信息和设备孪生信息。表的具体字段设计详见[这里](https://github.com/kubeedge/kubeedge/blob/master/docs/components/edge/devicetwin.md#tables)。



## 2. 源码分析

本节从源码出发，剖析DeviceTwin模块的内部实现。为了避免篇幅冗长，在不影响说明的前提下，会适当删减代码。

为了方便理解，读者需要提前了解：

* 从前一小节我们已经知道，DeviceTwin模块会在本地数据库中创建3张表，即Device Table、Device Attribute Table和Device Twin Table，分别记录设备的基本信息、设备的属性信息和设备孪生信息。这里需要注意，Device Table中存储的Device并不是云端基于Kubernetes CRD机制定义的[Device API](https://github.com/kubeedge/kubeedge/blob/master/cloud/pkg/apis/devices/v1alpha2/device_instance_types.go#L360)，而是重新定义了一个名字同为[Device](https://github.com/kubeedge/kubeedge/blob/master/edge/pkg/devicetwin/dttype/types.go#L16)的结构体，用以存储标准Device API中的**部分信息**。

  当前，CloudHub与EdgeHub中传递的设备并不是完整的Device API，而是另外重新定义了一个同名的结构体，用于云边的设备信息同步。具体如下所示：

  （1）基于Kubernetes CRD机制定义的Device API，位于cloud/pkg/apis/devices/v1alpha2/device_instance_types.go：

  ```go
  // Device is the Schema for the devices API
  // +k8s:openapi-gen=true
  type Device struct {
  	metav1.TypeMeta   `json:",inline"`
  	metav1.ObjectMeta `json:"metadata,omitempty"`
  
  	Spec   DeviceSpec   `json:"spec,omitempty"`
  	Status DeviceStatus `json:"status,omitempty"`
  }
  // DeviceSpec represents a single device instance. It is an instantation of a device model.
  type DeviceSpec struct {
  	// Required: DeviceModelRef is reference to the device model used as a template
  	// to create the device instance.
  	DeviceModelRef *v1.LocalObjectReference `json:"deviceModelRef,omitempty"`
  	// Required: The protocol configuration used to connect to the device.
  	Protocol ProtocolConfig `json:"protocol,omitempty"`
  	// List of property visitors which describe how to access the device properties.
  	// PropertyVisitors must unique by propertyVisitor.propertyName.
  	// +optional
  	PropertyVisitors []DevicePropertyVisitor `json:"propertyVisitors,omitempty"`
  	// Data section describe a list of time-series properties which should be processed
  	// on edge node.
  	// +optional
  	Data DeviceData `json:"data,omitempty"`
  	// NodeSelector indicates the binding preferences between devices and nodes.
  	// Refer to k8s.io/kubernetes/pkg/apis/core NodeSelector for more details
  	// +optional
  	NodeSelector *v1.NodeSelector `json:"nodeSelector,omitempty"`
  }
  // DeviceStatus reports the device state and the desired/reported values of twin attributes.
  type DeviceStatus struct {
  	// A list of device twins containing desired/reported desired/reported values of twin properties..
  	// Optional: A passive device won't have twin properties and this list could be empty.
  	// +optional
  	Twins []Twin `json:"twins,omitempty"`
  }
  ```

  （2）用于云边消息同步的Device结构体，分别位于cloud/pkg/devicecontroller/types/device.go和edge/pkg/devicetwin/dttype/types.go：

  ```go
  //Device the struct of device
  type Device struct {
  	ID          string              `json:"id,omitempty"`
  	Name        string              `json:"name,omitempty"`
  	Description string              `json:"description,omitempty"`
  	State       string              `json:"state,omitempty"`
  	LastOnline  string              `json:"last_online,omitempty"`
  	Attributes  map[string]*MsgAttr `json:"attributes,omitempty"`
  	Twin        map[string]*MsgTwin `json:"twin,omitempty"`
  }
  //MsgAttr the struct of device attr
  type MsgAttr struct {
  	Value    string        `json:"value"`
  	Optional *bool         `json:"optional,omitempty"`
  	Metadata *TypeMetadata `json:"metadata,omitempty"`
  }
  //MsgTwin the struct of device twin
  type MsgTwin struct {
  	Expected        *TwinValue    `json:"expected,omitempty"`
  	Actual          *TwinValue    `json:"actual,omitempty"`
  	Optional        *bool         `json:"optional,omitempty"`
  	Metadata        *TypeMetadata `json:"metadata,omitempty"`
  	ExpectedVersion *TwinVersion  `json:"expected_version,omitempty"`
  	ActualVersion   *TwinVersion  `json:"actual_version,omitempty"`
  }
  ```

* 对于边缘侧存储的Device Table（对应`Device`结构体）、Device Attribute Table（对应`MsgAttr`结构体）和Device Twin Table（对应`MsgTwin`结构体），可以理解为，它们将Device CRD中的字段拆开并分别存储。目前，对Device这一自定义资源的处理方式与对内置资源（比如Pod、Service）的处理方式不同，后者将一个完整的API对象存储在名为[`Meta`](https://github.com/kubeedge/kubeedge/blob/master/edge/pkg/metamanager/dao/meta.go#L17)的表中。（至于为何这样设计，而不统一起来，原因不得而知）

* 另外需要指出的是，在当前的Device CRD设计中，并不包含`State`（表示设备的状态，即“online”、“offline”和“unknown”）和`LastOnline`（设备的最后在线时间戳）这两个字段；而在云与边的设备信息同步中（即`Device`结构体）却又包含了这两个字段。后期是否要考虑把这两个字段加入到Device CRD中呢？（此处存疑）

* 云边通信（即CloudHub与EdgeHub之间）的消息结构体是`model.Message`，如下所示：

  ```go
  // Message struct
  type Message struct {
  	Header  MessageHeader `json:"header"`
  	Router  MessageRoute  `json:"route,omitempty"`
  	Content interface{}   `json:"content"`
  }
  ```

  但在DeviceTwin模块中，传输的消息结构体为`DTMessage`，如下所示：

  ```go
  //DTMessage the struct of message for communicating between cloud and edge
  type DTMessage struct {
  	Msg      *model.Message
  	Identity string //表示节点ID
  	Action   string //代表该条消息执行的动作，根据不同的Action将消息转发至对应的DeviceTwin子模块
  	Type     string //这个貌似也没啥用
  }
  ```



### 2.1. 代码入口

代码入口：edge/pkg/devicetwin/devicetwin.go

```go
// Start run the module
func (dt *DeviceTwin) Start() {
	dtContexts, _ := dtcontext.InitDTContext()
	dt.DTContexts = dtContexts
  err := SyncSqlite(dt.DTContexts) //将存储在本地数据库中的device信息载入DTContexts
	...
	dt.runDeviceTwin() //运行各个子模块
}
```

可以看到，DeviceTwin模块的启动主要做了两件事情，一是将存储在本地数据库（sqlite）中的设备信息载入至`DTContexts`中；二是运行各个子模块。

（1）`SyncSqlite()`负责同步设备元数据，即从本地数据库中查询所有设备，并将这些设备载入至`DTContext`中。`DTContext`是一个关键的数据结构，负责存储在DeviceTwin模块中的上下文信息。具体在之后的讨论涉及。

```go
// SyncSqlite sync sqlite
func SyncSqlite(context *dtcontext.DTContext) error {
	...
	rows, queryErr := dtclient.QueryDeviceAll() //从本地数据库中查询所有设备
	...
	for _, device := range *rows {
		err := SyncDeviceFromSqlite(context, device.ID) //将设备保存至DTContext中
		...
	}
	return nil
}
```

（2）`runDeviceTwin()`则包含3个方面的内容，分别是 1）注册并启动子模块；2）向子模块分发消息；3）健康检查。

```go
func (dt *DeviceTwin) runDeviceTwin() {
	moduleNames := []string{dtcommon.MemModule, dtcommon.TwinModule, dtcommon.DeviceModule, dtcommon.CommModule}
  //1.注册并启动子模块
	for _, v := range moduleNames {
		dt.RegisterDTModule(v)
		go dt.DTModules[v].Start()
	}
	go func() {
		for {
			...
			if msg, ok := beehiveContext.Receive("twin"); ok == nil {
				klog.Info("DeviceTwin receive msg")
        //2.消息转发
				err := dt.distributeMsg(msg)
				if err != nil {
					klog.Warningf("distributeMsg failed: %v", err)
				}
			}
		}
	}()

  //3.健康检查，如有必要则会重启子模块
	for {
		select {
		case <-time.After((time.Duration)(60) * time.Second):
			//range to check whether has bug
			for dtmName := range dt.DTModules {
				health, ok := dt.DTContexts.ModulesHealth.Load(dtmName)
				if ok {
					now := time.Now().Unix()
					if now-health.(int64) > 60*2 {
						klog.Infof("%s health %v is old, and begin restart", dtmName, health)
						go dt.DTModules[dtmName].Start()
					}
				}
			}
			for _, v := range dt.HeartBeatToModule {
				v <- "ping"
			}
		case <-beehiveContext.Done():
			for _, v := range dt.HeartBeatToModule {
				v <- "stop"
			}
			klog.Warning("Stop DeviceTwin ModulesHealth load loop")
			return
		}
	}
}
```



### 2.2. 消息转发

函数入口：edge/pkg/devicetwin/process.go

```go
//distributeMsg distribute message to diff module
func (dt *DeviceTwin) distributeMsg(m interface{}) error {
   msg, ok := m.(model.Message)
   ...
   message := dttype.DTMessage{Msg: &msg}
   //如果msg的parentID不为空，说明该消息是一个确认消息(response message)
   //如果是云端下发的消息，parentID均为空
   if message.Msg.GetParentID() != "" {
      klog.Infof("Send msg to the %s module in twin", dtcommon.CommModule)
      confirmMsg := dttype.DTMessage{Msg: model.NewMessage(message.Msg.GetParentID()), 
                                     Action: dtcommon.Confirm}
      if err := dt.DTContexts.CommTo(dtcommon.CommModule, &confirmMsg); err != nil {
         return err
      }
   }
   //根据msg的source和resource type进行分类
   //并填充message
   if !classifyMsg(&message) { //关键~
      return errors.New("Not found action")
   }
   if ActionModuleMap == nil {
      initActionModuleMap()
   }
	
   //根据消息的Action，将消息转发至对应的子模块中
   if moduleName, exist := ActionModuleMap[message.Action]; exist {
   		...	
      if err := dt.DTContexts.CommTo(moduleName, &message); err != nil {
         return err
      }
   } 
   ...
   return nil
}
```

具体来看`classifyMsg()`这一方法，搞懂它至关重要。

首先，会根据消息源(`msg.Router.Source`)判断进入哪个分支。比如，设备的信息是由云端的`devicecontroller`组件发送过来的，因此`msg.Router.Resource`就等于`devicecontroller`。紧接着，会根据消息体所传送的资源类型(`msg.Router.Resource`)进一步分类，从而确定对应的`Action`是什么，最后根据不同的`Action`将消息转发至不同的子模块中。

```go
func classifyMsg(message *dttype.DTMessage) bool {
	if EventActionMap == nil {
		initEventActionMap()
	}
	var identity string
	var action string
	msgSource := message.Msg.GetSource()
	if strings.Compare(msgSource, "bus") == 0 { //如果消息源来自”eventbus“
		...
	}
  //如果消息源来自"edgemgr"或"devicecontroller"
  else if (strings.Compare(msgSource, "edgemgr") == 0) || 
            (strings.Compare(msgSource, "devicecontroller") == 0) {
		...
    //根据message.Msg.Router.Resource进行分类
		if strings.Contains(message.Msg.Router.Resource, "membership/detail") {
			message.Action = dtcommon.MemDetailResult
			return true
		} else if strings.Contains(message.Msg.Router.Resource, "membership") { // 标记1
			message.Action = dtcommon.MemUpdated
			return true
		} else if strings.Contains(message.Msg.Router.Resource, "twin/cloud_updated") {
			message.Action = dtcommon.TwinCloudSync
			resources := strings.Split(message.Msg.Router.Resource, "/")
			message.Identity = resources[1] //nodeID
			return true
		} else if strings.Contains(message.Msg.Router.Operation, "updated") {
			resources := strings.Split(message.Msg.Router.Resource, "/")
			if len(resources) == 2 && strings.Compare(resources[0], "device") == 0 {
				message.Action = dtcommon.DeviceUpdated
				message.Identity = resources[1]
			}
			return true
		}
		return false
	} else if strings.Compare(msgSource, "edgehub") == 0 {
		...
}
```

我们以云端DeviceController与边缘端通信作为示例，加以说明。（DeviceController分析详见xx）

假设云端[新增一个设备](https://github.com/kubeedge/kubeedge/blob/master/cloud/pkg/devicecontroller/controller/downstream.go#L388)，就会构造这样一条消息，并发送至边缘端。

```go
//cloud/pkg/devicecontroller/controller/downstream.go -> deviceAdded()
msg := model.NewMessage("")
resource, err := messagelayer.BuildResource(device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions[0].Values[0], "membership", "")
msg.BuildRouter(modules.DeviceControllerModuleName, constants.GroupTwin, resource, model.UpdateOperation)
msg.Content = xxx //这里存放消息传递的有效数据
```

其中，`device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions[0].Values[0]`表示在创建设备实例时通过`NodeSelector`所绑定的边缘节点，假设为”edgenode_1“，那么，就会构造出：

```
resource = "node/edgenode_1/membership"
```

而`msg.BuildRouter(modules.DeviceControllerModuleName, constants.GroupTwin, resource, model.UpdateOperation)`则定义了消息的路由信息，如下：

```go
msg.Router.Source = "devicecontroller"  //表示消息源
msg.Router.Group = "twin"               //表示消息发送的组别
msg.Router.Resource = "node/edgenode_1/membership" //表示消息的资源类型，也表示topic类型
msg.Router.Operation = "update"
```

基于此，消息的路由信息就很明确了。我们再回过头看上边的`classifyMsg()`方法，不难发现，这条消息会进入到「标记1」所处的分支，从而标记该条消息对应的`Action`是`dtcommon.MemUpdated`。由于在`ActionModuleMap`中已经定义了`Action`与子模块之间的映射关系，因此，接下来就会根据这条消息的`Action`转发至对应的子模块，然后由那个子模块来执行具体的逻辑。



### 2.3. 子模块逻辑

本节重点分析四个子模块的具体执行逻辑。

#### membership

`membership`模块实际上就是负责管理边缘节点上的设备。当有新增设备或删除设备，都是由这一组件负责处理。

```go
//Start worker
func (mw MemWorker) Start() {
	initMemActionCallBack() //初始化DTMessage.Action与回调函数的映射关系
	for {
		select {
		case msg, ok := <-mw.ReceiverChan:
			if !ok {
				return
			}
			if dtMsg, isDTMessage := msg.(*dttype.DTMessage); isDTMessage {
        //根据Action，调用对应的callback函数
				if fn, exist := memActionCallBack[dtMsg.Action]; exist { 
					_, err := fn(mw.DTContexts, dtMsg.Identity, dtMsg.Msg)
					...
				} 
			}

		case v, ok := <-mw.HeartBeatChan:
			...
			if err := mw.DTContexts.HeartBeat(mw.Group, v); err != nil {
				return
			}
		}
	}
}
```

`membership`模块的启动逻辑非常简单。首先，它会初始化消息的`Action`与回调函数的映射关系，即`initMemActionCallBack()`；随后，**根据接收到的消息的`Action`调用不同的回调函数，也就是去执行具体的动作**。同时，还需要收发心跳信息。这里主要关注该模块可执行哪些动作：

* dealMembershipGet：该方法主要用于获取内存中(即`DTContext`)存储的设备信息。主要负责与`eventbus`组件的交互，当`eventbus`组件订阅了指定的topic后，就是通过该方法获取到设备信息的。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/membership-get_hokqyt.png" style="zoom:60%;" />
  </center>

* dealMembershipUpdate：该方法负责更新边缘节点的设备信息。比如云端控制面新增或删除了一个设备，会下发更新信息，最终就是通过这个方法来执行的。并且在更新完成后，会把最新的结果发送给`eventbus`。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/membership-update_trephr.png" alt="membership-update" style="zoom:60%;" />
  </center>

* dealMembershipDetail：待定。有点没看懂它的目的。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/membership-detail_kemjxs.png" alt="membership-detail" style="zoom:60%;" />
  </center>

这里需要再强调一下！前面已经提过，边缘端拿到的Device，并不是云端基于Kubernetes CRD机制定义的Device API。为了获取到设备基本信息，设备属性信息和设备孪生信息，又额外定义了许多的结构体用于存储这些信息，代码中有很多序列化与反序列的操作。不要被绕晕了。

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/devicecontroller_s2vsmm.png" alt="devicecontroller" style="zoom:13%;" />
</center>

#### twin

`twin`模块主要负责所有设备孪生相关的操作。比如，设备孪生更新（device twin update）、设备孪生获取（device twin get）和设备孪生同步至云端（device twin sync-to-cloud）。该模块的执行逻辑与`membership`模块的逻辑是一模一样的，我们主要看一下它支持执行哪些动作：

* dealTwinUpdate：该方法更新指定设备的设备孪生信息。更新信息可以来自云端（比如云端控制面想要改变等的开/关状态），通过`edgehub`接收到；也可以来自MQTT，通过`eventbus`接收到（`mapper`会在devicetwin update topic上发布信息）。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567385/sel/devicetwin-update_tpzrxp.png" alt="devicetwin-update" style="zoom:60%;" />
  </center>

* dealTwinGet：获取指定设备的设备孪生信息。该方法负责与`eventbus`交互。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567385/sel/devicetwin-get_lamrk3.png" alt="devicetwin-get" style="zoom:60%;" />
  </center>

* dealTwinSync：该方法负责将设备孪生信息同步至云端。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567387/sel/sync-to-cloud_bglfnr.png" alt="sync-to-cloud" style="zoom:60%;" />
  </center>



#### device

device 模块主要负责执行设备相关的操作，比如处理设备状态更新和设备属性更新。该模块可执行的动作有如下几个：

- dealDeviceAttrUpdate：更新设备的属性。设备属性的更新由云端发起。不过就目前(2021-01-04)来看，这个方法还没用起来，云端devicecontroller的downstream部分并没有看到更新设备属性的操作。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567385/sel/device-update_jydjbb.png" alt="device-update" style="zoom:60%;" />
  </center>

- dealDeviceStateUpdate：更新设备的状态（state）、最后一次在线时间（last online）。设备状态的更新是由`eventbus`主动发起的。更新后，会把更新结果同步至云端和边缘端的`eventbus`。

  <center>
  <img src="https://res.cloudinary.com/rachel725/image/upload/v1611567385/sel/device-state-update_akz55j.png" alt="device-state-updater" style="zoom:60%;" />
  </center>
  
  

#### communication

模块主要负责DeviceTwin和其他组件的通信。该模块可执行的动作有如下几个：

* dealSendToCloud：用于向云端发送数据。在该函数内部，首先确保边缘侧与云端的连接是否正常，随后向edgehub发送消息（基于beehive框架），再由edgehub向云端发送。
* dealSendToEdge：内部调用`beehiveContext.Send()`方法，直接向`eventbus`组件发送消息。
* dealLifeCycle： 负责检查边缘与云端的状态是否正常连接，如果正常连接，则设置`DTContext.state`的状态为`connected`；反之则标志其为`unconnected` 。
* dealConfirm： 用于处理消息的确认。源码中我看到只有在[这里](https://github.com/kubeedge/kubeedge/blob/master/edge/pkg/devicetwin/process.go#L52)使用到了。



### 2.4. 总结

DeviceTwin模块源码整理如下：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/devicetwin_xhjwnk.png" alt="devicetwin" style="zoom:20%;" />
</center>
