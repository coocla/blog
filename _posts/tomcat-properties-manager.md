title: tomcat配置文件参数，一机多实例管理配置
categories: Linux
tags: [linux, tomcat]
date: 2013-09-29 22:33:00
---
## 了解Tomcat
Tomcat不是一个完整意义上的Jave EE（j2ee）服务器，因为它没有提供完整的Java EE企业应用平台的API。但是由于Tomcat遵循apache开源协议，并且对当前Java开发框架开源组件Structs、Spring和Hibernate等实现完美支持，因此tomcat被众多企业用来部署配置众多的Java应用程序，实现替代一些商业的Java应用服务器。
## Tomcat的目录结构
要部署使用tomcat，则必须了解tomcat的目录结构以及各目录的作用。这里以tomcat7为例<!--more-->
安装Tomcat，此处不在阐述,进入tomcat安装目录下：
```
|-- bin
|   |-- bootstrap.jar tomcat启动时所依赖的一个类，在启动tomcat时会发现Using CLASSPATH: 是加载的这个类
|   |-- catalina-tasks.xml 定义tomcat载入的库文件，类文件
|   |-- catalina.bat
|   |-- catalina.sh                  tomcat单个实例在Linux平台上的启动脚本
|   |-- commons-daemon-native.tar.gz            jsvc工具，可以使tomcat已守护进程方式运行，需单独编译安装
|   |-- commons-daemon.jar            jsvc工具所依赖的java类
|   |-- configtest.bat
|   |-- configtest.sh         tomcat检查配置文件语法是否正确的Linux平台脚本
|   |-- cpappend.bat
|   |-- daemon.sh tomcat已守护进程方式运行时的，启动，停止脚本
|   |-- digest.bat
|   |-- digest.sh
|   |-- setclasspath.bat
|   |-- setclasspath.sh
|   |-- shutdown.bat
|   |-- shutdown.sh tomcat服务在Linux平台下关闭脚本
|   |-- startup.bat
|   |-- startup.sh          tomcat服务在Linux平台下启动脚本
|   |-- tomcat-juli.jar
|   |-- tomcat-native.tar.gz  使tomcat可以使用apache的apr运行库，以增强tomcat的性能需单独编译安装
|   |-- tool-wrapper.bat
|   |-- tool-wrapper.sh
|   |-- version.bat
|   `-- version.sh 查看tomcat以及JVM的版本信息
|-- conf 顾名思义，配置文件目录
|   |-- catalina.policy 配置tomcat对文件系统中目录或文件的读、写执行等权限，及对一些内存，session等的管理权限
|   |-- catalina.properties 配置tomcat的classpath等
|   |-- context.xml tomcat的默认context容器
|   |-- logging.properties 配置tomcat的日志输出方式
|   |-- server.xml        tomcat的主配置文件
|   |-- tomcat-users.xml        tomcat的角色(授权用户)配置文件
|   `-- web.xml tomcat的应用程序的部署描述符文件
|-- lib
|-- logs 日志文件默认存放目录
|-- temp
|   `-- safeToDelete.tmp
|-- webapps           tomcat默认存放应用程序的目录，好比apache的默认网页存放路径是/var/www/html一样
|   |-- docs tomcat文档
|   |-- examples                     tomcat自带的一个独立的web应用程序例子
|   |-- host-manager              tomcat的主机管理应用程序
| |   |-- META-INF           整个应用程序的入口，用来描述jar文件的信息
| |   |   `-- context.xml     当前应用程序的context容器配置，它会覆盖tomcat/conf/context.xml中的配置
| |   |-- WEB-INF  用于存放当前应用程序的私有资源
| |   |   |-- classes  用于存放当前应用程序所需要的class文件
|       |   | |-- lib          用于存放当前应用程序锁需要的jar文件
| |   |   `-- web.xml 当前应用程序的部署描述符文件，定义应用程序所要加载的serverlet类，以及该程序是如何部署的
|   |-- manager                  tomcat的管理应用程序
|   |-- ROOT              指tomcat的应用程序的根，如果应用程序部署在ROOT中，则可直接通过http://ip:port 访问到
`-- work 用于存放JSP应用程序在部署时编译后产生的class文件
```
## 了解tomcat的主配置文件(server.xml)结构及含义
如下图所示，前端请求被tomcat直接接收或者由前端的代理，通过HTTP，或者AJP代理给Tomcat，此时请求被tomcat中的connector接收，不同的connector和Engine被service组件关联起来，在一个Engine中定义了许多的虚拟主机，由Host容器定义，每一个Host容器代表一个主机，在各自的Host中，又可以定义多个Context，用此来定义一个虚拟主机中的多个独立的应用程序。
[](http://7xk38j.com1.z0.glb.clouddn.com/tomcat/170420310.jpg)
## 单实例应用程序配置一例
规划：
网站网页目录：/web/www      域名：www.test1.com
论坛网页目录：/web/bbs     URL：bbs.test1.com/bbs
网站管理程序：$CATALINA_HOME/wabapps   
URL：manager.test.com    
允许访问地址：172.23.136.*
*conf/server.xml*
```
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JasperListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  <GlobalNamingResources>
  <!-- 全局命名资源，来定义一些外部访问资源，其作用是为所有引擎应用程序所引用的外部资源的定义 --!>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  <!-- 定义的一个名叫“UserDatabase”的认证资源，将conf/tomcat-users.xml加载至内存中，在需要认证的时候到内存中进行认证 -->
  <Service name="Catalina">
  <!-- # 定义Service组件，同来关联Connector和Engine，一个Engine可以对应多个Connector，每个Service中只能一个Engine --!>
    <Connector port="80" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
    <!-- 修改HTTP/1.1的Connector监听端口为80.客户端通过浏览器访问的请求，只能通过HTTP传递给tomcat。  -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="Catalina" defaultHost="test.com">
    <!-- 修改当前Engine，默认主机是，www.test.com  -->
    <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
    </Realm>
    # Realm组件，定义对当前容器内的应用程序访问的认证，通过外部资源UserDatabase进行认证
      <Host name="test.com"  appBase="/web" unpackWARs="true" autoDeploy="true">
      <!--  定义一个主机，域名为：test.com，应用程序的目录是/web，设置自动部署，自动解压    -->
        <Alias>www.test.com</Alias>
        <!--    定义一个别名www.test.com，类似apache的ServerAlias -->
        <Context path="" docBase="www/" reloadable="true" />
        <!--    定义该应用程序，访问路径""，即访问www.test.com即可访问，网页目录为：相对于appBase下的www/，即/web/www，并且当该应用程序下web.xml或者类等有相关变化时，自动重载当前配置，即不用重启tomcat使部署的新应用程序生效  -->
        <Context path="/bbs" docBase="/web/bbs" reloadable="true" />
        <!--  定义另外一个独立的应用程序，访问路径为：www.test.com/bbs，该应用程序网页目录为/web/bbs   -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="/web/www/logs"
               prefix="www_access." suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        <!--   定义一个Valve组件，用来记录tomcat的访问日志，日志存放目录为：/web/www/logs如果定义为相对路径则是相当于$CATALINA_HOME，并非相对于appBase，这个要注意。定义日志文件前缀为www_access.并以.log结尾，pattern定义日志内容格式，具体字段表示可以查看tomcat官方文档   -->
      </Host>
      <Host name="manager.test.com" appBase="webapps" unpackWARs="true" autoDeploy="true">
      <!--   定义一个主机名为man.test.com，应用程序目录是$CATALINA_HOME/webapps,自动解压，自动部署   -->
        <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="172.23.136.*" />
        <!--   定义远程地址访问策略，仅允许172.23.136.*网段访问该主机，其他的将被拒绝访问  -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="/web/bbs/logs"
               prefix="bbs_access." suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        <!--   定义该主机的访问日志      -->
      </Host>
    </Engine>
  </Service>
</Server>
conf/tomcat-users.xml
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
  <role rolename="manager-gui" />
  <!--  定义一种角色名为：manager-gui    -->
  <user username="cz" password="manager$!!110" roles="manager-gui" />
  <!--  定义一个用户的用户名以及密码，并赋予manager-gui的角色    -->
</tomcat-users>
```
由以上配置不难看出存在的一个问题。如果我们想要对其中一个应用程序的配置做一些修改，那么就必须重新启动tomcat，那样势必就会影响到另外两个应用程序的正常服务。因此以上配置是不适合线上使用的，因此需要将其配置为多实例，每个实例只跑一个独立的应用程序，那样我们应用程序之间就不会在互相受到影响。但是我们将面临这样一个问题，80端口只能被一个HTTP/1.1 Connector监听，而三个tomcat实例则至少需要3个HTTP/1.1 Connector，这样我们就需要一个前端代理做分发处理，接收HTTP 80端口的请求，按域名通过每个tomcat实例的AJP/1.3 Connector传递请求。而前端的代理选择apache，基于这样的思路，我们还可以做到tomcat的负载均衡，而且apache会将接收的HTTP超文本传输报文重新封装成二进制格式通过AJP/1.3 协议传递给后端的tomcat处理，在效率上也是有明显的提升。

## 结合apache构造多实例tomcat集群apache结合tomcat的方式主要有三种：mod_jk，ajp_proxy，
http_proxy(以http协议代理给tomcat)，而当前使用最多的还是要数mod_jk。因为mod_jk出现的较早，技术已经相当成熟，而且具备集群节点健康检测功能，支持大型的AJP数据包。

安装apache,tomcat这里不在详述
安装tomcat-connectors
```
# tar zxvf tomcat-connectors-1.2.30-src.tar.gz
# cd tomcat-connectors/src
# ./configure --with-apxs=/usr/local/apache/bin/apxs
# make && make install
```
### 单独建立httpd-jk.conf
单独建立httpd-jk.conf文件用来配置mod_jk的相关设置
```
vim /usr/local/apache2/conf/extra/httpd-jk.conf
```
配置apache装载mod_jk.so模块
```
LoadModule jk_module modules/mod_jk.so
```
指定保存了worker相关工作属性定义的配置文件
```
JkWorkersFile /usr/local/apache/conf/extra/workers.properties
```
定义mod_jk模块的日志文件
```
JkLogFile /usr/local/apache/logs/mod_jk.log
```
定义mod_jk模块日志的记录级别
```
JkLogLevel info
```
### 建立worker相关工作属性定义的配置文件
vim /usr/local/apache/conf/extra/workers.properties
```
worker.list=Cluster1,stat
worker.web2.port=8003
worker.web2.host=172.23.138.19
worker.web2.type=ajp13
worker.web2.lbfactor=1

worker.web3.port=8003
worker.web3.host=172.23.136.144
worker.web3.type=ajp13
worker.web3.lbfactor=1
worker.Cluster1.type=lb
worker.Cluster1.balance_workers=web2,web3
worker.Cluster1.sticky_session = 1

worker.stat.type=status
worker.list=Cluster2
worker.manager2.port=7003
worker.manager2.host=172.23.138.19
worker.manager2.type=ajp13
worker.manager2.lbfactor=1

worker.manager3.port=7003
worker.manager3.host=172.23.136.144
worker.manager3.type=ajp13
worker.manager3.lbfactor=1
worker.Cluster2.type=lb
worker.Cluster2.balance_workers=manager2,manager3
worker.Cluster2.sticky_session = 1
```
### 配置后端tomcat多实例
使用脚本快速部署tomcat实例，修改新实例的AJP/1.3 Connector的监听端口，HTTP/1.1 Connector的监听端口以及Server容器的监听端口。脚本内容如下：
```
#!/bin/bash
# when:2013-01-21
# who: blog.coocla.org
#
Java_Home=/usr/java/jdk1.7.0_10
Tomcat_Home=/usr/local/tomcat_7
Tomcat_User=tomcat
New_instance=/usr/local/new
if [ ! -d $New_instance ];then
        mkdir -p $New_instance
else
        echo "The parh alreadly exists..."
        exit
fi
id $Tomcat_User 2&> /dev/null & useradd -r $Tomcat_User
cp -r $Tomcat_Home/conf $New_instance
mkdir -p $New_instance/{logs,temp,webapps/ROOT,work}
cat > $New_instance/tomcat.sh << EOF
#!/bin/sh
JAVA_HOME=`echo $Java_Home`
JAVA_OPTS="-Xms64m -Xmx128m"
CATALINA_HOME=`echo $Tomcat_Home`
CATALINA_BASE=`echo $New_instance`
export JAVA_HOME JAVA_OPTS CATALINA_HOME CATALINA_BASE
su `echo $Tomcat_User` \$CATALINA_HOME/bin/catalina.sh \$1
EOF
cat > $New_instance/webapps/ROOT/index.jsp << EOF
<html><body><center>
<h1>This is a new tomcat instance!</h1>
</br>
Now time is: <%=new java.util.Date()%>
</center>
</body></html>
EOF
chown $Tomcat_User:$Tomcat_User -R $New_instance
```
现在后端每台tomcat节点的配置状况如下：
www.test.com实例：
```
<Server port="8000">
<Connector port="8001" protocol="HTTP/1.1">
<Connector port="8003" protocol="AJP/1.3">
</Server>
```
manager.test.com实例：
```
<Server port="7000">
<Connector port="7001" protocol="HTTP/1.1">
<Connector port="7003" protocol="AJP/1.3">
</Server>
```
使用新实例中的tomcat.sh进行启动每个实例
### 配置多域名的负载均衡
vim /usr/local/apache/conf/extra/httpd-vhosts.conf 
```
NameVirtualHost *:80
<VirtualHost *:80>
ServerName www.test.com
JkMount /* Cluster1
</VirtualHost>
<VirtualHost *:80>
ServerName manager.test.com
JkMount /* Cluster2
JkMount /status stat
</VirtualHost>
```

到此基于多域名多实例的tomcat负载均衡集群构建完成，启动apache即可，基于当前结构还可结合持久会话管理器(PersistentManager)来实现会话持久的效果。





</br>



