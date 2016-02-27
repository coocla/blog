title: Kubernetes技术研究容器监控监测
categories: Kubernetes
tags: [docker, kubernetes, monitor, Liveness]
date: 2015-09-17 20:49:00
---
大部分的应用程序我们在部署的时候都会适当的添加监控，对于运行载体容器则更应该如此。kubernetes提供了 liveness probes来检查我们的应用程序。它是由节点上的kubelet定期执行的。

首先说一下Pod的整个生命阶段：
* Pending：表示集群系统正在创建Pod，但是Pod中的container还没有全部被创建，这其中也包含集群为container创建网络，或者下载镜像的时间；
* Running：表示pod已经运行在一个节点商量，并且所有的container都已经被创建。但是并不代表所有的container都运行，它仅仅代表至少有一个container是处于运行的状态或者进程出于启动中或者重启中；
* Succeeded：所有Pod中的container都已经终止成功，并且没有处于重启的container；
* Failed：所有的Pod中的container都已经终止了，但是至少还有一个container没有被正常的终止(其终止时的退出码不为0)<!--more-->

对于liveness probes的结果也有几个固定的可选项值：
* Success：表示通过检测
* Failure：表示没有通过检测
* Unknown：表示检测没有正常进行

Liveness Probe的种类：
* ExecAction：在container中执行指定的命令。当其执行成功时，将其退出码设置为0；
* TCPSocketAction：执行一个TCP检查使用container的IP地址和指定的端口作为socket。如果端口处于打开状态视为成功；
HTTPGetAcction：执行一个HTTP默认请求使用container的IP地址和指定的端口以及请求的路径作为url，用户可以通过host参数设置请求的地址，通过scheme参数设置协议类型(HTTP、HTTPS)如果其响应代码在200~400之间，设为成功。

当前kubelet拥有两个检测器，他们分别对应不通的触发器(根据触发器的结构执行进一步的动作)：
* Liveness Probe：表示container是否处于live状态。如果LivenessProbe失败，LivenessProbe将会通知kubelet对应的container不健康了。随后kubelet将kill掉container，并根据RestarPolicy进行进一步的操作。默认情况下LivenessProbe在第一次检测之前初始化值为Success，如果container没有提供LivenessProbe，则也认为是Success；
* ReadinessProbe：表示container是否以及处于可接受service请求的状态了。如果ReadinessProbe失败，endpoints controller将会从service所匹配到的endpoint列表中移除关于这个container的IP地址。因此对于Service匹配到的endpoint的维护其核心是ReadinessProbe。默认Readiness的初始值是Failure，如果一个container没有提供Readiness则被认为是Success。

对于LivenessProbe和ReadinessProbe用法都一样，拥有相同的参数和相同的监测方式。
* initialDelaySeconds：用来表示初始化延迟的时间，也就是告诉监测从多久之后开始运行，单位是秒
* timeoutSeconds: 用来表示监测的超时时间，如果超过这个时长后，则认为监测失败

当前对每一个Container都可以设置不同的restartpolicy，有三种值可以设置：
* Always: 只要container退出就重新启动
* OnFailure: 当container非正常退出后重新启动
* Never: 从不进行重新启动

如果restartpolicy没有设置，那么默认值是Always。如果container需要重启，仅仅是通过kubelet在当前节点进行container级别的重启。

最后针对LivenessProbe如何使用，请看下面的几种方式，如果要使用ReadinessProbe只需要将livenessProbe修改为readinessProbe即可：
```
apiVersion: v1
kind: Pod
metadata:
  name: probe-exec
  namespace: coocla
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/health
      initialDelaySeconds: 5
      timeoutSeconds: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: probe-http
  namespace: coocla
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
        host: www.baidu.com
        scheme: HTTPS
      initialDelaySeconds: 5
      timeoutSeconds: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: probe-tcp
  namespace: coocla
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      initialDelaySeconds: 5
      timeoutSeconds: 1
      tcpSocket:
        port: 80
```
我们使用上面的construct创建资源：
```
# kubectl create -f probe.yaml
# kubectl get event
```
通过event我们可以发现对应的container由于probe监测失败一直处于循环重启中，其事件原因：unhealthy
[](http://7xk38j.com1.z0.glb.clouddn.com/liveness_probes1C154BAA-20AA-4F30-985D-0A4913A483BA.png)







</br>