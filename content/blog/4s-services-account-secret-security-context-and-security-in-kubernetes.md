+++
id= "588"

title = "4S: Services Account, Secret, Security Context and Security in Kubernetes"
description = "Service Account, Secrets和Security Contexts作为保证kubernetes集群Security的策略被引入，相关代码一直处于快速变更与迭代中。本文谨从design和初级实践的视角对其进行概略性的分析，以飨读者。"
tags= [ "security" , "Kubernetes" ]
date= "2015-07-30 13:41:07"
author = "何思玫"
banner= "img/blogs/588/Kubernetes_Security.png"
categories = [ "Kubernetes" ]

+++

Service Account, Secrets和Security Contexts作为保证kubernetes集群Security的策略被引入，相关代码一直处于快速变更与迭代中。本文谨从design和初级实践的视角对其进行概略性的分析，以飨读者。

<!--more-->


## **1\. 集群安全（Security in Kubernetes)**

众所周知，集群安全的首要关注点无疑是隔离性。进程之间的相互隔离，进程与集群基础设施的严格界限，用户与管理员之前的天然角色区分，都应该被考虑到隔离性的范畴内。 统而言之，集群安全性必须考虑如下几个目标：

 (1) 保证容器与其运行的宿主机的隔离。 

(2) 限制容器对于基础设施及其它容器的影响权限，运行拥有特权模式的容器是不被推荐的行为。 

(3) 最小权限原则——对所有组件权限的合理限制。 

(4) 通过清晰地划分组件的边界来减少需要加固和加以保护的系统组件数量。 

(5) 普通用户和管理员的角色区分，同时允许在必要的时候将管理员权限委派给普通用户。 

(6) 允许集群上运行的应用拥有secret data。 涉及安全，authentication和authorization是不能绕过的两个话题。下面我们就先来了解一下k8s在这两个issue所提供的支持。

## **2\. Authentication**

k8s目前支持三种认证方式，包括certificates/tokens/http basic auth。 client certificate authentication是双向认证的方式，可以经由easyrsa等证书生成工具生成服务器端并客户端证书。 Token authentication：单向认证方式，为kube-apiserver提供`- -token_ auth_file`，格式为一个有3columns的csv file：token,user name,user id。[此处](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides/logging-elasticsearch.md)为使用该认证方法的常见的elasticsearch case。 Basic authentication：传入明文用户名密码作为apiserver的启动参数，不支持在不重启apiserver的前提下进行用户名/密码修改。 

更多细节详见[官方相关文档](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/admin/authentication.md)。

## **3\. Authorization**

`--authorization-mode`，apiserver的参数，用于定义对secure port设置何种authorization policy，包括三种AlwaysAllow/AlwayDeny/ABAC，第一种policy允许所有对apiserver的API request，与之相反，第二种则会block所有的API request，第三种则为Attribute-Based Access Control，即对于不同request attribute，有不同的access control。 下面我们着重讨论ABAC mode。

### **3.1 Request Attributes**

在考虑authorization时，一个request有5种attribute需要考虑： - user - group - readOnly - resource（如只访问API endpoint，如/api/v1/namesapces/default/pods，或者其它杂项endpoint，如/version，此时的resource是空字符串） - namespace

### **3.2 Policy File Format**

对于ABAC mode，还需要specify `--authorization-policy-file`参数，例子参见[此处](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/pkg/auth/authorizer/abac/example_policy_file.jsonl)。 除此之外，apiserver还可以通过调用Authorizer(interface)来决定是否允许某个API操作。

## **4\. UserAccount**

这个概念适用于所有希望与k8s集群交互的个体（通常可以认为是human），因此用户名通常是human-readable，目前并不作为一个代码中的类型单独出现，一般通过config file来读取和感知。

## **5\. Service Account**

Service Account概念的引入是基于这样的使用场景：运行在pod里的进程需要调用k8s API（如scheduler/replication controller/minitor system等）以及非k8s API的其它服务（如image repository/被mount到pod上的NFS volumes中的file等）。我们使用Service Account来为pod提供id。 Service Account和第4节中提及的User account可能会带来一定程度上的混淆，下面我们就先从概念的层面将其sort it out。

*   user account通常是为human设计的，而service account则是为跑在pod里的process。
*   user account是global的，即跨namespace使用；而service account是namespaced的，即仅在belonging的namespace下使用。
*   创建一个新的user account通常需要较高的特权并且需要经过比较复杂的business process（即对于集群的访问权限的创建），而service account则不然。
*   不同的auditing consideration。

### **5.1 Design Overview**

Service Account包括如下几个元素：

*   name,用以作为id
*   principal，用以authenticated及authorized
*   security context，定义linux capabilities等与系统相关的参数。
*   secrets

### **5.2 Use Cases**

通常我们在使用`kubectl`binary来与集群交互时，我们（User Account）都会通过apiserver的认证（如果该集群设置了认证方式的话），可以被看作某个特定的User Account。类似地，当pod希望与apiserver进行交互时，也需要使用特定的认证机制，即Service Account。 当创建pod且没有经过特殊设置时，它将会被默认地分发到该namespace下default的service account，可以通过`kubectl get pods/podname -o yaml`的命令行查看`spec.serviceAccount`的field）。

#### **5.2.1 default service account**

每个namespace在创建时都会有一个default的service Account，如下所示

```shell
$ kubectl get serviceAccounts --all-namespaces
NAMESPACE     NAME      SECRETS
default       default   1
kube-system   default   1
```


#### **5.2.2 Create your own service Account**

用户可以方便地通过yaml文件等形式创建自己的serviceAccount，如

```shell
$ cat > /tmp/serviceaccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF

$ kubectl create -f /tmp/serviceaccount.yaml
serviceaccounts/build-robot
```


