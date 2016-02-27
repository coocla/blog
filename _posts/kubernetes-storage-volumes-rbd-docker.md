title: Kubernetes技术研究持久化存储
categories: Kubernetes
tags: [docker, kubernetes, volume]
date: 2015-08-30 22:24:00
---
docker本身设计之初是用来执行一个app，抑或是一个应用程序，在其运行结束后，将销毁一切数据，但是这明显不是我们想要的，docker也想到了这个，因此其本身提供一个-v的参数，用来将外部的存储挂载到container中，用来保存我们的持久化的数据。kubernetes最为其集群管理工具，自然也想到了这些，而且还提供了更强大的功能，基于其插件化的设计，kubernetes针对volume的driver支持是十分丰富的。本文就简单描述几个driver的设计思想和使用方法：<!--more-->

### emptyDir
如果Pod配置了emptyDir类型Volume， Pod 被分配到Node上时候，会创建emptyDir，这好像是基于pod的一块内存分配。只要Pod运行在Node上，emptyDir都会存在（Pod中任何一个容器挂掉不会导致emptyDir丢失数据），但是如果Pod从Node上被删除（Pod被删除，或者Pod发生迁移），emptyDir也会被删除，并且永久丢失。

创建一个pod并且使用emptyDir volume:
```
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: empty
      mountPath: /mnt
  volumes:
  - name: empty
    emptyDir:
      Medium: ""   # 这里可以使用空或者使用"memory"关键字，来制定不通的存储媒介
```
创建完后，我们通过进入container中查看/mnt所挂载的类型
```
# kubectl exec -p test-emptydir -it /bin/bash
root@test-emptydir:/# findmnt -m /mnt
```
![](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes_volume/v1.jpg)
![](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes_volume/v2.jpg)
当我们分别使用Medium为空或者为memory时，我们发现container所创建device类型是不一样的，及实现方式也是不一样的。

### hostPath
hostPath这个就很好理解，它好比就是docker自身的-v参数，允许挂载container所在的Node上的文件或者目录到Pod里面去。但是极不建议使用这种方式，因为这样会使你所创建的资源极度与你Node上的文件系统结构十分耦合，在调度过程中，大部分情况我们是无法预测到资源被调度到的具体的Node，所以如果对Node的文件系统目录维护不好，这将导致你的资源创建失败。
```
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: hostpath
      mountPath: /mnt
  volumes:
  - name: hostpath
    hostPath:
      Path: /data
```
```
# kubectl exec test-hostpath -it /bin/bash
root@test-hostpath:/# findmnt -m /mnt
```
![](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes_volume/v3.jpg)
可以看到，/mnt目录所挂载的是我们Node中的/data目录，由于Node中/data所在的文件系统类型是XFS，以及挂载的参数，我们在container中也可以看到具体的详情的。

### Ceph RBD
GlusterFS是rbd允许将Rados块设备挂载到Pod中。想较emptyDir，rbd将持久存储你的数据，即使设备卸载，pod删除等，数据都会被保存。rbd其实是由ceph分布式存储提供的，他本身提供对象存储，块存储，文件系统存储，这里暂且不讲述关于ceph的特性。但是有一点我们需要了解，同一个rbd(这里暂且就理解为云硬盘)可以被多个客户端以只读的方式挂载，但是只允许一个客户端以读写的方式进行挂载。
再次之前我们需要准备一个ceph集群，具体的步骤可以参考官网上使用ceph-deploy快速部署一个集群，当然也可以使用github上的ceph image创建ceph container
其次我们需要完成以下其他配置：
1. 安装ceph客户端
在kubernetes的集群中的所有节点安装ceph客户端，因为在创建pod时，默认我们是不能提前确定pod会被创建到哪个节点上。
```
# yum install ceph
```
2. 创建挂载rbd的用户的secret
在客户端进行rbd挂载时需要进行用户认证，采用用户名和其keyring。默认使用ceph-deploy部署创建一个admin用户，对应keyring为/etc/ceph/ceph.client.admin.keyring
将keyring里面的值用base64进行编码
```
# awk '/key/{print $3}' /etc/ceph/ceph.client.admin.keyring  | base64
QVFCWll0MVZvRddf5R2hBQUlzMm92VGMvdmMyaFzaWjl2dFhsS1E9PQo=
```
使用编码后的key创建secret
```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: ceph-client-admin-keyring
data:
  key: QVFCWll0MVZvRddf5R2hBQUlzMm92VGMvdmMyaFzaWjl2dFhsS1E9PQo=
```
3. 在ceph集群的rbd pool中创建一个img，大小为100M
```
# rbd create new —size 100
# rbd ls
img
new
```
4. 将该img挂载并且格式化为ext4的文件系统
> 注意这里需要使用一个内核支持rbd模块服务器进行挂载，否则将无法直接挂载
```
# modprobe rbd
# lsmod | grep rbd
rbd                    73133  2
libceph               235953  1 rbd

# rbd map rbd/new
# rbd showmapped
id pool image snap device
0  rbd  img   -    /dev/rbd0
1  rbd  new   -    /dev/rbd1

# mkfs.ext4 /dev/rbd1
# rbd unmap /dev/rbd1
```
5. 创建pod并且将刚才已经格式化的img挂载到container中
```
apiVersion: v1
kind: Pod
metadata:
  name: test-rbd
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: rbd
      mountPath: /mnt
  volumes:
  - name: rbd
    rbd:
      monitors:
      - 10.10.2.121:6789
      pool: rbd
      image: new
      user: admin
      secretRef:
        name: ceph-client-admin-keyring
      fsType: ext4
```
这里还可以使用keyring选项来代替secretRef，用来表示认证的keying文件在节点的位置，这就使我们必须要手动维护这个keyring文件在所有的节点上都存储在相同的位置。这很明显大大增加了我们的维护成本，因此不建议使用该方法。
6. 在pod运行的节点上查看rbd挂载的情况
```
# rbd showmapped
0  kube test  -    /dev/rbd0
1  rbd  new   -    /dev/rbd1
```
7. 登陆到container中查看挂载情况
```
# kubectl exec test-rbd -it /bin/bash
root@test-rbd:/# findmnt /mnt/
```
![](http://7xk38j.com1.z0.glb.clouddn.com/kubernetes_volume/v4.jpg)
8. 在/mnt中创建一个文件
```
root@test-rbd:/# echo ’This is RBD’ > /mnt/index
```
9. 删除该pod，并将该img挂载查看里面的内容
```
# rbd map rbd/new
# rbd showmapped
0  rbd  img   -    /dev/rbd0
1  rbd  new   -    /dev/rbd1

# mount /dev/rbd1/ /mnt/
# ls /mnt/
index  lost+found
# cat /mnt/index
This is RBD
```








</br>