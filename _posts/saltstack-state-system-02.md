title: 如何使用SaltStack states
categories: SaltStack
tags: [saltstack]
date: 2013-11-18 16:15:54
---
如何使用salt states ?
许多强大并且复杂的工程解决方案都是建立在简单的原则上，salt states就做到了这一点，保持极度的简单原则
salt state系统的核心是sls文件(salt state file)，sls文件是表示一个系统应该处于的状态，并且利用一个简单的格式设置这些数据，这经常被称为状态管理。

It is all just data
其实sls文件只是一个数据结构，对于sls文件的深入理解将会有助于对salt state的深入理解和应用。
sls文件事实上只是一个字典、列表、字符串或者数字。通过对配置进行数据结构化，作为其编写人员货开发人员它都变得清晰精确，容易理解。<!--more-->

The top file
所有的state file都可以通过top.sls文件来分配给不同的主机进行使用，这个文件是整体的入口描述。

Default data - yaml
默认情况下salt 表现的数据格式采用的最简单的序列化格式-YAML
典型的sls文件经常看起就像yaml

> 这些daemons使用一些通用的服务和包名称，不同的架构经常使用不同的名称和安装包。例如apache在红帽系统中应该是httpd。salt对于底层服务管理使用init下的脚本，系统命令名，配置文件等，执行service.get_all函数来获取对应服务器上一组可用的服务名称。
```
apache:
  pkg:
    - installed
  service:
    - running
    - require:
      - pkg: apache
```
这个sls数据将确保包名为apache的包被安装，并且确保apache服务处于运行状态，这个组件可以被解释为一个简单的方法。

第一行为这组数据设置一个ID，可以通过ID来对其进行调用。
第二行是用来声明salt state开始的状态，所以它将分别使用 包管理(pkg)和服务(service)，这个pkg状态管理，通过系统本地的软件包管理器进行软件包安装，service服务管理，将会管理系统上的守护进程。
第三行和第五行是运行的函数，这些函数被定义在pkg和service状态中，在这里，apache包会被安装，并且apache守护进程将会运行。
最后一行，require是一个必要的声明，用来定义状态之间的依赖，它能确保apache服务成功安装后才会启动apache守护进程。

Adding configs and users
当建立一个像apache的web服务器时，应该有很多组件被添加，apache的配置文件将会需要进行管理，并且对其进行一个用户和组的设置。
```
apache:
  pkg:
    - installed
  service:
    - running
    - watch:
      - pkg: apache
      - file: /etc/httpd/conf/httpd.conf
      - user: apache
  user.present:
    - uid: 87
    - gid: 87
    - home: /var/www/html
    - shell: /bin/nologin
    - require:
      - group: apache
  group.present:
    - gid: 87
    - require:
      - pkg: apache

/etc/httpd/conf/httpd.conf:
  file.managed:
    - source: salt://apache/httpd.conf
    - user: root
    - group: root
    - mode: 644
```
这个例子扩展了上面的那个例子，其中包括了一个配置文件，一个用户、组和一个新的声明 watch。
添加更多的states是很容易的，因为新的用户和组的state都在apache ID下，这个用户和组将会称谓apache的用户和组，这个require状态将保证group被添加后正确的添加user，而这个组将会在apache包被安装后被添加。
现在需要声明对service服务改变的监视，并且现在将这三个states看作为一个对象。这个watch statement和require做了类似的事情，在其监视的状态运行之后正确在运行一个state，但它增加了一个额外的组件。watch statement将会在其监视的函数(state)中有任何的变化后运行这个state。所以如何这个package更新，这个配置文件改变或者这个user uid改变，然后这个service state watcher将会运行。service state watch仅仅是重启service，所以在这个案例中，配置文件中的一个改变也会触发restart。
Moving beyond a single sls
当在一个可扩展的环境下设置salt states，不止一个的sls将会被使用。上面的例子是在一个单一的sls文件中，当拥有两个或更多的sls文件时可以构建一个树状结构。上面的例子还引用了一个文件，使用一个强大的源 - salt://apache/httpd.conf。这个文件将会被使用。

sls文件的目录结构奠定了salt master，一个sls仅仅是一个文件或者下载服务器中的文件。

这个apache例子放置在salt文件服务器的根下：
```
apache/init.sls
apache/httpd.conf
```
所以这个httpd.conf在apache目录下仅仅是一个文件，并且被直接引用。

