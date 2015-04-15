---
layout:post
title:jenkins配置与使用总结
categories:[java]
tags:[java,安装]
---
    jenkins可以快速从源码进行项目构建，并帮助你将构建出来的程序部署到需要的位置，节约了很多人力操作……公司号称要后台全用java，却连这种基本的持续构成工具都没有推广使用，全部都由每个项目组自行摸索使用，恩。截止目前位置我只用其构建过tomcat的web项目以及普通java命令行项目。
    
    安装步骤如下：
    首先去官网下一个下来……我自己用的下载地址是http://ftp-chi.osuosl.org/pub/jenkins/war/1.595/jenkins.war。
    下完以后随便将这个玩意丢哪个目录去。运行方式有两种。一种是直接在命令行执行java -jar jenkins.war，还有一种是将其丢进tomcat中执行。我偷懒用得第一种方法。执行完以后访问127.0.0.1:8080页面，ok没问题。
    随后配置jenkins的用户登录和安全。总不能丢公网上谁都能访问是吧……虽然这东西不应该能在公网访问到……选择页面左侧菜单的 Manage Jenkins，进入管理页面。选择Configure Global Security。勾上Enable security，选上Jenkins’ own user database 和 Allow users to sign up两项。继续勾上Matrix-based security。新建一个用户。然后在该页面右上角有login in链接，点开这个页面，选择创建一个账户，账户名字是刚才新建用户的名字，然后填写密码，以后就可以用这个用户登录……
    随后还需要在配置页面配置git，svn，maven插件和安装地址等。

    正常的使用不再赘述，比如项目编译完后要部署并执行，普通的jar包需要自行编写shell脚本等方式处理，部署tomcat的话，jenkins有提供一个比较好用的插件，具体安装和使用步骤我参考了http://www.ylzx8.cn/qiyeruanjian/industry/1466767.html。

    下面是个人使用时做得配置：
    一般来说我再jenkins构建完代码后，我需要把程序丢到某个目录然后重启这个程序。但是在jenkins中，它默认会杀掉一切用户开启的进程，这和我的目的不符。我在stackoverflow上找到两种解决办法。一种是在jenkins的config页面，在Global properties选项增加设置name：BUILD_ID value：dontKillMe /usr/local/tomcat/bin/catalina.sh start。添加这行可以让“/usr/local/tomcat/bin/catalina.sh start”这个命令启动的进程不被jenkins杀掉。还有一种是我索性现在用的，在用命令行启动jenkins的时候我会加上参数-Dhudson.util.ProcessTree.disable=true。这个参数可以关掉jenkins杀进程的功能。
    jenkins启动后，数据一般会保存在用户home目录的.jenkins目录下，公司服务器这个目录磁盘很小，我需要把它挪到别的地方，可以在/etc/profile 增加一行配置export JENKINS_HOME=xxxx；保存，退出后执行：source  /etc/profile，然后重启jenkins，这样就可以让jenkins把数据保存在这个目录中。我的一个web项目构建了300次居然在jenkins的目录下耗费了13g得磁盘空间……