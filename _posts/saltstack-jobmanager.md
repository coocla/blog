title: SaltStack job manager
categories: SaltStack
tags: [saltstack, job manager, schedule]
date: 2014-03-24 16:06:52
---
Salt Minions 在Salt的cachedir中维护着一个proc目录，这个目录下用以存放着已已经执行过的job id命名的文件；这些文件包含当前运作任务的详细信息，它允许使用jobs方法进行查找。他们位于cachedir下的pro目录中，默认配置为*/var/cache/salt/proc*

jobs方法被定义在saltutil模块中，包含以下方法：
1. running 返回proc目录下所有运行job的数据
2. find_job 返回指定job id的相关数据
3. signal_job 允许向指定的job id任务发送一个信号
4. term_job 向指定的job id发送一个termaination信号(SIGTERM,15)来控制进程
5. kill_job 向指定的job id发送一个kill信号(SIGKILL,9)<!--more-->

## jobs runner
jobs runner包含有很多函数
1. active 返回当前系统上正在运行的jobs
```
# salt-run jobs.active
```
2. lookup_jid 执行后的结果数据发送回master，会被缓存在本地24小时，具体缓存时长可以通过keep_jobs参数控制，此时可以使用lookupjid 指定jid来查看该job当时执行结束后返回的结果
```
# salt-run jobs.lookup_jid <job id number>
```
3. list_jobs 返回截止当前时间本地所缓存的所有job数据
```
# salt-run jobs.list_jobs
```

## SCHEDULING JOBS
schedule任务调度系统，可以将其理解为类似于linux系统上的crontab，可以执行任何的执行函数在minions上或者在master上任何的runner

开启schedule，只需要在master或者minion的配置文件中开启schedule参数，或者为minion定义pillar数据 指定maxrunning用来开启对某个任务最多执行次数的限制。该值默认为1。

对于在minion上执行state，可以通过位置参数或者按照yaml的字典格式提供命名参数，例如：
1. STATES
```
schedule:
  log-loadavg:
    function: cmd.run
    seconds: 3660
    args:
      - 'logger -t salt < /proc/loadavg'
    kwargs:
      stateful: False
      shell: True
```
2. HIGHSTATES
```
schedule:
  highstate:
    function: state.highstate
    minutes: 60
```
在minion的配置文件中或者pillar数据中，设置highstate，每60分钟运行一次，时间声明可以指定seconds, minutes, hours, days
3. RUNNERS
```
schedule:
  overstate:
    function: state.over
    seconds: 35
    minutes: 30
    hours: 3
```
配置在master的配置文件中
4. SCHEDULER WITH RETURNER
```
schedule:
  uptime:
    function: status.uptime
    seconds: 60
    returner: mysql
  meminfo:
    function: status.meminfo
    minutes: 5
    returner: mysql
```
schedule也可以将minon收集的数据通过returner收集到mysql数据库中.

> 需要注意的是：当你通过pillar首次或者新增加schedule的定义时，记得要将pillar数据同步至对应minion上去。
 

```
# salt <tgt> saltutil.refresh_pillar
```






</br>