但是当使用不仅仅是单一的sls文件时，多个组建可以被添加到工具箱中，考虑下这个SSH例子：
```
ssh/init.sls

openssh-client:
    pkg.installed

/etc/ssh/ssh_config:
    file.managed:
        - user: root
        - grouop: root
        - mode: 644
        - source: salt://ssh/ssh_config
        - require:
            - pkg: openssh-client

ssh/server.sls

include:
    - ssh

openssh-server：
    pkg.installed

sshd:
    service.running:
        - require:
            - pkg: openssh-client
            - pkg: openssh-server
            - file: /etc/ssh/banner
            - file: /etc/ssh/sshd_config

/etc/ssh/sshd_config：
    file.managed:
        - user: root
        - group: root
        - mode: 644
        - source: salt://ssh/sshd_config
        - require:
            - pkg: openssh-server

/etc/ssh/banner:
    file:
        - managed
        - user: root
        - group: root
        - mode: 644
        - source: salt://ssh/banner
        - require:
             - pkg: openssh-server
```
> 注意我们使用了两个类似的方法通过salt进行文件的管理，在/etc/ssh/sshd_config state上，我们使用了file.managed进行定义，然而在/etc/ssh/banner state上，我们使用了file状态并且添加了一个managed托管属性，进行state的声明定义。两种方法产生的结果都一样，第一种方式使用file.managed其实只是一个缩写，你可以将其理解为一个快捷方式。

现在我们的state树状结构看起来像这样：
```
apache/init.sls
apache/httpd.conf
ssh/init.sls
ssh/banner
ssh/ssh_config
ssh/sshd_config
```
这个例子介绍了include statement。include语句包含另外一个文件，以便于发现其需要require或watch的组建，以及很快会被证明的-扩展

这个include 语句允许states交叉链接，当一个sls拥有include语句，他的意思是扩展到包含的内容，包括sls文件。

注意一些sls文件名为init.sls。而另一些则不是。

Extending included sls data
有时sls数据需要扩展，也许apache服务需要更多的资源控制，在某些情况下被不同的文件所设置

在这些例子中，首先将添加一个自定义的banner给sshd，其次在apache中添加更多的监视器从mod_python中。
```
ssh/custom-server.sls

include:
    - ssh.server

extend:
    /etc/ssh/banner:
        file:
            - source: salt://ssh/custom-banner

python/mod_python.sls；

include：
    - apache

extend:
    apache:
        service:
            - watch:
                - pkg: mod_python

mod_python:
    pkg.installed
```
这个custom-server.sls文件使用扩展语句，将会使用下载的custom-banner内容覆盖之前下载的内容。因此在使用配置的banner内容将会被改变

在新的mod_python sls中定义了mod_python包被添加，但是更重要的是 extend apache service也将watch到这个改变。
在扩展中使用require 或 watch
extend语句不同于require或watch。它只是起附加扩展的作用，并不是取代必要的组件。

Understanding the render system
因为sls数据是简单的数据序列，它并不能完全代表YAML。salt默认使用YAML，因为它非常简单，容易学习和使用。但是只要提供一个渲染器模块，sls文件可以以任何格式呈现出来。

缺省的渲染系统是yaml_jinja渲染器。yaml_jinja渲染器将首先使用jinja2模版系统，然后通过yaml解析器，这里的好处是，当个创建一个sls文件将使用了完整的编程构造方法。

其他的渲染器可以使用yaml_mako 和yaml_wempy。它将替代默认的jinja模版系统。更引人注目的是pure python 的py和pydsl的渲染器。py渲染器允许在sls文件中写纯pure python，允许最大的灵活性和强大的sls数据。而pydsl渲染器提供了一种灵活的、特定于域的语言用于编写数据在sls中编写python。

注释：这个默认引擎不只是在sls文件中可用，他们也可以用于文件。可以被用于file.managed状态。使文件管理更具有动态和灵活性。一些例子中使用模版管理文件，可以在文档中找到的他们。

Getting to know the default - yaml_jinja
默认渲染器-yaml_jinja允许使用jinja模版系统。一个jinja模版系统的指导可以在这里找到。http://jinja.pocoo.org/docs

