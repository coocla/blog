title: Nginx利用nginx_tcp_proxy_module代理sshd服务
categories: Nginx
tags: [nginx tcp proxy]
date: 2014-09-08 03:31:53
---
Nginx，一款当前拥有“土豪金”身份的web服务器软件和反向代理软件，以其高性能，文档性，丰富功能模块，结构简单，低资源消耗的特性，以绝对性优势拥有“土豪金”这一名誉称号。

Nginx默认不支持基于tcp协议的代理，中午饭后谷歌一把，发现了国内的一个牛人开发了一个tcp代理的第三方模块，赶紧膜拜下！哪天我也能像人家那样挥一挥手写个模块用用那该多好了……

#### 入正题：

我的kvm只有一个公网IP，创建了VM后，VM只分配了一个私有地址，但是我又不想使用kvm的console控制口，以及那极不安全的vnc桌面。因此，代理sshd服务的需求有了。

#### 入手：

sshd服务是基于tcp协议的，因此需要nginx支持tcp协议代理，谷歌一把找到了这个第三方代理模块nginx_tcp_proxy_module，其Github地址：https://github.com/yaoweibin/nginx_tcp_proxy_module

<!-- more -->
#### 安装部分:

下载nginx源码包，下载nginx_tcp_proxy_module模块源码
```
# wget http://nginx.org/download/nginx-1.2.1.tar.gz
# git clone https://github.com/yaoweibin/nginx_tcp_proxy_module.git
# tar zxf nginx-1.2.1.tar.gz
# cd nginx-1.2.1
# patch –p1 < /root/nginx_tcp_proxy_module/tcp.patch
```
> 注：这里有个问题，刚开始我使用的nginx的源码包在进行安装补丁的时候提示如下错误：
 

```
can't find file to patch at input line 5
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|diff --git a/src/core/ngx_log.c b/src/core/ngx_log.c
|index c0485c6..bfb1f5c 100644
|--- a/src/core/ngx_log.c
|+++ b/src/core/ngx_log.c
--------------------------
File to patch:
```
后来更换了源码包后再进行补丁安装后就正常了


在你本机nginx的编译选项上增加以下参数：

```
--add-module=/root/nginx_tcp_proxy_module
```

随后，make && make install

 
### 配置部分：

根据个人习惯，以下为我的配置，小伙伴们根据语法习惯自行修改即可

1. 在nginx的主配置文件nginx.conf中的一级块中添加如下内容：
```
tcp {
    include tcp_proxy/*.conf;
}
```
2. 在其相对位置创建tcp_proxy目录，并在该目录下创建一个配置文件，名为:proxy_sshd.conf，内容如下：
```
upstream sshd {
    server 192.168.6.110:22;
}

server {
    listen 22;
    proxy_pass sshd;
}
```
3. Description

其实它的配置和nginx的配置是严格一致的，它所带的一些参数配置我了解的还不多，不过在Github上作者也做出的有详细介绍。各位小伙伴可以去感受下。再附传送门：https://github.com/yaoweibin/nginx_tcp_proxy_module





