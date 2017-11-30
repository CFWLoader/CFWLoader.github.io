---
title: Fedora 25配置资料整合
date: 2017-11-29 08:53:10
tags:
	- Linux
	- Fedora 25
categories: 系统运维
---

## 网络配置

### WiFi热点

Fedora 25直接用`gnome`的网络界面管理里面可以将无线网卡设置为热点，但是会没有响应，[解决办法](http://bytefreaks.net/gnulinux/fedora-25-with-gnome-3-making-a-wi-fi-hotspot)在此。

用`nm-connection-editor`命令打开界面，填好`SSID`，以及设置`Security`中的WiFi密码，然后回到`gnome`中的网络管理界面开启热点即可。