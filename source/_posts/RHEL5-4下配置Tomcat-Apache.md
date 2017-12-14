---
title: RHEL5.4下配置Tomcat+Apache
date: 2017-11-02 09:52:44
tags:
	- RHEL
	- Linux
	- Tomcat
	- Apache
  - 系统运维
categories: Linux
---

笔者一直为一个项目进行后期开发以及维护，因为基础服务提供者的质量太差，导致最近作为服务器的虚拟机又崩溃了。而且崩溃最严重的地方是虚拟存储服务，以致于现今需要在一台新的虚拟服务器上重新部署项目。因为上一年也经历过该情况，所以将部署过程写成博客，以防再次发生时没有部署文档参考。

## 环境支持
1. RHEL 5.4
1. gcc
1. ssh，Windows下可选择XShell。

## 所需工具
1. MySQL，笔者部署用祖传的`MySQL-5.6.25-1.rhel5.x86_64.rpm-bundle.tar`
1. JDK-1.7.0_21
1. Apache Tomcat 7.0.62
1. [Apache 2.2.34](http://mirror.bit.edu.cn/apache//httpd/httpd-2.2.34.tar.gz)
1. [Apache Active MQ 5.7.0](http://archive.apache.org/dist/activemq/apache-activemq/5.7.0/apache-activemq-5.7.0-bin.tar.gz)
1. Tomcat Connector - [JK-1.2.42](http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.42-src.tar.gz)

### 工具说明

因为这是一个实际部署的项目，所以用的工具都是一直传下来的，不能随意更换，不然很有可能导致服务器无法正常运行项目。例如笔者最近部署贪方便，使用了JDK1.8编译项目，就算能够通过设置参数让JDK1.8编译等级为1.7,但是始终无法编译成功。以下针对使用的工具进行简单说明:

1. 这些工具版本最好是固定这个组合，别的版本组合不能确保项目能够成功部署运行，有些工具可能太老了，已经不提供官方下载了，只能通过别的方法获取，或者尝试使用近似的版本。
1. 笔者也想全程使用ssh远程控制服务器，但是因为是通过堡垒机登录的服务器，所以涉及文件传输的时候使用`scp`等命令是无法正常工作的，而祖传下来的方法是在Windows下使用X-Shell登录服务器，而在服务器使用`rz -be`命令接收文件，`sz ${文件路径}`将文件发送到本地机器上。
1. MySQL，JDK，Tomcat，Apache是常用的工具，这里不加以说明。
1. Apache Active MQ，百度上是介绍是用来集群进行消息传递的，但是在该项目的主要提供邮件发送服务。
1. Tomcat Connector，用于提供Apache的负载均衡时，Apache与Tomcat的连接。

### 初始准备

笔者将所需的工具先传输到了`/home/trans/recv`目录下，其中包含打包好的项目，名为`Demo.war`。
虚拟服务器上只有一个`root`账户，注意操作全程是在`root`权限下进行。

## 安装MySQL

在工具所在的目录下，执行:
``` bash
$ mkdir mysql-bundle
$ tar xvf MySQL-5.6.25-1.rhel5.x86_64.rpm-bundle.tar -C ./mysql-bundle
$ cd mysql-bundle
```
解压文件并进入目录。

执行命令:
``` bash
$ rpm -ivh MySQL-server-5.6.25-1.rhel5.x86_64.rpm --nodeps --force
$ rpm -ivh MySQL-client-5.6.25-1.rhel5.x86_64.rpm --nodeps --force
```
安装MySQL服务器以及客户端，里面加了`--force`参数是为了跳过GPG keys检测，不然无法成功安装。

该版本的MySQL安装之后，`root`的默认密码不为空，执行:
``` bash
$ cat ~/.mysql_secret
```
查看文本，获取初始密码。

设置MySQL `root`账户的初始密码:
``` bash
$ mysqladmin -u root -p password ${新密码}
```
之后会询问之前的密码，输入在.mysql_secret看到的密码即可修改成功。

进入mysql建立项目所需的数据库和表，注意建库的时候使用:
``` bash
$ CREATE DATABASE '${库名}' DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
```
指定库的编码为`UTF-8`，或者根据项目需要更改编码。

至此，MySQL的安装以及初始配置完成。

## 安装JDK

卸载系统自带的open JDK：
``` bash
$ rpm -qa | grep gcj
```
看到的一行同时具有`java`和`gcj`字样的，就是卸载目标使用`yum remove`命令卸载即可。
系统中也有可能存在其他的jdk，同样卸载即可。


展开JDK压缩包，并指定到:
``` bash
$ tar zxvf jdk-7u21-linux-x64.gz -C /usr/local/Java/
```

添加`Java`的系统变量:
``` bash
$ vim /etc/profile
```

添加到文件尾:
``` code
export JAVA_HOME=/usr/local/Java/jdk1.7.0_21
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```

可以:
``` bash
$ source /etc/profile
```
更新一下系统变量。

## 安装Apache

因为是一个实际部署的站点，故首先需要对本机的路由进行一些修改:
``` bash
$ vim etc/hosts
```
内容修改为:
``` code
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.0.1   ${站点名}
```
后推出编辑器。

展开压缩包并进入目录:
``` bash
$ tar zxvf httpd-2.2.34.tar.gz
$ cd httpd-2.2.34
```

创建安装目录并使用命令安装:
``` bash
$ mkdir /usr/local/apache
$ ./configure--prefix=/usr/local/apache --enable-so --enable-proxy --enable-proxy_http=shared --enable-module=so --enable-mods-shared=all --enable-proxy-ajp=shared  --enable-proxy-balancer -with-mpm=worker
$ make && make install
```
进入漫长的等待编译过程，中间可能会碰上`apr`、`apr-util`、`pcre`缺失，在这该虚拟机提供的环境暂不缺失，故在这里不展开细说。

安装完毕后开始配置Apache。
``` bash
$ cd /usr/local/apache/conf
$ vim httpd.conf
```
首先修改`ServerName`行为:
``` code
ServerName ${站点名}:${端口号}
```
确定`DocumentRoot`为:
``` code
DocumentRoot "/usr/local/apache/htdocs"
```

将以下:
``` code
Include conf/extra/httpd-mpm.conf
Include conf/extra/httpd-vhosts.conf
Include conf/extra/httpd-default.conf
```
的注释去除，启用这些配置。

修改`extra/httpd-vhost.conf`:
``` bash
$ vim extra/httpd-vhost.conf
```
内容为:
``` code
<VirtualHost *:80>
    DocumentRoot "/usr/local/apache/htdocs"
    ServerName ${站点名}
   <Directory "/usr/local/apache/htdocs">
        Options Indexes FollowSymLinks
        AllowOverride None
   </Directory>
</VirtualHost>

```
保存退出。

之后可以使用:
``` bash
$ /usr/local/apache/bin/apachectl start
```
启动httpd，在别的机器上访问该站点，可以看到返回的网页，其中有字符串类似于"Hello world!"，证明站点上的apache部署成功。
使用命令:
``` bash
$ /usr/local/apache/bin/apachectl stop
```
停止apache服务，进行Tomcat的安装配置。

## 安装Tomcat

回到存放工具压缩包的目录，执行:
``` bash
$ tar zxvf apache-tomcat-7.0.62.tar.gz -C /opt/
$ cd /opt/apache-tomcat-7.0.62/
```
解压并开始配置tomcat。
修改`conf/catalina.sh`，在文件配置信息开始段添加:
``` code
JAVA_OPTS="-Xms1024m -Xmx4096m -XX:PermSize=256m -XX:MaxPermSize=512m"
export TOMCAT_HOME=/opt/apache-tomcat-7.0.62
export CATALINA_HOME=/opt/apache-tomcat-7.0.62
export JRE_HOME=/usr/local/Java/jdk1.7.0_21/jre
export JAVA_HOME=/usr/local/Java/jdk1.7.0_21
```
一个是告知tomcat，该系统的jdk的位置;第二个功能，是配置JVM的启动参数，上一年重新配置该服务器的时候，一切配置正常，从外部访问该站点却死活返回不了完整的页面，经过两三天的折腾，终于发现是tomcat启动JVM的默认内存太小，以至于部署项目以及初始化tomcat过程虚拟机的内存就爆满了，不断地触发GC，使用`top`命令可以看到虚拟机的CPU占用率极高，当时也没有HTTP访问。刚开始从tomcat的log里面找不到明显的异常记录，就把解决问题的方向指向了apache整合tomcat中去，配置JK模块修改了很多个版本都不行。最后还是老师兄厉害，整个部署错误就在tomcat的log的一句`Memory Exceeded`里面，而不是常见的一大段的异常报告文本段。
今年这一次重新部署也类似，不过有了一年的经验，找出了单句严重错误日志，是tomcat在初始化一个监听器的时候缺少了这个类的实现，还有零散的错误，也是之前提到了这个是我用JDK1.8编译项目出的问题，用祖传的JDK1.7-21编译就没问题了。

一般的教程到这里会让启动tomcat下的`bin/startup.sh`，然后通过本地浏览器参看tomcat是否成功配置，但是我们处于没有图形化界面的服务器中，所以这里笔者假定tomcat是能够正常启动的，或者可以先把apache的根目录设置为`/opt/apache-tomcat-7.0.62/webapps`，之后从能够使用浏览器的外部机器访问站点，如果能够访问到tomcat的主页的话就没问题了。

将要发布的项目的war复制到tomcat下，执行:
``` bash
$ cp /home/trans/recv/Demo.war /opt/apache-tomcat-7.0.62/webapps/
```

修改`conf/server.xml`:
``` bash
$ vim conf/server.xml
```
其中有将`port`值为`8080`以及`8009`的`Connector`配置段修改为:
``` code
 <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" useBodyEncodingForURI="true" URIEncoding="UTF-8"/>
			   
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" useBodyEncodingForURI="true" URIEncoding="UTF-8" protocolHandlerClassName="org.apache.jk.server.jkCoyoteHandler"/>
```
因为Apache整合tomcat中用的JK模块是用`AJP`进行通信的，所以`8009`端口配置要正确，`AJP`的版本要和稍后配置的`workers.properties`中的版本一致，这里采用的`1.3`。
继续修改`server.xml`中的`Engine`:
``` code
<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat1">
```
这里要注意的是`jvmRoute`的值，这里需要稍后配置的`workers.properties`中的`worker`保持一致，否则JK模块无法正常找到该tomcat完成整合。
继续修改`Host`段:
``` code
<Host name="localhost" appBase="/usr/local/apache/htdocs/Demo" autoDeploy="true" unpackWARS="true" xmlNamespaceAware="false" xmlValidation="false">
        <Context path="/" docBase="/opt/apache-tomcat-7.0.62/webapps/Demo.war" reloadable="true" debug="0" crossContext="true"/>
</Host>
```
这里配置了项目发布到apache中的本地位置，以及项目的存放位置等配置。

其实应该还有集群时的`Cluster`段的配置，当年老师兄传给我的时候说这段是不需要配的，因为项目根本不是分布式的，虽然apache那边是配置好的，但是机器只有一台，等以后有机会再配置集群的`Session`问题也不迟。这个项目都已经老年时期了，所以是没可能的了，所以需要配置集群的看官抱歉了，这里没有，只能自行百度了。

至此，tomcat的配置完成。

## 编译安装JK模块到Apache中

回到工具包的存放目录，执行:
``` bash
$ tar zxvf tomcat-connectors-1.2.42-src.tar.gz
$ cd tomcat-connectors-1.2.42-src/native
```
解压tomcat connector并进入目录。
编译JK模块:
``` bash
$ chmod 755 buildconf.sh
$ ./buildconf.sh
$ ./configure --with-apxs=/usr/local/apache/bin/apxs
$ make && make install
```
耐心等待过程的完成,完成后apache的`modules`目录下会出现一个`mod_jk.so`的动态库。

## Apache整合Tomcat

继续打开:
``` bash
$ vim /usr/local/apache/conf/httpd.conf
```

在`Include`段添加一句:
``` code
Include conf/mod_jk.conf
```

在`conf`目录下新建一个`mod_jk.conf`文件，添加如下内容:
``` code
LoadModule jk_module modules/mod_jk.so
JkWorkersFile conf/workers.properties
```

在`conf`目录下新建一个`workers.properties`文件，添加如下内容:
``` code
workers.java_home=/usr/local/Java/jdk1.7.0_21
ps=/

worker.list=controller,tomcat1

#Set properties for tomcat1
worker.tomcat1.type=ajp13  
worker.tomcat1.host=localhost  
worker.tomcat1.port=8009
worker.tomcat1.lbfactor=50  
worker.tomcat1.cachesize=10  
worker.tomcat1.cache_timeout=600  
worker.tomcat1.socket_keepalive=1  
worker.tomcat1.socket_timeout=300

worker.controller.type=lb
worker.controller.balance_workers=tomcat1
worker.controller.sticky_session=false
```

随后修改`conf/extra/httpd-vhosts.conf`:
``` code
<VirtualHost *:80>
    DocumentRoot "/usr/local/apache/htdocs/Demo"
    ServerName ${站点名}
   <Directory "/usr/local/apache/htdocs/Demo">
        Options Indexes FollowSymLinks
        AllowOverride None
   </Directory>
  JkMount /* tomcat1
  JkMount /*.* tomcat1
</VirtualHost>
```

至此，配置已经给好了，剩下的就是启动测试了。

## 设置tomcat和apache为系统服务

创建`/etc/init.d/tomcat`，添加内容:
``` code
# !/bin/sh
# tomcat
# chkconfig: 345 63 37
# description: tomcat.
# processname tomcat 7.0

export JAVA_HOME=/usr/local/Java/jdk1.7.0_21
export CLASSPATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib
. /etc/rc.d/init.d/functions
. /etc/sysconfig/network
case $1 in
start)
sh /opt/apache-tomcat-7.0.62/bin/startup.sh
;;
stop)
sh /opt/apache-tomcat-7.0.62/bin/shutdown.sh
;;
restart)
sh /opt/apache-tomcat-7.0.62/bin/shutdown.sh
sh /opt/apache-tomcat-7.0.62/bin/startup.sh
;;
esac
exit 0
```

创建`/etc/init.d/tomcat`,添加内容:
``` code
# !/bin/sh
. /etc/rc.d/init.d/functions
. /etc/sysconfig/network
case $1 in
start)
sh /usr/local/apache/bin/apachectl start
;;
stop)
sh /usr/local/apache/bin/apachectl stop
;;
restart)
sh /usr/local/apache/bin/apachectl restart
;;
esac
exit 0
```

赋予执行权限:
``` bash
$ chmod a+x /etc/init.d/tomcat /etc/init.d/httpd
```

让`service`感知到新的服务:
``` bash
$ service reload
```

然后就可以用:
``` bash
$ service httpd start
$ service tomcat start
```
以及`restart`等选项便捷启动。

老师兄传的是，反复使用上面命令启动httpd以及tomcat，直到`/usr/local/apache/htdocs`下生成了`Demo`目录，并且在该目录下有一个`ROOT`的目录，然后就可以从外部机器访问该站点发布的项目了。

至此，一个Apache整合Tomcat的实战就完成了。

## 安装ActiveMQ

因为项目用到了邮件发送，依赖于ActiveMQ的支持。不过这个安装也很简单:
``` bash
$ tar zxvf /home/trans/recv/apache-activemq-5.7.0-bin.tar.gz -C /opt/
$ cd /opt/apache-activemq-5.7.0/
$ bin/activemq start
```
然后基本上就不用管了。

其他还有一些祖传的脚本，用于项目发布，以及上年经过灾害后我写来用于备份数据的脚本，可惜这一次毁得太彻底了，我都懒得再写了，就让这个项目静静地走向生命周期的结束吧。

## 总结

1. 在一个真正的部署环境下，之能够查看工具提供的日志了，看日志很重要，而不是像平时的在代码写一句输出到控制台那么简单地捕捉到异常，学会看日志很重要，不然就会重现上一年我死活部署了三天，就是因为没看到那么一小条日志。
1. 备份机制，你要假定灾害随时发生，并且都是最坏情况，幸好上一年有了经历灾害的经历，自己写了一份备份脚本，不然今年的国庆节我也不可能玩得那么放心了。