title: Smokeping的Master.Slave.线上配置
categories: Monitor
tags: [monitor, smokeping slave, nginx]
date: 2013-10-20 20:10:04
---
## 前言
通常情况下，smokeping主机和被监控的主机之间都会存在网络延迟和丢包情况。因此现在比较热火的分布式，多节点监控架构似乎迫在眉睫，而smokeping完全支持master/slave多节点监控
## 优点：
smokeping的主从结构，默认是开启master和slave所有的检测指针去检测远程主机(当然这个选项也是有个参数可以控制，只让slave去检测远程主机)。一个master可以管理多个slave，而且slave配置起来也很简单
slave从master上获取自己的配置信息，所有的检测数据以及web呈现都在master上，slave只负责按照从master获取的配置信息进行数据检测，所以说master/slave的架构也只需要维护好master的配置文件即可，其他的信息slave都会动态获取到。<!--more-->
Smoking 检测分布式的检测方式是被动模式，由从节点启动时获得主节点的config 文件，然后进行数据检测收集，收集完毕后直接将数据提交给主节点。主从通信验证是通过类似于rsync的密码认证方式，在启动slave节点时指定--shared-secret=filename 来和主进行密码验证
## 架构
slave通过master的web接口与master保持正常的通信，slave在初始化启动连接到master的时候，master会告诉slave的作业内容，当slave完成了一轮作业内容时便会将结果返回给master，如果分配给slave的任务发生了改变，master会告诉slave，其他slave已经交付的结果
其实一个slave就是一个单独的实例，slave的配置信息来自于master,不是来自于本地配置文件(这样就减少了大量的维护成本)，slave在完成每一轮的作业任务后，就会尝试连接master提交自己的结果。如果无法连接到master，这个结果将会和下一轮的结果一块发送给master，master收到结果后，将检测的数据存储在一个以perl的可存储形式的文件中，以便于重启了smokeping实例后，不会丢失这些数据。
```

   [slave 1]     [slave 2]      [slave 3]
        |             |              |
        +-------+     |     +--------+
                |     |     |
                v     v     v
              +---------------+
              |    master     |
              +---------------+
```
## Master配置部分
配置一个主从结构，需要在master的配置文件中添加slave的部分，所有slave需要被定义在master的slave区块中(了解配置文件中的几大区块，猛戳这里)并且每一个slave需要用户一个具有唯一名称的菜单名(层次名)，对于slave所设置的章节名一定要和slave的名称保持一致。

