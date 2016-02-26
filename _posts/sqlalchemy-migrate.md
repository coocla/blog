title: ORM模型结构自动扩展(升降级)工具SQLAlchemy-migrate
categories: Python
tags: [Python, migrate]
date: 2015-04-05 11:57:38
---
## 前言
在一个项目中，随着版本的递增需求的增加，大部分也会涉及到项目本身的数据结构发生变化。对于已经上线的项目数据是尤为重要的，那么就要求项目本身的一个版本控制与升降机需要达到一个灵活安全的层次。那么今天的主角登场*SQLAlchemy-migrate*.
*SQLAlchemy-migrate*是基于SQLAlchemy的数据库结构版本控制工具，它提供的sqlalchemy.versioningAPI及一个migrate的命令。
## 概念
关于SQLAlchemy-migrate的几个概念：
### repository
仓库，一个项目中可以存在多个repository，一个单独的repository是sqlachemy-migrate工作的最小单元，由migrate.py以及配置文件migrate.cfg以及修改数据结构的脚本或者SQL文件。同一个项目可以升级/降级至某个repository中的某个version.<!--more-->
### repository name
每个repository的名字，在同一个project中必须是全局唯一的.
### version
数据结构的版本号.
##工作流程
```
                       +---------+                                 
                       | version |                                 
                       +--+----+-+                                 
                          ^    |                                   
                          |    |                                   
           version_control|    |upgrade/downgrade                  
                          |    |                                   
                          |    |                                   
                          |    v                                   
                +--------------+------------+                      
                | migrate_version[db table] |                      
                +------------+--------------+                      
                             |                                     
                             |                                     
      +-----------------+----+----------+-------------------+      
      |                 |               |                   |      
      v                 v               v                   v      
+-----+----+       +----+-----+    +----+-----+        +----+-----+
| version1 |       | version2 |    | version3 |        | version4 |
+----------+       +----------+    +----------+        +----------+
```
> 以下操作如果不明确说明均是通过migrate.versioning API获取。

## 实战
1. 创建一个repository
2. 通过engine初始化repository的migrate version
3. 按照标准为version 脚本定义upgrade(升级)、downgrade(降级) 方法。
4. 通过指定版本号调用upgrade/downgrade方法对比当前version，按照相差版本号从小到大依次执行对应的脚本，从而达到对数据结构的升级/降级。
> 好的习惯是定义好升级对应的降级操作，当然downgrade非必须，可以在downgrade方法中引发一个异常，例如：raise NotImplementedError("Unsupport downgrade.")

### 安装
```
pip install sqlalchemy-migrate
```
### 创建一个空的repository
```
#-*- coding:utf-8 -*-
import os

from migrate.versioning import api
from migrate.versioning.repository import Repository

_REPOSITORY = None

def find_db_repository(name, **opts):
    global _REPOSITORY
    path = os.path.join(os.path.abspath(os.path.dirname(__file__)), name)
    if not os.path.exists(path):
        Repository.create(path, name, **opts)
    if _REPOSITORY is None:
        _REPOSITORY = Repository(path)
    return _REPOSITORY

if __name__ == '__main__':
    repo_name = 'fm'
    find_db_repository(repo_name)
```
### 初始化db version control
```
#-*- coding:utf-8 -*-
import os

import sqlalchemy

from migrate.versioning import api
from migrate.versioning.repository import Repository

_REPOSITORY = None
_ENGINE = None
sql_connection = 'mysql://fm:fm@localhost/fm'

def find_db_repository(name, **opts):
    global _REPOSITORY
    path = os.path.join(os.path.abspath(os.path.dirname(__file__)), name)
    if not os.path.exists(path):
        Repository.create(path, name, **opts)
    if _REPOSITORY is None:
        _REPOSITORY = Repository(path)
    return _REPOSITORY

def get_engine():
    global _ENGINE
    if _ENGINE is None:
        engine_args = {
            "pool_recycle": 5,
            "echo": False,
            "convert_unicode": True,
        }
        _ENGINE = sqlalchemy.create_engine(sql_connection, **engine_args)
        _ENGINE.connect()
    return _ENGINE

def db_version_control(name, version=None):
    repository = find_db_repository(name)
    api.version_control(get_engine(), repository, version)
    return version

if __name__ == '__main__':
    repo_name = 'fm'
    #find_db_repository(repo_name)
    db_version_control(repo_name)
```
> 执行后，将会在数据库fm中创建一张名为migrate_version的表，用来标示当前项目相对的repository所处的版本号。

