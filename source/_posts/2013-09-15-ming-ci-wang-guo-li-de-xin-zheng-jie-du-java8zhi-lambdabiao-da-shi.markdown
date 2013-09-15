---
layout: post
title: "名词王国里的新政-解读Java8之lambda表达式"
date: 2013-09-15 08:36
comments: true
categories: lambda
---

前几天在reddit上看到Java8 M8 Developer Preview版本已经发布了，不免想要尝鲜一把。Developer Preview版本已经所有Feature都完成了，Java8的特性可以在这里看到[http://openjdk.java.net/projects/jdk8/features](http://openjdk.java.net/projects/jdk8/features)，下载地址：[http://jdk8.java.net/download.html](http://jdk8.java.net/download.html)。

<!--more-->

## 下载及配置

Intellij IDEA已经完美支持Java8了。首先打开Project Structure，在Project里设置新的JDK路径，并设置Modules=>Source=>Language Level为8.0即可。

现在我们可以使用Java8编写程序了！但是当我们开开心心编写完，享受到高级的lambda表达式后，运行程序，会提示：`java: Compilation failed: internal java compiler error`！这是因为javacc的版本还不对，在Compiler=>Java Compiler里将项目对应的javacc版本选为1.8即可。

什么？你说你用Eclipse？好像目前还没有稳定版！想尝鲜的，可以看看这个地址[http://stackoverflow.com/questions/13295275/programming-java-8-in-eclipse](http://stackoverflow.com/questions/13295275/programming-java-8-in-eclipse)，大致是先checkout Eclipse JDT的beta java8分支，然后在Eclipse里运行这个项目，从而启动一个支持java8的Eclipse…不过应该难不倒作为geek的你吧！

## 体验lambda表达式

好了，我们开始体验Java8的新特性-lambda表达式吧！现在我们的匿名类可以写成这样子了：

{% codeblock lang:java java8中的lambda %}
	new Thread(() -> {
		System.out.println("Foo");
	}).start();
{% endcodeblock %}

而之前的写法只能是这样子：

{% codeblock lang:java 以前的java匿名函数 %}
	new Thread(new Runnable() {
		@Override
		public void run() {
			System.out.println("Foo");
		}
	}).start();
{% endcodeblock %}

这样一看，我们似乎就是匿名类写起来简单了一点啊？而第二种方法，借助便捷的IDE，好像编写效率也没什么差别？博主开始也是这样认为，仔细学习之后，才知道其中的奥妙所在！

这里有一个重要的信息，就是**`()->{}`这里代表一个函数，而非一个对象。**可能这么说比较抽象，我们还是代码说话吧：

{% codeblock lang:java 函数作为参数 %}
	public class LambdaTest {

	    private static void bar(){
	        System.out.println("bar");
	    }

	    public static void main(String[] args) {
	        new Thread(LambdaTest::bar).start();
	    }

	}
{% endcodeblock %}

看懂了么？这里`LambdaTest::bar`代表一个函数(用C++的同学笑了)，而new Thread(Runnable runnable)的参数，可以接受是一个函数作为参数！

是不是觉得很神奇，颠覆了Java思维？在剖析原理以前，博主暂且卖个关子，我们先来讲讲什么是lambda表达式。

## 什么是lambda表达式

### lambda表达式的由来

絮叨几句，现代编程语言的lamdba表达式都来自1930年代初，阿隆佐·邱奇(Alonzo Church)提出的λ演算(Lambda calculus)理论。λ演算的核心思想就是“万物皆函数”。一个λ算子即一个函数，其一般形式是`λx.x + 2`。一个λ算子可以作为另一个λ算子的输入，从而构建一个高阶的函数。λ演算是函数式编程的鼻祖，大名鼎鼎的编程语言Lisp就是基于λ演算而建立。用过Lisp的应该都清楚，它的语法很简单，但是却有包容万物的能力。

可能搞计算机的对邱奇比较陌生，但是提起和邱奇同时代的另外一个人，大家就会觉得如雷贯耳了，那就是阿兰·图灵。邱奇成名的时候，图灵还是个大学生。邱奇和图灵一起发表了邱奇-图灵论题，并分别提出了λ演算和图灵机，加上哥德尔提出的递归函数一起，在理论上确定了什么是可计算性。至于什么是可计算性，其实博主也说不清楚，但是现代所有计算机程序语言，都可以认为是从三种之一发展而来，并与之等价的。仅此一点，其影响深远，可想而知。当年教我们《计算理论》的是一个德高望重的教授，人称宋公，每次讲到那个辉煌的年代，总是要停下来，神情专注的感叹一句：“伟大啊！”想想确实挺伟大，人家图灵大学时候就奠定了现代计算机的基础，而我们那会大概还在打DOTA…

附上大神们的照片，大家感受一下：

![turing etc.][1]

### 现代编程语言中的lambda表达式

好了扯远了，神游过了那个伟大的时代，我们继续思考如何编代码做需求吧…

现代语言的lambda表达式，大概具备几个特征(博主自己归纳的，如有不严谨，欢迎指正)：

1. 函数可作为输入；
2. 函数可作为输出；
3. 函数可作用在函数上，形成高阶函数。
4. 函数支持lambda格式的定义。

其实有了1、2，3也就是顺水推舟的事情，而4其实没有太大的必要性，因为一般语言都有自己的函数定义方式，4仅仅是作为一种补充。当然实现了4的语言，一般都会说：“你看我实现了lambda表达式！”(望向Java8和Python同学)

## 在Java8中使用lambda表达式

### FunctionalInterface

Java中的lambda无法单独出现，它需要一个接口来盛放。这个接口必须使用@FunctionalInterface作为注解，并且只有一个未实现的方法。等等，什么叫接口中未实现的方法？恭喜你，猜对了！Java8的接口也可以写实现了。是不是觉得Interface和AbstractClass更加傻傻分不清楚了？但是AbstractClass是无法使用@FunctionalInterface注解的，官方的解释是为了防止AbstractClass的构造函数做一些事情，可能会导致一些调用者意料不到的事情发生。

好了，我们来看一点代码，Runnable接口现在变成了这个样子：

{% codeblock lang:java Runnable接口新定义 %}
	@FunctionalInterface
	public interface Runnable {
	    public abstract void run();
	}
{% endcodeblock %}

这里我们可以将任意无参数的lambda表达式赋值给Runnable:

{% codeblock lang:java 向runnable传递闭包 %}
	Runnable runnable = () -> {
		System.out.println("Hello lambda!");
	};
	runnable.run();
{% endcodeblock %}
        
lambda表达式本质上是一个函数，所以我们还可以用更加神奇的赋值：

{% codeblock lang:java 向runnable传递闭包 %}
	public class HelloLambda {

	    private static void hellolambda() {
	        System.out.println("Hello lambda!");
	    }

	    public static void main(String[] args) {
	        Runnable runnable = HelloLambda::hellolambda;
	        runnable.run();
	    }
	}
{% endcodeblock %}
	
这里看到这里，大家大概明白了，lambda表达式其实只是个幌子，更深层次的含义是：函数在Java里面可以作为一个实体进行表示了。这就意味着，在Java8里，函数既可以作为函数的参数，也可以作为函数的返回值，即具有了lambda演算的所有特性。

### Function系列API

看到这里，可能大家会有疑问？什么样的函数和什么样的lambda表达式属于同一类型？答案是参数和返回值的类型共同决定函数的类型。例如Runnable的run方法不接受参数，也没有返回值，那么Runnable接口则可以用任意没有参数且没有返回值的函数来赋值。这样概念上来说，Runnable表示的含义就**从一个对象变成了一个方法**。

这一点在Java8中的java.util.function包里的代码得到了验证。以最具有代表性的Function接口为例：

{% codeblock lang:java Function接口 %}
	@FunctionalInterface
	public interface Function<T, R> {

	    R apply(T t);

	}
{% endcodeblock %}
	
有了Function，我们可以这样写：

{% codeblock lang:java Function接口的赋值 %}
	Function<Integer,String> convert = String::valueOf;
	String s = convert.apply(1);
{% endcodeblock %}
	
这个东东是不是很像Javascript中的函数对象？

可惜的是，这里的Function算是个半成品，它只能表示一个有单个参数，并有非void返回值的函数。像System.out.println()这种方法，因为返回值为void，是无法赋值为Function的！

怎么办？java.util.function包提供了一个不那么完美的解决方案：多定义几个FunctionalInterface呗！

于是，在Java8里有了：

* Supplier: 没有参数，只有返回值的函数
* Consumer: 一个参数，返回值为void的函数
* BiFunction: 两个参数，一个返回值的函数
* BiConsumer: 两个参数，没有返回值的函数
* ...

对于这些个API，我也没有什么力气吐槽了，反正我也想不出更好的方法…大家趁机，多学几个单词吧，嗯。

## 总结：名词王国的新政

相信很多同学都看过这篇著名的文章：[名词王国里的死刑](http://lcwangchao.github.io/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B/2012/07/02/excution_in_the_kingdom_of_nouns/)。这篇文章吐槽了Java里，动词(方法)在Java里总是要依附于某个名词(对象/类)存在。

现在动词在名词王国终于有了一个身份了。当然这个动词需要先取得一个名词的身份(FunctionInterface)，然后才能名正言顺的幸存下来。好在Oracle国王预先为他们留了一些身份(Function、Consumer、Supplier、BiFunction...)，所以大多数动词都已经找到了自己的位置。System.out.println(String)现在是Consumer<String>了，String.valueOf(Integer)现在是Function<Integer,String>了，Collection.size()现在是Supplier<Integer>了…。要为一些较长参数的方法获取一个身份，也是挺容易的(定义一个新的FunctionInterface接口)。

我相信这个影响是深远的。例如下面一段代码，可以同一行代码将一个List<Integer>转换成一个List<String>：

{% codeblock lang:java List<Integer>转换成List<String> %}
	List<String> strings = intList.stream().map(String::valueOf).collect(Collectors.<String>toList());
{% endcodeblock %}
	
当然问题也存在。因为包含了闭包等因素，FunctionInterface的序列化/反序列化会是一个相当复杂的事情。熟悉Java的开发者，也会因为lambda的引入，带来了一些困惑。俗话说活到老学到老，我倒是不介意这个新功能，你说呢？

本系列文章还有余下几部分，敬请期待：

[lambda表达式与闭包]()

[Java8 lambda表达式原理分析]()

## 参考文献：

1. [http://blog.sciencenet.cn/blog-414166-628109.html](http://blog.sciencenet.cn/blog-414166-628109.html)
2. [http://www.global-sci.org/mc/issues/3/no2/freepdf/80s.pdf](http://www.global-sci.org/mc/issues/3/no2/freepdf/80s.pdf)
3. [http://en.wikipedia.org/wiki/Lambda_calculus](http://en.wikipedia.org/wiki/Lambda_calculus)

  [1]: http://static.oschina.net/uploads/space/2013/0914/160058_HLj3_190591.png