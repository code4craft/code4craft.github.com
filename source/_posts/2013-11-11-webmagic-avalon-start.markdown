---
layout: post
title: "WebMagic-Avalon计划启动"
date: 2013-11-11 07:50
comments: true
categories: 
---
一直以来都有个想法，想要将爬虫做到可配置化，同时可复用这些配置。然后将这些配置做成可复用，可分享，可搜索的，这样就不会经常有一堆人重复劳动了！

双11的时候启动这个项目，希望解放苦逼的程序员，以后多点时间去谈恋爱，陪家人！

<!--more-->

### 1. 将爬虫脚本化

用尽量自然，简单的语言达到爬虫配置的目的，初定使用JRuby，雏形大概是这样子：

```java
    title = css "div.BlogTitle h1"
    content = css "div.BlogContent"
    urls "http://my\\.oschina\\.net/flashsword/blog/\\d+"
```

见博客：[在webmagic中加入了自定义语言](http://my.oschina.net/flashsword/blog/175349)

### 2. 完整的管理后台

完整的管理后台，包括：

* 规则的选择

* 爬虫的管理

* 爬虫的监控

### 3. 脚本分享

一个脚本要做到可分享，可能包括几个描述性内容：

* 适配的URL

* 简单描述

* 抽取结果

最后做到一个平台，可以自由发布和搜索脚本。