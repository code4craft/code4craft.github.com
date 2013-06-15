---
layout: post
title: "使用webmagic抓取页面并保存为wordpress文件"
date: 2013-06-10 18:05
comments: true
categories: 爬虫 webmagic
---
之前做过一年的爬虫，当年功力不够，写的代码都是一点一点往上加。后来看了下据说是最优秀的爬虫[`scrapy`](http://www.oschina.net/p/scrapy)的结构，山寨了一个Java版的爬虫框架。

这个框架也分为Spider、Schedular、Downloader、Pipeline几个模块。此外有一个Selector，整合了常用的抽取技术(正则、xpath)，支持链式调用以及单复数切换，因为受够了各种抽取的正则，在抽取上多下了一点功夫。

<!-- more -->

废话不多，上代码。在webmagic里直接实现PageProcessor接口，即可实现一个爬虫。例如对我的点点博客[http://progressdaily.diandian.com/](http://progressdaily.diandian.com/)进行抓取：

        public class DiandianBlogProcessor implements PageProcessor {

            private Site site;

            @Override
            public void process(Page page) {
                //a()表示提取链接，as()表示提取所有链接
                //getHtml()返回Html对象，支持链式调用
                //r()表示用正则表达式提取一条内容，rs()表示提取多条内容
                //toString()表示取单条结果，toStrings()表示取多条
                List<String> requests = page.getHtml().as().rs("(.*/post/.*)").toStrings();
                //使用page.addTargetRequests()方法将待抓取的链接加入队列
                page.addTargetRequests(requests);
                //page.putField(key,value)将抽取的内容加入结果Map
                //x()和xs()使用xpath进行抽取
                page.putField("title", page.getHtml().x("//title").r("(.*?)\\|"));
                //sc()使用readability技术直接抽取正文，对于规整的文本有比较好的抽取正确率
                page.putField("content", page.getHtml().sc());
                page.putField("date", page.getUrl().r("post/(\\d+-\\d+-\\d+)/"));
                page.putField("id", page.getUrl().r("post/\\d+-\\d+-\\d+/(\\d+)"));
            }

            @Override
            public Site getSite() {
                //site定义抽取配置，以及开始url等
                if (site == null) {
                    site = Site.me().setDomain("progressdaily.diandian.com").setStartUrl("http://progressdaily.diandian.com/").
                            setUserAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_2) AppleWebKit/537.31 (KHTML, like Gecko) Chrome/26.0.1410.65 Safari/537.31");
                }
                return site;
            }
        }

然后实现抓取代码：

        public class DiandianProcessorTest {

            @Test
            public void test() throws IOException {
                DiandianBlogProcessor diandianBlogProcessor = new DiandianBlogProcessor();
                //pipeline是抓取结束后的处理
                //ftl文件放到classpath:ftl/文件夹下
                //默认放到/data/temp/webmagic/ftl/[domain]目录下
                FreemarkerPipeline pipeline = new FreemarkerPipeline("wordpress.ftl");
                //Spider.me()是简化写法，其实就是new一个啦
                //Spider.pipeline()设定一个pipeline，支持链式调用
                //ConsolePipeline输出结果到控制台
                //FileCacheQueueSchedular保存url，支持断点续传，临时文件输出到/data/temp/webmagic/cache目录
                //Spider.run()执行
                Spider.me().pipeline(new ConsolePipeline()).pipeline(pipeline).schedular(new FileCacheQueueSchedular(diaoyuwengProcessor.getSite(), "/data/temp/webmagic/cache/")).
                        processor(diaoyuwengProcessor).run();
            }
        }
        
跑一遍之后，将所有输出的文件，合并到一起，并加上wp的[头尾](https://github.com/code4craft/webmagic/tree/master/webmagic-samples/src/main/resources)，就是wordpress-backup.xml了！

代码已开源[https://github.com/code4craft/webmagic](https://github.com/code4craft/webmagic)<strike>有什么邪恶用途你懂的…</strike>