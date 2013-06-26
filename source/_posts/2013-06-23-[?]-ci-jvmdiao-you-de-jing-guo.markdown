---
layout: post
title: "BlackHole开发日记-一次压力测试及JVM调优的经过"
date: 2013-06-23 06:44
comments: true
categories: 
---
BlackHole开发很久了，目前稳定性、性能都还可以了，但是作为一个Java程序，内存开销一直是硬伤，动不动100M内存下去了，对于单机用户实在是不太友好。

怎么办？优化先从分析开始！

### 获取内存信息
	
获取内存信息一般使用jmap。

	jmap -histo pid

这种方式获取到得比较简略，我们可以先把内存dump下来，再进行离线分析。jhat是一个离线内存分析工具，会开启一个web服务以供展示。

	jmap -dump:file=dumpfile pid
	jhat -J-Xmx512m dumpfile
	
<!-- more -->
	
访问http://127.0.0.1:7000即可。默认是Class列表，翻到页面底部可以查看其他功能。
	
参考资料：

[http://www.lhelper.org/newblog/?tag=jhat](http://www.lhelper.org/newblog/?tag=jhat)
[http://blog.csdn.net/gtuu0123/article/details/6039474](http://blog.csdn.net/gtuu0123/article/details/6039474)

### 详细分析

以下分析仅针对JDK 7 HotSpot虚拟机。jmap -histo的结果：

	num     #instances         #bytes  class name
	----------------------------------------------
     1:         28490        4057072  <constMethodKlass>
     2:         28490        3882464  <methodKlass>
     3:          2630        2820064  <constantPoolKlass>
     4:         46350        2412600  <symbolKlass>
     5:         32778        2372824  [C
     6:          2630        1990744  <instanceKlassKlass>
     7:          3418        1955208  [I
     8:         15491        1911568  [B
     9:          2347        1845400  <constantPoolCacheKlass>
    10:         19246         615872  java.lang.String
    11:           256         561152  [Lnet.sf.ehcache.store.chm.SelectableConcurrentHashMap$HashEntry;
    12:          2930         304720  java.lang.Class
    13:          3740         247280  [S
    14:          4321         221520  [[I
    
其中[开头表示数组，[C [I [B 分别是char[] int[] byte[]。

constMethodKlass、都实现自sun.jvm.hotspot.oops.Klass，用于在永久代里保存类的信息。

换到JDK6之后，发现永久代的消耗下去了。

	num     #instances         #bytes  class name
	----------------------------------------------
     1:         10823        3072312  [B
     2:         16605        2318720  <constMethodKlass>
     3:         18687        1388088  [C
     4:         16605        1328608  <methodKlass>
     5:         27595        1296832  <symbolKlass>
     6:          1699         940392  <constantPoolKlass>
     7:          2520         883408  [I
     8:          1699         724944  <instanceKlassKlass>
     9:          1472         565136  <constantPoolCacheKlass>
    10:           256         561152  [Lnet.sf.ehcache.store.chm.SelectableConcurrentHashMap$HashEntry;
    11:         12148         291552  java.lang.String
    12:          4505         288320  net.sf.ehcache.Element
    13:          7290         233280  java.lang.ThreadLocal$ThreadLocalMap$Entry
    14:          1946         186816  java.lang.Class
    15:          4509         180360  net.sf.ehcache.store.chm.SelectableConcurrentHashMap$HashEntry

查看一下总的内存开销(参考资料:
[http://yytian.blog.51cto.com/535845/574527](http://yytian.blog.51cto.com/535845/574527))

    ps -e -o 'pid,comm,args,pcpu,rsz' | grep java |  sort -nrk5
    1239 java            java -jar -Djava.io.tmpdir=  0.1 52816
或者：
	
	top -pid pid
    
查看到只有52m，看来JDK7占用内存果然增加了！

### 压力测试

使用140,000个随机域名做压力测试。发现之前使用的JDK7 Developer Preview u04，在短时间产生大量对象的时候，GC会失去作用，内存迅速飙升到200M，后来更新到1.7.0u25，稳定到了130m，qps大概在49000~50000之间。

尝试使用ConcurrentHashMap代替EhCache，qps提高到50000~51000之间，变化不明显，EhCache还是相当优秀的。

为了提高吞吐量，查看GC情况:

	sudo jstat -gcutil 2960
	S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
	0.00   0.75  30.35  12.06  72.92     25    0.063     0    0.000    0.063

140,000个请求产生了25次YGC。

参考了关于JVM的调优文章[http://blog.csdn.net/kthq/article/details/8618052](http://blog.csdn.net/kthq/article/details/8618052)

重新设置新生代为200m，并开启并行收集：

	JVM_OPTION="-XX:+UseParallelGC -XX:NewSize=200m"
	
140,000个请求只产生了3次YGC，但是qps变化并不明显。看来新生代设置大之后，虽然YGC少了，但是一次回收的时间多了，最终其实没啥影响啊，那还是弄小一点好了。

看来折腾JVM效果不明显，除了内存开销之外，其他没有明显变化。