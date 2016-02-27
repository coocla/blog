title: SaltStack cron状态模块
categories: SaltStack
tags: [saltstack]
date: 2013-11-11 21:36:41
---
salt.states.cron 是一个管理Unix系统计划调度任务的命令

cron的状态管理需要一些参数来声明，时间参数需要声明：分钟，小时，天，月，星期。用户还要声明该动作是编辑还是新定义。

默认情况下，这个时间参数都是"*"，执行用户是"root"，当修改一个当前存在的cron job，这个名字的声明必须是全局唯一的，所以修改当前一个已存在的cron job看想起来像下面这样<!--more-->：
```
date > /tmp/crontest:
  cron.present:
    - user: root
	- minute: 5
```
修改为
```
date > /tmp/crontest:
  cron.present:
    - user: root
	- minute: 7
	- hour: 2
```
然后现有的这个cron将会被更新，但是如果这个cron命令被改变，那么一个新的cron job将会被添加到这个用户的crontab中。

此外，时间参数(分钟，小时等)可以通过使用随机random，而不是使用一个特定的值。例如，在定义cron state中使用random关键字定义minute参数，相同的cron job可以推送到数百或数千台主机，并且他们就分别使用一个随机生成的分钟。当cron job同时访问一个网络资源这将会是很有帮助的，因为所有的主机同时运行这个工作时是不可取的。
```
/path/to/cron/script:
  cron.present:
    - user: root
	- minute: random
	- hour: 2
```
因为salt假设在未定义的模板参数值都是"*"，当添加一个参数到state并且设置其值为"random"，那么该参数的值将会由随机生成的这个数字代替之前默认的"*"，然而如果这个参数的值在minion上已经是一个具体的数字，那么使用random关键字，将不会修改minion上的配置.
```
salt.states.cron.absent(name, user='root', **kwargs)
该states可以验证指定的cron job是否在指定的用户下存在，如果name匹配了，将会移除这个匹配到的cron job
name
  在用户crontab中不应该出现的命令
user
  需要修改的cron用户名字，默认是"root"用户
```

```
salt.states.cron.file(name, source_hash='', user='root', template=None, context=None, replace=True, defaults=None, env=None, backup='', **kwargs)

该states提供文件管理功能，为crontab文件预制一个函数(模板等)，分配到某个用户上

name
  这个源文件将被用于crontab,这个源文件可以被托管在任何的salt-master上，或者一个HTTP或FTP服务器上。对于托管在salt文件服务器上，如果这个文件位于文件服务器上的一个名为spam的目录下，并且名字为eggs,那么这个源字符串应该是sakt://spam/eggs。如果这个文件托管在HTTP或FTP服务器上，则依赖于source_hash这个参数。

source_hash
  这可以使一个文件，其中包含一个源哈希字符串，或者是一个哈希字符串，源哈希字符串是跟在哈希散列算法：
md5=e138491e9d5b97023cea823fe17bac22

user
  被设置crontab的用户，默认是root

template
  命名模板将会按照模板类型的语法进行渲染下载下来的文件，现在只支持jinja 和 mako

context
  覆盖默认的上下文变量传递给模板

replace
  如果这个crontab将被替换，如果错误，这条命令将会被忽略，如果对于指定的用户存在一个crontab，默认是True

defaults
  默认传递给模板的上下文内容

backup
  当覆盖了用户的crontab将会开启备份模式
```

```
salt.states.cron.present(name, user='root', minute='*', hour='*', daymonth='*', month='*', dayweek='*')

该states验证指定的cron job是否存在于指定的用户下，关于更多详细设置crontab的参数，请查看系统crontab文档，大多数Unix系统的cron文档，可以使用man 5 crontab的man page中找到

name
  应该被执行的cron job的命令

user
  需要修改的cron用户名字，默认是"root"用户

minute
  这个参数是被设置cron的分钟部分，这里可以是您的系统支持的任何有效的字符串。默认值是 *

hour
  这个参数是被设置cron的小时部分，默认值是 *

daymonth
  这个参数是被设置cron的天数的部分，默认值是 *

month
  这个参数是被设置cron的月份部分，默认值是 *

dayweek
  这个参数是被设置cron的星期部分，默认值是 *
```
附实例一枚:
```
# cat /srv/salt/top.sls
base:
    "*":
        - manager.cron

# cat /srv/salt/manager/cron/init.sls
restart_ks.sh:
    file.managed:
        - source: salt://resource/restart_ks.sh
        - name: /usr/local/sbin/restart_ks.sh
jobs:
    cron.present:
	    - name: sh /usr/local/sbin/restart_ks.sh
        - user: root
        - minute: 40
        - hour: 0
        - require:
            - file.managed: restart_ks.sh
```