title: Kubernetes技术研究资源限制
categories: Kubernetes
tags: [docker, kubernetes, limits, quota]
date: 2015-08-30 21:52:00
---
## 前言
kubernetes的调度算法仅仅会将新创建的资源调度到拥有足够CPU和内存的节点上，因此kubernetes为用户提供了两种方法来定义资源的限制。

### 资源限制
首先基于计算类的资源限制，称为resouce limit.
当指定一个pod时，可以指定每一个container可以使用多少CPU和RAM，当container拥有资源限制时，调度器可以调度出更加适合该pod运行的节点，并且针对资源的抢占也可以很好的控制。

> 资源的类型分为两个：CPU、memory
> CPU的限制的CPU的核数比重，内存的最小单位是字节

从kubernetes源代码中可以看到关于资源限制的单位如下图：<!--more-->
[](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes_limits/084C3B95-37AB-46D3-84C3-D4FE538FE5F3.jpg)
CPU和RAM统称为计算资源，一般计算资源拥有这些特性：可以衡量，可以申请，可以分配，可以消耗。它不同于API资源。
```
ResourceName	Description
cpu	Total cpu limits of containers
memory	Total memory limits of containers
```

### 容器和Pod的资源限制：
每一个container和pod都可以通过
    spec.container[x].resources.limits.cpu
    spec.container[x].resources.limits.memory
来进行资源的限制
> 这里的x用来表示列表的索引

### 资源限制的生效规则：
在相同的集群中，如果没有设置对应的值，那么将会使用默认的值。默认的值依赖于所处集群的namespace配置，针对资源限制只能作用在单个容器中，pod的资源限制则是其中container的资源限制总和，如果没有设置对应值则视为0。

### 创建一个默认的限制：
我们针对default的namespace创建一个针对容器的默认的资源限制(在创建容器时如果没有指定，那么其资源限制规则将继承我们创建的这个默认的)，另外，我们还可以针对该namespace中资源限制创建一个范围：最大(max)，最小(min)

```
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - default:
      cpu: 100m
      memory: 512Mi
    max:
      cpu: 1024m
      memory: 10240Mi
    min:
      cpu: 100m
      memory: 256Mi
    type: Container
    type: Pod
    type: ReplicationController
```
创建一个带有资源限制的的pod(这里我们没有指定它所属的namespace，默认属于default)：
```
apiVersion: v1
kind: Pod
metadata:
  name: limit
spec:
  containers:
  - name: web
    image: nginx
    resources:
      limits:
        cpu: 100m
        memory: 500Mi
```
### 集群的调度与Pod的资源限制：
当创建pod时，kubernetes集群调度器会选择一个节点来运行pod，每一个节点针对每一种资源类型都有一个最大容量，一定数量的CPU和Memory可以提供给pod。调度器需要保证节点上资源限制的总和必须小于节点的总容量。如果调度器对节点的容量检查失败，即使节点的内存和CPU资源使用很低，调度也仍然会拒绝将pod运行到该节点上。
当kubelet启动container时，如果该container有针对CPU和memory资源的限制，那么kubelet将会使用docker中的资源限制—cpu-shares  —memory参数将定义的限制作用到container上
spec.container[].resources.limits.cpu的值就会除以1024作为—cpu-shares的值,即资源权重
spec.container[].resources.limits.memory的单位Ki=KB、Mi=MB、Gi=GB很容易理解

## 限制隔离
第二种资源限制，在多个团队或用户共享一个集群时，可以为团队或用户分配不通的资源配额，称为resource quota。
1. 不同的团队属于不同的namespaces
2. 用户使用计算资源限制在他们的pod上
3. 管理员为每一个namespace创建不通的资源限制
4. 如果使用的配额超过了当前namesapce拥有的资源限制，HTTP API将会返回403
5. 如果某一namespace配置了资源限额，但是创建pod时没有指定资源配额，那么HTTP API将会返回403，注意：如果要这样使用，请使用LimitRange、ResourceQuota这个两个admission controller来进行确保数据的强一致。可以看到apiserver返回给我们的信息如下：

```
[{
  "metadata": {},
  "status": "Failure",
  "message": "error when creating \"pod.yaml\": Pod \"limit2\" is forbidden: Limited to 5 pods",
  "reason": "Forbidden",
  "details": {
    "name": "limit2",
    "kind": "Pod"
  },
  "code": 403
}]
```
### 利用资源配额我们可以做什么：
在一个容量为32G内存、16核的集群中。配置团队A使用20G内存和10核新，团队B使用10G内存和4核，并且预留出2G内存和2核，以备未来进行分配
### 资源抢占时的调度策略：
在资源紧张的情况下，kubernetes采用先到先得的策略予以分配
### 开启资源配额：
配置apiserver的配置文件
使用—admission-control=ResourceQuota参数来启动apiserver
配额支持限制以下资源：
```
ResourceName	Description
pods	Total number of pods
services	Total number of services
replicationcontrollers	Total number of replication controllers
resourcequotas	Total number of resource quotas
secrets	Total number of secrets
persistentvolumeclaims	Total number of persistent volume claims
```
### 创建一个Quota:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: default
spec:
  hard:
    cpu: 4
    memory: 32G
    pods: 30
    services: 100
    replicationcontrollers: 80
    resourcequotas: 10
```
### 查看对应Quota的状态
```
# kubectl describe quota my-quota
Name:			my-quota
Namespace:		default
Resource		Used		Hard
--------		----		----
cpu			200m		4
memory			268435456	32G
pods			5		30
replicationcontrollers	2		80
resourcequotas		2		10
services		2		100
```

当前的一些quota可能还不能满足需求例如：

1. 将当前集群的资源按照比例分配给多个team
2. 资源的限制有一个缓冲的范围，软限制和硬限制
3. 检查一个namespace的资源用量，自动的添加节点或增加配额
类似这些需求，都可以参照ResouceQuota这种堆叠式方式，编写controller算法来实现。






</br>



