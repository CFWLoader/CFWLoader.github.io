---
title: hexo+github配置
date: 2017-10-31 10:47:44
tags: 
	- github
	- hexo
	- 工具配置
	- Fedora 25
categories: Linux
---
## 需要的环境

1. git
1. npm
1. nodejs

笔者使用的是Fedora 25，所以使用dnf命令就可以把这个两个工具快速安装了，nodejs也可以通过dnf快速安装到系统中。

## Github上的准备

在github上建立一个与自己用户名对应的仓库，仓库名为"${用户名}.github.io"。
在本地上新建一个目录，用于存储hexo博客的工程内容，我就建立了一个"blog"作为hexo的根目录，只需要记得根目录路径即可，很多操作直接在根目录上使用hexo命令。

## 安装Hexo

在前面提及的工具已经安装的情况下，使用：
``` bash
$ sudo npm install -g hexo
```
安装hexo。

## 初始化以及配置Hexo工程

进入blog目录，执行
``` bash
$ hexo init
```
初始化目录。

生成静态页面:
``` bash
$ hexo generate
```

或者简化的命令参数:
``` bash
$ hexo g
```

然后可以使用:
``` bash
$ hexo server
```

使用浏览器访问[http://localhost:4000](http://localhost:4000)即可看到搭建起来的本地博客。

### 将Hexo本地工程部署到远程站点上

在blog本地根目录下有一个名为`_config.yml`的配置文件，用于配置整个hexo工程。

使用:
``` bash
$ vim _config.yml
```
打开， 其他平台或者编辑器的使用者请自行使用熟悉的编辑器。

到了文件末尾会有一段`deploy`开头的配置。
修改成如下:
``` code
deploy:
	type: git
	repo: ${之前在github仓库上创建的仓库的git或者http链接}
	branch: master
```

例如我的配置:
``` code
deploy:
	type: git
	repo: git@github.com:CFWLoader/CFWLoader.github.io.git
	branch: master
```

然后执行命令:
``` bash
$ npm install hexo-deployer-git --save
```
保存配置。

最后，执行:
``` bash
$ hexo deploy
```
将本地的修改提交到github仓库上，并会部署到站点上运行应用。

使用浏览器访问建立的仓库名则可以看到部署的博客站点，例如我的站点是[http://cfwloader.github.io](http://cfwloader.github.io)

至此github + hexo的基本配置结束。

## 部署事项

每一次编辑并发布，请按照下列命令执行:
``` bash
$ hexo clean
$ hexo generate
$ hexo deploy
```

## 开启Hexo文章分类功能

在项目目录下，编辑`scaffolds/post.md`，修改内容为:
``` code
---
title: {{ title }}
date: {{ date }}
tags:
categories:
---
```

之后使用`hexo new`产生新文章时，在categories中填入文章的分类，hexo会自动产生分类栏。

因为一般我们采用的是中文，所以产生出来的页面的路径也包含中文，如果需要将分类与路径分离，可以修改`_config.yml`中`category_map`，有兴趣的可以自己去探索探索，这里不展开介绍了。

## Hexo标签功能

Hexo自带的模板的文章头就有`tags`选项，之前学习搭建hexo的时候别的文章提到有多个标签的时候有两种填法。
一种是:
``` code
tags: [tag1, tag2, ..., tag`n`]
```

另一种是:
``` code
tags:
	-tag1
	-tag2
	...
	-tag`n`
```

笔者使用的`hexo-cli`版本是`1.0.4`，只有第二种方法才能正确产生期望的标签样式。故采用第二种方法。

## 常用命令

新建文章:
``` bash
$ hexo new "${文章名}"
```

新建页面:
``` bash
$ hexo new page "${页面名}"
```

生成页面:
``` bash
$ hexo generate
```
或者
``` bash
$ hexo g
```

部署应用到站点上:
``` bash
$ hexo deploy
```

更多:
``` bash
$ hexo help
```