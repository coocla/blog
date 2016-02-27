title: 深入了解rsync, 文件同步的军刀
categories: Linux
tags: [linux, rsync]
date: 2013-09-25 14:03:49
---
## rsync 简介
rsync（remote synchronize）是一个远程数据同步工具，可通过 LAN/WAN 快速同步多台主机之间的文件。也可以使用 rsync 同步本地硬盘中的不同目录。
rsync 是用于替代 rcp 的一个工具，rsync 使用所谓的 rsync算法 进行数据同步，这种算法只传送两个文件的不同部分，而不是每次都整份传送，因此速度相当快。 您可以参考 How Rsync Works A Practical Overview 进一步了解 rsync 的运作机制。
rsync 的初始作者是 Andrew Tridgell 和 Paul Mackerras，目前由 http://rsync.samba.org 维护。
rsync 支持大多数的类 Unix 系统，无论是 Linux、Solaris 还是 BSD上 都经过了良好的测试。 CentOS系统默认就安装了 rsync 软件包。<!--more--> 此外，在 windows 平台下也有相应的版本，如 cwrsync 和DeltaCopy 等。
rsync 具有如下的基本特性：
1. 可以镜像保存整个目录树和文件系统
2. 可以很容易做到保持原来文件的权限、时间、软硬链接等
3. 无须特殊权限即可安装
4. 优化的流程，文件传输效率高
5. 可以使用 rsh、ssh 方式来传输文件，当然也可以通过直接的 socket 连接
6. 支持匿名传输，以方便进行网站镜象

在使用 rsync 进行远程同步时，可以使用两种方式：远程 Shell 方式（建议使用 ssh，用户验证由 ssh 负责）和 C/S 方式（即客户连接远程 rsync 服务器，用户验证由 rsync 服务器负责）。
无论本地同步目录还是远程同步数据，首次运行时将会把全部文件拷贝一次，以后再运行时将只拷贝有变化的文件（对于新文件）或文件的变化部分（对于原有文件）。
本节重点介绍 rsync 客户命令的使用，有关 rsync 服务器的配置和使用请参见下节。
rsync 在首次复制时没有速度优势，速度不如 tar，因此当数据量很大时您可以考虑先使用 tar 进行首次复制，然后再使用 rsync 进行数据同步。

## 镜像、备份和归档
### 实施备份的两种情况：
* 需保留备份历史归档：在备份时保留历史的备份归档，是为了在系统出现错误后能恢复到从前正确的状态。这可以使用完全备份和增量备份来完成。
    * 可以使用 tar 命令保存归档文件。
    * 为了提高备份效率，也可以使用 rsync 结合 tar 来完成。
* 无需保留备份历史归档：若无需从历史备份恢复到正确状态，则只备份系统最“新鲜”的状态即可。这可以简单地使用 rsync 同步来完成。此时通常称为镜像。镜像可以分为两种：
    * 被镜像的目录在各个主机上保持相同的位置。此时一般是为了实施负载均衡而对多个主机进行同步镜像。例如：将主机 A 的 /srv/www 目录同步到主机 B 的 /srv/www 目录等。
    * 被镜像的目录在各个主机上不保持相同的位置。例如：主机 A 和主机 B 都运行着各自的业务，同时又互为镜像备份。此时主机 A 的 /srv/www 目录同步到主机 B 的 /backups/hosta/www 目录；主机 B 的 /srv/www 目录同步到主机 A 的 /backups/hostb/www 目录

### rsync 命令
rsync 是一个功能非常强大的工具，其命令也有很多功能选项。rsync 的命令格式为：
1. 本地使用: rsync [OPTION...] SRC... [DEST]
2. 通过远程 Shell 使用：
    rsync [OPTION...] [USER@]HOST:SRC... [DEST]
    rsync [OPTION...] SRC... [USER@]HOST:DEST
3. 访问 rsync 服务器:
    rsync [OPTION...] [USER@]HOST::SRC... [DEST]
    rsync [OPTION...] SRC... [USER@]HOST::DEST
    rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
    rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST
    
* SRC: 是要复制的源位置
* DEST: 是复制目标位置
* 若本地登录用户与远程主机上的用户一致，可以省略 USER@
* 使用远程 shell 同步时，主机名与资源之间使用单个冒号“:”作为分隔符
* 使用 rsync 服务器同步时，主机名与资源之间使用两个冒号“::”作为分隔符
* 当访问 rsync 服务器时也可以使用 rsync:// 
* “拉”复制是指从远程主机复制文件到本地主机
* “推”复制是指从本地主机复制文件到远程主机
* 当进行“拉”复制时，若指定一个 SRC 且省略 DEST，则只列出资源而不进行复制

