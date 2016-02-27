title: Kubernetes技术研究Secret资源
categories: Kubernetes
tags: [docker, kubernetes, secret]
date: 2015-08-21 13:20:07
---
## 前言
作为kubernetes中一个重要的资源secret，它的设计初衷是为了解决container在访问外部网络或外部资源时验证的问题，例如访问一个git仓库，连接一个数据库，设置一些密码配置等，需要额外验证的场景。它被作为一种资源的形式被设计，由kubernetes集群统一管理，从字面意思来看已经表现了敏感，安全等特性，因此集群对待这类资源的管理需要额外的保护，对其内容进行加密是十分有必要的。当前，如果集群中的某些几点被恶意攻击或泄露，那么对于secret的影响则只会影响到该节点上的一些数据安全，无可厚非，如果主节点被攻陷，那么....大家都懂。<!--more-->

## secret的创建
* 用户自己手动创建，大部分情况用来存储用户私有的一些信息
* 集群系统自动创建，用来作为集群中各个组件之间通信的身份校验使用

## secret的类型
* 0paque，基于base64编码的，长用来存储密码，公钥之类的
* kubernetes.io/dockercfg，用来存储私有docker registry 登陆验证时使用
* kubernetes.io/service-account-token，用来创建集群服务账号的token，用作集群各组件通信时的令牌

### 0paque
此类型的secret中data字段是一个map类型，其key在之后的挂载会作为文件名使用，其命名规范需要满足DNS_SUBDOMAIN的规范，其value会作为文件内容使用，但是会经过base64解码。因此这里我们需要填入的是真是内容经过base64编码后的内容。在未来版本，可能会支持直接将其value的值以环境变量的方式存在于container中，如果目前有这种需要则需要用户通过文件读取的方式自行将其添加到环境变量中。

在整个集群里面还有一种资源叫做namespace，每一个secret和pod都存在于namespace中，在后续pod中挂载secret时，需要满足对应的secret和pod必须存在于同一个namespace中，另外为了防止对集群进程的内存占用，每一个secret的内容都被限制在1M大小之内。

* 需求：
我们有一个rsync的服务，rsync需要一个密码文件的认证，该密码文件名为：.rsyncpass，密码内容为：password
* 实现：
1. 针对该密码创建对应的secret，并生成密码内容经过base64编码后的字符串
```
# echo 'password'|base64
cGFzc3dvcmQK
```
```
apiVersion: v1
kind: Secret
metatdata:
  name: my-rsync-pass
type: 0paque
data:
  .rsyncpass: cGFzc3dvcmQK
```
```
# kubectl create -f password.yaml
secrets/my-rsync-pass
```
2. 创建rsync服务并将该secret挂载至对应的container中
```
apiVersion: v1
kind: Pod
metadata:
  name: my-rsync-service
spec:
  containers:
  - name: rsync
    image: rsync
    volumeMounts:
    - name: rsync-pass
      mountPath: /mnt/
      readOnly: true
  volumes:
  - name: rsync-pass
    secret:
      secretName: my-rsync-pass
```
```
# kubectl create -f pod.yaml
```
3. 检查secret是否挂载成功
```
# kubectl exec my-rsync-service -i -t -- bash -il
# ls -al /mnt/
.rsyncpass
# cat /mnt/.rsyncpass
password
```

### kubernetes.io/dockercfg
此类型的secret通常用来作为私有registry的验证，在kubernetes创建containers时对image进行pull操作之前对registry的验证，通常情况下对于有验证的registry我们都会使用docker login REGISTRY来进行登录认证，认证通过后，会在当前用户家目录下生成一个.dockercfg的文件，里面会记录关于每个registory的验证信息。

* 场景：

我们有一个私有的docker registry，地址为https://docker.coocla.org，我们需要基于这个仓库创建一些container，但是我们不可能在所有的集群几点上都执行一遍docker login。此时secret要发挥其作用了。

* 实现
1. 选择一台集群节点，利用docker login来生成.dockercfg(1.6版本以后就生成在.docker/config.json里了)
```
# docker login docker.coocla.org
docker login docker.coocla.org
Username: dev
Password:
Email:
WARNING: login credentials saved in /root/.docker/config.json
Login Succeeded
```
2. 将.docker/config.json里面的内容通过base64进行编码
```
# cat /root/.docker/config.json | base64
ewoJImF1dGhzIjogewoJCSJkb2NrZXIuY29vY2xhLm9yZyI6IHsKCQkJImF1dGgiOiAiWkdWMk9t
UnZZMnRsY2c9PSIsCgkJCSJlbWFpbCI6ICIiCgkJfQoJfQp9
```
> 注意：这里我们发现经过base64编码后是两段字符串，但是我们在定义secret时需要将其合并成一行，不允许有回车出现。
 

3. 创建对应的secret
```
apiVersion: v1
kind: Secret
metadata:
  name: docker.coocla.org.key
type: kubernetes.io/dockercfg
data:
  .dockercfg: ewoJImF1dGhzIjogewoJCSJkb2NrZXIuY29vY2xhLm9yZyI6IHsKCQkJImF1dGgiOiAiWkdWMk9tUnZZMnRsY2c9PSIsCgkJCSJlbWFpbCI6ICIiCgkJfQoJfQp9
```

4. 创建pod，通过指定参数imagePullSecrets来指定拉取镜像时所需要的认证内容
```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - name: mongodb
    image: "docker.coocla.org/mongo:2.4"
  imagePullSecrets:
  - name: docker.coocla.org.key
```

### kubernetes.io/service-account-token
关于这类的secret，本文暂时不做过多介绍，这里涉及到集群中的另外一种资源serviceaccount。在日常操作时也很少用到，暂时不做多的介绍。

附：Secret和Pod的生命周期和依赖关系：

1. 在调用apiserver创建pod时，apiserver是不会检查pod中引用的secret是否存在(在对应的namespace中)，在pod被scheduler调度到某个节点进行创建的时候，kubelet会尝试寻找对应的secret，如果此时secret不存在或者因网络原因未能成功获取到对应的secret，那么kubelet会周期的尝试拉取pod中所依赖的secret，并且产生一个event(集群中的另外一种资源)，pod中对应container的创建也会因此被阻塞。
2. 当pod中的container成功创建后，此时如果修改了其secret的内容，那么运行中的container所挂载的secret是不会随之改变的。





</br>








