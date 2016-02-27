title: SQLAlchemy外键的使用一.外键的定义
categories: Python
tags: [python, sqlalchemy 外键]
date: 2014-08-20 21:50:08
---
## 前言
SQLAlchemy orm可以将数据库存储的数据封装成对象，同时，如果封装的好的话，所有的数据库操作都可以封装到对象中。这样的代码在组织结构上会非常的清晰，并且相对与使用sql语句在sql注入方面会极具降低。
SQLAlchemy中的映射关系有四种，分别是一对多，多对一，一对一，多对多;
实现这种映射关系只需要外键(ForeignKey),和relationship

## 一对多
```
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, CHAR
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, backref

Base = declarative_base()

class Parent(Base):
    __table__ = "parent"
    id = Column(Integer, Primary_key=True)
    name = Column(CHAR(50))
    child = relationship("child", backref="parent")

class Child(Base):
    __table__ = "child"
    id = Column(Integer, Primary_key=True)
    name = Column(CHAR(50))
    parent_id = Column(Integer,ForeignKey('parent.id'))
```
## 多对一
```
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, CHAR
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, backref

Base = declarative_base()

class Parent(Base):
    __table__ = "parent"
    id = Column(Integer, Primary_key=True)
    name = Column(CHAR(50))

class Child(Base):
    __table__ = "child"
    id = Column(Integer, Primary_key=True)
    name = Column(CHAR(50))
    parent_id = Column(Integer,ForeignKey('parent.id'))
    parent = relationship("parent", backref="child")
```
由此可见，一对多与多对一的区别在于其关联(relationship)的属性在多的一方还是一的一方；如果relationship和ForeignKey都定义在多的一方,那就是多对一,如果relationship定义在一的一方那就是一对多。
## 多对一来
relationship相当于给Child实例增加了一parent属性，在查询过程中，要查询一个Child所关联的所有Parent,可以这样做:
```
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine

def get_engine():
    engine = create_engine("mysql://User:Password@Host/Database?charset=utf8")

_MARK = None

def get_session(autocommit=True, expire_on_commit=False):
    global _MARK
    if _MARK is None:
        Session = sessionmaker(bind=get_engine(),\
            autocommit = autocommit,\
            expire_on_commit = expire_on_commit)
        _MARK = Session()
    return _MARK

def model_query(*args, **kwargs):
    session = kwargs.get("session") or get_session()
    return session.query(*args)

model_query(Child).filter(Child.id=1).parent
```
backref相当于给Parent实例增加了一个child属性，在查询过程中，如果要查询一个Parent所关联对所有Child，可以这样做：
```
model_query(Parent).filter(Parent.id=1).child
```




</br>