### 常用选项：
```

选项	说明
-a, ––archive	归档模式，表示以递归方式传输文件，并保持所有文件属性，等价于 -rlptgoD (注意不包括 -H)
-r, ––recursive	对子目录以递归模式处理
-l, ––links	保持符号链接文件
-H, ––hard-links	保持硬链接文件
-p, ––perms	保持文件权限
-t, ––times	保持文件时间信息
-g, ––group	保持文件属组信息
-o, ––owner	保持文件属主信息 (super-user only)
-D	保持设备文件和特殊文件 (super-user only)
-z, ––compress	在传输文件时进行压缩处理
––exclude=PATTERN	指定排除一个不需要传输的文件匹配模式
––exclude-from=FILE	从 FILE 中读取排除规则
––include=PATTERN	指定需要传输的文件匹配模式
––include-from=FILE	从 FILE 中读取包含规则
––copy-unsafe-links	拷贝指向SRC路径目录树以外的链接文件
––safe-links	忽略指向SRC路径目录树以外的链接文件（默认）
––existing	仅仅更新那些已经存在于接收端的文件，而不备份那些新创建的文件
––ignore-existing	忽略那些已经存在于接收端的文件，仅备份那些新创建的文件
-b, ––backup	当有变化时，对目标目录中的旧版文件进行备份
––backup-dir=DIR	与 -b 结合使用，将备份的文件存到 DIR 目录中
––link-dest=DIR	当文件未改变时基于 DIR 创建硬链接文件
––delete	删除那些接收端还有而发送端已经不存在的文件
––delete-before	接收者在传输之前进行删除操作 (默认)
––delete-during	接收者在传输过程中进行删除操作
––delete-after	接收者在传输之后进行删除操作
––delete-excluded	在接收方同时删除被排除的文件
-e, ––rsh=COMMAND	指定替代 rsh 的 shell 程序
––ignore-errors	即使出现 I/O 错误也进行删除
––partial	保留那些因故没有完全传输的文件，以是加快随后的再次传输
––progress	在传输时显示传输过程
-P	等价于 ––partial ––progress
––delay-updates	将正在更新的文件先保存到一个临时目录（默认为 “.~tmp~”），待传输完毕再更新目标文件
-v, ––verbose	详细输出模式
-q, ––quiet	精简输出模式
-h, ––human-readable	输出文件大小使用易读的单位（如，K，M等）
-n, ––dry-run	显示哪些文件将被传输
––list-only	仅仅列出文件而不进行复制
––rsyncpath=PROGRAM	指定远程服务器上的 rsync 命令所在路径
––password-file=FILE	从 FILE 中读取口令，以避免在终端上输入口令，通常在 cron 中连接 rsync 服务器时使用
-4, ––ipv4	使用 IPv4
-6, ––ipv6	使用 IPv6
––version	打印版本信息
––help	显示帮助信息
```
* 若使用普通用户身份运行 rsync 命令，同步后的文件的属主将改变为这个普通用户身份。
* 若使用超级用户身份运行 rsync 命令，同步后的文件的属主将保持原来的用户身份。

## rsync 的基本使用
### 在本地磁盘同步数据
```
# rsync -a --delete /home /backups
# rsync -a --delete /home/ /backups/home.0
```
在指定复制源时，路径是否有最后的 “/” 有不同的含义，例如：
* /home ： 表示将整个 /home 目录复制到目标目录
* /home/ ： 表示将 /home 目录中的所有内容复制到目标目录

### 使用基于 ssh 的 rsync 远程同步数据
1. 同步静态主机表文件
    ```
    # 执行“推”复制同步（centos5 是可解析的远程主机名）
[root@soho ~]# rsync /etc/hosts centos5:/etc/hosts

# 执行“拉”复制同步（soho 是可解析的远程主机名）
[root@centos5 ~]# rsync soho:/etc/hosts /etc/hosts
```
2. 同步用户的环境文件
    ```
    # 执行“推”复制同步
[osmond@soho ~]$ rsync ~/.bash* centos5:

# 执行“拉”复制同步
[osmond@cnetos5 ~]$ rsync soho:~/.bash* .
```
3. 同步站点根目录
    ```
    # 执行“推”复制同步
[osmond@soho ~]$ rsync -avz --delete /var/www root@192.168.0.101:/var/www

# 执行“拉”复制同步
[osmond@cnetos5 ~]$ rsync -avz --delete root@192.168.0.55:/var/www /var/www
    ```
    * 使用基于 ssh 的 rsync 同步数据可以使用 -e ssh 参数，当前的 CentOS 默认指定使用 ssh 作为远程Shell。若您在其他系统上执行 rsync 命令，为确保使用 ssh 作为远程 Shell，请添加 -e ssh 参数。
    * 通常 rsync 命令在后台以 cron 任务形式执行，为了避免从终端上输入口令需要设置 ssh。ssh 的设置方法请参考 安全登录守护进程。
 
