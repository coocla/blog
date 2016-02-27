title: Kubernetes资源创建yml语法
categories: Kubernetes
tags: [docker, kubernetes]
date: 2015-08-20 15:23:00
---
## 前言
在是用kubernetes中，我们对资源的创建大部分都是通过
```
kubelet create -f RESOURCE.yaml
```
刚开看的时候不免有一些迷茫，看不懂语法，不知道怎么写；今天本文就介绍一下kubernetes construct语法。

Construct语法其实就是由kubelet格式化成API的post data，提交给apiserver，因此这里支持yaml，json两种数据结构的文件。<!--more-->

## Pod资源
```
apiVersion: v1                                      指定api版本，此值必须在kubectl apiversion中
kind: Pod                                           指定创建资源的角色/类型
metadata:                                           资源的元数据/属性
  name: test                                        资源的名字，在同一个namespace中必须唯一
  labels:                                           设定资源的标签
    sex: boy                                        标签以key/value的结构存在
    age: 18
spec:                                               #specification of the resource content 指定该资源的内容
  restartPolicy: Never                              表明改容器仅仅运行一次，默认k8s的策略，在此容器退出后，会立即创建一个相同的容器
  volumes:                                          定义一组挂载设备
  - name: volume                                    定义一个挂载设备的名字
    hostPath: /data/www/html                        挂载设备类型为hostPath，路径为宿主机下的/data/www/html,这里设备类型支持很多种
  containers:                                       指定资源中的容器
  - name: container1                                容器的名字
    image: “docker.coocla.org/ubuntu:last”          容器使用的镜像地址
    volumeMounts:
      - mountPath: /mnt                             挂载到容器的某个路径下
        name: volume                                挂载设备的名字，与volumes[*].name 需要对应
    livenessProbe:                                  容器健康监测
    httpGet:                                        http形式监测，返回200-399之间，则认为容器正常
      path: /health
      port: 8080
    initialDelaySeconds: 15                         表明第一次检测在容器启动后多长时间后开始
    timeoutSeconds: 1                               检测的超时时间
    env:                                            指定容器中的环境变量
    - name: str                                     变量的名字
      value: "hello world”                          变量的值
    command: ["/bin/bash", "-c"]                    覆盖容器中的Entrypoint,对应Dockefile中的ENTRYPOINT
    args: ["/bin/echo", "$(str)"]                   对应Dockerfile中CMD参数
```
健康监测还支持另外一种方法：
```
exec:                                      执行命令的方法进行监测，如果其退出码不为0，则认为容器正常
  command:
  - cat
  - /tmp/health
initialDelaySeconds: 15
timeoutSeconds: 1
```
## ReplicationController的语法参数
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2                                      指定rc中pod的个数
  template:                                        指定rc中pod的模板，rc中的pod都是按照这个模板来创建的
  metadata:                                        指定rc中pod的元数据，注意这里不需要在指定pod的名字，它由rc复制生成
    labels:
      app: nginx
  spec:
    container:
    - name: nginx
      image: nginx
```
## Service
```
apiVersion: v1
kind: Service
metadata:
  name: nginxsvc
  labels:
    app: nginx
spec:                                          指定Service中的内容
  ports:                                       映射列表
  - port: 80                                   service的端口
    porotocal: TCP                             映射的协议类型，支持TCP/UDP
    targetPort: 80                             映射到pod的端口
    name: www.baidu.com                        该映射的名字
selector: 匹配器
port: 80
app: nginx 匹配label中app为nginx，port为80的pod
```
## Secret
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: 0paque                                      定义secret的类型，这里支持三种类型
  password: xxx|base64                            以key/value的形式定义，value需要经过base64编码才可以，在secret被挂载到container中后，会以key作为文件名，value的值经过base64解码作为内容，以文件的形式存在于container中
  username: xxx|base64

---
type: kubernetes.io/service-account-token         第二种secret类型，用作创建服务账号的token，用作进程间通信的认证

---
type: kubernetes.io/dockercfg                     第三种secret类型，用作在创建container，对docker registry的认证
data:
  .dockercfg: `cat ~/.dockercfg | base64`
```


以上为部分资源参数，当然还有更多的参数可以指定，及更多的资源可以通过定义construct来创建。






</br>