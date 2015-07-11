---
layout: post
title: "清理你的Mac"
date: 2015-07-11 09:44
comments: true
categories: 
---
之前图便宜，买了128G的Mac Air，结果最大的苦恼就是空间不够。今天终于抽出时间，写了一些小工具，清理一下无用的文件。

<!--more-->

## 1. 查找大文件夹

第一步其实是要查找哪些文件占据了磁盘空间。finder对于文件夹大小统计是无能为力的，这时候还要借助`du`来进行查找。

```shell
du -hd1 #查找一级目录下各文件大小
```

但是这样的工具始终不太方便，首先无法排序，也不太方便按照指定大小过滤。所以干脆自己写了一个脚本来做这个事情，利用了awk的一些功能。程序比较糙，用了很多临时文件。不过shell的风格就是：能够解决问题才是最重要的不是嘛。

```shell
#!/bin/sh
depth=$1
size=$2
sudo du -d$depth > dud.log
ls -al | grep -v '\.\.' > lsd.log
cat dud.log | awk -F'\t' -v size="$size" '$1>size {print$1,$2}' > dularge_tmp.log #awk无法直接用环境变量，所以使用-v先设置变量
cat lsd.log | awk -v size="$size" '$5>size {print$5,$9}' >> dularge_tmp.log 
sort -k1 -rn dularge_tmp.log > dularge_tmp2.log
human_readable_size dularge_tmp2.log > dularge.log
cat dularge.log
sudo rm dud.log
rm lsd.log
rm dularge_tmp2.log
rm dularge_tmp.log
```

因为awk无法直接用环境变量，所以使用-v先设置变量。而且`du`还有一个坑：du出来的数字，并不是字节数，根据观察，应该是0.5K为一个单位。于是在脚本中海做了一些转换。

```shell
cat dud.log | awk -F'\t' -v size="$size" '$1*512>size {print$1*512,$2}' > dularge_tmp.log #计算大小的时候，需要*512
```

## 2. 删除无效文件

一遍扫描下来，发现Java项目竟然占了很大的部分，原因是mvn编译好的war包和jar包。怎么办？删之：

```shell
find . -name "target" -type d -exec rm -rf {} \;
```

还有发现QQ聊天记录的图片占据了5G多空间。这里有点麻烦的就是，我也不太希望将这些文件删除，因为可能会有一些重要图片。怎么办呢？我是将所有文件备份，然后将时间多于一段时间的删除。使用finder做这件事太慢了，还是用shell吧。

```shell
find . -mtime +150 -exec rm {} \; #删除150天之前修改的文件
```

至此，空间释放出10G以上，又可以撑一段时间了…