title: 个人博客备份存储dropbox
categories: Linux
tags: [linux, 备份, dropbox]
date: 2013-10-15 22:28:00
---
## 前言
现在个人博客越来、个人网站越来越多，身边大部分人都扮演着小小站长的角色，每个人手下都有着不同配置的VPS，作为一个Linux运维人员，拥有一个VPS的小小站长，细心经营捣鼓着自己的服务器/VPS，努力而细心的“经营”着自己的这份工作与爱好。本着事事无常的态度，不可再生的数据显的尤为重要，“备份”这个名词，总会让人的内心得到一丝安全感。

备份是门大学问，根据备份方式可分逻辑备份和物理备份。
根据备份时刻可分冷备份和热备份<!--more-->
根据存储介质可分本地备份和异地备份，而异地备份具有容灾性

对于个人VPS的数据备份，如果只将网页数据和数据库数据打包保存在本机的一个目录下，并不能完全达到数据备份的意义。而异地备份在于备份的数据会被存放在至少3个物理节点上，已达到异地容灾性备份，而鉴于成本考虑，不可能在买一个VPS进行异地容灾备份。但这并不代表对于VPS用户做到异地备份就没有其他备份方案了。

Dropbox，一个提供同步本地文件的网络存储在线应用，它提供了丰富的API，我们可以通过API将自己VPS备份的数据立刻同步到DropBox中，而DropBox提供的免费空间最够大部分站长的备份空间需求。我们利用DropBox这样的第三方脚本，来实现同步与删除。
## DropBox
### 配置自己的Dropbox，创建一个app
下载DropBox脚本，到自己的VPS上
```
# wget https://raw.github.com/andreafabrizi/Dropbox-Uploader/master/dropbox_uploader.sh
# chmod +x dropbox_uploader.sh
# ./dropbox_uploader.sh
```
运行该脚本，根据以下提示进行DropBox的设置
```
This is the first time you run this script.

1) Open the following URL in your Browser, and log in using your account: https://www2.dropbox.com/developers/apps
在浏览器中打开，https://www2.dropbox.com/developers/apps
2) Click on "Create App", then select "Dropbox API app"
点击"Create App"按钮，然后选择"Dropbox API app"
3) Select "Files and datastores"
选择"Files and datastores"选项
4) Now go on with the configuration, choosing the app permissions and access restrictions to your DropBox folder
现在去进行配置，选择这个app对你整个Dropbox文件夹的访问权限
5) Enter the "App Name" that you prefer (e.g. MyUploader113012919723169)
输入一个你喜欢的App名称

Now, click on the "Create app" button.

When your new App is successfully created, please type the
App Key, App Secret and the Permission type shown in the confirmation page:

# App key: xxxxxxxxxx
# App secret: xxxxxxxxx
# Permission type, App folder or Full Dropbox [a/f]: a

> App key is xxxxxxxxxx, App secret is xxxxxxxxxx and Access level is App Folder, it's ok? [y/n]y

> Token request... OK

Please open the following URL in your Browser, and allow Dropbox Uploader
to access your DropBox folder:

--> https://www2.dropbox.com/1/oauth/authorize?oauth_token=iP0TqrmjjKcAUfwW

Press enter when done...

> Access Token request... OK

Setup completed!
```
[](http://7xk38j.com1.z0.glb.clouddn.com/dropbox/1.png)
[](http://7xk38j.com1.z0.glb.clouddn.com/dropbox/2.png)
[](http://7xk38j.com1.z0.glb.clouddn.com/dropbox/3.png)
到此为止，关于Dropbox的设置就结束了，接下来按照自己的情况编写备份同步脚本。

### 编写备份脚本
```
#!/bin/bash
#
# when:2013/10/15
# who:http://blog.coocla.org

TODAY=`date -I` # 获取当前日期
BACKUP_LOG=/data/backup/backup_${TODAY}.log # 备份日志
Expire=`date -d -7day +"%Y-%m-%d"` # 获取7天前的日期
MYSQL_USER="root" # Mysql用户
MYSQL_PASS="rootpass" # Mysql密码
MYSQL_DB=('blog' 'yunxiaojia') # 要备份的数据库名
BACK_DIR=/data/backup # 备份存放的目录
Dropbox=/${TODAY} # Dropbox上创建的app存放目录，这里的根(/)是指app的根目录
WEB_DATA=/data/www/wwwroot # 网页文件目录

#Create Today BackupDirectory
if [ ! -d $BACK_DIR/$TODAY ];then
mkdir $BACK_DIR/$TODAY
fi

#Backup Mysql DB
echo "###############################################################" > $BACKUP_LOG
echo "Backup Mysql DB." >> $BACKUP_LOG
echo "Start Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG
for db in ${MYSQL_DB[@]};do
/usr/bin/mysqldump -u$MYSQL_USER -p$MYSQL_PASS --skip-opt --add-drop-table --create-options -q -e --set-charset --routines --single-transaction --master-data=2 $db > ${TODAY}_${db}_full_back.sql --log-error=$BACKUP_LOG
done
tar zcf ${TODAY}_db_full_back.tar.gz *.sql
rm -f *.sql
mv ${TODAY}_db_full_back.tar.gz ${BACK_DIR}/${TODAY}/
echo "Stop Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG

#Backup Website Data
echo "###############################################################" >> $BACKUP_LOG
echo "Backup Website Data." >> $BACKUP_LOG
echo "Start Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG
cd $WEB_DATA
tar zcf ${TODAY}_web_full_back.tar.gz ./* && cd -
mv ${WEB_DATA}/${TODAY}_web_full_back.tar.gz ${BACK_DIR}/${TODAY}/
echo "Stop Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG

#Upload Dropbox
echo "###############################################################" >> $BACKUP_LOG
echo "Upload backup." >> $BACKUP_LOG
echo "Start Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG
/usr/local/sbin/dropbox_uploader.sh upload ${BACK_DIR}/${TODAY}/${TODAY}_db_full_back.tar.gz ${Dropbox}/${TODAY}_db_full_back.tar.gz >> $BACKUP_LOG
/usr/local/sbin/dropbox_uploader.sh upload ${BACK_DIR}/${TODAY}/${TODAY}_web_full_back.tar.gz ${Dropbox}/${TODAY}_web_full_back.tar.gz >> $BACKUP_LOG
echo "Stop Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG

#Delete old data
echo "###############################################################" >> $BACKUP_LOG
echo "Delete expire data." >> $BACKUP_LOG
echo "Start Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG
find ${BACK_DIR} -mtime +3 | xargs rm -rf
/usr/local/sbin/dropbox_uploader.sh delete /$Expire/ >> $BACKUP_LOG
echo "Stop Time : `date +%F" "%H:%M:%S`" >> $BACKUP_LOG
```
查看日志信息如下：
```
###############################################################
Backup Mysql DB.
Start Time : 2013-10-15 22:12:11
Stop Time : 2013-10-15 22:12:21
###############################################################
Backup Website Data.
Start Time : 2013-10-15 22:12:21
Stop Time : 2013-10-15 22:12:27
###############################################################
Upload backup.
Start Time : 2013-10-15 22:12:27
 > Uploading "/data/backup/2013-10-15/2013-10-15_db_full_back.tar.gz" to "/2013-10-15/2013-10-15_db_full_back.tar.gz"... DONE
 > Uploading "/data/backup/2013-10-15/2013-10-15_web_full_back.tar.gz" to "/2013-10-15/2013-10-15_web_full_back.tar.gz"... DONE
Stop Time : 2013-10-15 22:12:42
###############################################################
Delete expire data.
Start Time : 2013-10-15 22:12:42
 > Deleting "/2013-10-08"... FAILED
Stop Time : 2013-10-15 22:12:42
```
然后将该脚本和dropbox_uploader.sh脚本放置于/usr/local/sbin或同一目录下，然后定义crontab计划任务：
根据日志中开始时间和结束时间，小伙伴们可以定义crontab开始的时间，由于我的内容比较少，在1分钟只能即可完成，我的crontab设置如下：
```
59 23 * * * /bin/sh /usr/local/sbin/backup.sh > /dev/null 2>&1
```
小伙伴们也可以下载DropBox客户端安装在自己的PC上，然后进行同步，手动运行以上脚本或者等待计划任务执行后，待自动同步后，即可在windows下查看的到刚才同步上去的备份文件啦！

好了，小伙伴们快去创建自己的异地备份策略吧~
