title: 博客从Typecho迁移到hexo
categories: 其他
tags: [hexo]
date: 2016-02-26 16:11:07
---
## 前言
之前博客都是运行在wordpress/Typecho上面，这两种框架都是基于php编写，依赖于lnmp, 加之服务器网络有时不太稳定, 访问速度就时好时坏, 前段时间还被别人注入了php木马, 搞了很长时间才清理干净. 甚是头疼.

另外就是每当写博客发表时都要在重新排版, 于是索性换了*hexo* 其基于nodejs编写, 由markdown直接转换为纯静态的html, 访问速度会快一点, 安全等方面也不需要考虑太多. 本地编写结合github管理, 简直就是完美.

整体结构/流程:
本地---> github ---> vps

本地推送到github后由github的webhook来触发vps的更新部署脚本, 于是首先就想到了ngx_lua, 在nginx配置一个url */hookdeploy* 然后由lua来触发执行更新脚本.
<!--more-->
由于当前vps上nginx不支持lua所以需要先为当前nginx增加lua模块。

## 安装依赖包LuaJIT, nginx_lua, ngx_devel_kit
```
# wget http://luajit.org/download/LuaJIT-2.0.4.tar.gz
# tar zxf LuaJIT-2.0.4.tar.gz
# cd LuaJIT-2.0.4
# make && make install 
# cd ../
# wget https://codeload.github.com/openresty/lua-nginx-module/tar.gz/v0.10.0 -O nginx-lua-v0.10.0.tar.gz
# tar zxf nginx-lua-v0.10.0.tar.gz
# wget https://codeload.github.com/simpl/ngx_devel_kit/tar.gz/v0.2.19 -O ngx_devel_kit-v0.2.19.tar.gz
# tar zxf ngx_devel_kit-v0.2.19.tar.gz
```
下载当前nginx版本的源码包, 我这里就使用1.7.8版本

```
# wget http://nginx.org/download/nginx-1.7.8.tar.gz
# tar zxf nginx-1.7.8.tar.gz
# cd nginx-1.7.8
# ./configure --add-module=../ngx_devel_kit-v0.2.19 --add-module=../nginx-lua-v0.10.0
# make
```
> 使用nginx -V 查看当前版本nginx的编译参数

替换新编译的nginx二进制文件

```
# cp objs/nginx /usr/local/nginx/sbin/nginx
```

## 编写hexo的更新和部署脚本hookdeploy.sh
```
#!/bin/bash
ROOT=/data/www/wwwroot/coocla_new

function Pull(){
  cd $ROOT/source
  git pull origin master
}

function Deploy(){
  cd $ROOT
  hexo g
}

Pull
if [ $? -eq 0 ];then
  Deploy
fi
```

## 配置nginx的hook
```
location /hookdeploy {
    access_by_lua '
        local f = io.popen("/bin/bash", "/tmp/hookdeploy.sh")
        ngx.say("I get it")
    '
}
```