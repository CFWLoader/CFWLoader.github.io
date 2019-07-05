---
title: Git问题汇总
date: 2019-07-05 11:36:54
tags:
    - Git
    - Github
    - Windows
categories: 工具使用
---

因为有多次重装系统后配置系统环境的经历，因此笔者认为有必要总结其中的经验，节省下一次配置时使用搜索引擎的时间。

## 安装以及基本配置

[Windows系统的Git下载地址](https://git-scm.com/download/win)，依据默认配置一路下一步即可，进阶用户当然可以定制其中的配置，例如新增以VS Code为默认编辑器的选项。

进入git提供的命令行界面，建议配置基本信息：

``` bash
$ git config --global user.name "${你的用户名}"
$ git config --global user.email ${GitHub用户邮箱}
```

常用GitHub托管代码时，笔者倾向于使用SSH免密登录，因此使用：

``` bash
$ ssh-keygen -t rsa -C "${邮箱地址}"
```

然后一路回车就行了，生成的公私钥文件位于`~/.ssh/`（Windows 10为`C:\Users\${用户名}\.ssh`）文件夹内，打开其中`id_rsa.pub`文件，复制其中文本到[GitHub SSH配置](https://github.com/settings/keys)中，完成该步骤。

测试配置结果可以使用：

``` bash
$ ssh -T git@github.com
```

第一次测试成功会发现类似以下的信息：

``` text
The authenticity of host 'github.com (207.97.227.239)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)?
```

回车即可，若并未成功并且出现超时结果，可以参考本文的下一节。

## 使用SSH拉取远程仓库超时问题

笔者在最近一次（2019-07-05）配置git中发现，按上一节配置后并未成功拉取仓库代码，出现了超时：

``` text
ssh: connect to host github.com port 22: Connection timed out
```

结合个人使用经验以及谷歌搜索结果，使用以下命令测试：

``` bash
$ ssh -T -p 443 git@github.com
```

发现可以成功连接GitHub，因此判定为Git客户端的配置与GitHub服务器开放的端口不符。在此前安装Git时候并未出现过该问题，不过那时GitHub还没有被微软收购，其中原因暂不深究，解决问题为先。虽然有将仓库链接改为`https://`开头的解决方案，但是笔者不倾向于该方案，因此选择了以下方案：

在`~/.ssh/`（Windows 10为`C:\Users\${用户名}\.ssh`）文件夹内新建一个`config`文件（注意不要有后缀，仅一英文单词），向其中写入：

``` text
Host github.com
Hostname ssh.github.com
Port 443
```

即改变SSH的默认连接，再次运行：

``` bash
$ ssh -T git@github.com
```

测试通过，问题解决，让新配置的Git也可以通过SSH方式拉取仓库代码。