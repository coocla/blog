title: 如何使用Ansible 2-0 Python API
categories: Python
tags: [ansible, python api, ansible 2.0]
date: 2016-06-30 15:16:23
---
## 概览
Ansible 和 SaltStack 都提供了 Python 直接调用的API, 这方便了 Pythoner 对这些软件进行二次开发和整合, 此功能着实方便了不少, 比起 Python 代码中调用 shell 也略显专业!

然而 Ansible 在2.0版本后重构了大部分的代码逻辑, 启用了2.0版本之前的 Runner 和 Playbook 类, 使得广大同学之前的代码运行错误. 择日不如撞日, 今天中午对照(官方的文档)[http://docs.ansible.com/ansible/developing_api.html#python-api-2-0], 结合源代码, 对2.0版本之后的 Python API 做了下探究

## Adhoc
adhoc 其实就是执行 Ansible 模块, 通过 adhoc 我们可以方便快捷的完成一些临时的运维操作.

### 2.0 之前的调用
```
import ansible.runner
import json
runner = ansible.runner.Runner(
       module_name='ping',       # 模块名
       module_args='',           # 模块参数
       pattern='all',            # 目标机器的pattern
       forks=10             
    )
datastructure = runner.run()
data = json.dumps(datastructure,indent=4)
```
> 当然这里会去加载默认的 inventory

如果不想使用 inventory 文件或者想使用动态的 inventory, 则可以使用 host_list 参数代替
```
import ansible.runner
import json
runner = ansible.runner.Runner(
        host_list=["10.10.0.1"],    # 这里如果明确指定主机需要传递一个列表, 或者指定动态inventory脚本
        module_name='ping',            # 模块名
        module_args='',                # 模块参数
        extra_vars={"ansible_ssh_user":"root","ansible_ssh_pass":"xx"},
        forks=10
    )
datastructure = runner.run()
data = json.dumps(datastructure,indent=4)
```

### 2.0 之后的调用
```
import json
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor

loader = DataLoader()                     # 用来加载解析yaml文件或JSON内容,并且支持vault的解密
variable_manager = VariableManager()      # 管理变量的类,包括主机,组,扩展等变量,之前版本是在 inventory 中的
inventory = Inventory(loader=loader, variable_manager=variable_manager)
variable_manager.set_inventory(inventory) # 根据 inventory 加载对应变量

class Options(object):
    '''
    这是一个公共的类,因为ad-hoc和playbook都需要一个options参数
    并且所需要拥有不同的属性,但是大部分属性都可以返回None或False
    因此用这样的一个类来省去初始化大一堆的空值的属性
    '''
    def __init__(self):
        self.connection = "local"  
        self.forks = 1
        self.check = False

    def __getattr__(self, name):
        return None

options = Options()

def run_adhoc():
    variable_manager.extra_vars={"ansible_ssh_user":"root" , "ansible_ssh_pass":"xxx"}  # 增加外部变量
    # 构建pb, 这里很有意思, 新版本运行ad-hoc或playbook都需要构建这样的pb, 只是最后调用play的类不一样
    # :param name: 任务名,类似playbook中tasks中的name
    # :param hosts: playbook中的hosts
    # :param tasks: playbook中的tasks, 其实这就是playbook的语法, 因为tasks的值是个列表,因此可以写入多个task
    play_source = {"name":"Ansible Ad-Hoc","hosts":"10.10.0.1","gather_facts":"no","tasks":[{"action":{"module":"shell","args":"w"}}]}
    play = Play().load(play_source, variable_manager=variable_manager, loader=loader)
    tqm = None
    try:
        tqm = TaskQueueManager(
            inventory=inventory,
            variable_manager=variable_manager,
            loader=loader,
            options=options,
            passwords=None,
            stdout_callback='minimal',
            run_tree=False,
        )
        result = tqm.run(play)
        print result
    finally:
        if tqm is not None:
            tqm.cleanup()

if __name__ == '__main__':
    run_adhoc()
```

## Playbook
playbook 则类似于 SaltStack 中的 state

### 2.0 之前的调用
```
from ansible import callbacks
from ansible import utils
from ansible.playbook import PlayBook

stats = callbacks.AggregateStats()
callback = callbacks.PlaybookCallbacks()
runner_callbacks = callbacks.PlaybookRunnerCallbacks(stats)
pb = ansible.playbook.PlayBook(
    playbook="tasks.yml",
    stats=stats,
    callbacks=playbook_cb,
    runner_callbacks=runner_cb,
    check=True
)
pb.run()
```

### 2.0 之后的调用
```
import json
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor

loader = DataLoader()                     # 用来加载解析yaml文件或JSON内容,并且支持vault的解密
variable_manager = VariableManager()      # 管理变量的类,包括主机,组,扩展等变量,之前版本是在 inventory 中的
inventory = Inventory(loader=loader, variable_manager=variable_manager)
variable_manager.set_inventory(inventory) # 根据 inventory 加载对应变量

class Options(object):
    '''
    这是一个公共的类,因为ad-hoc和playbook都需要一个options参数
    并且所需要拥有不同的属性,但是大部分属性都可以返回None或False
    因此用这样的一个类来省去初始化大一堆的空值的属性
    '''
    def __init__(self):
        self.connection = "local"  
        self.forks = 1
        self.check = False

    def __getattr__(self, name):
        return None

options = Options()

def run_playbook():
    playbooks=['task.yaml']  # 这里是一个列表, 因此可以运行多个playbook
    variable_manager.extra_vars={"ansible_ssh_user":"root" , "ansible_ssh_pass":"xxx"}  # 增加外部变量
    pb = PlaybookExecutor(playbooks=playbooks, inventory=inventory, variable_manager=variable_manager, loader=loader, options=options, passwords=None)
    result = pb.run()
    print result

if __name__ == '__main__':
    run_playbook()
```

## 总结
新版本的部分代码重构和之前的变化还是蛮大的, 子系统的拆建, 解耦, 例如变量管理从之前的inventory中拆解出来. 但是总体的组件设计思想没有变化, 例如callback插件, 可以通过指定不同的插件, 来对结果进行收集/汇总.
研究的不深, 应该还有很多新的特性与使用方法, 暂时先写到这, 后续有什么有意思的发现在继续写.









</br>
