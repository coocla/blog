title: SaltStack使用state进行系统配置-3
categories: SaltStack
tags: [saltstack]
date: 2013-11-21 17:06:39
---
Templating sls modules
sls模块内联执行可能依赖于你的编程逻辑。默认的模板使用jinja2模板系统作为渲染器，可以通过修改主配置文件master来改变渲染器的配置

所有的state都通过一个模板系统完成最初的读取，使用模板系统，只需要添加一个模板标记，一个使用模板标记的sls例子：<!--more-->
```
{% for usr in ['moe','larry','curly'] %}
{{ usr }}:
  user.present
{% endfor %}
```
这个使用了模板的sls文件生成后结果如下：
```
moe:
  user.present
larry:
  user.present
curly:
  user.present
```
下面这个是一个更复杂的例子：
```
{% for usr in 'moe','larry','curly' %}
{{ usr }}:
  group:
    - present
  user:
    - present
    - gid_from_name: True
    - require:
      - group: {{ usr }}
{% endfor %}
```
Using grains in sls modules
通常情况下state在不同的系统有不同的表现。grains的信息也可以在sls模板文件中使用，例如，可以这样用：
```
apache:
  pkg.installed:
    {% if grains['os'] == 'RedHat' %}
    - name: httpd
    {% elif grains['os'] == 'Ubuntu' %}
    - name: apache2
    {% endif %}
```
Calling salt modules from templates
所有的state都是在minion上加载模板系统，这允许数据可以被实时的收集到目标系统上，它还允许在sls模块中允许shell命令。

模板内容也可以有效的使用salt函数：
```
moe:
  user:
    - present
    - gid: {{ salt['file.group_to_gid']('some_group_that_exists') }}
```
注意，上面这个例子要正常工作，请确保some_group_that_exists存在
下面的例子，使用network.hw_addr函数来检索eth的mac地址：
```
salt['network.hw_addr']('eth0')
```

Advanced sls module syntax
最后，我们将讨论在一些更负载的state树中的一些令人难以置信的技巧

Include语句
前面的例子展示了如何通过多个文件进行扩展。同样，require可以使用include语句进行多个文件中的state之间的依赖，例如：
```
python/python-libs.sls:

python-dateutil:
  pkg.installed
```
```
python/django.sls:

include:
  - python.python-libs

django:
  pkg.installed:
    - require:
      - pkg: python-dateutil
```
Extend语句
你可以使用extend语句修改之前的例子，例如当apache的vhosts文件发生了改变也重启apache
```
apache/apache.sls:

apache:
  pkg.installed
```
```
apache/mywebsite.sls:

include:
  - apache.apache

extend:
  apache:
    service:
      - running
      - watch:
        - file: /etc/httpd/extra/httpd-vhosts.conf

/etc/httpd/extra/httpd-vhosts.conf:
  file.managed:
    - source: salt://apache/httpd-vhosts.conf
```
extend它只是起追加作用，而不是取代任何必要的组件

Name语句
你可以使用name语句的值来覆盖ID的值，例如，这个例子更加有利于维护重写
```
apache/mywebsite.sls:

include:
  - apache.apache

extend:
  apache:
    service:
      - running
      - watch:
        - file: mywebsite

mywebsite:
  file.managed:
    - name: /etc/httpd/extra/httpd-vhosts.conf
    - source: salt://apache/httpd-vhosts.conf
```

Names语句
更强大的是使用names语句一次性的声明多个states来覆盖ID语句。这经常可以在模板中利用循环来实现，例如第一个例子中的循环可以写成下面这样的形式：
```
stooges:
  user.present:
    - names:
      - moe
      - larry
      - curly
```




</br>
