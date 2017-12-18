---
title: Fedora 25配置资料整合
date: 2017-11-29 08:53:10
tags:
	- Linux
	- Fedora 25
	- 工具配置
categories: Linux
---

## 网络配置

### WiFi热点

Fedora 25直接用`gnome`的网络界面管理里面可以将无线网卡设置为热点，但是会没有响应，[解决办法](http://bytefreaks.net/gnulinux/fedora-25-with-gnome-3-making-a-wi-fi-hotspot)在此。

用`nm-connection-editor`命令打开界面，填好`SSID`，以及设置`Security`中的WiFi密码，然后回到`gnome`中的网络管理界面开启热点即可。

### Chrome中导入XX-Net的http证书无效问题

Chrome在设置中导入了其`CA.crt`文件之后，使用http代理还是会报站点不信任问题，根据`Linux`证书管理的教程:
``` bash
$ sudo dnf install nss-tools		# 安装证书导入工具
```
然后导入`GAE`的证书:
``` bash
$ certutil -d sql:$HOME/.pki/nssdb -A -t TC -n "GoAgent XX-Net" -i ${XX-Net的安装目录}/data/gae_proxy/CA.crt
```
随后无论是重启系统，还是重复导入多次还是无效。

有时候想查看导入的证书的信息:
``` bash
$ certutil -d sql:$HOME/.pki/nssdb -L
```
会报:
``` text
certutil: function failed: SEC_ERROR_BAD_DATABASE: security library: bad database.
```
一般可能是用`~`来表示当前的用户的根目录，貌似这个`sql:`后面跟`~`会出错，用`$HOME`来代替一般可以，实在不行就只能重新建立一个库再导入了:
``` bash
$ mv ~/.pki/nssdb ~/.pki/nssdb.corrupted
$ mkdir ~/.pki/nssdb
$ chmod 700 ~/.pki/nssdb
$ certutil -d sql:$HOME/.pki/nssdb -N
```
笔者之前搜索了好长一段时间，最后找到一个原因说是`Fedora 25`不支持`SSL`中的某个算法(具体是什么因为有点久远找不到了)，所以就算导入了证书之后也没用，以下有两个代替方案:
1. 用`FireFox`，加载插件`AutoProxy`，随后导入`GAE`证书，就可以使用`HTTP`翻墙了。
1. 使用`XX-Net`中的`X-Tunnel`功能，这个协议比`HTTP`方便多了，不用导入直接使用。缺点是流量不是免费的，要么买，要么就[贡献自己的GAE](https://github.com/XX-net/XX-Net/wiki/DonateAppid)获得流量。

之前`GAE`没墙那么紧的时候试过，还是报不信任的错误，然而笔者今天整理资料做尝试的时候，因为`GAE`的`IP`已经基本给封得扫不出来了。所以笔者再重现这个问题的时候因为`GAE`没法用而不能得知有效性了。

因此，使用`XX-Net`笔者推荐使用其`X-Tunnel`功能。

## 安装TIM

万恶的企鹅一直不出Linux版的通信软件，而在日常中又经常使用，但是`WebQQ`又太烂了，不好用，一般的`Wine QQ`用的是`2013国际版`，轻量版的`TIM`去除了很多不必要的功能，成为了笔者常用的通信软件，于是动了在`Fedora`下安装一个的念头。

笔者在多番寻找之后终于找到一个可用的`Fedora 25`[TIM教程](https://gao4.pw/%E5%9C%A8fedora25%E4%B8%AD%E5%AE%8C%E7%BE%8Ewineqq)。

最后笔者在安装`Wine`重启系统之后无法进入系统，最后卸载了`Wine`才恢复了正常，所以说实在要用鹅厂软件的话，还是老老实实用虚拟机比较好。