### 在配置文件中启用slaves块的配置，并且定义你的slave节点，如下：
```
*** Slaves ***
secrets=/usr/local/smokeping/etc/smokeping_secrets.dist		# 定义通信用的秘钥文件，里面包含slave的名字以及对应密码
 
+ tuosi				# slave的名字
display_name=tuosi	# slave的别名
location=changzhou	# 这个字段用来定义slave主机的位置，类似于description
color=0000ff		# slave收集的数据在图像中显示的颜色，
```
### 将定义的slave节点分配给你需要监控的主机
```
*** Targets ***

++ changzhou		# 定义一个菜单，这个值将会作为data下的一个目录名被创建，属于这个菜单下所有数据都会被存放在这个目录下
menu = 拓思机房		# 定义web上显示的菜单名
title = 拓思机房	# 定义web上显示的头部名

+++ 29gui<-(xxx.xxx.xxx.xxx.xxx)		# 定义一个主机，这个主机的数据将会被存放在　data/changzhou/29gui目录下
menu = 29柜<-(xxx.xxx.xxx.xxx)			# web上显示的菜单名，可点击
title = 29柜<-(xxx.xxx.xxx.xxx)			# 图表头部名称
alerts = someloss						# 报警阀值
slaves = tuosi							# slave节点
host = xxx.xxx.xxx.xxx					# 被监控的主机IP或者域名
```
### 创建master和slave通信的秘钥文件
创建通信用的秘钥文件，内容为“slave的名字:密码”,这里需要注意秘钥文件的权限，由于smokeping的master/slave是通过smokeping程序进行验证的，所以这个秘钥文件的属主必须是smokeping进程的运行用户身份，并且权限为600.下面slave上的密码文件的权限也是一样，需要同样的权限归属，这点需要注意。
```
# echo "tuosi:helloworld" > /usr/local/smokeping/etc/smokeping_secrets.dist
# chown daemon:daemon /usr/local/smokeping/etc/smokeping_secrets.dist
# chmod 600 /usr/local/smokeping/etc/smokeping_secrets.dist
```
## Slave配置部分
slave端实际上不需要太多的配置，只需要将smokeping正确安装即可，具体可参照 Smokeping的配置安装 一文，进行到gmake install 即可~
是不是so easy~
### 创建master与slave的密码文件
```
# echo "helloworld" > /usr/local/smokeping/etc/secrets
# chown daemon:daemon /usr/local/smokeping/etc/secrets
# chmod 600 /usr/local/smokeping/etc/secrets
```
### 启动slave
```
可以使用/usr/local/smokeping/bin/smokeping --help
观察到与slave有关的几个参数如下：
--master-url		# 当smokeping运行在slave模式下，使用该项指定master的访问url（web接口，用以通信）
--slave-name		# 默认情况下，不指定改项时，slave连接到master后，master会以slave的hostname作为slavename，如果不希望这样做，就需要手动指定改选项
--shared-secret		# 和master通信认证的密码文件
--cache-dir			# 当smokeping运行在slave模式下，临时数据存放在master上的目录路径
--pid-dir			# slave模式下，其pid存放的目录路径。可选参数，默认继承--cache-dir参数的值
```
```
# /usr/local/smokeping/bin/smokeping --master-url=http://xxx.xxx.xxx.xxx/smokeping.cgi --cache-dir=/usr/local/smokeping/cache/ --shared-secret=/usr/local/smokeping/etc/secrets --slave-name=tuosi --logfile=/usr/local/smokeping/slave.log 
```
写成脚本，启动方便一点，内容如下：
```
#!/bin/bash
#
# when: 2013/10/18
# who: http://blog.coocla.org

SMKEPING=/usr/local/smokeping/bin/smokeping
MASTERURL=http://xxxx/smokeping.cgi
SLAVENAME=tuosi
CACHEDIR=/usr/local/smokeping/cache
SECRET=/usr/local/smokeping/etc/secrets
LOGFILE=/usr/local/smokeping/slave.log

if [ -f $PIDFILE ] ; then
    PID=`cat $PIDFILE`
    if kill -0 $PID 2>/dev/null ; then
        echo "smokeping is running with PID $PID"
        exit 0
    else
        echo "smokeping not running but PID file exists => delete PID file"
        rm -f $PIDFILE
    fi
else
    echo "smokeping (no pid file) not running"
fi

if $SMKEPING --master-url=$MASTERURL --slave-name=$SLAVENAME --cache-dir=$CACHEDIR --shared-secret=$SECRET --logfile=$LOGFILE > /dev/null; then
    echo "smokeping started"
else
	echo "smokeping could not be started"
fi
```
## 线上配置样例
接下来，来一发重量级的，实战smokeping配置文件内容分享：
```
*** General ***

owner    = daemon
contact  = czlinux@163.com
mailhost = localhost
sendmail = /usr/sbin/sendmail
# NOTE: do not put the Image Cache below cgi-bin
# since all files under cgi-bin will be executed ... this is not
# good for images.
imgcache = /usr/local/smokeping/cache
imgurl   = cache
datadir  = /usr/local/smokeping/data
piddir  = /usr/local/smokeping/var
cgiurl   = http://some.url.com/smokeping.cgi
smokemail = /usr/local/smokeping/etc/smokemail.dist
tmail = /usr/local/smokeping/etc/tmail.dist
# specify this to get syslog logging
syslogfacility = local0
# each probe is now run in its own process
# disable this to revert to the old behaviour
concurrentprobes = yes

*** Alerts ***
from = smokeping@gamewave.net
to = czlinux@163.com

+someloss
type = loss
# in percent
pattern = >3%,*12*,>3%,*12*,>3%
comment = detected loss 3 times over the last two hours
# 定义一个阀值，表示12次检测中。如果有3次大于3%的丢包

+rttdetect5
type = rtt
# in milliseconds
pattern = <5,<5,<5,<5,<5,>5,>5,>5,>5
comment = routing messed up again ?
# 定义一个阀值，表示前五次检测延迟小于5ms，从第六次延迟大于5ms
+rttdetect50
type = rtt
# in milliseconds
pattern = <50,<50,<50,<50,<50,>50,>50,>50,>50
comment = routing messed up again ?
# 定义一个阀值，表示前五次延迟小于50ms,从第六次延迟大于50ms

+rttdetect60
type = rtt
# in milliseconds
pattern = <60,<60,<60,<60,<60,>60,>60,>60,>60
comment = routing messed up again ?
# 定义一个阀值，表示前五次延迟小玉60ms,从第六次延迟大于60ms

+rttdetect80
type = rtt
# in milliseconds
pattern = <80,<80,<80,<80,<80,>80,>80,>80,>80
comment = routing messed up again ?
# 定义一个阀值，表示前五次延迟小玉80ms,从第六次延迟大于80ms
# 小伙伴们可以根据自己的业务重要程度集合网络状况，定义不通的丢包延迟阀值

*** Database ***

step     = 300
pings    = 60

# consfn mrhb steps total

AVERAGE  0.5   1  1008
AVERAGE  0.5  12  4320
    MIN  0.5  12  4320
    MAX  0.5  12  4320
AVERAGE  0.5 144   720
    MAX  0.5 144   720
    MIN  0.5 144   720

*** Presentation ***
charset = UTF-8
template = /data/smokeping/etc/basepage.html.dist

+ charts

menu = Charts
title = The most interesting destinations

++ stddev
sorter = StdDev(entries=>4)
title = Top Standard Deviation
menu = Std Deviation
format = Standard Deviation %f

++ max
sorter = Max(entries=>5)
title = Top Max Roundtrip Time
menu = by Max
format = Max Roundtrip Time %f seconds

++ loss
sorter = Loss(entries=>5)
title = Top Packet Loss
menu = Loss
format = Packets Lost %f

++ median
sorter = Median(entries=>5)
title = Top Median Roundtrip Time
menu = by Median
format = Median RTT %f seconds

+ overview

width = 600
height = 100
range = 12h

+ detail

width = 600
height = 200
#loss_background = yes
unison_tolerance = 2


"最后1天"       1d
"最后1周"       7d
"最后1个月"     30d

++loss_colors			# 定义图像中不同丢包率的表示颜色
0    00ff00    "0"
1    00b8ff    "1/100"
2    0059ff    "2/100"
3    5e00ff    "3/100"
4    7e00ef    "4/100"
5    ffff00    "5/100"
10   ff00ff    "10/100"
50   ff0000    "50/100"
#99   ff0000    "99/100"


*** Probes ***

+ FPing
binary = /usr/sbin/fping

*** Slaves ***
secrets=/data/smokeping/etc/smokeping_secrets.dist

+beijing
display_name=beijing
color=ff0000

+zqdx
display_name=zqdx
color=ff0000

+nbdx
display_name=nbdx
color=ff0000

# 定义三个slave节点
*** Targets ***

probe = FPing
menu = Top
title = blog.coocla.org
remark = 欢迎使用smokeping监控

#####################################################################

+ tuosikeji

menu  = 拓思科技
title = 拓思科技
nomasterpoll = yes

#--------------------------------------
++ tuosiqinye
menu  = 拓思-常州勤业机房
title = 拓思-常州勤业机房

+++ qinye11-beijing
menu  = 北京->11柜(xx.xxx.xxx.xxx)
title = 北京->11柜(xxx.xxx.xxx.xxx)
alerts = rttdetect80,someloss
slaves = beijing
host  = xxx.xxx.xxx.xxx

++++ qinye11-other
menu  = 11柜(xxx.xxx.xxx.xxx)
title = 11柜(xxx.xxx.xxx.xxx)
alerts =
slaves = nbdx zqdx czdx
host  = xxx.xxx.xxx.xxx

+++ qinye16-beijing
menu  = 北京->16柜(xxx.xxx.xxx.xxx)
title = 北京->16柜(xxx.xxx.xxx.xxx)
alerts = someloss,rttdetect80
slaves = beijing
host  = xxx.xxx.xxx.xxx

#-------------------------------------
++ tuosidianxin
menu  = 拓思-常州电信机房
title = 拓思-常州电信机房

+++ dianxinD03-beijing
menu  = 北京->D03-04柜(xxx.xxx.xxx.xxx)
title = 北京->D03-04柜(xxx.xxx.xxx.xxx)
alerts = someloss,rttdetect80
slaves = beijing
host  = xxx.xxx.xxx.xxx

++++ dianxinD03-other
menu  = D03-04柜(xxx.xxx.xxx.xxx)
title = D03-04柜(xxx.xxx.xxx.xxx)
alerts =
slaves = nbdx zqdx czdx
host  = xxx.xxx.xxx.xxx

+++ dianxinD05-beijing
menu  = 北京->D05柜(xxx.xxx.xxx.xxx)
title = 北京->D05柜(xxx.xxx.xxx.xxx)
alerts = someloss,rttdetect80
slaves = beijing
host  = xxx.xxx.xxx.xxx
```
## 总结
1. 定义好smokeping和web服务器的运行用户，因为会涉及一些权限；
2. smokeping的运行用户调用rrdtools进行数据收集绘图，所以存放rrd数据文件，以及图像文件的目录smokeping的运行用户一定要拥有写权限；
3. web前端对于该况图和监控图的展示，是通过web服务器的运行用户通过smokeping.cgi对所绘制的图像进行展示，所以web服务器的运行用户一定也要对图像目录拥有读写选项；
4. 对于web页面的中文显示，无外乎与设置web的字符集以及所在的操作系统要支持对应的字符集；
5. 对于其他的rrd文件不更新，web页面没有图像，或者web页面有图像但没有数据，这些一般都是因为以上权限设置不正确或者rrdtool安装不正确导致的；
6. 对于master/slave的架构，首先也明白其认证的原理，一定要将slave-name和master配置文件中配置的slave节点名以及密钥文件中的节点名相对应；
7. 对于master的密钥文件(包含slave节点名和密码)，slave的密码文件(只有密码)，要这样去思考，这样两个文件的内容都是smokeping通过smokeping的api进行交互通信的，那么交互通信的用户肯定是smokeping的运行用户，因此这两个文件的属主一定要属于smokeping的运行用户，其次要想到rsync的配置，其文件的属性权限一定要是600权限；
8. 对于smokeping的配置文件，由于是用perl写的，大家刚乍一看都觉得乱糟糟的，觉得这玩意很麻烦，其实，去理解了配置文件中各个区块的含义之后，对于整个配置就会一目了然~





</br>