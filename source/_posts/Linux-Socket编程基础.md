---
title: Linux Socket编程基础
date: 2018-02-27 11:27:43
tags:
	- 网络通信
    - TCP
    - Linux
    - C
    - C++
categories: 网络通信
---

之前完成了`Effective C++`系列之后，整个人就懒起来了，加之春节假期，几乎整个2月份就没有整理写博客了。

回到之前，因为`C++后台开发`基本上都要求熟悉`Linux`以及`Socket`，所以学习`TCP`以及`Socket`编程是不能避免的事情。

本篇博客首先介绍基本的接口，随后给出一个简单的单线程，面向单个用户的演示程序。

### 用到的接口
``` c++
// 创建一个Socket并返回对应的文件描述符。
int socket(int domain, int type, int protocol);

// 将指定的sockfd与指定地址addr绑定起来。
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// 使指定sockfd可以被监听，即支持后续的accept()函数。
int listen(int sockfd, int backlog);

// 通过sockfd接受外部的连接，成功接受后，addr会存有外部连接的地址信息。
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

注意`socket`并不一定是`TCP`的，也可以是`unix`等通信协议，无非是`TCP`用得比较多而已。

在编写的过程中，通过`man`得知使用上述函数所需要的头文件，并且其中带有样例，可以参考怎么初始化一些数据结构。