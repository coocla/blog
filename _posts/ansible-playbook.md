title: Ansible配置管理play-book
categories: Ansible
tags: [ansible,playbook]
date: 2014-09-08 11:31:07
---
## Play-book
Ansible是一个完全简单的模型驱动的配置管理、部署和命令执行框架;
对于ansible的playbook就是完全的配置管理框架，它类似于saltstack的state system;
对于被管理的节点不需要安装任何软件依赖及ansible.

使用playbook的准备工作:
* 选择一台机器作为你的管理节点，并且安装ansible
* 确保你的这台机器上拥有你所要管理机器的SSH秘钥，因为ansible使用SSH登录管理
* 创建主机文件，将你所要管理的节点信息加入到主机文件中<!--more-->
请确保以上三步已经被正确设置好，可以使用以下命令来验证：
```
# ansible testgroup -m ping
```
```
test1 | success >> {
    "changed": false,
    "ping": "pong"
}

lightcloud | success >> {
    "changed": false,
    "ping": "pong"
}
```
ansible连接到testgroup中的每一台主机，传送所需的模块至每个节点并执行，返回模块的输出内容。
ansible目前有很多管理模块，我们可以根据这些模块来写出等幂的playbook。

模拟这样的一个场景，使用ansible的playbook来管理节点的DNS配置文件。这里用到以下几个知识点：
* 使用template模块，使用模板文件动态生成dns配置文件中的nameserver
* 使用fetch模块，将被管理节点上的dns配置文件fetch到管理节点
* 模板的使用，编写模板文件
### Playbook
```
---
- hosts: testgroup
  vars:
      name: coocla
      resolver-servers:
      - 223.5.5.5
      - 223.6.6.6
  user: root
  tasks:
  - name: Deploy resolv.conf
    template: src=files/resolv.j2 desc=/etc/resolv.conf mode=0444
  - name: Fetch node file
    fetch: src=/etc/resolv.conf desct=/tmp/fetchfile
```
这里我们首先指定了*hosts(target)*，然后声明了两个变量*name*和*resolver-server*，并且使用root来运行此次的task, 使用tasks关键字来声明以下内容为需要运行的任务，任务详情则是调用对于功能的module
### Template
```
# resolv.j2 by {{ name | upper }}
# for  {{ ansible_eth0.ipv4.address }}
{% if domain is defined -%}
domain {{ domain }}
{% endif -%}
{% for ns in resolvers -%}
nameserver {{ ns }}
{% endfor %}
```
这里我们引用了定义的变量name，并使用过滤器赋值给upper将其转换为大写；其次我们引用了ansible的facts(类似于saltstack中的grains)，内置的facts变量可以使用setup模块来查看(ansible testgroup -m setup)。接下来则使用了jinja的if和for语法来对变量进行判断和循环引用.
目前为止一切工作准备就绪，现在可以把这些任务跑一跑了：
```
# ansible-playbook learn-template.yml
```
```
PLAY [testgroup] **************************************************************

GATHERING FACTS ***************************************************************
ok: [test1]
ok: [lightcloud]

TASK: [Depoly resolv.conf] ****************************************************
changed: [test1]
changed: [lightcloud]

TASK: [Fetch node file] *******************************************************
changed: [test1]
changed: [lightcloud]

PLAY RECAP ********************************************************************
lightcloud                 : ok=3    changed=2    unreachable=0    failed=0
test1                      : ok=3    changed=2    unreachable=0    failed=0
```
由以上输出可见，一共有个两个任务，在两个节点上均正常运行结束.
```
cat /tmp/fetchfile/xxxx/etc/resolv.conf
# resolv.in by COOCLA
# for xxxx
nameserver 223.5.5.5
nameserver 223.6.6.6
```

查看fetch到的文件内容也是我们之前预期的结果。
