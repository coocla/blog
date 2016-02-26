title: Python pwd和grp模块
categories: Python
tags: [python, pwd, grp]
date: 2014-04-22 11:38:46
---
## pwd
提供了一个Unix 密码数据库（/etc/passwd）的接口，这个数据库包含本地机器用户账户信息。

### pwd.getpwuid(uid)
返回对应uid的用户信息
### pwd.getpwnam(name)
返回对应name的用户信息
### pwd.getpwall()
返回所有用户信息<!--more-->

```
import pwd

def get_user():
  all_user = {}
  for user in pwd.getpwall():
    all_user[user[0]] = all_user[user[2]] = user
  return all_user

def userinfo(uid):
  return get_user()[uid]

print userinfo(0)
pwd.struct_passwd(pw_name='root', pw_passwd='x', pw_uid=0, pw_gid=0, pw_gecos='root', pw_dir='/root', pw_shell='/bin/bash')
print userinfo('root')
pwd.struct_passwd(pw_name='root', pw_passwd='x', pw_uid=0, pw_gid=0, pw_gecos='root', pw_dir='/root', pw_shell='/bin/bash')
```

## grp
提供了一个Unix 用户组/group（/etc/group）数据库的接口。
### grp.getgrgid(gid):
返回对应gid的组信息
### grp.getgrname(name):
返回对应group name的组信息
### grp.getgrall():
返回所有组信息

pwd和grp的用法都十分相似，对于操作linux系统用户和组十分方便灵活，推荐使用。



</br>