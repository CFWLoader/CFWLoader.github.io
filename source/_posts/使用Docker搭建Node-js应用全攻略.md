---
title: 使用Docker搭建Node.js应用全攻略
date: 2020-08-02 17:55:33
tags:
    - Docker
    - Node.js
categories: 工具使用
---

最近在了解Docker，学习Docker的[快速教程](https://docs.docker.com/get-started/)后，也仅仅只是学会一些简单的命令证明自己成功安装并且能够跑起Demo，官方也有简单的[DockerFile](https://docs.docker.com/get-started/part2/#sample-dockerfile)介绍。

在上面的简单准备之后，就需要开始实战开发自己的应用镜像。

准备好docker环境之后，第一个要做的事情就是下载node的docker镜像：

``` bash
# 拉取最新版node镜像
$ docker pull node
# 拉取指定版本node镜像
$ docker pull node:${version}
```
