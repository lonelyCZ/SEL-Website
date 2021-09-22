+++

id= "13"

title = "kubeedge源码分析系列（四）：DeviceController模块详解"
description = "DeviceController属于KubeEdge的云端组件，负责设备管理。KubeEdge利用Kubernetes提供的CRD机制，对真实的物理设备进行抽象，通过自定义一个名为`Device`的自定义资源（Custom Resource）来描述设备的元数据以及状态。而DeviceController，顾名思义，就是这一资源的的控制器，由它负责云边的设备信息同步。"
tags= [ "kubeedge"]
date= "2021-01-25 10:00:11"
author = "毛金勇"
banner= "img/blogs/13/devicecontroller.png"
categories = [ "kubeedge" ]

+++



本篇主要剖析了为自定义资源提供各种服务的控制器DeviceController的源码，分别从`upstream`和`downstream`两个独立的goroutine出发去追寻数据的流动过程。

<!--more-->



## 1. 概述

DeviceController属于KubeEdge的云端组件，负责设备管理。KubeEdge利用Kubernetes提供的[CRD机制](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)，对真实的物理设备进行抽象，通过自定义一个名为`Device`的自定义资源（Custom Resource）来描述设备的元数据以及状态。而DeviceController，顾名思义，就是这一资源的的控制器，由它负责云边的设备信息同步。在具体的实现中，DeviceController分为两个部分，会启动两个独立的goroutine，即`downstream`和`upstream`，其中`downstream`通过监听 Kubernetes API Server 将设备的状态更新由云端同步至边缘端；而`upstream`则负责订阅来自边缘端的消息，并将其同步至API Server中。

再具体分析这部分源码之前，需要明确两个概念，即`DeviceModel`和`DeviceInstance`。在KubeEdge中，DeviceController通过这两个概念对设备进行抽象。

