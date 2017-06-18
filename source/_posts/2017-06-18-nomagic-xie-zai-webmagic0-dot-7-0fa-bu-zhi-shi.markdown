---
layout: post
title: "No Magic - 写在WebMagic0.7.0发布之时"
date: 2017-06-03 20:13
comments: true
categories: 
---

![logo](http://webmagic.io/images/logo.jpeg)


过节三天，没有安排长途旅游，除了带女儿出去活动一下，终于有点时间写点业余代码了。

WebMagic这次终于有比较大的重构，其实要感谢英语培训班，因为大部分代码都是女儿上课的时候完成的。她上课两个小时，我就在旁边的咖啡店等两小时，可以没什么干扰的写两小时代码。这个版本前前后后也拖了2个月了，终于在前两天测试完成，主要部分没什么大问题了，也可以发一个版本。

这次发布主要针对两个诟病比较久的问题，一个是POST请求的处理，一个是代理的切换。老实说我自己遇到这两种场景都不多，所以之前一直用的是很简单的做法完成功能。

这次想重写，最难的部分就是如何定API。前前后后包括网友提交的，也有5、6个版本了。最终选择的API我还是比较满意的，好处就是它看起来很简单。好的API定义，应该给人的感觉是“啊，就应该是这样子的”。比如POST请求，有一个核心就是设置RequestBody。之前写了一版是根据参数和Content-Type去动态的做转换，例如Request.setParams(Object)，然后Request.setContentType("JSON")，就可以提交JSON。这个在Spring MVC等框架中也用的不少。后来改成了最基本的、最简单的方式：自己设置RequestBody，再将RequestBody的构造做封装，最终写起来是这样的：

```java
Request request = new Request("http://xxx/path");
request.setMethod(HttpConstant.Method.POST);
request.setRequestBody(HttpRequestBody.json("{'id':1}","utf-8"));
```

这个接口看起来一点也不高级，但是挺容易理解。这里JSON是序列化好的结果，其实HttpRequestBody.json()也是多余，直接将Content-Type传过来，连.json()也省了。只不过考虑到对功能有一些“友好性”的引导，毕竟手写Content-Type容易出错，所以加上了。

之前看《Unix编程艺术》，觉得Keep It Simple, Stupid挺好。现在写的代码、读的代码越来越越多，也越来越发现简单的好处。

现在回过头来看WebMagic有挺多设计不合理的地方(幸好能发现不然这几年白混了)。

刚开始写的时候是4年前，那会取名是受“ImageMagick”影响，“Magic”是希望它可以让Web数据的获取变得很方便，跟魔法一样。那会对模块化已经比较有心得，所以宏观上没什么大问题，但是局部的实现和API上，有很多“试图多做事”的做法。例如html.regex()可以不写捕获组序号，默认取第一个捕获组（如果有的话），@Target()的正则匹配也是自己定制过的，将URL中最常用的"."直接替换成了"\\."。在使用中，相信也给不少人带来了困扰，从issue中就看得出来。

当然WebMagic本身这么多人喜欢，也是我没料到的，看各种评价，主要还是文档写的比较全面详细，另外API比较简单。最近有次分享，一个朋友找到我跟我合影，说是用到框架收益良多，也是让我很惊喜。

这次0.7.0之后，我会陆续重构一些组件，让它们变得更加简单。

No Magic, just K.I.S.S.
 