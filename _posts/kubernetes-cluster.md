title: Kubernetes技术研究集群搭建
categories: Kubernetes
tags: [docker, kubernetes, etcd, cluster]
date: 2015-08-19 14:33:00
---
## 环境准备：
IP	角色	
数量
192.168.0.30/10.10.0.11
kubernetes master、etcd	
1
192.168.0.31/10.10.0.10
kubernetes node、etcd	
1
192.168.0.32/10.10.0.12
kubernetes node、etcd	1
## 安装软件包：<!--more-->
### Master
```
# yum install kubernetes-master etcd -y
```
### Node
```
# yum install kubernetes-node etcd -y
```
## 配置etcd集群：
* 192.168.0.30：
```
# [member]
ETCD_NAME=192.168.0.30
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.0.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.0.11:4001,http://127.0.0.1:4001"


# [cluster]
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_ADVERTISE_PEER_URLS=“http://10.10.0.11:2380"
ETCD_INITIAL_CLUSTER="192.168.0.30=http://10.10.0.11:2380,192.168.0.31=http://10.10.0.10:2380,192.168.0.32=http://10.10.0.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.0.11:4001"
```
针对几个URLS做下简单的解释：
* [member]
ETCD_NAME ：ETCD的节点名
ETCD_DATA_DIR：ETCD的数据存储目录
ETCD_SNAPSHOT_COUNTER：多少次的事务提交将触发一次快照
ETCD_HEARTBEAT_INTERVAL：ETCD节点之间心跳传输的间隔，单位毫秒
ETCD_ELECTION_TIMEOUT：该节点参与选举的最大超时时间，单位毫秒
ETCD_LISTEN_PEER_URLS：该节点与其他节点通信时所监听的地址列表，多个地址使用逗号隔开，其格式可以划分为scheme://IP:PORT，这里的scheme可以是http、https
ETCD_LISTEN_CLIENT_URLS：该节点与客户端通信时监听的地址列表
* [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS：该成员节点在整个集群中的通信地址列表，这个地址用来传输集群数据的地址。因此这个地址必须是可以连接集群中所有的成员的。
ETCD_INITIAL_CLUSTER：配置集群内部所有成员地址，其格式为：ETCD_NAME=ETCD_INITIAL_ADVERTISE_PEER_URLS，如果有多个使用逗号隔开
ETCD_ADVERTISE_CLIENT_URLS：广播给集群中其他成员自己的客户端地址列表

* 192.168.0.31：
```
# [member]
ETCD_NAME=192.168.0.31
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.0.10:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.0.10:4001,http://127.0.0.1:4001"


# [cluster]
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_ADVERTISE_PEER_URLS=“http://10.10.0.10:2380"
ETCD_INITIAL_CLUSTER="192.168.0.30=http://10.10.0.11:2380,192.168.0.31=http://10.10.0.10:2380,192.168.0.32=http://10.10.0.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.0.11:4001"
```

* 192.168.0.32:
```
# [member]
ETCD_NAME=192.168.0.32
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.0.12:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.0.12:4001,http://127.0.0.1:4001"


# [cluster]
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_ADVERTISE_PEER_URLS=“http://10.10.0.12:2380"
ETCD_INITIAL_CLUSTER="192.168.0.30=http://10.10.0.11:2380,192.168.0.31=http://10.10.0.10:2380,192.168.0.32=http://10.10.0.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.0.12:4001"
```
## 配置Master
### apiserver
*/etc/kubernetes/apiserver*
```
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
 KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet_port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://10.10.0.11:4001,http://10.10.0.10:4001,http://10.10.0.12:4001"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=172.42.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS=""
```
### 配置controller-manager
*/etc/kubernetes/controller-manager*
```
KUBELET_ADDRESS="--machines=10.10.0.11,10.10.0.12"

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--service-account-private-key-file=/var/run/kubernetes/apiserver.key --root-ca-file=/var/run/kubernetes/apiserver.crt"
```
### 创建kubernetes集群的证书目录
```
# mkdir -p /var/run/kubernetes
# chown -R kube.kube /var/run/kubernetes
```
### 启动Master上的三个服务
```
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
```
## 配置Node
分别在节点192.168.0.31 , 192.168.0.32上按照以下进行配置。现仅列出一个节点的具体配置。
### kubelet
*/etc/kubernetes/kubelet*
```
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname_override=192.168.0.31"

# location of the api-server
KUBELET_API_SERVER="--api_servers=http://10.10.0.11:8080"

# Add your own!
KUBELET_ARGS="--pod-infra-container-image=docker.io/kubernetes/pause:latest"
```
### config
*/etc/kubernetes/config*
```
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://10.10.0.11:8080"
```
### docker
*/etc/sysconfig/docker*
```
OPTIONS='--selinux-enabled=false --log-level=warning'
```
启动相关服务
```
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```
## 基础架构验证
### 在Master上查看集群信息
```
# kubectl get node
NAME           LABELS                                STATUS
192.168.0.31   kubernetes.io/hostname=192.168.0.31   Ready
192.168.0.32   kubernetes.io/hostname=192.168.0.32   Ready
```
### 创建一个pod
```
# kubectl run my-nginx --image=nginx --replicas=1 --port=80
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR       REPLICAS
my-nginx     my-nginx       nginx      run=my-nginx   1
```
## 网络配置
这里网络部分是以插件的形式配置在kubernetes集群中，这里选用flannel。
### 安装flannel
在node 192.168.0.31、192.168.0.32安装flannel
```
# yum install flannel -y
```
### 配置flannel
```
# cat /etc/sysconfig/flannel
FLANNEL_ETCD="http://10.10.2.120:4001"
FLANNEL_ETCD_KEY="/coreos.com/network"
```
### 为flannel创建分配的网络
```
# etcdctl mk /coreos.com/network/config '{"Network": "172.16.0.0/16"}'
```
重置docker0网桥的配置
删除docker启动时默认创建的docker0网桥，flannel启动时会获取到一个网络地址，并且配置docker0的IP地址，作为该网络的网关地址，如果此时docker0上配置有IP地址，那么flannel将会启动失败。
```
# ip link del docker0
```
开启防火墙使用iptables service

这里需要开启iptables，flanneld使container可以跨节点通信使用的是iptables的NAT功能，所以需要开启iptables，我这里个人比较喜欢使用iptables，对于新的管理服务firewalld，我选择使用iptables来替代。
```
# yum install iptables-services -y
# systemctl disable firewalld
# systemctl stop firewalld
# iptables -F
# iptables-save
# systemctl enable iptables
# systemctl start iptables
```
启动flannel
```
# systemctl enable flanneld
# systemctl start flanneld
```


至此kubernetes集群便配置完成





</br>