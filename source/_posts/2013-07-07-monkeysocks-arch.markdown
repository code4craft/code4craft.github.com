---
layout: post
title: "monkeysocks架构规划"
date: 2013-07-07 21:35
comments: true
categories: 
---
> monkeysocks的目标是为开发以及测试提供一个稳定的环境。它使用socks代理，将录制网络流量并本地保存，并在测试时将其重放。

## jsocks的改造

首先对公司一个项目进行了代理，测试结果：从开始启动到完成，只有4.7M的网络流量，本地空间开销不是问题。

今天把jsocks修改了下，将build工具换成了maven，并独立成了项目[https://github.com/code4craft/jsocks](https://github.com/code4craft/monkeysocks/jsocks)。后来算是把record和replay功能做完了，开始研究各种协议replay的可能性。

<!--more-->

replay时候，如何知道哪个请求对应响应包是个大问题。开始的方式是把request报文的md5作为key，response作为value。

## TCP协议分析

后来使用wiredshark结合程序日志来进行分析。

TCP协议栈大概是这样子：
![image](http://www.skullbox.net/diagrams/tcppacket.gif?dur=673)

下面是wiredshark抓包的截图，从ea开始才是应用层协议的内容。

![image](http://code4craft.github.io/images/posts/tcp-wiredshark.png)

## 应用层协议分析

实现replay后，拿HTTP协议做了测试，自己用程序写了个URLConnection，倒是能够实现replay，但是换到浏览器里就很难了，因为cookie总是会有些不一样(现在基本上所有站点都会写cookie吧)。如果不对应用层协议本身进行分析，那么进行包的伪造就很难了。

https协议对于重放攻击做了处理，每次的请求包都不一样，也无法replay成功，暂时略过。

后来对于测试中得重点协议--mysql的协议，进行了研究。

这是一个有状态的协议，状态转移图如下：

![image](http://dev.mysql.com/doc/internals/en/images/graphviz-db6c3eaf9f35f362259756b257b670e75174c29b.png)

详细介绍[http://dev.mysql.com/doc/internals/en/client-server-protocol.html](http://dev.mysql.com/doc/internals/en/client-server-protocol.html)，有点hold不住的感觉啊！

看了Authentication部分，会由server端发送一个随机数，来避免重放攻击。这个东西启发了我，因为主动权一般都是在server端，而我们要对client进行欺骗，难度就小了很多。

## 架构设计

后来决定把架构解耦了，fake server单独作为一个模块，可以单独启动成TCP server，也可以加入到jsocks里。最后架构是这样子：

![image](http://code4craft.github.io/images/posts/monkeysocks-arch.png)

fake servers的实现必定是个大坑，不过能把常用协议都了解一遍，本身也很有意思不是么？

##  开发计划：

 * 实现fake servers的TCP框架。

 * 研究并实现常用协议的fake server。

 * 确定持久化以及报文对应的策略。