通过\`kubectl get < resource > -o yaml获取详细信息

```shell
$ kubectl get serviceaccounts/build-robot -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-07-28T07:34:02Z
  name: build-robot
  namespace: default
  resourceVersion: "196476"
  selfLink: /api/v1/namespaces/default/serviceaccounts/build-robot
  uid: 0477793a-34fb-11e5-a412-005056b45394
secrets:
- name: build-robot-token-n0o98
```


可以看到，token（secret）被自动生成并ref了。如果在之后创建pod的时候ref这个自建的serviceAccount，在`spec.serviceAccount`中指定即可。 通过如下命令可以删除serviceAccount。

```shell
$ kubectl delete serviceaccount/build-robot
```


#### **5.2.3Adding Secrets to a service account**

注意到，我们可以不使用serviceAccount自建的secrets，而是选用我们自己生成的secrets，这是一种consume secret的方式。 首先创建secret，`kubectl create -f secret.yaml`

```yaml
//secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  data-1: dmFsdWUtMQ0K
  data-2: dmFsdWUtMg0KDQo=
```


然后创建一个comsume该secret，`kubectl create -f serviceaccount.yaml`

```yaml
//serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
secrets:
- name: test-secret
```


最后在pod的manifest中指定其serviceaccount，`kubectl create -f busybox4.yaml`

```yaml
//busybox4.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox4
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
    volumeMounts:
      - name: foo
        mountPath: /var/run/secrets/kubernetes.io/serviceaccount
        readOnly: true
  restartPolicy: Always
  serviceAccountName: build-robot
  volumes:
    - name: foo
      secret:
        secretName: test-secret
```


已了解如何创建一个引用自建secret的serviceaccount，但是如果在exsiting serviceaccount中添加引用，目前[官方文档](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/user-guide/service-accounts.md)mark it as TODO。笔者认为此用法应该与上例类似，但笔者采用这种方式consume secret时，serviceaccount似乎并不对该secret有维护作用。 Update:此用法确与上例类似，参见issue#12012，笔者后续会考虑对相关doc进行更新。

## **6\. Secrets**

secret是用于存放sensitive information的数据结构，可以通过manual或者automatic两种方式创建。 secret的use case主要有两种，或者作为volume mount到pod上，或者用于kubelet在为pod里的container拉取镜像时。 使用serviceAccount自建的secret是一种保证secure的推荐方式，当然可以由用户disable或者overridden，如果用户愿意承担额外的风险的话。

### **6.1 Secrets类型一览**

secret的类型目前有三种，如下所示

```go
type SecretType string

const (
    SecretTypeOpaque              SecretType = "Opaque"                                 // Opaque (arbitrary data; default)
    SecretTypeServiceAccountToken SecretType = "kubernetes.io/service-account-token"    // Kubernetes auth token
    SecretTypeDockercfg           SecretType = "kubernetes.io/dockercfg"                // Docker registry auth
)
```


Opaque类型是默认的用户自建类型，即任意string（实际上也并不是十分任意，`data` label的key/value必须满足特定的要求：Each key must be a valid DNS\_SUBDOMAIN or leading dot followed by valid DNS\_SUBDOMAIN. Each value must be a base64 encoded string as described in https://tools.ietf.org/html/rfc4648#section-4）。 第二种类型的例子如下所示

```yaml
{
    "apiVersion": "v1",
    "kind": "Secret",
    "metadata": {
        "name": "mysecretname",
        "annotations": {
            "kubernetes.io/service-account.name": "build-robot"
        }
    },
    "type": "kubernetes.io/service-account-token"
}    
```


第三种类型一般用于向private registry拉取镜像，在6.3节中再做阐述。

### **6.2 manually specify a secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: dmFsdWUtMg0K
  username: dmFsdWUtMQ0K
```


如上述所示，使用`kubectl create`命令即可创建一个secret。创建完毕之后，可以使用serviceAccount来consume，也可以手动添加到pod的manifest中，如下例所示：

```yaml
{
 "apiVersion": "v1",
 "kind": "Pod",
  "metadata": {
    "name": "mypod",
    "namespace": "myns"
  },
  "spec": {
    "containers": [{
      "name": "mypod",
      "image": "redis",
      "volumeMounts": [{
        "name": "foo",
        "mountPath": "/etc/foo",
        "readOnly": true
      }]
    }],
    "volumes": [{
      "name": "foo",
      "secret": {
        "secretName": "mysecret"
      }
    }]
  }
}
```


[More example](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/docs/user-guide/secrets) here。

### **6.3 Manually specifying an imagePullSecret**

这个方法仅在GKE/GCE或其它cloud-provider[场景](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/user-guide/images.md#specifying-imagepullsecrets-on-a-pod)下推荐使用，创建一个类型为kubernetes.io/dockercfg的secret，并且在pod manifest的imagePullSecrets label下使用。因为缺乏实验平台，在此不作赘述。 总而言之，目前k8s对于secret的support仍处于不十分完善的阶段，secret可能以多种方式暴露给外部用户甚至是attacker。

## **7\. Security context**

Security context是用以对容器进行限制，使得不同的运行容器之前能够实现较为明晰的隔离，以及降低其影响宿主机和其它容器的可能性。通俗而言，容器中的security context用于表征在创建及运行容器时，它能够使用及访问的资源参数。 该概念将会被用到如下两处：

*   kubelet用于创建及运行container。
*   作为serviceAccount的一部分定义在pod中。

这个概念具体如何使用在k8s场景中基本上可以认为在servicAccount中的securityContext label下，目前代码只实现了capabilities和privileged，在此不展开说明。 