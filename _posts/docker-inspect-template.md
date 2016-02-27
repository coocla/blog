title: Docker inspect模板语法
categories: Docker
tags: [docker, golang模板]
date: 2015-08-03 21:44:58
---
## 前言
学习使用Docker的过程，避免不了有时需要获取container的一些元数据。大家通常使用docker inspect命令来获取自己所需的元数据信息。
```
# docker help inspect
Usage: docker inspect [OPTIONS] CONTAINER|IMAGE [CONTAINER|IMAGE...]

Return low-level information on a container or image

  -f, --format=      Format the output using the given go template
  --help=false       Print usage
```
会发现有一个-f的参数，使用go模板来格式化输出内容。<!--more-->

Go语言全名golang，我个人也是才接触，感觉其语法比较难记比较蛋疼，没有python那么简明工整，但是为了学好/用好docker，熟悉下其语法还是有必要的。当前你也可以使用grep进行匹配搜索，应该也可以达到目的。但略微显得有些不高端。哈哈，废话不多说，进入主题。

模板其实是什么意思？就是组数据根据一个固定的模式两者进行合并输出，这就是模板。go template和常见的web开发模板引擎很类似，例如jinja2(tornado, django, flask都是用的这个模板引擎)。

## 启动一个容器
```
# docker run -d -p 80:5000 ubuntu ping 114.114.114.114 -name test
```
获取该容器的元数据
```
# docker inspect test
```
发现其输出内容是一个json字符串，里面包含了该容器的元数据
## 模板基本语法
```
* {{}} 语法用来处理指令，在双大括号外面的字符均被作为输出的内容进行输出
* .  用来表示当前上下文，大部分情况下多用来表示容器的整个元数据，但是也可以```
使用with语句改变当前的上下文内容

举个例子：
```
# docker inspect -f 'Hello World' test
Hello World
```
### 双大括号外面的字符串被作为标准输出进行输出
```
# docker inspect -f 'Container {{.Name}} Pid is {{.State.Pid}}' test
Container /test Pid is 24698
```
### 双大括号用来处理指令
```
# docker inspect -f '{{with .NetworkSettings}}{{.IPAddress}} Gatewy is {{.Gateway}} {{end}}' test
192.168.42.134 Gatewy is 192.168.42.129
```
> 使用with语句修改当前的上下文，注意使用with语法需要end结尾

### 模板逻辑比较
docker inspect中也支持一些逻辑比较运算。
```
eq 等于
ne 不等于
lt 小于
lt 小于等于
gt 大于
ge 大于等于
```
例如:
```
# docker inspect -f '{{eq .Name .Name}}' test
true
# docker inspect -f '{{eq .Age .Age}}' test
FATA[0000] template: :1:2: executing "" at <eq .Age .Age>: error calling eq: invalid type for comparison
```
由此可见，当对应的属性不存在，或者对应的值为null时，则不能进行比较

### 模板语法index
index类似python中通过索引获取列表中的元素
```
# docker inspect -f '{{.Config.Cmd}}' test
[ping 114.114.114.114]
```
在python中，这样的数据结构被称为列表，因此这里我暂时称它为列表，如果我们想要取列表的第一个值时应该怎么取?
```
# docker inspect -f '{{index .Config.Cmd 0}}' test
ping
# docker inspect -f '{{index .Config.Cmd 1}}' test
114.114.114.114

# docker inspect -f '{{json .State}}' test | jq '.'
{
  "StartedAt": "2015-08-03T06:57:18.343618856Z",
  "Running": true,
  "Dead": false,
  "Error": "",
  "ExitCode": 0,
  "FinishedAt": "0001-01-01T00:00:00Z",
  "OOMKilled": false,
  "Paused": false,
  "Pid": 24698,
  "Restarting": false
}
# docker inspect -f '{{index .State "Dead"}}' test
false
```
index 也可以这样使用，当数组的索引是一个非数字的字母时，可以这样来获取对应的值
```
# docker inspect -f '{{len .Config.Cmd}}' test
2
```
len函数获取当前数组的长度

### 模板数据类型
docker inspect 的数据类型可分为浮点型、字符串、布尔类型。在docker 1.7版本，新加入了整形
```
# docker inspect -f '{{.State.ExitCode}}' test
0
# docker inspect -f '{{eq .State.ExitCode 0}}' test
FATA[0000] template: :1:2: executing "" at <eq .State.ExitCode 0>: error calling eq: incompatible types for comparison
# docker inspect -f '{{eq .State.ExitCode 0.0}}' test
true
```
由此可以看出，容器元数据中，均为浮点类型，即使看起来像是整形。
### 模板逻辑语句
docker inspect 同时也支持使用go语言编写复杂的逻辑语句。
```
# docker inspect -f '{{if eq .State.Running true}}
{{range $p,$v := .NetworkSettings.Ports}}
{{$p}}-->{{json $v}}
{{end}}
{{end}}' test
5000/tcp-->[map[HostIp:0.0.0.0 HostPort:80]]
```
判断容器是否处于运行状态，如果是，循环打印出容器的端口映射情况
```
# docker inspect -f ‘{{if eq .State.Running true}}
{{range $p,$v := .NetworkSettings.Ports}}
{{$p}} —> {{json $v}}
{{end}}
{{else}}
{{.State.FinishedAt}}
{{.State.Error}}
{{end}’ test
5000/tcp-->[map[HostIp:0.0.0.0 HostPort:80]]
```
判断容器是否处于运行状态，如果是，循环打印出容器的端口映射情况，否则打印容器完成的时间，以及容器的错误信息
## 总结
其实这里就是引用了golang的html/template模块，更加复杂高级的处理，可以结合html/template模块的用法进行测试使用。

附送Hello world之Go版本:
```
package main

import "fmt"

func main(){
  fmt.Println("Hello World")
}
```

