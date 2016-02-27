title: SaltStack使用state进行系统配置
categories: SaltStack
tags: [saltstack, state]
date: 2013-11-19 10:55:47
---
如何使用salt states快速的配置一个系统。

本文将指导完成使用salt来配置一个minion运行apache http服务器并且使这个服务正确的运行。

在往下面进行之前，请确保你的salt已经安装并且已经工作。

Setting up the salt sate tree
states以文本文件的形式存储在master上，并且按照需求通过master的文件服务器传输给minions。state文件的集合构成了state tree<!--more-->

在使用salt时必须为其至少设置一个所谓的中央系统也就是state tree的第一个顶级点。编辑master的配置文件将以下行取消注释：
```
file_roots:
  base:
    - /srv/salt
```
> 如果你在部署FreeBSD的操作系统，这个 file_roots参数默认值为：/usr/local/etc/salt/states
 

重启salt master使这个配置改变生效：
```
pkill salt-master
salt-master -d
```

在master上，上一步中的目录并没有被创建，(默认/srv/salt)，创建一个名为top.sls文件，并添加一下内容：
```
base:
  '*':
    - webserver
```
这个top.sls文件是用来分隔环境的。默认环境是base，在base的环境集合定义了对minions的匹配，可以使用简单的*来指定所有的主机。

目标minions
salt可以通过glob、pcre正则表达式，或者通过grains来进行匹配任何的目标机器，例如：
```
base:
    'os:Fedora':
        - match: grain
        - webserver
```
在top.sls相同的目录下，创建一个名为webserver.sls的空文件，内容如下：
```
apache:                 # ID的声明
    pkg：               # state类型的声明
        - installed     # state中的函数声明
```
第一行，是对全局ID的声明，可以是任意的标识符，在这种情况下，使用需要被安装的软件包的包名来定义。
> 对于apache httpd web服务器的包名取决于操作系统的发行版，例如，在Fedora是httpd，但在Debian/Ubuntu是apache2
 

第二行，是对state的声明，定义了我们所要十一on个的salt states，在这个例子中，我们使用pkg state以确保一个给定的包被正确安装。
第三行，称为函数的声明，这个函数被定义在pkg state这个模块中。

Renderers渲染器
states的sls文件可以写成很多格式，salt只依赖于很简单的数据结构，并且不关心这些数据结构的建立，模版语言和DSLs到处都有，每个人都喜欢。
建立数据结构可以很简单的编写预期的任务，在本文中我们在jinja2的模版中使用YAML，这也是默认的格式。可以在master配置文件中修改默认渲染器

Install the package
接下来运行我们创建的state,在master上打开一个终端并且运行一下命令：
```
% salt '*' state.highstate
```
master将会指导所有的目标minions运行 state.highstate。当minion执行highstate，它将会下载top文件中匹配的内容，minion将表达式中匹配的内容下载、编译、执行。
一旦完成，minion将返回所有的动作执行结果和所有更改

```
SLS File Namespace(sls文件名称空间)
这个例子中，这个webserver.sls文件可以简单的写作webserver。这个sls文件名称空间遵循一些简单的规则：
1. 这个.sls可以被省略，例如webserver.sls可以被称谓webserver
2. 子目录可以更好的组织.
    a. 每个子目录都代表一个点
    b. webserver/dev.sls可以写作webserver.dev
3. 在子目录的存在一个名为init.sls文件，所以webserver/init.sls也可以简写为webserver
4. 如果webserver.sls和webserver/init.sls同时存在，那么webserver/init.sls将会被忽略，并且webserver.sls将会被成为webserver
```
Troubleshooting Salt
如果输出的内容不是你所预期的，以下的建议可以帮助你缩小问题。

启用日志记录
当你启用日志的debug模式，salt输出的信息将会很全
```
salt-minion -l debug
```
前台运行minion
不使用daemon模式(-d)启动minion,可以从输出看到其工作的任何细节
```
salt-minion &
```
增加salt运行的默认超时时间，例如，修改默认超时时间为60s：
```
salt -t 60
```
为了更好的达到这三个效果：
```
salt-minion -l debug &                  # 在minion上运行
salt '*' state.highstate -t 60          # 在master上运行
```





</br>