---
title: Spark配置笔记
date: 2018-10-12 21:26:21
tags:
    - RHEL 7.5
    - Spark
    - Hadoop
    - Java
    - Scala
categories: 工具使用
---

## 环境支持
1. RHEL 7.x
2. JDK 1.8或以上

## JDK的安装

这一步相信经常使用Java的人都有一定的配置经验，无非在RHEL下安装：

``` bash
yum install ${jdk发行版本}.rpm
```

有一些rpm包不支持在安装时自动配置`JAVA_HOME`等环境变量，可以使用：

``` bash
rpm -qa | grep -i java
```

找出安装包的完整名称，随后：

``` bash
rpm -ql ${上一句命令输出的jdk完整名称}
```

找到`JDK`的安装路径，随后在`/etc/profile`或其他脚本下加入：

``` bash
JAVA_HOME=${找到的JDK安装路径}
CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export CLASSPATH
... # 省略中间的其他命令
PATH=.:$PATH:$JAVA_HOME/bin
```

然后就是经典的测试是否成功配置：

``` bash
java -version
```

## Hadoop安装配置

由于笔者仅关注成功配置Spark后运行一些程序，暂时不涉及存储问题，这一步日后进行到了再记录。

## Spark安装配置

先到[官网](http://spark.apache.org/downloads.html)上下载对应的Spark版本，注意要和Hadoop的版本匹配，例如笔者下载[协同Hadoop2.7以上的最新2.3.2版](http://mirror.bit.edu.cn/apache/spark/spark-2.3.2/spark-2.3.2-bin-hadoop2.7.tgz).

成功下载后，使用`tar`命令（笔者习惯安装到`/opt`目录下，当然可以安装到`/usr/local`等目录）：

``` bash
tar zxf spark-2.3.2-bin-hadoop2.7.tgz -C /opt # 笔者在root权限下执行
cd /opt/spark-2.3.2-bin-hadoop2.7/bin
```

直接执行：

``` bash
./spark-shell
```

进入交互模式，证明安装成功。