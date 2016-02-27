title: SaltStack使用state进行系统配置-2
categories: SaltStack
tags: [saltstack]
date: 2013-11-20 21:55:30
---
你可以在一个声明的ID下面定义多个state语句，例如，我们可以快速的修改webserver.sls，并且如果apache没有运行将其启动。
在运行state.highstate之前，尝试停止apache，并且再次观察其输出内容。<!--more-->
```
apache:
  pkg:
    - installed
  service:
    - running
    - require:
      - pkg: apache
```

Expand the sls module
正如你所见，sls模块以.sls附加为文件扩展名并且从state的根下开始引用，同一个sls模块可以同时定义在一个目录。现在通过创建一个名为webserver的目录，并且将webserver.sls移动并且重命名到webserver/init.sls，你的state目录结构应该像下面这样：
```
|- top.sls
`- webserver/
   `- init.sls
```
组织sls模块
你可以将其他的.sls文件放置到state文件目录。这样的组织使你的state树在文件系统上显的更加简洁，例如：如果我们创建了一个webserver/django.sls文件，那么我们可以这样来引用它 webserver.django
此外，state提供了强大的扩展功能.

Require other states
现在我们已经安装了apache，并且处于工作状态，让我们添加一个HTML文件来定制我们的网站。但是如果没有webserver，它将不是一个有用的网站，所以除非apache已经安装并且运行否则我们不希望salt来安装这些HTML文件。像下面内容一样，将其包含在你的webserver/init.sls：
```
apache:
  pkg:
    - installed
  service:
    - running
    - require:
      - pkg: apache

/var/www/index.html:                        # 声明一个ID
  file:                                     # 声明state
    - managed                               # state中的函数名
    - source: salt://webserver/index.html   # 函数的参数
    - require:                              # 声明requisite
      - pkg: apache                         # requisite的引用
```
第九行声明一个新的ID，在这个例子中它是我们自定义的HTML文件要安装的位置(Note：在不同的操作系统或发行版上apache服务的默认位置可能不同，/srv/www也可能是个好位置。)
第十行声明使用的state。
第十一行声明使用第十行中的state的函数名。这个managed函数将从master上下载文件，并且按照指定的位置进行安装。
第十二行是函数的参数，在本例中利用managed函数中的source参数，指定需要从master上下载的文件的路径和名称
第十三行是require的声明
第十四行是require所要引用的state 和 ID，在这个例子它值的例子开头处的那个ID，这个声明告诉salt在不安装apache之前不要安装HTML文件

接下来，创建index.html文件，并且保存在webserver目录下：
```
<html>
    <head><title>Salt rocks</title></head>
    <body>
        <h1>This file brought to you by Salt</h1>
    </body>
</html>
```
最后，在调用一次state.highstate，minion将获取和执行highstate以及从master的文件服务器上获取我们的HTML文件
```
salt '*' state.highstate
```
现在验证apache中你自定义的HTML吧！

require VS watch
现在有两个依赖的声明，“require和watch”，不是所有的state都支持"watch"。service state不支持"watch"并且不支持通过观察一个条件进行重新启动一个服务

例如，如果你使用salt配置安装apache的虚拟主机，当配置文件发送改变时要重启apache，你可以修改之前的例子：
```
/etc/httpd/extra/httpd-vhosts.conf:
  file:
    - managed
    - source: salt://webserver/httpd-vhosts.conf

apache:
  pkg:
    - installed
  service:
    - running
    - watch:
      - file: /etc/httpd/extra/httpd-vhosts.conf
    - require:
      - pkg: apache
```




</br>
