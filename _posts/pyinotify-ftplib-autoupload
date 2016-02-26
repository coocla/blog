title: pyinotify结合ftplib自动上传新建的文件
categories: Python
tags: [python, pyinotify, ftplib]
date: 2014-05-05 14:51:34
---
### 应用场景
从国内往国外上传，因国际带宽影响，速度很慢，于是做了一个中转FTP，而自动上传需求也就诞生了

代码地址：https://github.com/coocla/linux/blob/master/ftp/autoupload_ftp.py  
sftp类型：https://github.com/coocla/linux/blob/master/sftp/autoupload_sftp.py<!--more-->

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-
# This is a new file will be automatically uploaded to the ftp python script.
# Author: http://blog.coocla.org

import pyinotify
import os
from ftplib import FTP

WATCHDIR="/tmp/"
HOST="xxxxx"
USER="xxxxx"
PASS="xxxxx"
pid_file="/tmp/monitor_upload.pid"
log_file="/tmp/monitor_upload.log"

class OnCreateClick(pyinotify.ProcessEvent):
  def process_IN_CREATE(self, event):
    abspath=os.path.join(event.path, event.name)
    if os.path.isdir(abspath):
      print "create directory:  %s" % event.name
    elif os.path.isfile(abspath):
      print "create file:  %s" % event.name
      relname=abspath.split(WATCHDIR)[1]
      relpath=os.path.dirname(relname)
      upload(abspath, relpath, event.name)

def main():
  wm = pyinotify.WatchManager()
  notifier = pyinotify.Notifier(wm, OnCreateClick())
  wm.add_watch(WATCHDIR, pyinotify.IN_CREATE, rec=True, auto_add=True)
  notifier.loop(daemonize=True, pid_file=pid_file, stdout=log_file)

def upload(abspath, relpath, filename):
  ftp = FTP()
  ftp.connect(HOST)
  ftp.login(USER, PASS)
  bufsize = 1024
  ftp.cwd("xxxxxxx")
  try:
    ftp.cwd(relpath)
  except:
    for remote_path in relpath.split(os.path.sep):
      ftp.mkd(remote_path)
      ftp.cwd(remote_path)
  filehandler = open(abspath, "rb")
  try:
    ftp.storbinary("STOR " + filename, filehandler, bufsize)
    print "Upload file %s success." % abspath
  except:
    print "Upload file %s error." % abspath
  filehandler.close()
  ftp.quit()

if __name__ == '__main__':
  if os.path.isfile(pid_file):
    os.remove(pid_file)
  main()
```



</br>