### 获取当前版本
```
#-*- coding:utf-8 -*-
import os

import sqlalchemy

from migrate.versioning import api
from migrate.versioning.repository import Repository
from migrate import exceptions as versioning_exceptions

_REPOSITORY = None
_ENGINE = None
sql_connection = 'mysql://fm:fm@localhost/fm'

def find_db_repository(name, **opts):
    global _REPOSITORY
    path = os.path.join(os.path.abspath(os.path.dirname(__file__)), name)
    if not os.path.exists(path):
        Repository.create(path, name, **opts)
    if _REPOSITORY is None:
        _REPOSITORY = Repository(path)
    return _REPOSITORY

def get_engine():
    global _ENGINE
    if _ENGINE is None:
        engine_args = {
            "pool_recycle": 5,
            "echo": False,
            "convert_unicode": True,
        }
        _ENGINE = sqlalchemy.create_engine(sql_connection, **engine_args)
        _ENGINE.connect()
    return _ENGINE

def db_version_control(name, version=None):
    repository = find_db_repository(name)
    api.version_control(get_engine(), repository, version)
    return version

def db_version(name):
    repository = find_db_repository(name)
    try:
        return api.db_version(get_engine(), repository)
    except versioning_exceptions.DatabaseNotControlledError:
        db_version_control(name)
        return api.db_version(get_engine(), repository)

if __name__ == '__main__':
    repo_name = 'fm'
    #find_db_repository(repo_name)
    #db_version_control(repo_name)
    print db_version(repo_name)
```
### 升级/降级
```
#-*- coding:utf-8 -*-
import os

import sqlalchemy

from migrate.versioning import api
from migrate.versioning.repository import Repository
from migrate import exceptions as versioning_exceptions

_REPOSITORY = None
_ENGINE = None
sql_connection = 'mysql://fm:fm@localhost/fm'

def find_db_repository(name, **opts):
    global _REPOSITORY
    path = os.path.join(os.path.abspath(os.path.dirname(__file__)), name)
    if not os.path.exists(path):
        Repository.create(path, name, **opts)
    if _REPOSITORY is None:
        _REPOSITORY = Repository(path)
    return _REPOSITORY

def get_engine():
    global _ENGINE
    if _ENGINE is None:
        engine_args = {
            "pool_recycle": 5,
            "echo": False,
            "convert_unicode": True,
        }
        _ENGINE = sqlalchemy.create_engine(sql_connection, **engine_args)
        _ENGINE.connect()
    return _ENGINE

def db_version_control(name, version=None):
    repository = find_db_repository(name)
    api.version_control(get_engine(), repository, version)
    return version

def db_version(name):
    repository = find_db_repository(name)
    try:
        return api.db_version(get_engine(), repository)
    except versioning_exceptions.DatabaseNotControlledError:
        db_version_control(name)
        return api.db_version(get_engine(), repository)

def db_sync(name, version=None):
    if version is not None:
        try:
            version = int(version)
        except ValueError:
            raise
    current_version = db_version(name)
    repository = find_db_repository(name)
    if version is None or version > current_version:
        return api.upgrade(get_engine(), repository, version)
    else:
        return api.downgrade(get_engine(), repository, version)

if __name__ == '__main__':
    repo_name = 'fm'
    #find_db_repository(repo_name)
    #db_version_control(repo_name)
    #print db_version(repo_name)
    db_sync(repo_name)
```
### 编写升级/降级脚本
升降机默认会调用对应repository目录下的version中的脚本，脚本名以版本号加下划线开头，里面分别定义的upgrade，downgrade方法对应升级和降级的操作，具体的代码跟和sqlalchemy定义models一样，可以做以下操作：
* create a column
* drop a column
* alter a column
* rename a column
* rename an index
* Create primary key constraint
* Drop primary key constraint
* Create foreign key contraint
* Drop foreign key constraint
* Create unique key contraint
* Drop unique key constraint
* Create check key contraint
* Drop check key constraint

在修改数据结构的同时，也可以修改对应字段的值，或者填补之前的空白字段到达not null，这里需要注意的时在添加unique时注意生产环境数据是否已经有了不唯一，如果不唯一则需要先修改数据然后在添加对应的unique key。

> 参考: http://sqlalchemy-migrate.readthedocs.org/en/latest/versioning.html




</br>