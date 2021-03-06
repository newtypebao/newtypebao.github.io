---
layout: post
title: 关于nginx中 warn conflicting server name localhost on 0.0.0.080, ignored异常的解决方案 
date: 2018-05-09
tag: nginx
---

最近项目中配置nginx时遇到一个很奇怪的问题：

首先，先说一下背景，目前的需求是需要把一个应用同时发布在80和443端口，nginx是通过yum方式默认安装的。<br />
在配置完2个server之后（80、443）,启动时候报如下这个警告： <br />
>nginx [warn] conflicting server name localhost on 0.0.0.080, ignored

这个似乎是有端口方面的冲突，用这个错误信息直接百度了一下,得到的信息：
>意思是重复绑定了server name,但这个警告不会影响到服务器运行。而且，这个重复绑定的意思是现在运行的nginx服务和将要加载的新配置中的重复，所以，这个警告其实是不必的。

按照baidu的指引，忽略这个警告，尝试访问应用，但是应用始终是报404的错误提示。

经过排查发现，每次启动nginx的时候（当时我在配置文件中搜索“80”关键字，并没有找到匹配项，我甚至怀疑见鬼了。。即使我在配置中没有配置80端口，80端口也是监听着的，这点非常奇怪。）

通过服务器上网络端口监控发现80端口确实是被nginx master进程所占用，终于顺藤摸瓜发现，原来默认安装的nginx.conf中有那么一段配置：
>include /etc/nginx/conf.d/*.conf;

而在这个目录中有一个名为default.conf的文件，其中配置了server并且listen 80。。

终于找到了罪魁祸首，那么知道原因就好办了，注释掉这段或者修改下default,nginx便可以正常工作了。

之前baidu上搜索错误关键字到靠前的信息都不合适这个场景。。特此总结，以供有相似经历的人分享。
