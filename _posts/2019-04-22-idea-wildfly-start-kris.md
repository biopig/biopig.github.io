---
layout: post
title: IDEA和Wildfly配置及项目部署
categories: Tool
description: IDEA和Wildfly配置及项目部署
keywords: IDEA,Wildfly
---
IntelliJ IDEA和Wildfly配置及项目部署

**1. eclipse项目导入IDEA**

因为KRIS项目是由KRISADMIN和KRM两个项目组成，所以先需要新建一个空的Project，然后将KRISADMIN和KRM作为Module导入

- (1)新建空的Project

File - New - Project - Empty Project - Next

![](/images/posts/tools/idea_new_project.jpg)

之后输入project name 以及选择路径
	
![](/images/posts/tools/idea_new_project_finish.png)

完成之后关闭弹出界面

- (2)把项目作为Module导入Project

![](/images/posts/tools/idea_import_module.png)

之后找到项目选择点击OK

![](/images/posts/tools/idea_module_next.png)

Next
	
![](/images/posts/tools/idea_module_next1.png)

Next

![](/images/posts/tools/idea_module_next2.png)

Next

![](/images/posts/tools/idea_module_next3.png)

选择JDK1.7，你电脑中如果已经安装JDK1.7，会自动识别到，Next - Finish

![](/images/posts/tools/idea_module_next4.png)

执行上面相同步骤，导入另外一个项目。在导入KRM的时候，这个可以不勾

![](/images/posts/tools/idea_module_next5.png)


- (3)项目配置

使用快捷键 Ctrl+Alt+Shift+S 进去配置页面

①设置Project SDK版本(1.7)

![](/images/posts/tools/idea_setting_next.png)

②Modules

(1) 根据项目修改Name，选择Test out path（都在classes目录下）

![](/images/posts/tools/idea_setting_next1.png)
	
(2) 选择报红error的移除

![](/images/posts/tools/idea_setting_next2.png)

(3)导入jar包	
	
![](/images/posts/tools/idea_setting_next3.png)

选择lib

![](/images/posts/tools/idea_setting_next4.png)

同样步骤完成另外一项目的Modules配置

③Facets

![](/images/posts/tools/idea_setting_next5.png)

选择一个项目，之后

![](/images/posts/tools/idea_setting_next6.png)

另外一个项目的配置一样，完成之后可以在Facets Web下看到两个配置

![](/images/posts/tools/idea_setting_next7.png)

③Artifacts

![](/images/posts/tools/idea_setting_next8.png)

选择一个我们导入module，之后

![](/images/posts/tools/idea_setting_next9.png)

Apply完成(这步感觉是war包生成配置)，另外一个项目也是同样的操作(注意war包路径)

![](/images/posts/tools/idea_setting_next10.png)



**2. 服务器**

(1)run - Edit...

![](/images/posts/tools/idea_server_next.png)

(2)

![](/images/posts/tools/idea_server_next1.png)

(3)选择wildfly的地址

![](/images/posts/tools/idea_server_next2.png)

(4)添加 Deployment(war包)

![](/images/posts/tools/idea_server_next3.png)

最后Ok，就可以启动了 

![](/images/posts/tools/idea_server_next3.png)

**3. 常用快捷键**

`Ctrl+N `：搜索java文件

`Ctrl+Shift+N` ：搜索所有文件

`Ctrl+Shift+R`：全局搜索

2019/4/22 13:19:19 