### 使用 rsync 从远程 rsync 服务器同步数据
下面以镜像 CentOS 和 Ubuntu 的软件库为例来说明。
您可以到如下站点查找离自己最近的提供 rsync 服务的镜像站点
* CentOS — http://www.centos.org/modules/tinycontent/index.php?id=13
* Ubuntu — https://launchpad.net/ubuntu/+archivemirrors
然后执行类似如下命令：
```
rsync -aqzH --delete --delay-updates \
rsync://mirror.centos.net.cn/centos /var/www/mirror/centos
rsync -azH --progress --delete --delay-updates \
rsync://ubuntu.org.cn/ubuntu /var/www/mirror/ubuntu/
rsync -azH --progress --delete --delay-updates \
rsync://ubuntu.org.cn/ubuntu-cn /var/www/mirror/ubuntu-cn/
```
为了每天不断更新，可以安排一个 cron 任务：
```
# crontab -e
# mirror centos at 0:10AM everyday
10 0 * * * rsync -aqzH --delete --delay-updates rsync://mirror.centos.net.cn/centos /var/www/mirror/centos/
# mirror ubuntu at 2:10AM everyday
10 2 * * * rsync -azH --progress --delete --delay-updates rsync://ubuntu.org.cn/ubuntu /var/www/mirror/ubuntu/
# mirror ubuntu-cn at 4:10AM everyday
10 4 * * * rsync -azH --progress --delete --delay-updates rsync://ubuntu.org.cn/ubuntu-cn /var/www/mirror/ubuntu-cn/
```
## 筛选 rsync 的传输目标
### 使用 --exclude/--include 选项
可以使用 ––exclude 选项排除源目录中要传输的文件；同样地，也可以使用 ––include 选项指定要传输的文件。
例如：下面的 rsync 命令将 192.168.0.101 主机上的 /www 目录（不包含 /www/logs 和 /www/conf子目录）复制到本地的 /backup/www/ 。
```
# rsync -vzrtopg --delete --exclude "logs/" --exclude "conf/" --progress \
backup@192.168.0.101:/www/ /backup/www/
```
又如：下面的 rsync 命令仅复制目录结构而忽略掉目录中的文件。
```
# rsync -av --include '*/' --exclude '*' \
backup@192.168.0.101:/www/ /backup/www-tree/
```
选项 ––include 和 ––exclude 都不能使用间隔符。例如：
```
--exclude "logs/" --exclude "conf/"
```
### 使用 --exclude-from/--include-from 选项
当 include/exclude 的规则较复杂时，可以将规则写入规则文件。使用规则文件可以灵活地选择传输哪些文件（include）以及忽略哪些文件（exclude）。
* 若文件/目录在剔除列表中，则忽略传输
* 若文件/目录在包含列表中，则传输之
* 若文件/目录未被提及，也传输之

在 rsync 的命令行中使用 ––exclude-from=FILE 或 ––include-from=FILE 读取规则文件。
规则文件 FILE 的书写约定：
* 每行书写一条规则 RULE
* 以 # 或 ; 开始的行为注释行

包含（include）和排除（exclude）规则的语法如下：
* include PATTERN 或简写为 + PATTERN
* exclude PATTERN 或简写为 - PATTERN

PATTERN 的书写规则如下：
* 以 / 开头：匹配被传输的跟路径上的文件或目录
* 以 / 结尾：匹配目录而非普通文件、链接文件或设备文件
* 使用通配符
* *：匹配非空目录或文件（遇到 / 截止）
* **：匹配任何路径（包含 / ）
* ?：匹配除了 / 的任意单个字符
* [：匹配字符集中的任意一个字符，如 [a-z] 或 [[:alpha:]]
* 可以使用转义字符 \ 将上述通配符还原为字符本身含义

## 例子
```
# 不传输所有后缀为 .o 的文件
- *.o

# 不传输传输根目录下名为 foo 的文件或目录
- /foo

# 不传输名为 foo 的目录
- foo/

# 不传输 /foo 目录下的名为 bar 的文件或目录
- /foo/bar
```

```
# 传输所有目录和C语言源文件并禁止传输其他文件
+ */
+ *.c
- *
```

```
# 仅传输 foo 目录和其下的 bar.c 文件
+ foo/
+ foo/bar.c
- *
```

将规则写入规则文件之后，如何在命令行上使用它呢？下面给出一个例子：
首先将下面的规则存入名为 www-rsync-rules 的文件
```
# 不传输 logs 目录
- logs/

# 不传输后缀为 .tmp 的文件
- *.tmp

# 传输 Apache 虚拟主机文档目录（/*/ 匹配域名）
+ /srv/www/
+ /srv/www/*/
+ /srv/www/*/htdocs/
+ /srv/www/*/htdocs/**

# 传输每个用户的 public_html 目录（/*/ 匹配用户名）
+ /home/
+ /home/*/
+ /home/*/public_html/
+ /home/*/public_html/**
# 禁止传输其他
- *
```
然后即可使用类似如下的 rsync 命令：
```
rsync -av --delete --exclude-from=www-rsync-rules / remotehost:/dest/dir
```





</br>