* **DeviceModel**：描述了设备的属性（properties），定义了访问这些属性的方式（property visitor）。我们可以将`DeviceModel`理解为一组设备的模板。`DeviceModel`的具体设计详见[这里](https://github.com/kubeedge/kubeedge/blob/master/docs/proposals/device-crd.md#device-model-type-definition)。
* **DeviceInstance**：表示一个真实的设备实例。通过引用`DeviceModel`，创建一个设备实例。其中，`Device Spec`表示的设备的期望状态，而`Device Status`则表示设备的真实状态。`DeviceInstance`的具体设计详见[这里](https://github.com/kubeedge/kubeedge/blob/master/docs/proposals/device-crd.md#device-instance-type-definition)。



## 2. 源码分析

### 2.1. 代码入口

为了避免篇幅冗长，代码有部分省略。

`DownstreamController`的定义如下，它的作用是watch kubernetes api server and send change to edge。

```go
// DownstreamController watch kubernetes api server and send change to edge
type DownstreamController struct {
	kubeClient   *kubernetes.Clientset
	messageLayer messagelayer.MessageLayer

	deviceManager      *manager.DeviceManager
	deviceModelManager *manager.DeviceModelManager
	configMapManager   *manager.ConfigMapManager

	crdClient *rest.RESTClient
}
```

`UpstreamController`的定义如下，它的所用是subscribe messages from edge and sync to k8s api server。

```go
// UpstreamController subscribe messages from edge and sync to k8s api server
type UpstreamController struct {
	crdClient    *rest.RESTClient
	messageLayer messagelayer.MessageLayer
	// message channel
	deviceStatusChan chan model.Message

	// downstream controller to update device status in cache
	dc *DownstreamController
}
```

模块启动入口：cloud/pkg/devicecontroller/devicecontroller.go，DeviceController 主体逻辑如下，

```go
func (dc *DeviceController) Start() {
	downstream, err := controller.NewDownstreamController()
	...
	upstream, err := controller.NewUpstreamController(downstream)
	...
	downstream.Start()
	// wait for downstream controller to start and load deviceModels and devices
	// TODO think about sync
	time.Sleep(1 * time.Second)
	upstream.Start()
}
```



### 2.2. downstream

#### downstream.Start()

```go
// Start DownstreamController
func (dc *DownstreamController) Start() error {
	klog.Info("Start downstream devicecontroller")

	go dc.syncDeviceModel()

	// Wait for adding all device model
	// TODO need to think about sync
	time.Sleep(1 * time.Second)
	go dc.syncDevice()

	return nil
}
```

可以看到，`downstreamcontroller`的启动函数非常简单，即由两个独立的goroutine分别运行`syncDeviceModel()`和`syncDevice()`。由于`DeviceModel`必须要在`Device`实例之前先创建好，因此`syncDeviceModel()`和`syncDevice()`的启动顺序不能变。不过目前的实现中，只是简单的通过sleep 1秒钟来解决两者的同步问题，这是比较粗糙的做法，这里也标注了 TODO，后续肯定是需要改进的。



#### syncDeviceModel()

```go
// syncDeviceModel is used to get events from informer
func (dc *DownstreamController) syncDeviceModel() {
	for {
		select {
		case <-beehiveContext.Done():
			klog.Info("stop syncDeviceModel")
			return
		case e := <-dc.deviceModelManager.Events():
			deviceModel, ok := e.Object.(*v1alpha2.DeviceModel)
			...
			switch e.Type {
			case watch.Added:
				dc.deviceModelAdded(deviceModel)
			case watch.Deleted:
				dc.deviceModelDeleted(deviceModel)
			case watch.Modified:
				dc.deviceModelUpdated(deviceModel)
			default:
				klog.Warningf("deviceModel event type: %s unsupported", e.Type)
			}
		}
	}
}
```

这部分的代码还是非常清晰的，从`deviceModelManager`获取`deviceModel`相关的事件，根据事件类型触发`deviceModel`的添加、删除、更新等操作：

* deviceModelAdded：该方法只会在本地缓存中添加一条记录，并不会真正的创建`deviceModel`实例；
* deviceModelUpdated：该方法会更新缓存，并执行`updateAllConfigMaps()`，只不过后者还没有实现；
* deviceModelDeleted：该方法目前只会在本地缓存中删掉这个记录，TODO中说明应该要删除这个`deviceModel`关联的所有设备，只不过目前还没有实现。



#### syncDevice()

`syncDevice()`部分的逻辑与`syncDeviceModel()`一致，从`deviceManager`获取`device`相关的事件，并根据事件类型触发`device`的添加、删除、更新等操作：

* deviceAdded：该方法相比于`deviceModelAdded()`要复杂很多。代码如下：

```go
// deviceAdded creates a device, adds in deviceManagers map, send a message to edge node if node selector is present.
func (dc *DownstreamController) deviceAdded(device *v1alpha2.Device) {
	dc.deviceManager.Device.Store(device.Name, device) //缓存
	if len(device.Spec.NodeSelector.NodeSelectorTerms) != 0 && 
  	 len(device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions) != 0 && 
     len(device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions[0].Values) != 0 {
		dc.addToConfigMap(device) //根据device API创建configMap，相当于把配置写进去
		edgeDevice := createDevice(device) //根据device API创建types.Device
		
    msg := model.NewMessage("")
		resource, err := messagelayer.BuildResource(device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions[0].Values[0], "membership", "")
		...
		msg.BuildRouter(modules.DeviceControllerModuleName, constants.GroupTwin, resource, model.UpdateOperation)

		content := types.MembershipUpdate{AddDevices: []types.Device{ //消息的内容
			edgeDevice,
		}}
		content.EventID = uuid.NewV4().String()
		content.Timestamp = time.Now().UnixNano() / 1e6
		msg.Content = content

		err = dc.messageLayer.Send(*msg) //发送消息
		...
	}
}
```

首先，和`deviceModelAdded()`的逻辑一样，会在`deviceManager`中缓存一份。

然后，判断与该设备绑定的边缘节点是否为空，如果非空，就进入下一步。

随后调用`addToConfigMap()`创建一个configMap，configMap的作用非常重要，在边缘端mapper中，所有与设备相关的配置（比如说protocol、protocolVisitor等等）都是通过configMap获取的。因此，当在云端添加一个device时，就需要更新或创建对应的configMap。不过`addToConfigMap()`方法不会直接将configMap更新至边缘节点，只是在云端更新了，然后由edgecontroller watch到configMap的更新后，再同步到边缘节点。

接着调用`createDevice(device)`，这一步的作用是，由标准的[Device API](https://github.com/kubeedge/kubeedge/blob/master/cloud/pkg/apis/devices/v1alpha2/device_instance_types.go#L360)转成[Device](https://github.com/kubeedge/kubeedge/blob/master/cloud/pkg/devicecontroller/types/device.go#L4)，注意这两者的区别！前者是基于Kubernetes CRD机制定义的自定义资源（Custom Resource），是标准的Kubernetes API对象；后者用于云边通信（即cloudhub与edgehub之间的消息传输），也就是说，边缘端拿到的Device对象并不是标准的Device API。设备这一块的处理和其他内置资源（比如Pod、Service）不同，对于内置资源，都是把完整的API对象发往边缘端的；而设备的处理则定义很多其他的结构体（主要就是Device、MsgTwin、MsgAttr）用于云边的device数据传输。

最后就是创建消息msg，并将其发送至边缘端。注意，msg.Content 中存放的是 types.MembershipUpdate{}，里面是一个新增设备的列表。和edgecontroller不同的是，edgecontroller同步至边缘的消息中，content是Pod、ConfigMap等对象。

* deviceDeleted：删除设备与新增设备的逻辑基本一致，不再赘述。
* deviceUpdated：设备更新的逻辑也比较复杂，该方法代码如下：

```go
// deviceUpdated updates the map, check if device is actually updated.
// If nodeSelector is updated, call add device for newNode, deleteDevice for old Node.
// If twin is updated, send twin update message to edge
func (dc *DownstreamController) deviceUpdated(device *v1alpha2.Device) {
	value, ok := dc.deviceManager.Device.Load(device.Name)//从缓存中取出old device
	dc.deviceManager.Device.Store(device.Name, device)
	if ok {
		cachedDevice := value.(*v1alpha2.Device)
		if isDeviceUpdated(cachedDevice, device) {
			// if node selector updated delete from old node and create in new node
			if isNodeSelectorUpdated(cachedDevice.Spec.NodeSelector, device.Spec.NodeSelector) {
				dc.deviceAdded(device)
				deletedDevice := &v1alpha2.Device{ObjectMeta: cachedDevice.ObjectMeta,
					Spec:     cachedDevice.Spec,
					Status:   cachedDevice.Status,
					TypeMeta: device.TypeMeta,
				}
				dc.deviceDeleted(deletedDevice)
			} else {
				// update config map if spec, data or twins changed
				if isProtocolConfigUpdated(&cachedDevice.Spec.Protocol, &device.Spec.Protocol) ||
					isDeviceStatusUpdated(&cachedDevice.Status, &device.Status) ||
					isDeviceDataUpdated(&cachedDevice.Spec.Data, &device.Spec.Data) {
					dc.updateConfigMap(device)
				}
				// update twin properties
				if isDeviceStatusUpdated(&cachedDevice.Status, &device.Status) {
					// TODO: add an else if condition to check if DeviceModelReference has changed, if yes whether deviceModelReference exists
					twin := make(map[string]*types.MsgTwin)
					addUpdatedTwins(device.Status.Twins, twin, device.ResourceVersion)
					addDeletedTwins(cachedDevice.Status.Twins, device.Status.Twins, twin, device.ResourceVersion)
					msg := model.NewMessage("")

					resource, err := messagelayer.BuildResource(device.Spec.NodeSelector.NodeSelectorTerms[0].MatchExpressions[0].Values[0], "device/"+device.Name+"/twin/cloud_updated", "")
					if err != nil {
						klog.Warningf("Built message resource failed with error: %s", err)
						return
					}
					msg.BuildRouter(modules.DeviceControllerModuleName, constants.GroupTwin, resource, model.UpdateOperation)
					content := types.DeviceTwinUpdate{Twin: twin}
					content.EventID = uuid.NewV4().String()
					content.Timestamp = time.Now().UnixNano() / 1e6
					msg.Content = content

					err = dc.messageLayer.Send(*msg)
					...
				}
			}
		}
	} else {
		// If device not present in device map means it is not modified and added.
		dc.deviceAdded(device)
	}
}
```

首先从缓存中取出old device，然后通过`isDeviceUpdated(cachedDevice, device)`比较new device与old device是否发生了更新；

如果是与设备绑定的NodeSelector发生了变化，处理方式是删除old device并添加new device；

如果是设备的spec, data or twins发生了变化，则更新configMap。前面已经提过了，边缘的Mapper就是靠configMap来获取设备的所有信息的；

另外，如果是设备孪生（twin）发生了变化，要需要向边缘端同步一条消息，而消息的内容是 DeviceTwinUpdate{Twin: twin}。



### 2.3. upstream

#### upstream.Start()

```go
// Start UpstreamController
func (uc *UpstreamController) Start() error {
	klog.Info("Start upstream devicecontroller")

	uc.deviceStatusChan = make(chan model.Message, config.Config.Buffer.UpdateDeviceStatus)
	go uc.dispatchMessage()

	for i := 0; i < int(config.Config.Buffer.UpdateDeviceStatus); i++ {
		go uc.updateDeviceStatus()
	}
	return nil
}
```

upstream的启动函数也非常简单，只做两件事情，分发消息并更新Device API（即`Device.Status`部分）。



#### dispatchMessage()

```go
func (uc *UpstreamController) dispatchMessage() {
	for {
		select {
		case <-beehiveContext.Done():
			klog.Info("Stop dispatchMessage")
			return
		default:
		}
		msg, err := uc.messageLayer.Receive()
		...
		resourceType, err := messagelayer.GetResourceType(msg.GetResource())
		...
		switch resourceType {
		case constants.ResourceTypeTwinEdgeUpdated:
			uc.deviceStatusChan <- msg
		default:
			...
		}
	}
}
```

首先接收消息，然后根据消息的`resourceType`进行转发，也就是把边缘端`devicetwin`更新上来的消息转发至`deviceStatusChan`中。



#### updateDeviceStatus()

```go
func (uc *UpstreamController) updateDeviceStatus() {
	for {
		select {
		case <-beehiveContext.Done():
			klog.Info("Stop updateDeviceStatus")
			return
		case msg := <-uc.deviceStatusChan:
			...
			msgTwin, err := uc.unmarshalDeviceStatusMessage(msg)
			...
			deviceID, err := messagelayer.GetDeviceID(msg.GetResource())
			...
			device, ok := uc.dc.deviceManager.Device.Load(deviceID)
			...
			cacheDevice, ok := device.(*v1alpha2.Device)
			...
			deviceStatus := &DeviceStatus{Status: cacheDevice.Status}
			for twinName, twin := range msgTwin.Twin {
				for i, cacheTwin := range deviceStatus.Status.Twins {
					if twinName == cacheTwin.PropertyName && twin.Actual != nil && twin.Actual.Value != nil {
						reported := v1alpha2.TwinProperty{}
						reported.Value = *twin.Actual.Value
						reported.Metadata = make(map[string]string)
						if twin.Actual.Metadata != nil {
							reported.Metadata["timestamp"] = strconv.FormatInt(twin.Actual.Metadata.Timestamp, 10)
						}
						if twin.Metadata != nil {
							reported.Metadata["type"] = twin.Metadata.Type
						}
						deviceStatus.Status.Twins[i].Reported = reported
						break
					}
				}
			}

			// Store the status in cache so that when update is received by informer, it is not processed by downstream controller
			cacheDevice.Status = deviceStatus.Status
			uc.dc.deviceManager.Device.Store(deviceID, cacheDevice)

			body, err := json.Marshal(deviceStatus)
			...
			result := uc.crdClient.Patch(MergePatchType).Namespace(cacheDevice.Namespace).Resource(ResourceTypeDevices).Name(deviceID).Body(body).Do(context.Background())
			...
			//send confirm message to edge twin
			resMsg := model.NewMessage(msg.GetID())
			nodeID, err := messagelayer.GetNodeID(msg)
			...
			resource, err := messagelayer.BuildResource(nodeID, "twin", "")
			...
			resMsg.BuildRouter(modules.DeviceControllerModuleName, constants.GroupTwin, resource, model.ResponseOperation)
			resMsg.Content = "OK"
			err = uc.messageLayer.Response(*resMsg)
			...
		}
	}
}
```

`updateDeviceStatus()`方法负责更新Device API，即`Device.Status`部分；然后再给边缘端发一个确认消息。



### 2.4. 总结

DeviceController部分的源码整理如下：

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1611567386/sel/devicecontroller_s2vsmm.png" alt="devicecontroller" style="zoom:20%;" />
</center>

