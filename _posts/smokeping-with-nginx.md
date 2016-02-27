title: CentOS中Smokeping+Nginx搭建
categories: Monitor
tags: [monitor, smokeping, nginx]
date: 2013-10-14 18:23:00
---
## 简介
smokeping是rrdtool的作者Tobi Oetiker的作品，采用多种方式对网络性能进行监控检测并告警，它支持较多的插件方式对网络的多项指标进行监控，并且支持Matser/Slave的架构，多个节点的监控数据可以在同一个图上展现。通过颜色和阴影表示网络延迟和丢包情况，图像很漂亮。适用于对多IDC机房网络的监控和网络性能的采集分析。<!--more-->

## Smokeping安装部分
安装smokeping依赖的一部分perl模块还有一些其他检测组件
```
# yum install rrdtool fping echoping curl perl-Net-Telnet perl-Net-DNS perl-LDAP perl-CGI-SpeedyCGI perl-libwww-perl perl-RadiusPerl perl-IO-Socket-SSL perl-Socket -y
# mkdir /root/packages
# wget http://oss.oetiker.ch/smokeping/pub/smokeping-2.6.8.tar.gz -P /root/packages
# cd /root/packages
# tar zxf smokeping-2.6.8.tar.gz
# cd smokeping-2.6.8
# ./configure --prefix=/usr/local/smokeping
```
抛出如下错误：
```
** Aborting Configure ******************************

If you know where perl can find the missing modules, set
the PERL5LIB environment variable accordingly.

FIRST though, make sure that 'perl' starts the perl
binary you want to use for SmokePing.

Now you can install local copies of the missing modules
by running

./setup/build-perl-modules.sh /usr/local/thirdparty

The RRDs perl module is part of RRDtool. Either use the rrdtool
package provided by your OS or install rrdtool from source.
If you install from source, the RRDs module is located
PREFIX/lib/perl
```
根据提示，运行./setup/build-perl-modules.sh /usr/local/thirdparty
可以查看./setup/build-perl-modules.sh脚本内容，发现其实就是在安装smokeping所依赖的一些perl模块
```
# export PERL5LIB=/usr/local/smokeping/thirdparty/lib/perl5/
# ./configure --prefix=/usr/local/smokeping
# /usr/bin/gmake install
```
## Smokeping简单配置部分
创建主配置文件
```
# cp /usr/local/smokeping/etc/config.dist /usr/local/smokeping/etc/config
# vim /usr/local/smokeping/etc/config
```
* *** :标示的区块属于不同类型的内容
* + :代表一级菜单 +下面的++是继承上面的+成为二级菜单。 而且可以有多个一级菜单和二级菜单。

第一部分General区块，属于基本配置
```
owner = daemon smokeping运行的用户
contact = admin@coocla.org smokeping管理员联系邮件地址
mailhost = localhost 邮件服务器地址
sendmail = /usr/sbin/sendmail 发送邮件件的二进制可执行程序
imgcache = /usr/local/smokeping/cache smokeping图片缓存
imgurl = cache 定义cgi程序显示图片的url目录
datadir = /usr/local/smokeping/data smokeping保存rrd文件的目录
piddir = /usr/local/smokeping/var 顾名思义，其pid目录
cgiurl = http://some.url/smokeping.cgi 完整的smokeping的url路径
smokemail = /usr/local/smokeping/etc/smokemail.dist 发送邮件的邮件内容模板
tmail = /usr/local/smokeping/etc/tmail.dist HTML邮件模板的路径
syslogfacility = local0 syslog日志记录的设备编号
```
第二部分Alter区块，属于报警配置
```
to = admin@coocla.org 报警邮件接收人地址
from = smokealert@company.xy 报警邮件发送人地址

+someloss 定义一个名为someloss的报警
type = loss 类型为丢包
pattern = >0%,*12*,>0%,*12*,>0% 对监控内容进行阀值的模式匹配
comment = loss 3 times in a row 检测12次，出现了3次丢包的情况，不论丢多少个包，就进行报警
```
第三部分Database区块，属于RRDTOOL数据库的配置
```
step = 300 步长，也就是多长时间为一个周期
pings = 20 ping的次数，这两项的组合意思是，每300秒进行20此的ping操作

# consfn mrhb steps total
AVERAGE 0.5 1 1008
AVERAGE 0.5 12 4320
MIN 0.5 12 4320
MAX 0.5 12 4320
AVERAGE 0.5 144 720
MAX 0.5 144 720
MIN 0.5 144 720
```
第四部分Presentation区块，属于网络状态，web显示的一些配置项
这块还没看，暂时先放这

第五部分Probes区块，属于Fping指针的配置
```
+ FPing
binary = /usr/sbin/fping
```
第六部分Slave区块，属于Matster,Slave架构的配置
暂时先将其注释起来，并连带上下文中所涉及的项注释下，否则待会启动时会报错

