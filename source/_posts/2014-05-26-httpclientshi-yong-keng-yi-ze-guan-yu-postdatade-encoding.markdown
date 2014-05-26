---
layout: post
title: "HttpClient使用坑一则——关于Post Data的encoding"
date: 2014-05-26 21:31
comments: true
categories: http
---
之前有个Http服务，使用Serlvet API实现的，现在遇到一个问题：一个同学使用HttpClient进行POST调用，结果中文传过来都是乱码。

<!--more-->

使用自制抓包工具[DP-IDEA](https://github.com/code4craft/dp-idea)抓包分析Http请求，发现类似`&value=%3f%3f`内容，并且Http头中Content-Type为`application/x-www-form-urlencoded`。上网查了才发现，Http Post请求也会默认做UrlEncoding，而HttpClient如果不设置，默认会用"ISO-8859-1"进行编码，于是改为UTF-8，问题解决！

PS: ISO-8859-1编码时，会将高位字符编码成63(16进制就是3F)，所以以后遇到这个可疑的字符，更加有迹可循一点。