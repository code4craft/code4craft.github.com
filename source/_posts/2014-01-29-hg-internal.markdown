---
layout: post
title: "代码管理进入大数据时代"
date: 2014-01-29 14:01
comments: true
categories: 
---
最近Facebook在它们的博客中发布了文章，说到它们扩展了Mercurial（一种代码管理工具），以管理它们日渐庞大的代码。Facebook的代码量有多大？[informationisbeautiful](http://www.informationisbeautiful.net/visualizations/million-lines-of-code/)这个网站发布过可视化一个软件代码量的介绍，Facebook的代码量大约是6000万行，超过了Windows Vista和Visual Studio 2012。

<!--more-->

当然这6000万行代码并没有保存在同一个仓库里。根据Facebook的文章，它们主代码仓库，大概有1700万行代码和44000个文件，并且每周有数千的提交。这样大的数据量下，我们常用的版本控制工具，包括git都不好使了，为什么呢？

我们先了解一下版本控制工具的原理。版本控制工具（Version Control System，以下简称VCS，请与CVS分开）从早期的集中式版本控制（CVS，SVN）发展到现在的分布式版本控制（git，Mercurial），使用方式、版本策略变化了很多，但是底层的原理变化并不大。VCS的底层大概可以分为两部分：元数据（版本信息）的管理和变更集（变更文件）的管理。

当我们进行一次提交的时候，首先VCS会产生一个版本号来唯一标记这次提交，在SVN里，这个版本号是一个自增整数，而在git里，它是一个40个字符的SHA1值，而在Mercurial则更复杂一点，它既有自增ID，又有SHA1值。然后VCS会扫描文件，看看哪些文件内容进行了修改，并把这次修改的索引信息保存下来，这就是这次提交的元数据。

而这次被修改的文件，它也会保存下来，在git中，整个项目一次提交的内容都会被压缩并存成一个文件，以该次提交的SHA1值作为文件名，并以树形结构保存下来，又叫tree object。在Mercurial里，它会把该项目的所有提交都打包压缩到changlog文件里，并以版本号来区分，这种格式又叫Revlog。相比之下，git的文件组织更加松耦合一点，所以甚至有人认为git是一种nosql数据库 :D

这里有点意思的是，代码君一直以为VCS对于单个文件的修改是“增量存储”的，而实际上，它是一个全量快照，但是经过了压缩的。后来想想也容易理解，如果是增量存储，那么切换版本的时候，需要把改变一个个摞上去，这个时间是难以接受的！在空间与时间之间，大家都一致选择了浪费空间换取时间！

那么当文件数量大的时候，会出现什么问题呢？

首先，每次提交的速度会变慢，因为每次提交都需要扫描所有文件系统，来判断是否有文件变更。如何解决呢？Facebook之前自己开发过一套监控文件变更的工具WatchMan。它会保存一个目录的文件树，并在操作系统那里注册-每当有文件改变时都会通知它，然后它会维护一个“是否更改”的记录。又是一个典型的“用空间换时间”的做法！然后它们定制了Mercurial，与WatchMan做了适配，于是每次提交的时候，Mercurial就无需扫描文件更改，只需要接收一个“变更文件列表”就行了！

其次，随着版本数增加，因为包含了所有历史文件，整个代码库会变得很巨大。用过git的朋友都知道，clone一个大项目的时间会很漫长，而有的时候，我们只需要最新的一个版本。其实大家都能想到做法是什么了吧？没错，定制过的Mercurial在clone和pull的时候，都只会复制最新版本以及元数据到本地，变更文件是不复制的，而在checkout到指定版本的时候，才会进行复制！有点意思的是，它们觉得这个复制都有点慢，于是使用了Memecached作为文件缓存。

另外，Mercurial大部分代码都是用Python开发，进行定制会比git方便不少，大家是不是有点想试一试呢？Mercurial的意思是“水银“，所以它又叫hg，它的代码地址是[http://selenic.com/repo/hg/](http://selenic.com/repo/hg/)。大家这就开始自己的VCS hack之旅吧！

参考资料：

* [https://code.facebook.com/posts/218678814984400/scaling-mercurial-at-facebook/](https://code.facebook.com/posts/218678814984400/scaling-mercurial-at-facebook/)
* [https://code.google.com/p/support/wiki/DVCSAnalysis](https://code.google.com/p/support/wiki/DVCSAnalysis)
* [http://mercurial.selenic.com/wiki/DeveloperInfo](http://mercurial.selenic.com/wiki/DeveloperInfo)
* [http://git-scm.com/book/en/Git-Internals-Git-Objects](http://git-scm.com/book/en/Git-Internals-Git-Objects)