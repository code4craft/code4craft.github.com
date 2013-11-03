---
layout: post
title: "MockSocks开发日志之三-为NIO设置Socks代理"
date: 2013-10-20 16:28
comments: true
categories: nio, bytecode, java
---
## 回顾

时隔3个月，MockSocks终于又能继续开发了。这个项目是目前为止做过的最有技术挑战的一个，目标是做成一个后端应用的fiddler，可以监控应用对外的网络流量、分析协议、重定向、并针对每个协议进行修改，同时可以录制和回放。项目也得到了部门总监和其他leader的肯定，可以多花心思弄弄好。

因为项目的核心是一个Socks代理，通过这个代理捕获双方的流量，并进行后续的操作。

<!--- more --->

## Server

Socks Server方面使用netty构建了一个，原本觉得无比复杂的东西，借助netty倒是变得很简单了，具体实现方式在这篇博客：[http://my.oschina.net/flashsword/blog/169361](http://my.oschina.net/flashsword/blog/169361)。学习netty期间用心写了几篇文章，也结交了一些朋友，倒是挺开心的。可惜netty系列文章没有完成，估计要等我MockSocks开发完才能继续了。

## 配置Client

server端开发完后，就轮到client端了。JDK的OIO是支持全局代理的，只需在JVM参数中配置`-DsocksProxyHost=xxx -DsocksProxyPort=xxx`即可。遗憾的是，这个配置对NIO是不起作用的。

后来考虑过几种办法：

1. 因为公司项目用到NIO的部分，主要也是通过netty做的。那么改netty的API，使其支持代理，是最简单的做法。使用netty构建一个socks client也是得心应手。但是这种做法不够彻底，且不具有通用性。

2. 修改NIO的接口SocketChannel.open()的实现，使其返回一个可以使用代理的SocketChannel。这种方法最彻底，但是涉及到JDK一些底层API，有些还没有暴露出来，实现难度有点大。

后来决定采用方法2，顺便学习一下。

`SelectorProviderImpl`、`SocketChannelImpl`都是VM的私有API，只有下载JDK源码才能看到。下载openjdk源码后，在`jdk/src/share/classes/`目录可找到。

`SelectorProvider.provider()`是JDK自己的一个扩展点，会根据不同的OS选择不同的SelectorProvider，OSX是KQueue。尝试自己写了一个SocketChannel的子类，做一个全局代理，结果被SelectorImpl摆了一道，里面要求必须实现`sun.nio.ch.SelChImpl`接口，而这个接口是包级可见的。

抱着侥幸心理，尝试将自己的新类写到`sun.nio.ch`包下，结果编译通过，加载提示无法访问其父类接口`sun.nio.ch.SelChImpl`，看来sun对自家的包是做了一些保留的。

看了一遍《深入理解Java虚拟机》关于ClassLoader那章，确定

尝试用agent修改代码。后来沮丧的发现，agent的`ClassFileTransformer`无法获取到系统级别的class，但是看ByteMan介绍，它倒是可以做到，有必要研究一番。
