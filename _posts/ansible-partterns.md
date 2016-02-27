title: Ansible匹配主机或组的Patterns
categories: Ansible
tags: [ansible patterns]
date: 2014-07-20 19:24:38
---
## Patterns
patterns意味着在ansible中管理哪些主机，也可以理解为，要与哪台主机进行通信，不过在playbooks中，它将以为着哪些主机需要应用特定的配置或者过程。SaltStack中的targeting
在命令行中，通常这样来使用
```
# ansible <pattern_goes_here> -m <module_name> -a <arguments>
# ansible webservers -m service -a "name=httpd state=restarted"
```
一个模式通常会用一个组来表示，这样可以在较少的文字中表示较多的主机(例如上面的例子)，主机都是“webserver”组中不管怎样，使用ansible，首先要知道如何告诉ansible，在你的inventory中有哪些主机。通过主机名或者组名都可以。<!--more-->

下面的通配模式用来表示inventory中的所有主机
* all
* *

利用通配符还可以指定一组具有规则特征的主机或主机名
* one.lightcloud.com
* one.lightcloud.com:two.lightcloud.com
* 192.168.1.50
* 192.168.1.*

下面的模式，用来知道一个地址或多个组。组名之间通过冒号隔开，表示“OR”的意思，意思是这两个组中的所有主机。
* webservers
* webservers:dbservers

当然你可以做出非的表达式，例如，目标主机必须在组webservers但不在phoenix组中
* webserver:!phoenix

你还可以做出交集的表达式，例如，目标主机必须即在组webservers中又在组staging中
* webservers:&staging

你还还可以把它们全部组合到一块
* webserver:dbservers:&staging:!phoenix

上面这个复杂的表达式最后表示的目标主机必须满足：在webservers或者dbservers组中，必须还存在于staging组中，但是不在phoenix组中。这些可以看作是SaltStack中Compound matchers
> 注意：在shell中，记得把 & ! 这些特殊符号进行转义。
 

在ansible-palybook命令中，你也可以使用变量来组成这样的表达式，但是你必须使用“-e”的选项来指定这个表达式。通常我们不这样用：
```
ansible-palybook -e webservers:!{{excluded}}:&{{required}}
```
你完全不需要使用这些严格的模式去定义组来管理你的机器。无论通过主机名，IP，组都可以使用通配符去匹配
* *.lightcloud.com
* *.com

他们也可以通过混合模式组合在一起
* *.lightcloud.com:*.com

你还可以在开头的地方使用"~"，用来表示这是一个正则表达式
* ~(web|db).*\.example\.com

最后，在ansible和ansible-playbook中，还可以通过一个参数"--limit"来明确指定排除某些主机或组
```
ansible-playbook site.yml --limit datacenter2
```



</br>

