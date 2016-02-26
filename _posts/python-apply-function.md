title: Python apply函数用法
categories: Python
tags: [python, apply]
date: 2013-11-11 14:31:46
---
函数格式为：apply(func,*args,**kwargs)

用途：当一个函数的参数存在于一个元组或者一个字典中时，用来间接的调用这个函数，并肩元组或者字典中的参数按照顺序传递给参数

解析：
* args是一个包含按照函数所需参数传递的位置参数的一个元组，是不是很拗口，意思就是，假如A函数的函数位置为 A(a=1,b=2),那么这个元组中就必须严格按照这个参数的位置顺序进行传递(a=3,b=4)，而不能是(b=4,a=3)这样的顺序
* kwargs是一个包含关键字参数的字典，而其中args如果不传递，kwargs需要传递，则必须在args的位置留空<!--more-->

> apply的返回值就是函数func函数的返回值

```
#coding:utf-8

def noargs_fun():
	print "No args functions"

def tup_fun(arg1,arg2):
	print arg1,arg2

def dic_fun(arg1=1,arg2=2):
	print arg1,arg2

if __name__ == '__main__':
	apply(noargs_fun)
	apply(tup_fun,("参数1","参数2"))
	kw={"arg1":"参数1","arg2":"参数2"}
	apply(dic_fun,(),kw)
```
运算输出
```
No args functions
参数1 参数2
参数1 参数2
[Finished in 0.7s]
```



</br>