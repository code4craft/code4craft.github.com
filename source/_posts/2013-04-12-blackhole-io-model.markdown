---
layout: post
title: "BlackHole开发日记-使用三种不同IO模型实现一个DNS代理服务器"
date: 2013-04-12 22:16
comments: true
categories: BlackHoleJ
---
BlackHoleJ是一个DNS服务器。他的一个功能是，对于它解析不了的DNS请求，它将请求转发到另外一台DNS服务器，然后再将其响应返回给客户端，起到一个DNS代理的作用。

![image](/images/posts/forward.png)

这个功能的实现经历了三个版本，也对应了三个经典的IO模型。

<!-- more -->

###BIO模型(Blocking I/O)

BlackHoleJ代理模式最开始的IO模型，实现很简单，当client请求过来时，新建一个线程处理，然后再线程中调用DatagramChannel发送UDP包，同时阻塞等待，最后接收到结果后返回。

	public byte[] forward(byte[] query) throws IOException {
		DatagramChannel dc = null;
		dc = DatagramChannel.open();
		SocketAddress address = new InetSocketAddress(configure.getDnsHost(),
				Configure.DNS_PORT);
		dc.connect(address);
		ByteBuffer bb = ByteBuffer.allocate(512);
		bb.put(query);
		bb.flip();
		dc.send(bb, address);
		bb.clear();
		dc.receive(bb);
		bb.flip();
		byte[] copyOfRange = Arrays.copyOfRange(bb.array(), 0, 512);
		return copyOfRange;
	}
	
其中dc.receive(bb)一步是阻塞的。因为请求外部DNS服务器往往耗时较长，所以为了达到快速响应，不得不开很多线程进行处理。同时每个线程都需要进行轮询dc.receive(bb)是否可用，会消耗更多CPU资源。

###Select模型(I/O multiplexing)

BlackHoleJ 1.1开始使用的IO模型。因为DNS使用UDP协议，而UDP其实是无连接的，所以所有请求以及响应复用一个DatagramChannel也毫无问题。同时预先使用DatagramChannel.bind(port)绑定某端口，那么对外部DNS服务器的转发和接收都可以使用这个端口。唯一需要做的就是通过DNS包的特征，来判断到底是哪个客户端的请求！而这个特征也很好选择，DNS包的headerId和question域即可满足需求。

发送方的伪代码大概是这样：

	public byte[] forward(byte[] queryBytes) {
		multiUDPReceiver.putForwardAnswer(query, forwardAnswer);
		forward(queryBytes);
		forwardAnswer.getLock.getCondition().await();
		return answer.getAnswer();
	}
	
接收方是一个独立的线程，代码大概是这样的：
	
	public void receive() {
		ByteBuffer byteBuffer = ByteBuffer.allocate(512);
		while (true) {
			datagramChannel.receive(byteBuffer);
			final byte[] answer = Arrays.copyOfRange(byteBuffer.array(), 0, 512);
			getForwardAnswer(answer).setAnswer(answer);
			getForwardAnswer(answer).getLock.getCondition().notify();
		}
	}

这里forwardAnswer是一个包含了响应结果和一个锁的对象(这里用到了Java的Condition.wait&notify机制，从而使阻塞线程交出控制权，避免更多CPU轮询)。还有一部分是multiUDPReceiver。这里multiUDPReceiver.putForwardAnswer(query, forwardAnswer)实际上是把forwardAnswer注册到一个Map里。

这样做的好处是仅仅在一个线程检查原本的多路I/O是否就绪，也就是I/O multiplexing。这跟Linux下select模型是一样的。

###AIO模型(Asynchronous I/O)

BlackHoleJ 1.1.3开始，使用了基于回调的AIO模型。这里建立了UDPConnectionResponser对象，里面封装了client的IP和来源端口号。每次收到外部DNS响应时，再根据响应内容找到这个client的IP和来源端口号，重新发送即可。

这实际上就是封装了callback的异步IO。

![image](/images/posts/aio.png)

发送方的伪代码大概是这样：

	public void forward(byte[] queryBytes) {
		multiUDPReceiver.putForwardAnswer(query, forwardAnswer);
		forward(queryBytes);
	}
	
接收方的代码大概是这样：

	public void receive() {
		ByteBuffer byteBuffer = ByteBuffer.allocate(512);
		while (true) {
			datagramChannel.receive(byteBuffer);
			final byte[] answer = Arrays.copyOfRange(byteBuffer.array(), 0, 512);
			getForwardAnswer(answer).getResponser().response(answer);
		}
	}
	
这里getResponser().response()直接将结果返回给客户端。

###测试：

使用queryperf进行了测试，使用AIO模型之后，仅仅单线程就达到了40000qps，比1.1.2效率高出了25%，而CPU开销却有了降低。