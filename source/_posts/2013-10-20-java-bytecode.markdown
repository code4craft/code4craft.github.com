---
layout: post
title: "Java字节码织入技术概述"
date: 2013-10-20 16:52
comments: true
categories: java,bytecode
---
十问：

* 什么是Java字节码？

<!-- more -->


Class文件的结构：

       ClassFile {
              u4             magic;
              u2             minor_version;
              u2             major_version;
              u2             constant_pool_count;
              cp_info        constant_pool[constant_pool_count-1];
              u2             access_flags;
              u2             this_class;
              u2             super_class;
              u2             interfaces_count;
              u2             interfaces[interfaces_count];
              u2             fields_count;
              field_info     fields[fields_count];
              u2             methods_count;
              method_info    methods[methods_count];
              u2             attributes_count;
              attribute_info attributes[attributes_count];
       }

![在此输入图片描述][1]

	javap -verbose Downloader
	
最开始是常量池，保存类名/方法/字面量等信息。

* Java字节码在JVM里如何保存？

字节码是类文件

* 字节码和ClassLoader的关系？

	-XX:+TraceClassLoading 

* 字节码修改能做的事？
* 字节码修改不能做的事？

	Java加载Class文件后会做"连接"操作，就是把符号表的彼此引用连接起来。

* 常用的字节码修改工具？ 


[1]: http://static.oschina.net/uploads/space/2013/1020/193328_GPX0_190591.png


## 参考书目

《深入理解Java虚拟机》

《The Java® Virtual Machine Specification-Java SE 7 Edition》