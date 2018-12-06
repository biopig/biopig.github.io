---
layout: post
title: 如何建立Github Pages自己的博客
categories: GitHub
description: 如何傻瓜式建立自己的github pages博客
keywords: GitHub Pages,GitHub
---
如何傻瓜式建立自己的github pages博客

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
