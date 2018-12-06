---
layout: post
title: 如何建立Github Pages自己的博客
categories: GitHub
description: 如何傻瓜式建立自己的github pages博客
keywords: GitHub Pages,GitHub
---
## 前言
写博客的地方很多，比如CSDN、简书等等。但是Github Pages可以定制下自己博客的外观，随心随意，github有很多很多开源优秀项目，可以学习，自己的博客还可以装逼啊。

### 1.在github创建一个仓库
    用户名.github.io
仓库的名字必须是 `github登录name.github.io `

例如(我的github名mojingman)：`mojingman.github.io`

### 2.下载仓库到本地

(1)先下载GitHubDesktopSetup.exe

[GitHubDesktopSetup下载](https://desktop.githubusercontent.com/releases/1.5.0-2f0c701f/GitHubDesktopSetup.exe) 完成之后安装登录自己的github账号

(2)`clone`刚刚创建的仓库到本地

![clone-code](https://i.imgur.com/rnYVfeC.png)


### 3.找自己喜欢的博客模板

 [github page官方建站指导](https://pages.github.com)介绍要让自己的博客变的很漂亮可以使用[Jekyll](https://jekyllrb.com/)，[了解如何设置Jekyll](https://jekyllrb.com/docs/)。

我比较懒，没有去自己从头配置Jekyll，然后直接用的别人配置好的。你可以在github上搜索下jekyll page，有很多很好看的模板，你可以`fork`到自己的github.我用的模板是[码志](https://github.com/mzlogin/mzlogin.github.io)

### 4.替换代码

* `fork`别人的模板仓库到自己github之后下载到本地(过程和第2步一样)。

![](https://i.imgur.com/X5rKMu7.png)

* 把fork模板仓库的所有的文件夹复制到自己创建的仓库

* 通过github客户端提交代码到github

![commit-code](https://i.imgur.com/MCsB9No.png)

* 访问`name.github.io`,例如 [mojingman.github.io](mojingman.github.io)看自己所用模板


### 5.修改信息以及配置
  
上面的步骤只是把别人的模板拿过来自己用，但是里面所有的数据都是别人，要修改成自己的数据或者自己配置，根据你所应用模板的文件夹具体修改，我这里只说明我所有模板的修改。

`_data` 文件夹下三个文件分别对应博客页面`链接、关于`的类容，可以通过TXT等工具打开直接修改成自己的信息，可以增加删除相关数据

`_drafts`文件夹下是博客草稿

`_includes`博客页面头和脚公共的主要布局，html自己有兴趣可以进行修改

`_layouts`对应博客`首页、分类、维基、链接、关于`页面的布局

`_posts`自己博客基本上全在这个文件夹，除了`template.md`文件其他文件都可以删除，因为其他都是别人的博客

`_wiki`维基页面的博客都在此文件夹下

`assets\images`文件下有个二维码可以换成自己的

`images`博客里面的图片以及图片资源都在此文件夹下

`pages`博客页面显示内容在此文件夹下，可以修改成自己的信息

`favicon.ico`网站logo，可以换成自己的

`_config.yml`整个博客的主要配置文件

小结：主要要修改的文件有：

①`_data`下所有文件

②`assets\images`二维码图片

③`_includes/sidebar-qrcode.html`文件第一行代码：`{% if site.url contains 'mojingman.github.io' %}`中的`mojingman.github.io`修改成自己的

④`pages` 页面显示内容可以修改成自己想要显示的文字，主要修改`about.md`文件，其他看自己

⑤`_config.yml`里`Main Configs、Author` 全部修改成自己的信息。

**评论模块：** 目前支持 disqus、gitment 和 gitalk，选用其中一种就可以了，推荐使用 gitalk。我用的是gitalk,Gitalk 需要一个 Github Application，[点击这里申请](https://github.com/settings/applications/new)。
![](https://i.imgur.com/7T0Oqm4.png)
点击创建

获取` Client ID` 和 `Client Secret`
![](https://i.imgur.com/dEmipF5.png)

修改`_config.yml`下 Comments内容

    gitalk:
    owner: mojingman
    repo: mojingman.github.io
    clientID: 刚刚申请的Client ID
    clientSecret: Client Secret
注意：owner是你的name，repo是你博客仓库名，若果你不做评论模块修改，那用的评论就是我的

⑥删除我的文章与图片。

  如下文件夹中除了 template.md 文件外，都可以全部删除，然后添加你自己的内容。

> * _posts 文件夹中是我已发布的博客文章。

> * _drafts 文件夹中是我尚未发布的博客文章。

> * _wiki 文件夹中是我已发布的 wiki 页面。
>
> * images 文件夹中是我的文章和页面里使用的图片。

该删除的删除，该修改的修改全部完成之后，上传所有的代码，如何上传代码上面有讲，至此所有配置完成了。这个个人微博搭建是在别人的模板基础上进行的，没有去看Jekyll的配置。你们有兴趣可以自己学习，傻瓜教程。

**[在线预览 &rarr;](https://mojingman.github.io)**

##怎么写博客

通过Markdown编辑器

网上找了很多编辑器比较好的一款是MarkdownPad，[点击此处下载资源](http://www.markdownpad.com/download.html)

破解：

    Email address :Soar360@live.com

    License key : 
    GBPduHjWfJU1mZqcPM3BikjYKF6xKhlKIys3i1MU2eJHqWGImDHzWdD6xhMNLGVpbP2M5SN6bnxn2kSE8qHqNY5QaaRxmO3YSMHxlv2EYpjdwLcPwfeTG7kUdnhKE0vVy4RidP6Y2wZ0q74f47fzsZo45JE2hfQBFi2O9Jldjp1mW8HUpTtLA2a5/sQytXJUQl/QKO0jUQY4pa5CCx20sV1ClOTZtAGngSOJtIOFXK599sBr5aIEFyH0K7H4BoNMiiDMnxt1rD8Vb/ikJdhGMMQr0R4B+L3nWU97eaVPTRKfWGDE8/eAgKzpGwrQQoDh+nzX1xoVQ8NAuH+s4UcSeQ==
![](https://i.imgur.com/IQkgfdX.png)


可以设置为中文![](https://i.imgur.com/0zZLXVU.png)

在`win10`和`win8`系统中软件会报错，需要下载一个组件`awesomium_v1.6.6_sdk_win`,[https://pan.baidu.com/s/1rIdXInA5iJOjQOE4cDuRaQ](https://pan.baidu.com/s/1rIdXInA5iJOjQOE4cDuRaQ)安装之后重启`MarkdownPad`就可以了。

小结：博客命名是 时间-搜索关键 这样，比如：`时间-github-page.md` 这样你在页面搜索的时候，搜索github和page就能搜索到这篇博客

在博客顶部还需要加入下面字段，回去自动归类和显示标题等：
   ![](https://i.imgur.com/h3Rgf68.png)

2018/12/6 11:43:36