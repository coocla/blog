title: 搭建私有的docker镜像仓库registry
categories: Docker
tags: [docker, registry]
date: 2015-08-01 12:16:02
---
Docker的创建及运行依赖于image(镜像)，而image的来源与存储媒介是docker hub，docker hub是官方公布出来的一个公共的image仓库，而对于企业应用，因为各种原因，这个公共docker hub很明显并不能解决和满足各地企业/个人的使用。

Docker官方在github上有一个项目 docker-registry，其作用是可以提供类docker hub功能的服务。企业或个人可以在自己的网络环境下，利用docker-registry搭建私有的docker hub。<!--more-->

## 安装docker-registry
系统环境：Centos 7
架构组成：docker registry(1台)   docker(2台)
配置文件：*/etc/docker-registry.yml  /etc/sysconfig/docker-registry*
```
# yum install docker-registry
```
docker-registry采用sqlite存储仓库中的元数据，sqllite的db file默认将数据保存在/tmp目录下，修改存储目录为/data/docker-registry
```
# mkdir -p /data/docker-registry
# sed -i '/sqlalchemy_index_database/{s@tmp@data/docker-registry@g}' /etc/docker-registry.yml
```
docker-registry中的镜像数据存储默认采用本地存储，存储在/var/lib/docker-registry
可以修改该路径将镜像存储在其他目录下
这里支持多种存储方式:
* local  - 本地文件系统
* s3  - AWS S3
* ceph-s3 -  ceph集群，使用S3的API
* azureblob -  Microsoft Azure Blob Storage
* gcs  - Google cloud
* swift  - OpenStack Swift
* glance  - OpenStack Glance
* elliptics  - Elliptices key/value

修改docker-registry的监听地址为127.0.0.1
```
sed -i '/REGISTRY_ADDRESS=/{s@\(.*=\).*@\1127.0.0.1@g}'/etc/sysconfig/docker-registry

# systemctl enable docker-registry
# systemctl start docker-registry
# systemctl -l staus docker-registry
```
加入开机启动，并且启动docker-registry，查看启动状态

验证运行是否正常
```
# curl 127.0.0.1:5000
```
## 配置Nginx代理，及添加Basci Auth的验证
```
# yum install nginx httpd-tools -y
```
生成认证文件
```
# mkdir /etc/nginx/private
# htpasswd -cb /etc/nginx/private/docker-auth.pass docker  dockerPass
```
配置nginx代理
*/etc/nginx/nginx.conf*
```
upstream docker-registry {
    server 127.0.0.1:5000;
}
```
*/etc/nginx/conf.d/docker-registry.conf*
```
server {
    listen 5000;
    server_name docker.coocla.org;
    access_log /data/logs/docker_access.log main;
    error_log /data/logs/docker_error.log;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    client_max_body 0;
    
    location / {
        auth_basic "Auth";
        auth_basic_user_file private/docker-auth.pass;
        proxy_pass http://docker-registry;
    }
    location /_ping {
        auth_basic off;
        proxy_pass http://docker-registry;
    }
    location /v1/_ping {
        auth_basic off;
        proxy_pass http://docker-registry;
    }
}
```
验证Nginx代理是否配置成功
```
# curl -u docker:dockerPass http://docker.coocla.org:5000
```
## 为Nginx代理配置ssl加密
生成CA的key
```
# mkdir certs && cd certs
# openssl genrsa -out CA.key 2048
```
通过生成的CA的key生成一个CA的证书
```
# openssl req -x509 -new -nodes -key CA.key -days 10000 -out CA.crt
```
为web服务的域名生成一个key
```
# openssl genrsa -out docker.coocla.org.key 2048
```
利用生成的docker.coocla.org.key生成对应的证书
```
# openssl req -new -key docker.coocla.org.key -out docker.coocla.org.csr
```
> 这里在填写Common Name的时候一定要与nginx的server_name保持一致

利用之前的CA的证书为域名的证书进行签发
```
# openssl x509 -req -in docker.coocla.org.csr -CA CA.crt -CAkey CA.key -CAcreateserial -out docker.coocla.org.crt -days 10000
```
将通过CA签发后的证书和对应的key拷贝到nginx的配置目录下
```
# cp docker.coocla.org.crt docker.coocla.org.key  /etc/nginx/private
```
配置Nginx开启ssl
/etc/nginx/conf.d/docker-registry.conf
```
server {
    listen 443;
    …
    ssl on;
    ssl_certificate /etc/nginx/private/docker.coocla.org.crt;
    ssl_certificate_key /etc/nginx/private/docker.coocla.org.key;
}
```
重载Nginx配置文件
```
# systemctl reload nginx
```
> 因为nginx的证书是由一个不被信任的CA(咱们自己随便生成的)签发的，因此其他服务器在访问的时候，会被提示证书不可信，请求会中断。因此需要将刚才咱们生成的CA的证书，加入到客户端的CA证书列表中。

将之前生成的CA证书的内容，追加到客户端的可信任CA证书列表中
CA证书内容类似：
```
-----BEGIN CERTIFICATE-----
MIID3TCCAsWgAwIBAgIJAPDDWEAD3cmpMA0
...
-----END CERTIFICATE-----
```
追加到客户端的可信任CA证书列表中*/etc/ssl/certs/ca-bundle.crt*
测试新添加的证书是否已经生效
```
# docker login https://docker.coocla.org
```
输入账号密码后。如果出现Login Succeeded字样则表示成功。
## 使用新搭建的docker-registry
从mirror中获取一个镜像，具体设置方法详见：http://dockone.io/article/160
获取一个镜像
```
# docker pull ubuntu
```
通过这个镜像运行一个容器
```
# docker -it ubuntu /bin/bash
```
在该容器中创建一个文件
```
# echo 'Hello ubuntu'  >  /root/new_image
```
退出该容器
```
# exit
```
将该容器当前的状态提交为一个镜像
```
# docker commit $(docker ps -ql) hello-ubuntu
```
> 执行上述命令时，请确保当前服务器上有且仅有一个容器，查看当前服务器上的容器列表：docker ps --all

查看当前镜像列表
```
# docker images
```
你应该发现现在有一个名为hello-ubuntu的新镜像
docker在推送新的镜像时，必须为这个镜像打上一个标签
```
# docker tag hello-ubuntu docker.coocla.org/hello-ubuntu
```
标签一般由四部分组成: [REGISTRYHOST/][USERNAME/]NAME[:TAG]
打好标签后将该镜像推送到我们的私有registry中
```
# docker push docker.coocla.org/hello-ubuntu
```
在另外一台客户端上，我们用新推送的镜像启动一个容器，并检查容器中/root下是否存在new_image的文件，并且文件内容为 Hello ubuntu
```
# docker run -d docker.coocla.org/hello-ubuntu cat /root/new_image
```
重新登录docker-registry机器，查看/var/lib/docker-registry/
该目录由images和repositories两个目录组成。

* images:用来存放每一层镜像相对于上一次数据改变的元数据
* repositories:用来存放整个registry中镜像的属性，索引，标签信息

由此也可以说docker registry其实是由两部分组成：Index，registry
在执行docker pull的命令时，首先会从index中获取镜像的信息，然后根据这些信息，通过registry将镜像拉取到本地。
 

到此，一个私有的docker hub就搭建完成了。






</br>