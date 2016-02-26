title: Ansible主机与组的定义Inventory
categories: Ansible
tags: [ansible inventory]
date: 2014-07-16 10:11:07
---
## Inventory

Ansible的Inventory文件，可以理解为saltstack中的salt-key中的所有minion的列表以及用户自定义的nodegroup的概念，默认情况下这个文件是/etc/ansible/hosts，到目前为止，以上仅仅是Inventory文件的小小一部分作用，其实他的作用远远不止这些:)
Hosts and Groups

对于/etc/ansible/hosts最简单的定义格式像下面:<!--more-->
```
mail.lightclooud.com

[webservers]
web1.lightcloud.com
web2.lightcloud.com

[cloudservers]
cloud1.lightcloud.com
cloud2.lightcloud.com
```
* 想当然，标准的FQDN自然表示的是主机的主机名或者域名
* 中括号中的名字代表组名，你可以根据你自己的需求将庞大的主机分成具有标识的组。

如果某些主机的SSH运行在自定义的端口上，ansible使用Paramiko进行ssh连接时，不会使用你SSH配置文件中列出的端口，但是如果修改ansible使用openssh进行ssh连接时将会使用：
```
badwolf.lightcloud.com:5309
```
假如你想要为某些静态IP设置一些别名，类似于SaltStack中minion配置文件中id的参数配置。你可以这样做：
```
jumper ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50
```
假如你有很多主机遵循某一种模式，你还可以这样来表示他们：
```
[webservers]
web[1:50].lightcloud.com
```
* 表示从web1到web50，共计50台主机

当然你还可以通过字母来定义可变的范围，例如：
```
[database]
db-[a:f].lightcloud.com
```
对于每台主机的连接类型，连接用户等这些你都可以完全自定义，例如：
```
[targets]
localhost ansible_connection=local
cloud1.lightcloud.com ansible_connection=ssh ansible_ssh_user=light
cloud2.lightcloud.com ansible_connection=ssh ansible_ssh_user=cloud
```
而对于上面的定义方式，只能算是我们针对每台主机的一个快速定义，随后将会讲述如何在host_vars目录下，以单个文件存储它们。
Host Variables

如上面提到，我们目前可以轻易的给某台或多台主机赋予变量，提供给后面的playbooks使用：
```
[lightcloud]
web1 http_port=202 maxRequestsPerChild=404
web2 http_port=303 maxRequestsPerChild=606
```
## Groups Variables

变量也可以通过组名，应用到组内的所有成员：
```
[lightcloud]
host1
host2

[lightcloud:vars]
ntp_server=ntp.lightcloud.com
proxy=proxy.lightcloud.com
```
## 组的子组,组变量
主机组可以包含主机组，主机的变量可以通过继承关系，继承到最高等级的组的变量。定义主机组之间的继承关系我们使用":children"来表示：
```
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeas:children]
atlanta
raleigh

[southeas:vars]
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```
## Splitting Out Host and Group Specific Data
Ansible并不建议将主机的变量都存储在Inventory文件中，这些变量也可以存储在单个文件中，而这些变量文件必须遵循YAML语法，假设Inventory文件没有被修改，路径为*/etc/ansible/hosts*

如果主机的名字“foosball”,两个组的名字分别是“raleigh”，“raleigh”，以下路径的yaml文件中的变量将会应用到对应的主机上：
```
/etc/ansible/group_vars/raleigh
/etc/ansible/group_vars/webservers
/etc/ansible/host_vars/foosball
```
> Tip：在Ansible 1.2或更高版本中，group vars和host

vars目录可以存在于playbook目录或者Inventory目录，如果两个目录都存在同一主机的相同定义，那么playbook目录将会第二次被加载，也就是说playbook中定义的将会覆盖Inventory中对应的定义
List of Behavioral Inventory Parameters

Ansible内置了一些关于连接主机的变量，设置以下变量控制ansible与远程主机：

* ansible_ssh_host  ansible通过ssh连接的IP或者FQDN
* ansible_ssh_port  SSH连接端口
* ansible_ssh_user  默认SSH连接用户
* ansible_ssh_pass  SSH连接的密码（这是不安全的，ansible极力推荐使用--ask-pass选项或使用SSH keys）
* ansible_sudo_pass  sudo用户的密码
* ansible_connection  SSH连接的类型：local,ssh,paramiko，在ansible 1.2之前默认是paramiko，后来智能选择，优先使用基于ControlPersist的ssh（支持的前提）
* ansible_ssh_private_key_file  SSH连接的公钥文件
* ansible_shell_type  指定主机所使用的shell解释器，默认是sh，你可以设置成csh, fish等shell解释器
* ansible_python_interpreter  用来指定python解释器的路径
* ansible\_\*\_interpreter  用来指定主机上其他语法解释器的路径，例如ruby，perl等

例如:
```
some_host         ansible_ssh_port=2222     ansible_ssh_user=manager
aws_host          ansible_ssh_private_key_file=/home/example/.ssh/aws.pem
freebsd_host      ansible_python_interpreter=/usr/local/bin/python
ruby_module_host  ansible_ruby_interpreter=/usr/bin/ruby.1.9.3
```
