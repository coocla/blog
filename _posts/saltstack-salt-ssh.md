title: SaltStack又一新功能salt-ssh
categories: SaltStack
tags: [saltstack]
date: 2013-12-13 12:04:44
---
在0.17.0版本，salt引入了一个新的传输系统salt-ssh，这个系统使用ssh协议，并且使salt完全基于ssh进行远程命令执行，在salt-master进行远程调用就绕过了salt-minion。

注意：salt ssh系统并不完全取代salt的通信系统。它只是简单的基于ssh协议，不在依赖于ZeroMQ和salt-minion，但是由于salt ssh完全是基于ssh协议的，所以在进行远程执行，ssh的执行效率要远远低于ZeroMQ

salt ssh是很容易使用的。简单的创建一个所需连接的操作系统的roster文件，并且运行salt-ssh命令以类似的方式作为salt的命令。

SALT SSH ROSTER
用作远程执行的这个roster系统在salt中很容易定义
创建roster系统是因为salt-ssh需要依赖这个名单来确定哪些系统如何连接并且执行。
因为这个roster系统属于“热插拔”类型，它可以增强附着在任何现有的系统中来收集关于哪些服务器目前可用以及可以使用salt-ssh来连接的信息，默认这个名单文件在/etc/salt/roster

HOW ROSTERS WORK
roster系统，将targets编译成一个数据结构提供给salt ssh。这个targets是一个列表，并且包含如何连接到这个系统的各种属性。salt对roster module唯一的要求就是要求targets返回数据结构。<!--more-->

Targets Data
这个roster target的信息可以像下面的格式存储：
```
<Salt ID>:   # target的id，类似于minion配置文件中"id"字段选项
    host:    # 远程主机的IP地址或者域名
    user:    # 登陆远程主机的用户名
    passwd:  # 对于用户名的面膜

    # 可选参数
    port:    # 远程主机的ssh端口
    sudo:    # 通过sudo命令运行命令
    priv:    # ssh的私钥文件的路径, 默认是salt-ssh.rsa
    timeout: # ssh等待响应的超时时间
```
简单的创建roster文件，默认文件/etc/salt/roster
```
web1: 192.168.42.1
```
上面是一个十分基本的roster文件，包含salt id对应的ip地址。一个包含更多信息的roster文件如下：
```
web1:
  host: 192.168.42.1 # IP或者域名
  user: fred         # 远程命令执行的用户身份
  passwd: foobarbaz  # ssh登陆密码，如果省略将会使用秘钥登陆
  sudo: True         # 是否sudo到root，默认不开启
web2:
  host: 192.168.42.2
```
CALLING SALT SSH
salt-ssh命令可以很容易执行相同的salt命令
```
salt-ssh '*' test.ping
```
salt-ssh和salt遵循相同的命令语法，并且可以使用标准的函数，输出内容和salt一样，并且许多相同的flags是可用的，具体可以查看http://docs.saltstack.com/ref/cli/salt-ssh.html 可用的选项。

Raw Shell Calls
默认salt-ssh在远程系统上运行salt远程支持模块。但是salt-ssh也可以运行原始的shell命令：
```
salt-ssh '*' -r 'ifconfig'
```
STATES VIA SALT SSH

salt-ssh也可以运行salt state。当使用salt-ssh时，state系统抽象出和salt相同的接口。定义标准的salt公式的目的是将salt-ssh和salt进行无缝的工作，反之亦然。
标准的salt states walkthroughs函数只需简单的用salt-ssh命令代替salt命令

TARGETING WITH SALT SSH
事实上salt-ssh于salt的目标匹配方法不同，在写的时候只支持glob和regex，其他的匹配模式也将逐步实现

RUNNING SALT SSH AS NON-ROOT USER
默认情况下，salt读取/etc/salt下的所有配置文件，如果使用普通用户运行salt ssh或者需要改变一些目录的权限，那么将会返回一个“没有权限”的消息。你必须修改两个参数：pki_dir，cachedir。这些都应该是这个普通用户具有写权限的完整路径。






</br>