第七部分Targets区块，属于监控目标的配置
```
probe = FPing 指定监控指针

menu = Top 定义一个菜单，类型为Top，titile 注释等信息，均可自定义
title = Network Latency Grapher
remark = Welcome to the SmokePing website of xxx Company. \
Here you will learn all about the latency of our network.

+ Test
menu= Targets 定义一个一级菜单
#parents = owner:/Test/James location:/

++ James 定义一个主机为James
menu = James 菜单名为James
title =James
alerts = someloss 报警阀值为上文定义的someloss
#slaves = boomer slave2
host = blog.coocla.org 监控的主机blog.coocla.org
```
创建必要的目录
```
# mkdir /usr/local/smokeping/{var,cacahe,data}
# chown daemon.daemon -R /usr/local/smokeping
```
下载启动脚本，并启动smokeping
```
# wget http://oss.oetiker.ch/smokeping/pub/contrib/smokeping-start-script -P /etc/init.d/smokeping
# chmod +x /etc/init.d/smokeping
# /etc/init.d/smokeping start

注：请按照自己的安装环境修改脚本中的smokeping命令的路径以及pid的存放路径
```
1. 观察cache目录下回立即生成两个文件rrdtool.png smokeping.png和刚才配置文件中以定义的那个监控主机的名字为名的目录被创建
2. data目录下也会生成一个和定义的主机名一致的目录，并且观察目录里面的rrd文件每5分钟是否正常更新一次

## Nginx-cgi配置安装部分
由于nginx不能直接执行外部可执行程序，需要nginx支持CGI，也就是将CGI的处理请求反向代理到后端的处理器，对于CGI的请求处理可以使用nginx-fcgi，其项目地址是：https://aur.archlinux.org/packages/nginx-fcgi/?setlang=zh_CN
这里我们只需要该项目中的nginx-fcgi.txt，nginx-fcgi.txt是一个用Perl脚本写的wrapper实例，所以，操作系统必须要安装Perl程序以及相关模块，下载下来后，可以看到其中有以下部分内容
use FCGI;
use Getopt::Long;
use IO::All;
use Socket;

说明其依赖于 FCGI，Getopt::Long，IO::All，Socket 这些模块，安装
```
# wget http://search.cpan.org/CPAN/authors/id/F/FL/FLORA/FCGI-0.74.tar.gz
# wget http://search.cpan.org/CPAN/authors/id/G/GB/GBJK/FCGI-ProcManager-0.19.tar.gz
# wget http://search.cpan.org/CPAN/authors/id/J/JV/JV/Getopt-Long-2.42.tar.gz
IO-All 依赖于 IO-String
# wget http://search.cpan.org/CPAN/authors/id/F/FR/FREW/IO-All-0.48.tar.gz
# wget http://search.cpan.org/CPAN/authors/id/G/GA/GAAS/IO-String-1.08.tar.gz
Socket 依赖于 ExtUtils::Constant
# wget http://search.cpan.org/CPAN/authors/id/P/PE/PEVANS/Socket-2.012.tar.gz
# wget http://search.cpan.org/CPAN/authors/id/N/NW/NWCLARK/ExtUtils-Constant-0.23.tar.gz

# tar zxf FCGI-0.74.tar.gz
# cd FCGI-0.74
# perl Makefile.PL
# make && make install
```
按照这种方式进行安装以上几个包，根据个人环境，仔细解决依赖关系。全部正常安装完后。

将nginx-fcgi.txt重命名为nginx-fcgi并放置于/etc目录下，创建nginx-fcgi启动脚本fcgi
```
# mv nginx-fcgi.txt /etc/nginx-fcgi
# vim /etc/init.d/fcgi
# chmod +x /etc/init.d/fcgi
# chkconfig --add fcgi
# /etc/init.d/fcgi start
```
## Nginx配置部分
根据Nginx官方文档内容http://wiki.nginx.org/SimpleCGI，创建cgi配置文件
```
# vim /usr/local/nginx/conf/
fastcgi_param QUERY_STRING $query_string;
fastcgi_param REQUEST_METHOD $request_method;
fastcgi_param CONTENT_TYPE $content_type;
fastcgi_param CONTENT_LENGTH $content_length;
fastcgi_param GATEWAY_INTERFACE CGI/1.1;
fastcgi_param SERVER_SOFTWARE nginx;
fastcgi_param SCRIPT_NAME $fastcgi_script_name;
fastcgi_param REQUEST_URI $request_uri;
fastcgi_param DOCUMENT_URI $document_uri;
fastcgi_param DOCUMENT_ROOT $document_root;
fastcgi_param SERVER_PROTOCOL $server_protocol;
fastcgi_param REMOTE_ADDR $remote_addr;
fastcgi_param REMOTE_PORT $remote_port;
fastcgi_param SERVER_ADDR $server_addr;
fastcgi_param SERVER_PORT $server_port;
fastcgi_param SERVER_NAME $server_name;
```
创建虚拟主机配置文件
```
# vim /usr/local/nginx/conf/vhosts/smokeping.conf
server
{
listen 8090;
server_name smokeping.coocla.org;
root /usr/local/smokeping/;
access_log /data/logs/smokeping.access.log main;
error_log /data/logs/smokeping.error.log;

location ~ .*\.(cgi|fcgi)$ {
root /usr/local/smokeping/htdocs/;
fastcgi_index index.cgi;
fastcgi_pass unix:/tmp/nginx-fcgi.sock;
include fcgi_params;
}
}
# /usr/local/nginx/sbin/nginx -s reload
```

访问，http://IP/smokeping.fcgi 即可访问到如下信息
[](http://7xk38j.com1.z0.glb.clouddn.com/smokeping/smokeping2.png) 