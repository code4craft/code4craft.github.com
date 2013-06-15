---
layout: post
title: "BlackHole开发日志--防止DNS污染"
date: 2013-02-25 21:13
comments: true
categories: BlackHole gfw
---
### 

-------

### DNS污染原理

DNS污染是比DNS劫持更加难以防御的一种攻击，受攻击者访问网站时可被导向其他域名，例如某“不存在的网站”被导向了一个“不存在的IP地址”。

DNS污染的原理如下：

DNS查询也是一个经典的请求-回答模式。首先，客户端发起DNS查询，这是一个UDP包。路由器在转发IP包时，对其内容做解析，若发现其是使用53端口的UDP包，并且其内容符合某些特征(普通的DNS请求都是明文)，则从旁路直接返回一个伪造的应答，将其应答指向某个特定IP。因为这个返回速度非常快，所以先于正常请求到达客户端。而客户端收到一个返回包，就认为得到了答案，不再继续接收，而正确的请求结果就被忽略了！

![image](/images/posts/dns-poison.png)

<!-- more -->

一般防止DNS污染有几种方法：

* 修改系统hosts文件

	系统的hosts文件可以配置某个域名对于的IP地址，并且会优先于DNS服务器的响应，所以此方法稳定高效，缺点是host地址需要不断更新。

* 改用TCP协议而不是DNS协议发送DNS请求

	DNS也支持TCP协议传输，而TCP协议没有收到污染，所以可以改用TCP作为DNS下层协议。缺点是TCP速度慢，并且DNS服务器支持度有限。代表工具：Tcp-DNS-proxy [https://github.com/henices/Tcp-DNS-proxy](https://github.com/henices/Tcp-DNS-proxy)

* 使用IPv6地址发送DNS请求

	某些站点可以通过IPv6地址访问。代表工具：dnsproxycn [http://code.google.com/p/dnsproxycn/](http://code.google.com/p/dnsproxycn/)

* 根据污染特征过滤伪造DNS答案

	先截获DNS污染结果，然后分析其特征，然后将判断为污染的DNS包过滤掉。代表工具：AntiDnsPollution [http://www.williamlong.info/archives/2184.html](http://www.williamlong.info/archives/2184.html)

### BlackHole解决方案

BlackHole解决方案是第4种：向一个不存在的地址发送DNS查询，正常情况下不会有应答；若触发DNS污染，则只会有伪造包返回。记录这个伪造包的特征(一般是A记录的IP地址)，加入黑名单，下次如果收到这些包，则直接过滤。

其实开发BlackHole时不知道有AntiDnsPollution这款工具，完成了才知道，采取的方法不谋而合。只不过BlackHole做的更复杂一点，增加了一些功能：

* DNS污染黑名单持久化

	所有污染IP都会存入"安装路径/blacklist"文件，每次重启可继续读入，也支持手工修改。
	
* 可用IP导出host

	BlackHole会对IP地址做可达性判断(根据ICMP协议请求)，存在DNS污染的域名，若正确DNS地址中，同时有可达IP，则会产生一条"IP domain"的DNS记录到"安装路径/safebox"文件中，与hosts文件格式一致，可以粘贴进去，从而无须再启动BlackHole。
	
* 支持多DNS服务器请求

	BlackHole可以同时向多个DNS服务器请求，并采用最先返回的正确结果作为答案返回；同时后台会继续接收响应，并根据最终答案中IP地址的可用性进行判断，去掉不可用的IP。你可以将BlackHole的转发DNS配置为：一个ISP提供的服务器，速度较快；另一个权威的DNS服务器，结果较可信。对于大多数请求，ISP提供的DNS可以满足需要，从而降低查找时间。同时如果在公司内网中使用，还可以将公司内部DNS服务器地址配置进去，这样可以保证内部配置的某些DNS的有效性。
	
* 支持缓存

	BlackHole使用ehcache作为缓存，并且可以持久化，下次启动时可直接载入上次缓存结果。
	
* 可配置

	BlackHole还是修改hosts文件的替代方案。通过修改"config/zones"可以自定义DNS拦截规则，支持通配符"*"。

BlackHole的源码地址：[https://github.com/code4craft/blackhole](https://github.com/code4craft/blackhole) 




