title: Kubernetes 集群原理剖析
categories: Kubernetes
tags: [docker, kubernetes]
date: 2015-08-02 11:10:00
---
## 什么是Kubernetes
Kubernetes是Google开源的容器集群管理系统，其提供应用部署、维护、 扩展机制等功能，利用Kubernetes能方便地管理跨机器运行容器化的应用。而且Kubernetes支持GCE、vShpere、CoreOS、OpenShift、Azure等平台上运行，也可以直接部署在物理主机上。
举个例子：
openstack用来管理虚拟化(kvm、xen、vmware等)
kubernetes用来管理容器(docker)<!--more-->

## Kubernetes概念(角色)组成
### Pod
* Pod是kubernetes的最小操作单元，一个Pod可以由一个或多个容器组成
* 同一个Pod只能运行在同一个主机上
* 同一个Pod共享着相同的volumes,network命名空间

### ReplicationController（RC）
* RC是用来管理Pod的，每个RC可以由一个或多个的Pod组成，在RC被创建后，系统将会保持RC中的可用Pod的个数与创建RC时定义的Pod个数一致，如果Pod个数小于定义的个数，RC会启动新的Pod，反之则会杀死多余的Pod
* RC通过定义的Pod模板被创建，创建后对象叫做Pods(也可以理解为RC)，可以在线的修改Pods的属性，以实现动态缩减/扩展Pods的规模或属性
* RC通过label关联对应的Pods，通过修改Pods的label可以删除对应的Pods在需要对Pods中的容器进行更新时，RC采用一个一个的替换原则来更新整个Pods中的Pod

### Label
* Label是用于区分Pod、Service、RC的key/value键值对
* Pod、Service、RC可以有多个label，但是每个label的key只能对应一个value
* 整个系统都是通过Label进行关联，得到真正需要操作的目标

### Service
* Service也是Kubernetes的最小操作单元，是真实应用服务的抽象
* Service通常用来将浮动的资源与后端真实提供服务的容器进行关联
* Service对外表现为一个单一的访问接口，外部不需要了解后端的规模与机制
* Service其实是定义在集群中一组运行Pod集合的抽象资源，它提供所有相同的功能。当一个Service资源被创建后，将会分配一个唯一的IP(也叫集群IP)，这个IP地址将存在于Service的整个生命周期，Service一旦被创建，整个IP无法进行修改。Pod可以通过Service进行通信，并且所有的通信将会通过Service自动负载均衡到所有的Pod中的容器

## Kubernetes组成
kubernetes中的组成类似于openstack，由不同的角色(组件)组成，共同支撑整个系统的运行。
主要包括：kubectl、kube-apiserver、kube-controller-manager、kube-scheduler、kube-proxy、kubelet，当然这些并不能组成一个完整的kubernetes系统，整个系统中的信息还需要一个存储介质Etcd，网络服务Flannel(可选)

相较于OpenStack中的组件：
```
kubectl                     nova/glance/neutron/cinder等        客户端命令行工具
kube-apiserver              nova-api                            REST API服务
kube-controller-manager     keystone…                           多个控制器的合体
kube-scheduler              nova-scheduler                      任务调度，命令下发
kube-proxy                  iptables                            网络代理转发
kubelet                     nova-compute                        容器/虚拟化的管理
etcd                        mysql                               信息存储
flannel                     dnsmasq                             IP地址的分配
```
### Kubectl
一个命令行工具，将接收到的命令，格式化后，发送给kube-apiserver，作为对整个平台操作的入口。
### Kube-apiserver
作为整个系统的控制入口，以REST API的形式公开。它可以横向扩展在高可用架构中。
### Kube-controller-manager
用来执行整个系统中的后台任务，它其实是多个控制进程的合体。大致包括如下
* Node Controller    负责整个系统中node up或down的状态的响应和通知
* Replication Controller    负责维持Pods中的正常运行的pod的个数
* Endpoints Controller    负责维持Pods和Service的关联关系
* Service Account & Token Controllers    负责为新的命名空间创建默认的账号和API访问Token

### Kube-scheduler
负责监视新创建的Pods任务，下发至未分配的节点运行该任务
### Kube-proxy
kube-proxy运行在每个节点上，它负责整个网络规则的连接与转发，使kubernetes中的service更加抽象化
### Kubelet
kubelet运行在每个节点上，作为整个系统的agent，监视着分配到该节点的Pods任务，(通过apiserver或者本地配置文件)，负责挂载Pods所依赖的卷组，下载Pods的秘钥，运行Pods中的容器(通过docker)，周期获取所有容器的可用状态，通过导出Pod和节点的状态反馈给REST系统

大概可以用以下这幅图来表示：
![](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes/kubernetes.png)


一组共享上下文的应用程序叫做一个pod，在上下文中，程序也可以应用单独的cgroup隔离。一个pod的模型就是一组运行指定应用的容器环境(逻辑主机)，他可以容纳一个或多个应用程序，但是在一个容器世界里，这表现的相对较耦合。它们会运行在相同的物理主机或虚拟主机上
pod中的上下文是结合Linux命令空间来定义的，这里包含:
* pod namespace(pod中的应用程序可以看到其他的进程)
* network namespace(应用程序获得相同的IP和端口空间)
* ipc namespace(pod中应用程序可以使用SystemV IPC或者POSIX消息队列来通信)uts namespace(pod中的应用程序共享主机名)

资源共享和通信
pod中所有的应用程序使用相同的网络命名空间，应用程序间可以使用localhost来发现其他程序及通信。每一个pod都有一个IP地址，用来和其他物理节点及跨网络的容器进行通信。
pod作为部署的最小单位，支持水平扩展和复制




</br>