title: django ORM数据模型的定义
categories: Python
tags: [python, django orm]
date: 2014-05-06 17:40:05
---
## 前言
数据模型的定义同样遵循python类的继承，介绍以下几种定义方式：
* 使用单个表。整个继承树共用一张表。使用唯一的表，包含所有基类和子类的字段。
* 每个具体类一张表，这种方式下，每张表都包含具体类和继承树上所有父类的字段。因为多个表中有重复字段，从整个继承树上来说，字段是冗余的。
* 每个类一张表，继承关系通过表的JOIN操作来表示。这种方式下，每个表只包含类中定义的字段，不存在字段冗余，但是要同时操作子类和所有父类所对应的表。<!--more-->

## 每个类一张表
```
from django.db import models

class Person(models.Model):
  name = models.CharField(max_length=20)
  sex = models.BooleanField(default=True)

class teacher(Person):
  subject = models.CharField(max_length=20)

class student(Person):
  course = models.CharField(max_length=20)
```
对应SQL
```
BEGIN;
CREATE TABLE "blog_person" (
    "id" integer NOT NULL PRIMARY KEY,
    "name" varchar(20) NOT NULL,
    "sex" bool NOT NULL
)
;
CREATE TABLE "blog_teacher" (
    "person_ptr_id" integer NOT NULL PRIMARY KEY REFERENCES "blog_person" ("id"),
    "subject" varchar(20) NOT NULL
)
;
CREATE TABLE "blog_student" (
    "person_ptr_id" integer NOT NULL PRIMARY KEY REFERENCES "blog_person" ("id"),
    "course" varchar(20) NOT NULL
)
;

COMMIT;
```
## 每个具体类一张表，父类不需要创建表
```
from django.db import models

class Person(models.Model):
  name = models.CharField(max_length=20)
  sex = models.BooleanField(default=True)

  class Meta:
    abstract = True

class teacher(Person):
  subject = models.CharField(max_length=20)

class student(Person):
  course = models.CharField(max_length=20)
```
对应SQL
```
BEGIN;
CREATE TABLE "blog_teacher" (
    "id" integer NOT NULL PRIMARY KEY,
    "name" varchar(20) NOT NULL,
    "sex" bool NOT NULL,
    "subject" varchar(20) NOT NULL
)
;
CREATE TABLE "blog_student" (
    "id" integer NOT NULL PRIMARY KEY,
    "name" varchar(20) NOT NULL,
    "sex" bool NOT NULL,
    "course" varchar(20) NOT NULL
)
;

COMMIT;
```
## 自定义创建的表名
```
from django.db import models

class Person(models.Model):
  name = models.CharField(max_length=20)
  sex = models.BooleanField(default=True)

  class Meta:
    abstract = True

class teacher(Person):
  subject = models.CharField(max_length=20)

  class Meta:
    db_table = "Teacher"

class student(Person):
  course = models.CharField(max_length=20)

  class Meta:
    db_table = "Student"
```
对应SQL
```
BEGIN;
CREATE TABLE "Teacher" (
    "id" integer NOT NULL PRIMARY KEY,
    "name" varchar(20) NOT NULL,
    "sex" bool NOT NULL,
    "subject" varchar(20) NOT NULL
)
;
CREATE TABLE "Student" (
    "id" integer NOT NULL PRIMARY KEY,
    "name" varchar(20) NOT NULL,
    "sex" bool NOT NULL,
    "course" varchar(20) NOT NULL
)
;

COMMIT;
```
## 代理模型，为子类增加方法，但不能增加属性
```
from django.db import models

class Person(User):
  class Meta:
    proxy = True

  def some_function(self):
    do_something
```
这样的方式不会改变数据存储结构，但可以纵向的扩展子类Person的方法，并且基础User父类的所有属性和方法。
## 通过子类增加属性
例如要利用django的User类，但是我们还需要额外增加一些字段，但有不想去破坏原生自带的User类，就可以通过以下方法去实现

```
from django.contrib.auth.models import User
from django.db import models

class Person(models.Model):
  user = models.OneToOneField(User)

  blog = models.CharField(max_length=100)
  location = models.CharField(max_length=200)
```




</br>