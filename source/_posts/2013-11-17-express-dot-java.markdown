---
layout: post
title: "小轮子一枚-高仿express的Java服务器"
date: 2013-11-17 18:36
comments: true
categories: 
---
之前做了个Java项目[MockSocks](https://github.com/code4craft/mocksocks)，要做UI，用Swing写实在是又low又费劲，跟前端同事聊起node-webkit，觉得很不错。但是我大部分业务都在Java上，于是就涉及到Java与js通信问题。

当然最常用的解决方案就是用Java写一个Web后端。但是这样解决太重，大部分时间都要花费在web的配置上，最终还要使用一个容器去启动它，程序流程也无法由我来控制了。

其实挺喜欢JMX的控制方式，只是用其他语言连接它成本有点高。于是就想仿照JMX的方式写一个Web Server，同时可嵌入到应用中。直接使用Jetty又太原生态了，URL路由/参数映射和转换总是要做的，于是参考了express的语法，就有了一个非常小的Web框架[express.java](https://github.com/code4craft/express.java)。

<!--more-->

本来开始雄心勃勃的要用netty自己写一个，但是后来遇到尴尬的地方：自己写一套HttpMessage类，设计API其实挺麻烦的，设计得好更是需要时间。如果要重用HttpServletRequest/Response呢，实现起来又太费劲。于是后来还是直接用Jetty写了，就不重复造轮子了。

Web框架已经到了汗牛充栋的地步，所以也没想跟谁谁比，完成的是自己的需求就够了。这东西不支持任何servlet规范(HttpServletRequest/Response两个对象基于servlet 3.0)，要的就是简单。

这个WebServer可以在程序内启动，由`UrlRouter`来完成路由，并路由到对应的`Controller`上。比较大的特色就是支持动态增加Controller和映射，这样对于新增是非常方便的。例如我有个service里有个状态`count`，那么我可以这么写：

{% codeblock lang:java ServiceMonitor %}
public class ServiceMonitor {

	private int count;

	private WebServer webServer;

	public ServiceMonitor(WebServer webServer) {
		this.webServer = webServer;
        monitor();
	}

	private void monitor() {
		webServer.get("/service/count", new AjaxController() {
			@Override
			public Object ajax(ParamMap params) {
				return ResultMap.create().put("count", count);
			}
		});
	}

	public static void main(String[] args) throws Exception {
		WebServer server = WebServer.jettyServer().port(8080);
		ServiceMonitor serviceMonitor = new ServiceMonitor(server);
		server.start();
		for (int i = 0; i < 1000; i++) {
            serviceMonitor.count = i;
            Thread.sleep(1000);
		}
	}
}
{% endcodeblock %}

当然这样分散式分配其实会带来一些url的管理问题，不过小项目呢，应该是更方便了。没有想过用这个写web应用，所以目前的定位就是这样子了。

顺便玩了玩angular.js以及less、boostrap、node-webkit等东东。

![mocksocks][1]

  [1]: http://static.oschina.net/uploads/space/2013/1117/212244_eFUQ_190591.png

这只猴子是一个博客中找到的，出自一个自由设计师之手，貌似已经卖给某个客户了[http://blog.coghillcartooning.com/2436/monkey-cartoon-character-sketch/](http://blog.coghillcartooning.com/2436/monkey-cartoon-character-sketch/)。估计真正开源的时候，会把它换掉吧，目前仅仅自己用用，也就无所谓了。

