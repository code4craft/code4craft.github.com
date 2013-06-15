---
layout: post
title: "Mac使用ssh socks代理翻墙"
date: 2013-01-05 21:46
comments: true
categories: gfw
---
听说最近VPN封的厉害，都改用ssh翻墙了。在Mac下实验了一把，总算鼓捣成功。总结如下：

<!-- more -->

使用ssh作为socks proxy：

有很多软件都可以实现这个功能，例如Secret Socks，SSH Tunnerl Manager。Secret Socks不支持自动连接，同时每次都要输入密码，太麻烦了。SSH Tunnerl Manager也是要输入密码。不过SSH Tunnerl Manager有个好处，更改配置后它会把SSH命令都显示出来。原来Mac下SSH直接就可以支持socks代理，大概这样配置：

	ssh -N -p 22 -C -c -D 7070 username@hostname
	
但是这样依然无法保存密码，后来搜索到一个软件sshpass，下载编译之，运行却提示找不到/usr/libexec/ssh-askpass，后来从网上找到一个人的解决方案，发现其实这就是提供密码的脚本，于是直接在里面敲上：

	#! /bin/sh 

	echo yourpassword
	
然后使用

	sshpass -pyourpassword ssh -N -p 22 -C -c -D 7070 username@hostname
	
就可以了！sshpass貌似自带断线重连功能，相当给力！

还有就是autoproxy的配置，在Chrome下使用SwitchySharp，在规则列表里加上[https://autoproxy-gfwlist.googlecode.com/svn/trunk/gfwlist.txt](https://autoproxy-gfwlist.googlecode.com/svn/trunk/gfwlist.txt)，选择AutoProxy兼容列表即可，当然对应情景模式别忘了选成你配好的本地代理！

最后将启动脚本ln -s到/Library/StartupItems/，至此大功告成。