使用渲染器有一些非常有用的数据传递，对于以渲染器为基础的模版引擎。下面这三个关键组建是可以使用渲染器的。salt grains pillar。salt对象允许任何salt函数从模版内容进行调用，grains允许grains从模版中访问，一些例子如下：
```
apache/init.sls

apache:
  pkg.installed:
    {% if grains['os'] == 'RedHat'%}
    - name: httpd
    {% endif %}
  service.running:
    {% if grains['os'] == 'RedHat'%}
    - name: httpd
    {% endif %}
    - watch:
      - pkg: apache
      - file: /etc/httpd/conf/httpd.conf
      - user: apache
  user.present:
    - uid: 87
    - gid: 87
    - home: /var/www/html
    - shell: /bin/nologin
    - require:
      - group: apache
  group.present:
    - gid: 87
    - require:
      - pkg: apache

/etc/httpd/conf/httpd.conf:
  file.managed:
    - source: salt://apache/httpd.conf
    - user: root
    - group: root
    - mode: 644
```
这个例子很简单，如果grains中系统名是红帽，那么对于apache来说，包的名称和服务就是httpd
一个更有意思的方法在jinja中被找到，在模块中设置MooseFS分布式操作系统chunkserver
```
moosefs/chunk.sls：

include:
  - moosefs

{% for mnt in salt['cmd.run']('ls /dev/data/moose*').split() %}
/mnt/moose{{ mnt[-1] }}:
  mount.mounted:
    - device: {{ mnt }}
    - fstype: xfs
    - mkmnt: True
  file.directory:
    - user: mfs
    - group: mfs
    - require:
      - user: mfs
      - group: mfs
{% endfor %}

/etc/mfshdd.cfg:
  file.managed:
    - source: salt://moosefs/mfshdd.cfg
    - user: root
    - group: root
    - mode: 644
    - template: jinja
    - require:
      - pkg: mfs-chunkserver

/etc/mfschunkserver.cfg:
  file.managed:
    - source: salt://moosefs/mfschunkserver.cfg
    - user: root
    - group: root
    - mode: 644
    - template: jinja
    - require:
      - pkg: mfs-chunkserver

mfs-chunkserver:
  pkg:
    - installed
mfschunkserver:
  service:
    - running
    - require:
{% for mnt in salt['cmd.run']('ls /dev/data/moose*') %}
      - mount: /mnt/moose{{ mnt[-1] }}
      - file: /mnt/moose{{ mnt[-1] }}
{% endfor %}
      - file: /etc/mfschunkserver.cfg
      - file: /etc/mfshdd.cfg
      - file: /var/lib/mfs
```
这个例子展示了更多的可用变量在jinja中，循环多用于动态检测可用的硬盘驱动器和设置安装，并且salt对应被多次用来调用shell命令来手机数据。

Introducing the python and the PyDSL Renderers
有时候选择默认渲染器可能并不能全完来完成所需的任务，当这种情况发生时，可以使用python的渲染器。同城一个YMAL渲染器应该用于大多数的sls文件。但文件设置为使用另外一个sls渲染器可以很容易的添加到树状结构中。

这个例子展示了非常基本的python sls文件：
```
python/django.sls

#!py

def run():
    '''
    Install the django package
    '''
    return {'include': ['python'],
            'django': {'pkg': ['installed']}}
```
这是一个非常简单的例子，第一行代表的是渲染器，意思是声明不是用默认的渲染器，使用py渲染器。然后运行定义的函数，从这个函数中返回的值必须满足salt的数据结构，或者更好的被称为salt hightstate数据结构

另外使用pydsl渲染器，可以将上面的例子写的更简单：
```
python/django.sls:
#!pydsl

include('python', delayed=True)
state('django').pkg.installed()
```
这个python示例在YAML中看上去像这样：
```
include:
  - python

django:
  pkg.installed
```
这个例子清楚的说明了使用YAML作为默认渲染器是一个明智的决定，灵活的能力使你可以在任何地方获取pure python sls

Running and debugging salt states
一旦规则在sls准备好，他们应该被测试，以确保他们能够正常工作。调用这些规则，只需要在命令行中执行salt '*' state.highstate。如果你返回的只有主机名但是没有其他任何返回。很可能在一个或多个sls文件中存在问题。在minion，使用salt-call命令执行 salt-call state.highstate -l debug 来调试检查输出的错误信息，这应该有助于解决这个问题。minions也可以在前台执行调试模式：salt-minion -l debug







</br>