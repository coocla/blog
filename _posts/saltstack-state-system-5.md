title: SaltStack使用state进行系统配置-4
categories: SaltStack
tags: [saltstack]
date: 2013-11-22 17:!2:19
---
本文将讲述salt如何通过建立一个file_roots来建立起工作的根，对于不同的"推广""开发""测试""生产"等场景。

Salt fileserver path inheritance
salt允许在同一个环境下设置多个根目录，比如下面的例子，它使用本地的目录以及一个通过NFS共享的目录
In the master config file (/etc/salt/master)<!--more-->
```
file_roots:
  base:
    - /srv/salt
    - /mnt/salt-nfs/base
```
在同一个环境下salt的fileserver可以包含多个根的列表。如果相同的文件在多个根下出现，那么最顶端的根将会生效，例如，如果/srv/salt/foo.txt 和 /mnt/salt-nfs/base/foo.txt都存在，那么使用salt://foo.txt时，这个foo.txt取的是 /srv/salt/foo.txt

Environment configuration
配置多个环境如下：
```
file_roots:
  base:
    - /srv/salt/prod
  qa:
    - /srv/salt/qa
    - /srv/salt/prod
  dev:
    - /srv/salt/dev
    - /srv/salt/qa
    - /srv/salt/prod
```
从上可以看出/srv/salt/prod继承了所有的环境，所以这个路径可以用在任何的环境下。/srv/salt/qa可以用于两个环境下，/srv/salt/dev则只能用再一个环境下。

基于顺序的定义，新的state文件可以放置到/srv/salt/dev下，并且推送到dev主机上进行测试。

这些state文件也可以被移动到/srv/salt/qa下，那么现在他们就可以用再开发和测试环境中，并且允许他们被推送到测试主机上进行测试。

如果这些文件移动到/srv/salt/prod下，那么这些文件将可以用在现在这三个所有的环境下

Practical example
一个简单的网站，被安装到/var/www/foobarcom。下面是一个部署的top.sls：
```
base:
  'web*prod*':
    - webserver.foobarcom
qa:
  'web*qa*':
    - webserver.foobarcom
dev:
  'web*dev*':
    - webserver.foobarcom
```
使用pillar,按照角色分配给主机：
```
/srv/pillar/top.sls:

base:
  'web*prod*':
    - webserver.prod
  'web*qa*':
    - webserver.qa
  'web*dev*':
    - webserver.dev
```
```
/srv/pillar/webserver/prod.sls:

webserver_role: prod
/srv/pillar/webserver/qa.sls:

webserver_role: qa
/srv/pillar/webserver/dev.sls:

webserver_role: dev
```
最后，编写部署网站的sls模块：
```
/srv/salt/prod/webserver/foobarcom.sls:

{% if pillar.get('webserver_role', '') %}
/var/www/foobarcom:
  file.recurse:
    - source: salt://webserver/src/foobarcom
    - env: {{ pillar['webserver_role'] }}
    - user: www
    - group: www
    - dir_mode: 755
    - file_mode: 644
{% endif %}
```
根据上面的这个sls模块所定义，那么网站的源代码就应该放置在下面这个位置：
/srv/salt/dev/webserver/src/foobarcom.

首先，让我们先部署到dev的环境下，使用state.highstate
```
salt --pillar 'webserver_role:dev' state.highstate
```
然而，如果并不想通过top.sls文件将所有的states配置部署到minions(假如当前我们已经当以很多的了state)，使用state.sls仅仅不是foobarcom这个state
```
salt --pillar 'webserver_role:dev' state.sls webserver.foobarcom
```
一旦网站需要部署到dev和qa，那么只需要把文件从/srv/salt/dev/webserver/src/foobarcom移动到/srv/salt/qa/webserver/src/foobarcom，并且执行以下命令：
```
salt --pillar 'webserver_role:qa' state.sls webserver.foobarcom
```
最后网站被qa测试通过，就可以将文件从/srv/salt/qa/webserver/src/foobarcom 移动到/srv/salt/prod/webserver/src/foobarcom，执行以下命令：
```
salt --pillar 'webserver_role:prod' state.sls webserver.foobarcom
```
由于salt的fileserver的继承，即使从/srv/salt/prod移走，你仍然可以通过相同的url(salt://)同时从qa和dev环境下访问







</br>